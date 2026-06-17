# System Architecture

## Motivation

While modern Large Language Models (LLMs) feature massive context windows capable of processing thousands of tokens, relying solely on session based memory presents significant architectural limitations.

The Challenge:
- **Volatility**: Context windows last only for a short period of time. Once a session is terminated, the memory buffer is cleared. Users are then forced to restate their preferences and constraints. 

- **Computational Overhead**: Continuously injecting a massive and static document of user history into the prompt for every single interaction introduces high latency and computational waste of the tokens, which leads to hardware and financial constraints due to the processing of execessive tokens during inference time.

CortexDB introduces a persistent and offline architecture. Instead of relying on a bloated context window, the system stores permanent facts about the user locally and utilizes a two step Retrieval Augmented Generation (RAG) pipeline. This ensures that only the most contextually relevant subset of historical data is dynamically injected into the LLM’s active prompt at inference time.

---

## Architecture Overview

CortexDB follows a simple memory lifecycle inspired by how humans convert short-term experiences into long-term memories. Instead of treating every conversation as an isolated session, the system continuously decides what should be remembered, what should be ignored, and when previously stored memories should be brought back into context.

The entire process can be divided into four stages.

### 1. Buffer (Short-Term Memory)

During an active conversation, CortexDB temporarily stores all messages in an in-memory buffer.

Think of this as the system's shortterm memory. It only exists while the session is running and acts as a working area where the agent can keep track of the ongoing dialogue. At this stage, nothing is permanently saved. The goal is simply to maintain enough context for the model to respond throughout the current interaction.

**Responsibilities:**

- Store the current conversation transcript.
- Track user and assistant messages in chronological order.
- Provide context for the active session.
- Discard the temporary buffer once the session ends.

---

### 2. Recall (Context Injection)

Whenever a new user query arrives, CortexDB first checks its long term memory (SQLite) before asking the language model to generate a response.

The system searches for previously stored information that may be relevant to the current conversation. These memories are then silently injected into the prompt so that the assistant can answer with awareness of past interactions.

From the user's perspective, this creates the feeling that the AI remembers previous conversations.

**Responsibilities:**

- Search the memory vault for relevant information.
- Retrieve user preferences, goals, or important facts.
- Inject retrieved memories into the LLM context.
- Generate more personalized responses.

---

### 3. Extract (Post-processing)

When a session ends, CortexDB analyzes the complete conversation and decides what is worth remembering permanently. Instead of saving every single message, an extraction model filters out temporary or low-value information and focuses only on high-signal facts.

For example:

**Remember**

- "I am preparing for software engineering placements."
- "I prefer Python over Java."
- "I enjoy project-based learning."

**Ignore**

- "Hello"
- "Thank you"
- Small talk and temporary statements.

This stage transforms raw conversations into structured memories.

**Responsibilities:**

- Analyze the conversation transcript.
- Identify important user facts.
- Categorize information into meaningful memory types.
- Discard low-value information.

---

### 4. Store (Long-Term Consolidation)

After important facts have been extracted, CortexDB stores them inside its long term memory vault.

Before saving, the system checks whether similar memories already exist. Duplicate information is removed, outdated information can be updated, and only novel memories are preserved.

The final memory is then encrypted and committed to persistent storage.

Over time, this allows CortexDB to build a continuously evolving profile rather than a collection of disconnected conversations.

**Responsibilities:**

- Detect duplicate memories.
- Update outdated information.
- Encrypt stored data.
- Persist memories to long-term storage.

---

In simple terms, CortexDB follows a continuous loop:

**Buffer → Recall → Extract → Store**

This cycle enables the system to gradually build persistent memory instead of starting from scratch every time a new session begins.


## Security and Storage

CortexDB is designed to store longterm memories about a user. Since these memories may contain sensitive information such as preferences, goals, work history, or personal details, security cannot be treated as an optional feature.

Storing this data as plain text creates a serious privacy risk.

### The Initial Approach

The simplest implementation was to use a local SQLite database and directly store extracted memories as strings.

For example:

```text
memory.db

ID | Memory
-------------------------------
1  | User prefers Python
2  | User is preparing for placements
3  | User enjoys project-based learning
```

While this approach is easy to implement, it introduces a major security problem.

---

### The Problem

SQLite databases are simply files stored on the local machine.

If someone gains access to the file system, they can open the database and immediately read every stored memory.

This means a person's entire historical profile could be exposed without needing to compromise the AI application itself.

For a system whose primary purpose is to remember user information, this is an unacceptable vulnerability.

---

### The Solution: Encryption at Rest

To protect stored memories, CortexDB encrypts all memory entries before they are written to the database.

The project uses **AES-GCM (Advanced Encryption Standard - Galois/Counter Mode)**.

AES-GCM was chosen because it provides two important guarantees:

- **Confidentiality:** Unauthorized users cannot read the stored data.
- **Integrity:** The system can detect if the data has been modified or corrupted.

This means CortexDB not only protects the contents of a memory but also verifies that the memory has not been tampered with.

---

### Why AES-GCM Instead of AES-CBC?

Traditional AES-CBC focuses primarily on encryption. AES-GCM goes one step further by adding built-in authentication.

This allows CortexDB to answer two questions whenever a memory is retrieved:

1. Can this data be decrypted?
2. Can this data be trusted?

If either check fails, the memory is rejected.

---

### Storage Schema

CortexDB uses SQLite as its local persistent storage layer. SQLIte was preferred over a dedicated vector database for the follwoing reasons:

- **Simplicity**: The use case for this system involves a single user storing a relatively small number of memories.

- **Scale**: Vector databases are used in ases where there are millions of embeddings, multiple users and large number of retrieval tasks to be performed in real time. 

- **Not only a vector storage**: We need to manage encrypted queries, timestamps and deduplication which is structured data already handled well by RDBMS.

The database schema is intentionally kept minimal.

```sql
CREATE TABLE IF NOT EXISTS memories (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    encrypted_text BLOB NOT NULL,
    vector_embedding TEXT NOT NULL,
    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

Each field serves a specific purpose:

| Column | Description |
|--------|-------------|
| `id` | Unique identifier for every memory entry |
| `encrypted_text` | The encrypted memory data stored as a binary BLOB |
| `vector_embedding` | The semantic embedding used for similarity search and memory retrieval |
| `timestamp` | The time at which the memory was created |


Before a memory is stored, it passes through the encryption pipeline. The `vector_embedding` is stored separately because CortexDB still needs a way to perform semantic search and retrieve relevant memories during future conversations. As a result, the system can efficiently search for memories while ensuring that the original text remains protected.


### How It Works

#### 1: Generate a secret key

When CortexDB is started for the first time, it generates a secure 32-byte cryptographic key. This key is stored separately and is required to encrypt and decrypt all future memories.

---

#### 2: Encrypt before storing

Before a memory is saved, CortexDB encrypts it. During encryption, AES-GCM automatically generates:

- A **Nonce** (a random value used once)
- An **Authentication Tag**
- The encrypted data (**Ciphertext**)

The original text is never written directly to the database.

---

#### 3: Store encrypted data

Instead of storing readable text, CortexDB stores encrypted binary data.

The database stores the following components together inside a binary BLOB field:

```text
Nonce + Authentication Tag + Ciphertext
```

As a result, opening the database file will not reveal any meaningful information.

---

#### 4: Verify before retrieval

Whenever CortexDB retrieves a memory, it first verifies its authenticity.

If even a single bit has been modified, the authentication check will fail.

In that situation:

- The memory is rejected.
- The system refuses to inject corrupted data into the AI context.
- An attacker cannot silently manipulate stored memories.

This acts as an additional safety layer that prevents compromised information from influencing future responses.

---

In short, CortexDB follows a simple principle:

> **Memories should not only be persistent, they should also be private and trustworthy.**

---

## Semantic Retrieval: A two stage memory search pipeline

One of the biggest challenges in building a memory system is retrieving the correct information from past conversations. People rarely repeat the exact same words every time they ask a question. Instead, they use synonyms, abbreviations, or completely different phrasing.

A simple keyword search cannot handle this.

For example, consider the following stored memory:

```text
The user codes in React.
```

Later, the user asks:

```text
What UI framework am I using?
```

Even though both statements are related, the word **React** never explicitly appears alongside **UI framework**.

A traditional SQL query such as:

```sql
SELECT * FROM memories
WHERE text LIKE '%framework%';
```

would fail to find the correct answer.

---

### Initial Approach: Single-Stage Vector Search

To solve this problem, CortexDB converts text into numerical embeddings.

The project uses the `all-MiniLM-L6-v2` embedding model, which transforms sentences into 384-dimensional vectors.

Instead of matching exact words, the system compares mathematical representations of meaning.

When a user asks a question:

1. Convert the query into an embedding.
2. Compare it against all stored memory embeddings.
3. Retrieve the most similar memories using cosine similarity.

This works well for many cases but introduces a new problem.

---

### The Limitation: Retrieval Failure

Vector search is very good at finding broad topical similarities, but it is not perfect at logical reasoning.

For example:

**Stored Memory**

```text
The user codes in React.
```

**User Query**

```text
What UI framework am I using?
```

The embedding model may fail to rank this memory highly enough because **framework** and **React** do not always occupy nearby positions in vector space without additional context.

As a result, important memories can be missed.

---

### The Solution: A Two-Stage Retrieval Pipeline

Instead of relying on a single retrieval step, CortexDB uses a two stage approach.

The idea is simple:

1. Retrieve a broad set of possible memories.
2. Rerank them using deeper semantic reasoning.

This combines speed with accuracy.

---

### Stage 1: Candidate Generation

The `all-MiniLM-L6-v2` embedding model performs a fast vector similarity search.

Instead of retrieving only the top 3 memories, CortexDB retrieves the top 10 candidate memories.

Think of this stage as casting a wide net.

At this stage, the results are only loosely related.

---

### Stage 2: Semantic Reranking

The 10 candidate memories are then passed into a [Cross-Encoder model](https://www.sbert.net/examples/cross_encoder/applications/README.html) (`ms-marco-MiniLM-L-6-v2`).

Unlike vector search, a Cross-Encoder evaluates the query and each memory together.

Instead of asking:

> Are these two pieces of text generally similar?

It asks:

> Does this memory directly answer this specific question?

This allows the system to perform deeper semantic reasoning.

For example:

```text
Query:
"What UI framework am I using?"

Candidate Memory:
"The user codes in React."
```

The Cross-Encoder can infer that **React is a UI framework**, assign a higher relevance score, and move it to the top of the ranking.

---

In simple terms:

> **First search broadly, then think deeply before remembering.**


---

## The Encoding Gate (Semantic Deduplication)

One of the biggest challenges in a long term memory system is deciding what not to remember.
If every extracted fact is stored without any filtering, the memory vault quickly fills with duplicate information and becomes less useful over time.

### The Initial Approach

At the end of every session, CortexDB extracted important facts from the conversation and directly inserted them into the database.

For example:

```text
Session 1:
The user codes in React.

Session 2:
The user is building a React project.

Session 3:
The user utilizes React.
```

While these appear different, they essentially communicate the same idea.

Over time, the database would become cluttered with multiple variations of identical memories.

---

### The Problem: Memory Bloat

Duplicate memories create two major issues.

#### 1. Redundant storage

The database grows unnecessarily large by storing repeated information.

#### 2. Poor retrieval quality

When the retrieval system searches for relevant memories, these duplicates begin to dominate the top results.

For example:

```text
Query:
What technologies am I using?

Retrieved Results:

1. The user codes in React.
2. The user utilizes React.
3. The user is building a React project.
```

Instead of returning diverse information, the system repeatedly returns the same concept.

This pushes other useful memories out of the context window.

---

### Why a Simple Similarity Threshold Wasn't Enough

An obvious solution is to compare embeddings using cosine similarity.

```text
If similarity > 0.85
Hence, Mark as duplicate
```

However, this is unreliable.

Paraphrasing slightly changes the vector representation, allowing near identical facts to bypass the threshold.

As a result, semantic duplicates can still enter the database.

---

### The Solution: The Encoding Gate

CortexDB introduces an Encoding Gate, a multi stage filtering pipeline that runs before any memory is inserted.

Instead of relying on a single technique, multiple checks are combined together.

The idea is simple:

> **A memory must pass several validation checks before it earns a place in long-term storage.**

---

### Layer 1: Substring Containment

The system first checks whether one sentence already exists inside another.

Examples:

```text
Existing Memory:
The user codes in React.

New Memory:
The user codes in React and TypeScript.
```

Since one statement already contains the other, the system flags it as a potential duplicate.

This quickly removes nested memories.

---

### Layer 2: Jaccard Similarity (Word Overlap)

Next, the text is tokenized into sets of unique words.

The overlap between the two vocabularies is then measured.

For example:

```text
Sentence A:
The user codes in React

Sentence B:
The user is building a React project

Word Sets:

{user, codes, react}

{user, building, react, project}
```

If the overlap exceeds a predefined threshold (for example, 60%), the memory is treated as a semantic duplicate.

This allows CortexDB to detect duplicates even when sentence structures differ.

---

### Layer 3: Vector Similarity

Finally, CortexDB applies embedding-based cosine similarity.

This acts as a final safety net for detecting conceptual duplicates that bypass lexical checks.

If the similarity exceeds the threshold, the memory is discarded.

---

This multi layered approach allows CortexDB to maintain a clean, diverse, and high-quality memory storage instead of accumulating repetitive information.

In simple terms:

> **Not every extracted fact deserves to become a permanent memory.**
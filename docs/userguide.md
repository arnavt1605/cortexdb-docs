# User Guide

CortexDB is designed to work as a lightweight command-line assistant that runs entirely on your local machine.

Once installed, you can interact with it just like any other terminal application. During conversations, CortexDB automatically builds and manages your encrypted memory vault in the background.

This guide explains how to manage your memories and switch between different local AI models.

---

## Managing Your Memory Storage

Although CortexDB learns automatically from conversations, you always maintain full control over your stored data.

The following builtin commands allow you to inspect and manage your memory storage directly from the chat interface:

### 1. View Stored Memories

To see all the facts that CortexDB has learned over time, use:

```bash
show memories
```

What happens:

- The command is intercepted internally and is not sent to the AI model.
- CortexDB decrypts your local memory vault.
- A numbered list of all stored memories is displayed.

Example:

```text
1. The user prefers Python over Java.
2. The user is preparing for placements.
3. The user enjoys project-based learning.
```

---

### 2. Clear All Memories

To permanently delete your entire memory, run:

```bash
clear memories
```

Before deletion, CortexDB will ask for confirmation.

```text
Are you sure you want to delete all memories? (y/n)
```

This extra step helps prevent accidental data loss.

Once confirmed, all stored memories are permanently removed from the local database.

---

### 3. Delete a specific memory

Instead of clearing all memories, the user can delete some specific memories which are of no use or have become obsolete.

To delete a memory, use the following command:

```bash
delete memory <memory-id>
```

For example:

```text
delete memory 3
```

---

### 4. Memory function toggle

If the user wants to temporarily disable the memory logging feature from the current session, to prevent the reading and storing of any of the current chat sessions, the following command can be used:

```bash
memory off
```

To turn the memory logging feature on again, use the following command:

```bash
memory on
```

## Using Different Local Models

CortexDB is model agnostic, meaning it can function with any local model of your choice. 

It is designed to work with any model available through Ollama.

By default, the application uses [`llama3.1:8b`](https://ollama.com/library/llama3.1:8b), but you can easily switch to another model depending on your hardware and preferences.

### Launch With A Specific Model

Use the `--model` (or `-m`) argument when starting the application.

Example:

```bash
cortexdb --model gemma2
```

or

```bash
cortexdb -m gemma2
```

---

## Handling Missing Models

If you try to load a model that is not installed, CortexDB will safely handle the error instead of crashing.

Example:

```text
Model 'phi3' was not found.

Installed models:

- llama3.1
- mistral
- gemma2

Please install the model first:

ollama pull phi3
```

---

## Downloading New Models

To install a new model, use the standard Ollama command.

```bash
ollama pull <model_name>
```

Example:

```bash
ollama pull qwen3:4b
```

After the download completes, you can immediately launch CortexDB with the new model.

```bash
cortexdb --model qwen3:4b
```

---

## Available System Commands

| Command | Description |
|---------|-------------|
| `show memories` | Display all stored memories |
| `clear memories` | Permanently delete all memories |
| `cortexdb --model <model_name>` | Launch CortexDB with a specific model |
| `delete memory  <memory_id>`  | Delete a specific memory from the database |
| `memory on` | To turn on the memory logging feature |
| `memory off` | To turn on the memory logging feature |

> **Note:** All memories are stored locally and encrypted before being written to disk. No data is sent to external servers unless you explicitly configure a cloud based model provider.
# Release Notes

All notable changes to CortexDB are documented here.

This project follows Semantic Versioning.

---

## v1.3.0 - Async Processing & Better Memory Retrieval

- **Asynchronous fact extraction:** Conversations are now added to a local queue and processed in the background, allowing the CLI to return control immediately.

- **Background worker:** Added a separate worker process that continuously monitors and processes queued transcripts.

- **Worker locking:** Added a lock mechanism to prevent multiple workers from running simultaneously and avoid database conflicts.

- **Similarity thresholding:** Added a minimum similarity score before injecting memories into the LLM to prevent irrelevant context from being used.

---

## v1.2.0 - More User Control

- **Delete individual memories:** Users can now delete specific memories using:

```bash
delete memory <id>
```

- **Incognito mode:** Added:

```bash
memory off
memory on
```

When memory is turned off, CortexDB neither reads from nor writes to longterm memory.

---

## v1.1.0 - Memory Management

- **View stored memories:**

```bash
show memories
```

Displays all saved memories from the encrypted vault.

- **Clear all memories:**

```bash
clear memories
```

Permanently deletes all stored memories after confirmation.

---

## v1.0.0 - Initial Release

- Initial release of CortexDB CLI.

- AES-GCM encryption for secure local storage.

- Two-stage retrieval pipeline using embeddings and reranking.

- Encoding Gate to prevent duplicate and noisy memories from being stored.
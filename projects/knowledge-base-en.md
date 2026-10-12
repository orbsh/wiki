# Knowledge Base (SurrealDB RAG)

> **Languages:** [English](knowledge-base-en.md) (primary) · [中文](knowledge-base.md)

## Architecture Decisions

### ADR-005: Use SurrealDB's built-in HTTP functions to call external APIs

- **Status**: Accepted
- **Context**: The vector-ingestion flow needs to call an external embedding API such as Ollama
- **Decision**: Issue HTTP requests directly from SurrealDB through the built-in `http::post()` / `http::get()` functions, instead of proxying them through a backend application
- **Rationale**:
  - **Less data in transit**: Vector data is bulky (a single bge-m3 record is ~4KB); a direct call avoids pointless forwarding through the backend and saves bandwidth and memory
  - **Interface hiding**: Applications only call the `fn::ollama::embed()` function and need not know Ollama exists
  - **Centralized configuration**: Ollama's address, timeout, model and other settings are managed in one place in the database's `config` table
  - **Lower maintenance cost across applications**: Multiple applications only need to talk to one database interface instead of each managing its own HTTP calls
  - **Decoupling**: Swapping the embedding model or service only requires editing the function definition in the database; applications change nothing
  - **Transactional consistency**: The HTTP request can run inside a transaction, making "embed + store" atomic
  - **Access control**: SurrealDB's permission system decides who may call the embedding function
- **Execution model**:
  - **Async and non-blocking**: SurrealDB runs on the tokio async runtime, so HTTP requests execute asynchronously
  - **It does not block other queries**: An HTTP request inside one query does not block other concurrent queries
  - **Serial within a single query**: Multiple HTTP calls in the same query run one after another; there is no native parallel syntax yet
  - **No impact in the current scenario**: `fn::ollama::embed()` issues one HTTP request per call, so head-of-line blocking does not arise
- **Related files**:
  - `communicate-skills/skills/surrealdb-query/assets/ollama.surql` (the function implementation)
  - `SPECIFICATION3.md` — ADR-004 (unified storage architecture, part of the same principle of reducing external dependencies)
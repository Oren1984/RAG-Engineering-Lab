# Known Limitations

This document lists the current limitations of the RAG Engineering Lab.
These are intentional design boundaries for v1, not bugs.

---

## Embedding and Retrieval

### Mock embeddings are not semantic

`MockEmbeddingProvider` produces vectors seeded from SHA-256 text hashes.
The vectors do not capture meaning — "chunking strategy" and "splitting approach"
produce entirely unrelated vectors. Retrieval rankings under mock mode reflect
hash proximity, not relevance.

**Impact:** MRR and nDCG scores with mock embeddings are not interpretable as
measures of semantic retrieval quality. Only Recall@K (binary hit check) is
meaningful in mock mode.

**Workaround:** Replace `MockEmbeddingProvider` with `HuggingFaceEmbeddingProvider`
for semantically meaningful results.

---

### FAISS availability depends on the environment

`faiss-cpu` is listed in `requirements.txt` but does not install cleanly on
all platforms (notably some ARM and older Linux configurations). Wheels are not
available for all Python/OS combinations.

**Impact:** `FaissVectorStore` raises `ImportError` with a clear message if
`faiss-cpu` is not installed. The system falls back to `ChromaVectorStore` in the
Streamlit UI when FAISS is unavailable.

**Workaround:** Install with `pip install faiss-cpu`. If the wheel is unavailable,
use ChromaDB as the vector store.

---

### HybridRetriever is not implemented

`src/retrieval/hybrid_retriever.py` is a documented stub. Calling it raises
`RetrievalError("Hybrid retrieval is planned for a future version.")`.

Hybrid sparse+dense retrieval (e.g. BM25 + dense) is out of scope for v1.

---

### CrossEncoderReranker requires sentence-transformers

`CrossEncoderReranker` is available but requires `sentence-transformers` and
a valid cross-encoder model. The first use downloads the model weights.
In environments without internet access or with limited disk space, it may fail.

**Default:** `PassThroughReranker` is used in v1, which returns results unchanged.

---

## System Design

### No LLM generation

This is a **retrieval-only** system. There is no LLM inference, no answer
generation, and no summarisation in v1. The `expected_answer` field in the
ground truth is informational only and is not used in scoring.

**By design.** The focus is retrieval quality measurement, not generation quality.

---

### No multi-user authentication or session isolation

The Streamlit app runs as a single-user local application. There is no login,
no session isolation between browser tabs, and no role-based access control.

**Impact:** Running the app in a shared or public environment would expose the
pipeline state to all users.

**By design.** The system is intended for local development and evaluation.

---

### Single-document index per pipeline run

`RAGPipeline.build_index()` indexes one document per call. The vector store
accumulates chunks across calls, but there is no document management, versioning,
or deduplication at the pipeline level.

**Impact:** Calling `build_index()` on the same document twice adds duplicate
chunks to the store (ChromaDB upserts prevent this; FAISS does not).

**Workaround:** Call `pipeline.clear()` before re-indexing.

---

### Pattern-based PII redaction is not exhaustive

`src/security/pii_redaction.py` uses regular expressions to detect common PII
patterns (email addresses, phone numbers, basic name patterns). It does not use
an NER model or a dedicated PII detection service.

**Impact:** Unusual or non-standard PII formats may not be redacted.

---

## Infrastructure

### Docker requires a running Docker daemon

`docker compose up --build` requires Docker Desktop (Windows/macOS) or the
Docker daemon (Linux) to be running. The container exposes port 8501.

**Impact:** The Dockerised workflow is not available in environments where
Docker is not installed or the daemon is stopped.

---

### No cloud deployment configuration in v1

There is no Kubernetes manifest, Helm chart, Terraform module, or cloud provider
configuration included in v1. The system is designed for local execution.

**By design.** Cloud deployment would require persistent volume management,
secret injection, and authentication that are out of scope for v1.

---

### Windows ChromaDB file-lock on cleanup

ChromaDB's `PersistentClient` holds open file handles on Windows. Deleting the
database directory while the client is active (e.g. at the end of a test) may
raise `PermissionError: [WinError 32]`.

**Mitigation:** All test code uses `tempfile.TemporaryDirectory(ignore_cleanup_errors=True)`
(Python 3.10+) and `shutil.rmtree(ignore_errors=True)` for cleanup.

---

## Scope Boundaries

The following capabilities are explicitly out of scope for v1 and are not planned
for near-term implementation:

| Capability | Status |
|-----------|--------|
| LLM agents | Not planned — this is not an agent system |
| MCP (Model Context Protocol) | Not applicable |
| CrewAI or LangChain | Not used |
| Chat interface | Not planned |
| FastAPI or REST API | Not planned |
| Kubernetes / Terraform | Not planned |
| SaaS or multi-tenant deployment | Not planned |
| BM25 / hybrid retrieval | Planned for a future version |
| Answer quality evaluation | Planned — requires LLM integration |

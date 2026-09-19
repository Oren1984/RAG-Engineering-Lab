# Documentation Index

This folder contains the technical documentation for the RAG Engineering Lab,
supplementing the project overview in the top-level [README.md](../README.md).

| Document | Covers |
|----------|--------|
| [architecture.md](architecture.md) | Layer-by-layer system design: ingestion, chunking, embeddings, vector stores, retrieval, pipeline, evaluation, security, observability |
| [API_KEYS_GUIDE.md](API_KEYS_GUIDE.md) | Environment variables, `.env` setup, optional API keys, and mock-mode configuration |
| [evaluation_report.md](evaluation_report.md) | Retrieval evaluation design, metric definitions (Recall@K, Precision@K, MRR, nDCG@K), and a baseline mock-mode run |
| [experiment_plan.md](experiment_plan.md) | How to design controlled experiments comparing chunking strategies, embedding providers, and vector stores |
| [security_observability.md](security_observability.md) | Input validation, prompt-safety warnings, PII redaction, the `METRICS` singleton, and health checks |
| [testing_strategy.md](testing_strategy.md) | Current test scaffolding, intended test layers, and manual smoke-check procedures |
| [known_limitations.md](known_limitations.md) | Intentional v1 design boundaries and scope exclusions |

All documentation reflects the current state of the codebase. Where a document
describes something that is planned but not yet implemented (for example, the
test suite), it says so explicitly rather than describing aspirational behavior
as if it already exists.

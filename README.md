# RAG Engineering Lab

** Oren Salami | AI Systems Engineer **

A local-first RAG retrieval evaluation system for comparing chunking strategies, embedding providers, vector stores, and retrieval quality using measurable IR metrics.

---

## Project Overview

The **RAG Engineering Lab** is a modular, local-first system for building and evaluating the retrieval layer of Retrieval-Augmented Generation systems.

The project focuses on the technical backbone of RAG:

```text
Ingest → Clean → Chunk → Embed → Store → Retrieve → Evaluate
```

It includes a Streamlit UI, mock-first execution mode, ChromaDB and FAISS support, multiple embedding providers, retrieval evaluation metrics, demo notebooks, Docker support, and a standalone static portfolio site.

This is not a full LLM chat application.
It is a retrieval quality lab designed to measure, compare, and explain how different RAG retrieval configurations behave.

---

## Purpose

The purpose of this project is to demonstrate how to design, implement, evaluate, and present a professional RAG retrieval system.

This project demonstrates:

- How to build a modular RAG retrieval pipeline
- How to compare chunking strategies
- How to support multiple embedding providers
- How to abstract vector stores behind a common interface
- How to evaluate retrieval quality using ground-truth questions
- How to run a RAG system locally without paid APIs
- How to package the project with Docker
- How to present a technical AI system with notebooks, documentation, and a static portfolio page

---

## Core Capabilities

- Local-first RAG retrieval pipeline
- Streamlit UI with four main tabs
- Document ingestion for PDF, TXT, Markdown, HTML, and HTM
- Text cleaning and normalization
- Fixed and recursive chunking strategies
- Mock embeddings for offline deterministic execution
- HuggingFace embedding support
- Optional OpenAI embedding provider
- Optional Gemini embedding provider
- ChromaDB vector store support
- FAISS vector store support with metadata sidecar
- Retrieval flow with ranked chunks
- Optional reranker interface
- Ground-truth evaluation workflow
- Recall@K, Precision@K, MRR, and nDCG@K metrics
- CSV export for evaluation results
- Security validation for files and queries
- Prompt-safety warning layer
- Optional PII redaction
- Observability metrics and health checks
- Docker Compose support
- Demo notebooks
- Static portfolio site

---

## Architecture & System Design

The system is built around a thin orchestration layer called RAGPipeline.

The pipeline owns the active embedding provider, chunker, vector store, and optional reranker. This guarantees that the same embedding provider instance is used for both indexing and querying.

Raw Document
    ↓
Document Loader
    ↓
Text Cleaner
    ↓
Chunker
    ↓
Embedding Provider
    ↓
Vector Store
    ↓
Retriever
    ↓
Evaluation Engine

Main layers:
app/
└── Streamlit UI

src/core/
└── Config, contracts, DTOs, exceptions, logging

src/ingestion/
└── Document loading, cleaning, metadata extraction

src/chunking/
└── FixedChunker and RecursiveChunker

src/embeddings/
└── Mock, HuggingFace, OpenAI, Gemini providers

src/vectorstores/
└── ChromaDB and FAISS implementations

src/retrieval/
└── Retriever, reranker, hybrid retriever stub

src/pipeline/
└── RAGPipeline orchestrator

src/evaluation/
└── Metrics, ground truth loader, evaluator

src/security/
└── File validation, query validation, prompt safety, PII redaction

src/observability/
└── Metrics, tracing, health checks

The architecture is intentionally modular so that each component can be replaced without rewriting the full system.

See [docs/architecture.md](docs/architecture.md) for a full layer-by-layer breakdown, including class-level design decisions.

---

## Documentation

Detailed technical documentation lives in [docs/](docs/):

| Document | Covers |
|----------|--------|
| [docs/architecture.md](docs/architecture.md) | Layer-by-layer system design |
| [docs/API_KEYS_GUIDE.md](docs/API_KEYS_GUIDE.md) | Environment variables and API key setup |
| [docs/evaluation_report.md](docs/evaluation_report.md) | Metric definitions and a baseline mock-mode evaluation run |
| [docs/experiment_plan.md](docs/experiment_plan.md) | How to design controlled retrieval experiments |
| [docs/security_observability.md](docs/security_observability.md) | Security controls and the observability layer in detail |
| [docs/testing_strategy.md](docs/testing_strategy.md) | Test layers, current test scaffolding status, and manual smoke checks |
| [docs/known_limitations.md](docs/known_limitations.md) | Intentional v1 design boundaries and scope exclusions |

---

## Tech Stack

Layer	            Technologies
Language	      Python 3.11
UI	              Streamlit
Pipeline	      Custom modular RAGPipeline
Ingestion	      PyPDF, BeautifulSoup
Chunking	      FixedChunker, RecursiveChunker
Embeddings	      Mock, SentenceTransformers, OpenAI, Gemini
Vector Stores	ChromaDB, FAISS
Evaluation	      Recall@K, Precision@K, MRR, nDCG@K
Data Handling	Pandas, JSON, CSV
Config	      python-dotenv, Pydantic Settings
Observability	Custom METRICS singleton, timing, health checks
Security	      File validation, query validation, prompt safety, PII redaction
Packaging	      Docker, Docker Compose
Notebooks	      Jupyter notebooks
Static Site	      HTML, CSS, Vanilla JavaScript
Testing	      Pytest (scaffolded; see docs/testing_strategy.md), mock-mode manual smoke checks

---

## Repository Structure

rag-retrieval-evaluation-lab/
├── app/
│   └── streamlit_app.py
├── data/
│   ├── ground_truth/
│   │   └── questions.json
│   ├── raw/
│   └── processed/
├── docs/
│   ├── README.md
│   ├── architecture.md
│   ├── API_KEYS_GUIDE.md
│   ├── evaluation_report.md
│   ├── experiment_plan.md
│   ├── security_observability.md
│   ├── testing_strategy.md
│   └── known_limitations.md
├── notebooks/
│   ├── rag_retrieval_demo.ipynb
│   └── rag_evaluation_experiments.ipynb
├── results/
├── src/
│   ├── chunking/
│   ├── core/
│   ├── embeddings/
│   ├── evaluation/
│   ├── ingestion/
│   ├── observability/
│   ├── pipeline/
│   ├── retrieval/
│   ├── security/
│   └── vectorstores/
├── static-site/
│   ├── index.html
│   ├── styles.css
│   ├── app.js
│   └── README.md
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── vector_db/
│   ├── chroma/
│   └── faiss/
├── .env.example
├── Dockerfile
├── docker-compose.yml
├── requirements.txt
└── README.md

---

## How to Run

Clone the repository:

```bash
git clone https://github.com/Oren1984/rag-retrieval-evaluation-lab.git
cd rag-retrieval-evaluation-lab
```

Create a virtual environment:

```bash
python -m venv .venv
```

Activate it:

# Windows
```bash
.venv\Scripts\activate
```

# macOS / Linux
```bash
source .venv/bin/activate
```

Install dependencies:

```bash
pip install -r requirements.txt
```

Create an environment file:

```bash
cp .env.example .env
```

See [docs/API_KEYS_GUIDE.md](docs/API_KEYS_GUIDE.md) for API key and environment setup.

Run the Streamlit app:

```bash
streamlit run app/streamlit_app.py
```

Default URL:

http://localhost:8501

If port 8501 is busy:

```bash
streamlit run app/streamlit_app.py --server.port 8511
```

Then open:

http://localhost:8511

---

## Docker

Build and run with Docker Compose:

```bash
docker compose up --build
```

To stop:

```bash
docker compose down
```

Runtime folders are mounted:
data/
vector_db/
results/

---

## Evaluation Workflow

The evaluation layer measures retrieval quality against ground-truth questions.

Default ground truth file:

data/ground_truth/questions.json

Evaluation flow:

1. Build an index from a document
2. Load ground-truth questions
3. Run each question through the retriever
4. Compare retrieved doc IDs against relevant doc IDs
5. Compute IR metrics
6. Export results to CSV

Metrics:

Metric	      Meaning
Recall@K	      Did the relevant document appear in the top-K results?
Precision@K	      How many of the top-K results were relevant?
MRR	            How early did the first relevant result appear?
nDCG@K	      How good was the ranking quality compared to the ideal order?

Output:

results/evaluation_results.csv

See [docs/evaluation_report.md](docs/evaluation_report.md) for metric definitions, a baseline mock-mode run, and known limitations of the evaluation, and [docs/experiment_plan.md](docs/experiment_plan.md) for how to run controlled comparisons across chunkers, embedding providers, and vector stores.

---

## Demo Notebooks

The project includes two educational notebooks:

Notebook	Purpose
notebooks/rag_retrieval_demo.ipynb	End-to-end retrieval pipeline walkthrough in mock mode
notebooks/rag_evaluation_experiments.ipynb	Ground-truth evaluation workflow and metric explanation

Run notebooks from the repository root:
jupyter notebook

Both notebooks run in mock mode by default and do not require OpenAI or Gemini API keys.

---

## Static Portfolio Site

The repository includes a standalone static portfolio page:

static-site/

Run locally:

```bash
cd static-site
python -m http.server 8000
```

Open:

'http://localhost:8000'

The static site is built with:

HTML + CSS + Vanilla JavaScript

No framework, no build step, and no npm install are required.

The static site presents:

- Project overview
- Architecture
- Capabilities
- Evaluation metrics
- Security and observability
- Testing and reliability
- Tech stack
- Project boundaries

---

## Testing

Run tests:

```bash
pytest
```

Run specific layers:
pytest tests/unit/
pytest tests/integration/
pytest tests/e2e/

Run coverage if configured:

```bash
pytest --cov=src
```

The preferred validation path is mock mode:

MOCK_MODE=true
EMBEDDING_PROVIDER=mock

This allows tests and smoke checks to run without paid API keys or external services.

The test directories (`tests/unit/`, `tests/integration/`, `tests/e2e/`) currently provide scaffolding only — see [docs/testing_strategy.md](docs/testing_strategy.md) for the current test coverage status, intended test layers, and manual smoke-check procedures.

---

## Observability

The project includes lightweight local observability.

Tracked metrics include:

- Documents loaded
- Chunks created
- Query count
- Error count
- Embedding latency
- Retrieval latency
- Rerank latency
- Last error
- Selected embedding provider
- Selected vector store

The Streamlit System Status tab displays runtime metrics and readiness information.

Observability modules:

src/observability/metrics.py
src/observability/tracing.py
src/observability/health.py

See [docs/security_observability.md](docs/security_observability.md) for full detail on both the observability and security layers.

---

## Security

The system includes practical local security controls.

Security features:

- File extension validation
- Upload size limit
- Safe filename handling
- Path traversal prevention
- Query validation
- Top-K validation
- Chunk size and overlap validation
- Prompt-safety warning patterns
- Optional PII redaction
- API keys loaded from environment variables only
- No secrets hardcoded in source code

This is not an enterprise security product, but it demonstrates secure defaults and safe local execution practices.

---

## What This Project Demonstrates

This project demonstrates my ability to:

- Design modular AI retrieval systems
- Build a local-first RAG architecture
- Separate ingestion, chunking, embeddings, storage, retrieval, and evaluation
- Work with vector stores such as ChromaDB and FAISS
- Implement provider-based architecture
- Build mock-safe systems that do not depend on paid APIs
- Add retrieval evaluation with standard IR metrics
- Build a practical Streamlit interface
- Add observability and validation controls
- Package systems with Docker
- Create portfolio-ready documentation, notebooks, and a static project site

---

## Current Status

Status: Portfolio-ready / Local-first / Validated through manual mock-mode smoke checks

Completed:

- Core RAG pipeline
- Streamlit UI
- Mock embedding provider
- ChromaDB vector store
- FAISS vector store
- HuggingFace provider support
- Optional OpenAI/Gemini provider support
- Evaluation engine
- Demo notebooks
- Static portfolio site
- Docker build support
- Documentation suite

The project is ready for portfolio presentation and technical review.

---

## Known Limitations

- Mock embeddings are deterministic but not semantic
- Evaluation with mock embeddings validates plumbing, not semantic retrieval quality
- HuggingFace models may download on first use
- FAISS installation may vary by platform
- Docker requires Docker Desktop or a running Docker daemon
- No multi-user authentication
- No LLM generation in v1
- No chat interface
- No agents, MCP, or CrewAI
- No Kubernetes or Terraform deployment in v1
- No cloud deployment automation in v1

See [docs/known_limitations.md](docs/known_limitations.md) for the full list with rationale and workarounds.

---

## Final Note

This project is part of my AI Engineering portfolio.

It focuses on the retrieval layer of RAG systems and demonstrates how to build a measurable, modular, local-first retrieval evaluation lab.

The goal is not to create another generic chatbot.
The goal is to show how retrieval quality can be engineered, tested, evaluated, and presented clearly.

---

## License

This project is licensed under the MIT License. See the LICENSE file for details.

---

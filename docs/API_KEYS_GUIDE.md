# API Keys & Environment Setup Guide — RAG Engineering Lab

This guide explains how to configure local environment variables and optional API keys for running the project safely.

## Required Variables

The project supports local/demo execution without paid API keys.

For offline-safe local usage, use:

- `MOCK_MODE=true`
- `EMBEDDING_PROVIDER=mock`

In this mode, real external API keys are optional and can be left empty.

Core non-secret runtime variables used by the project include:

- `APP_ENV`
- `LOG_LEVEL`
- `MOCK_MODE`
- `EMBEDDING_PROVIDER`
- `EMBEDDING_MODEL`
- `VECTOR_STORE`
- `CHUNK_STRATEGY`
- `CHUNK_SIZE`
- `CHUNK_OVERLAP`
- `TOP_K`
- `USE_RERANKER`
- `DATA_DIR`
- `VECTOR_DB_DIR`
- `RESULTS_DIR`
- `MAX_UPLOAD_MB`

## Optional API Keys

The repository currently uses the following optional external API key variables:

- `OPENAI_API_KEY` (required only when `EMBEDDING_PROVIDER=openai`)
- `GEMINI_API_KEY` (required only when `EMBEDDING_PROVIDER=gemini`)

No other API key variables from common provider lists are used in runtime configuration.

## Local .env Setup

From the repository root, create a local `.env` file:

```bash
cp .env.example .env
```

For Windows PowerShell:

```powershell
Copy-Item .env.example .env
```

Then edit `.env` locally for your machine.

Never commit real secrets.

## Example .env Structure

Use placeholders only:

```dotenv
APP_ENV=local
LOG_LEVEL=INFO

MOCK_MODE=true
EMBEDDING_PROVIDER=mock
EMBEDDING_MODEL=mock-384

VECTOR_STORE=chroma
CHUNK_STRATEGY=recursive
CHUNK_SIZE=500
CHUNK_OVERLAP=100
TOP_K=5
USE_RERANKER=false

OPENAI_API_KEY=your_openai_key_here
GEMINI_API_KEY=your_gemini_key_here

DATA_DIR=data
VECTOR_DB_DIR=vector_db
RESULTS_DIR=results
MAX_UPLOAD_MB=10
```

## Demo / Offline Mode

The project can run without paid API keys.

Recommended offline/local demo settings:

```dotenv
MOCK_MODE=true
EMBEDDING_PROVIDER=mock
```

In Streamlit, keep mock mode enabled and select the `mock` provider for deterministic local behavior.

If you switch to `openai` or `gemini`, set the corresponding key in `.env`.

## Security Best Practices

- Never commit `.env`.
- Keep `.env` in `.gitignore`.
- Use GitHub Secrets for CI/CD variables.
- Rotate keys immediately if exposed.
- Do not paste real API keys into README, screenshots, notebooks, or docs.
- Use placeholder values in documentation.

## Troubleshooting

### Missing API key

- Symptom: errors like `OPENAI_API_KEY not set` or `GEMINI_API_KEY not set`.
- Fix: add the correct key to `.env`, or switch back to mock mode.

### Invalid API key

- Symptom: provider authentication/authorization failures.
- Fix: verify the key value, project/account permissions, and provider billing status.

### .env file not loaded

- Symptom: app behaves as if defaults are still active.
- Fix: ensure `.env` exists at repository root and restart the app.

### Wrong variable name

- Symptom: key appears set but is ignored.
- Fix: use exact names: `OPENAI_API_KEY` and `GEMINI_API_KEY`.

### Wrong working directory

- Symptom: relative paths or `.env` are not detected.
- Fix: run commands from the repository root.

### Docker Compose not receiving environment variables

- Symptom: container does not see expected settings.
- Fix: ensure `.env` is present before `docker compose up --build`, and recreate containers after changes.
# NexusRAG — Agent Guide

## Dev Setup

```bash
cp .env.example .env          # set GOOGLE_AI_API_KEY or switch LLM_PROVIDER=ollama
docker compose -f docker-compose.services.yml up -d   # PG:5433 + ChromaDB:8002
python3 -m venv venv && source venv/bin/activate && pip install -r backend/requirements.txt
cd frontend && pnpm install && cd ..
# Optional: python backend/scripts/download_models.py  # ~2.5GB
```

## Run Dev Servers

```bash
./run_bk.sh   # Backend → uvicorn app.main:app --reload --port 8080
./run_fe.sh   # Frontend → pnpm dev, :5174, proxies /api → :8080
```
Or full Docker: `docker compose up -d`

## Tests & Lint

- **No tests exist** — skip `pytest` / `jest`
- Frontend: `cd frontend && pnpm lint && pnpm build` (build = `tsc -b` + `vite build`)
  - TS strict: `noUnusedLocals` + `noUnusedParameters` on — unused imports/vars break build
- Backend: no lint/typecheck tool in `requirements.txt` — review imports & syntax manually

## Key Config (`.env`)

Non-obvious vars:

| Var | Default | Note |
|---|---|---|
| `LLM_PROVIDER` | `gemini` | or `ollama` |
| `KG_EMBEDDING_PROVIDER` | `gemini` | or `ollama` / `sentence_transformers` (local) |
| `OLLAMA_ENABLE_THINKING` | `false` | Set true for thinking models (qwen3.5) |
| `NEXUSRAG_KG_LANGUAGE` | `English` | Overridable per workspace in UI |
| `NEXUSRAG_DOCUMENT_PARSER` | `docling` | or `marker` (lighter, better math) |
| `NEXUSRAG_PROCESSING_TIMEOUT_MINUTES` | `10` | Stale processing docs → FAILED |

Gemini: 2.5 → auto thinking_budget_tokens, 3.x → `LLM_THINKING_LEVEL`. Ollama: native tool calling auto-detected via probe.

## API Endpoints

All under `/api/v1`. Swagger: `http://localhost:8080/docs`

Non-obvious: `GET /config/status` (active providers), `POST /rag/chat/{workspace_id}/rate` (thumbs up/down), `POST /rag/chat/{workspace_id}/stream` (SSE, supports `metadata_filter`).

## MCP Server

```bash
cd mcp-server && pnpm install && npx tsc && node dist/index.js
```
Standalone TS, Streamable HTTP at `/mcp`, port 8000.

## Alembic

Only 1 migration (`custom_metadata`). New: `cd backend && alembic revision --autogenerate -m "msg"`. Apply: `alembic upgrade head`.

## Agent Gotchas

- Backend config reads `.env` from **project root** (not `backend/`)
- Frontend TS strict kills build on unused imports/vars
- ML models cache at `~/.cache/huggingface/` (Docker layer too)
- Doc images served from `backend/data/docling/` under `/static/doc-images/`

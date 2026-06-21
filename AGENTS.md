# NexusRAG — Agent Guide

## Dev Setup

```bash
cp .env.example .env          # edit GOOGLE_AI_API_KEY or switch LLM_PROVIDER=ollama
docker compose -f docker-compose.services.yml up -d   # PostgreSQL:5433 + ChromaDB:8002
python3 -m venv venv && source venv/bin/activate && pip install -r backend/requirements.txt
# Optional: python backend/scripts/download_models.py  # ~2.5GB (bge-m3 + reranker)
cd frontend && pnpm install && cd ..
```

## Run Dev Servers

```bash
./run_bk.sh   # Backend: uvicorn app.main:app --reload --port 8080 (from backend/)
./run_fe.sh   # Frontend: pnpm dev (port 5174, proxies /api → :8080)
```
Or full Docker: `docker compose up -d`

## Project Structure

- `backend/app/` — Python FastAPI
  - `main.py` — app entry, lifespan (auto-creates tables from `schema.sql`), static files mount
  - `api/` — routers: workspaces, documents, rag (chat/query), config
  - `core/config.py` — pydantic-settings, reads `.env` from project root
  - `core/database.py` — SQLAlchemy async engine + session
  - `services/` — nexus_rag_service (orchestrator), deep_retriever (hybrid), vector_store (ChromaDB), knowledge_graph_service, embedder, reranker, chunker, document_parser/
  - `scripts/` — download_models.py, eval_rag.py, eval_ragas_synthetic.py
  - `schema.sql` — raw SQL executed at startup if no alembic_version table
  - `alembic/` — only 1 migration (custom_metadata). New migrations: `cd backend && alembic revision --autogenerate -m "msg"`
- `frontend/` — React 19 + Vite 7 + TailwindCSS 4 + TypeScript 5.9
  - `@/` path alias → `src/`
  - `package.json`: build = `tsc -b && vite build`, lint = `eslint .`
- `mcp-server/` — standalone TypeScript MCP, port 8000, Streamable HTTP transport
- `tasks/todo.md` — active work plan (pgvector support)

## Key Config (`.env`)

| Var | Default | Notes |
|---|---|---|
| `LLM_PROVIDER` | `gemini` | `gemini` or `ollama` |
| `KG_EMBEDDING_PROVIDER` | `gemini` | Also `ollama` or `sentence_transformers` (fully local) |
| `NEXUSRAG_DOCUMENT_PARSER` | `docling` | `docling` (default) or `marker` (lighter, better math) |
| `NEXUSRAG_ENABLE_KG` | `true` | Toggle knowledge graph |
| `NEXUSRAG_DEDUP_ENABLED` | `true` | Content-hash + near-duplicate filter |
| `NEXUSRAG_VECTOR_PREFETCH` | `20` | Candidates before rerank |
| `NEXUSRAG_RERANKER_TOP_K` | `8` | Final results |
| `NEXUSRAG_PROCESSING_TIMEOUT_MINUTES` | `10` | Stale docs → FAILED |

Gemini models auto-select thinking mode (2.5 → budget, 3.x → level). Ollama: native tool calling auto-detected via probe.

## API Endpoints

All under `/api/v1`. Swagger: `http://localhost:8080/docs`

Key:
- `POST /documents/upload/{workspace_id}` — supports `custom_metadata` list
- `POST /rag/query/{workspace_id}` — hybrid search, supports `metadata_filter`
- `POST /rag/chat/{workspace_id}/stream` — SSE streaming chat, supports `metadata_filter`
- `GET /rag/graph/{workspace_id}` — KG visualization data

ML models cache at `~/.cache/huggingface/` (also Docker layer).

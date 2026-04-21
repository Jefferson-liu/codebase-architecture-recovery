# CLAUDE.md — Project Context for AI Agents

## What this repo is

A self-hosted codebase assistant. You point it at a Git repository, it indexes the code (file summaries, embeddings, architecture analysis), and exposes a chat UI where you can ask questions about the codebase. The agent answers using MCP tools that query the indexed knowledge — not hallucination.

## Architecture — 4 Docker services

```
postgres (5433)
  └── context-backend (8000)   ← Python/FastAPI — repo ingestion, MCP server, knowledge DB
        └── agent-backend (4111) ← Node/Express + Mastra — Claude agent, chat API
              └── frontend (3000)  ← React/Vite — chat UI, GitHub panel, repo browser
```

All services are configured via a single `docker.env` file (not committed — see `.env.example`).

## Submodule repos

| Directory | Repo | Role |
|---|---|---|
| `context-retrieval-backend/` | `Jefferson-liu/context-retrieval-backend` | Python backend |
| `context-retrieval-agent-test/` | `Jefferson-liu/context-retrieval-agent-test` | Node agent + frontend |

Always work inside the submodule directories, not the root. The root repo only holds `docker-compose.yml`, `docker.env`, and submodule pointers.

## context-retrieval-backend (Python/FastAPI)

**Entry point:** `main.py` — mounts all routers, starts background runners.

**Key paths:**
- `config/settings.py` — all env var defaults and validation
- `services/repo_knowledge/` — the core pipeline
  - `file_summarizer.py` — per-file LLM summarization (ReAct agent)
  - `file_summary_service.py` — orchestrates parallel summarization runs
  - `mcp_app.py` — FastMCP server exposing tools to the agent
  - `mcp_tools_service.py` — implements: `overview`, `architecture`, `file_tree`, `list_modules`, `get_module`, `search_file_summaries`, `get_file_summary`, `get_file`, `grep_code`, `glob_files`
- `infrastructure/database.py` — SQLAlchemy async session setup
- `routers/repo_knowledge/` — REST endpoints for triggering and monitoring pipeline runs

**Pipeline stages** (each is a background runner with its own run table):
1. File ingestion — chunks source files into DB
2. File summarization — LLM summaries per file
3. Embedding — vector embeddings for semantic search
4. Repo full summary — README-style overview + Mermaid architecture diagram
5. Repo manager — groups files into architectural modules

**Model config** (all via env vars in `config/settings.py`):
- `REPO_SUMMARY_MODEL_PROVIDER` — `gemini` | `anthropic` | `openai` (default: `gemini`)
- `REPO_EMBED_MODEL_PROVIDER` — same options (default: `gemini`)
- Corresponding API keys: `GEMINI_API_KEY`, `ANTHROPIC_API_KEY`, `PERSONAL_OPENAI_KEY`

## context-retrieval-agent-test (Node/TypeScript)

**Structure:**
```
backend/src/
  agent/agent.ts        ← Mastra Agent setup, model cache, MCP tool wrappers
  routes/chat.ts        ← POST /api/chat (SSE streaming), thread management
  index.ts              ← Express app, proxy /api/context → context-backend
frontend/src/
  App.tsx               ← Main chat UI, mermaid rendering, model selector
  components/           ← GitHubPanel, RepoBrowser, RunDiagnosticsModal, ContextBar
  hooks/useGitHub.ts    ← GitHub OAuth + repo indexing state
```

**Agent model selection:** Frontend sends `model` field in POST body. `getAgentForModel(model)` in `agent.ts` returns a cached Mastra Agent for that model. Available: `claude-haiku-4-5`, `claude-sonnet-4-6` (default), `claude-opus-4-7`.

**MCP scope injection:** Every MCP tool call has `run_id` injected from `scope-store.ts` so tools query the correct indexed snapshot. The LLM never sees or controls `run_id`.

**Agent instructions rule:** The agent must never answer codebase questions from training data. Every claim must be grounded in tool output. Use `get_file_summary` for high-level context; use `get_file` / `grep_code` only when making specific code references.

## Environment

Configuration lives in `docker.env` (gitignored). Template at `.env.example`.

The minimum required keys:
- `ANTHROPIC_API_KEY` — for the agent (all Claude models)
- `GEMINI_API_KEY` + `GOOGLE_GENERATIVE_AI_API_KEY` — for backend summarization/embeddings if `REPO_SUMMARY_MODEL_PROVIDER=gemini`

`REPO_INGEST_ALLOWED_ROOTS` must list the container paths of any repos you want to index (comma-separated). Mount those paths in `docker-compose.yml` under the `context-backend` volumes section.

## Common tasks

**Rebuild after backend changes:**
```bash
docker compose build context-backend && docker compose up -d context-backend
```

**Rebuild after agent/frontend changes:**
```bash
docker compose build agent-backend frontend && docker compose up -d agent-backend frontend
```

**View logs:**
```bash
docker compose logs -f context-backend
docker compose logs -f agent-backend
```

**Index a repo via API:**
```bash
curl -X POST http://localhost:8000/repo-knowledge/pipeline-runs \
  -H "Content-Type: application/json" \
  -d '{"repo_path": "/repos/my-project"}'
```

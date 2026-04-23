# Context Retrieval Agent

A self-hosted codebase assistant. Point it at a Git repository, let it index the code, then chat with an AI agent that answers questions grounded in your actual source — not hallucinations.

**Stack:** Python/FastAPI backend · Node/Mastra agent · React/Vite frontend · Postgres + pgvector · MCP (Model Context Protocol)

---

## Table of contents

1. [How it works](#how-it-works)
2. [Quick start](#quick-start)
3. [Configuration](#configuration)
4. [Indexing a repository](#indexing-a-repository)
5. [Using the chat UI](#using-the-chat-ui)
6. [Architecture](#architecture)
7. [Running outside Docker](#running-outside-docker)
8. [Deploying to Railway](#deploying-to-railway)
9. [Updating](#updating)
10. [Debugging & common errors](#debugging--common-errors)
11. [Edge cases](#edge-cases)
12. [Extending the system](#extending-the-system)
13. [Repository layout](#repository-layout)

---

## How it works

1. You connect a GitHub repo (or mount a local path) and trigger indexing.
2. The backend runs a pipeline: **file ingestion → per-file LLM summaries → embeddings → repo-wide summary → architectural module grouping**.
3. Each stage writes to its own Postgres tables, scoped by a `run_id`.
4. The agent connects to an MCP server exposed by the backend. It gets tools for: repo overview, architecture diagram, file tree, module list, file summaries, file content, grep, and glob.
5. You chat with the agent. `run_id` is auto-injected on every tool call so answers are scoped to the correct snapshot. The agent's instructions forbid answering from training data — every claim comes from a tool call.

---

## Quick start

### Prerequisites

- **[Docker Desktop](https://docs.docker.com/get-docker/)** running
- **Anthropic API key** (required — used by the chat agent)
- **Gemini API key** (required by default — used for backend summarization and embeddings)

### 1. Clone with submodules

```bash
git clone --recurse-submodules https://github.com/Jefferson-liu/CR.git
cd CR
```

Already cloned without `--recurse-submodules`? Run:

```bash
git submodule update --init --recursive
```

### 2. Configure environment

```bash
cp .env.example docker.env
```

Edit `docker.env` and fill in at minimum:

```env
ANTHROPIC_API_KEY=sk-ant-...
GEMINI_API_KEY=AIza...
GOOGLE_GENERATIVE_AI_API_KEY=AIza...   # same value as GEMINI_API_KEY
REPO_INGEST_ALLOWED_ROOTS=/repos
```

### 3. Mount a repo for indexing

Edit `docker-compose.yml`, find the `context-backend` service, and replace the default `repos:/repos` volume with a bind mount:

```yaml
context-backend:
  volumes:
    - /absolute/path/to/your/project:/repos/my-project:ro
```

The container path must start with a prefix listed in `REPO_INGEST_ALLOWED_ROOTS`.

### 4. Start the stack

```bash
docker compose up --build
```

Services come up at:

| Service | URL | Purpose |
|---|---|---|
| Frontend | http://localhost:3000 | Chat UI |
| Agent API | http://localhost:4111 | Chat + proxy to backend |
| Context API | http://localhost:8000 | Pipeline, REST, MCP |
| Postgres | localhost:5433 | Knowledge DB (pgvector) |

### 5. Index and chat

1. Open http://localhost:3000.
2. Click **GitHub** in the sidebar.
3. Enter a repo path (e.g. `/repos/my-project`) and trigger indexing.
4. Wait for all stages to turn green in the diagnostics modal.
5. Pick the repo from the sidebar and start chatting.

---

## Configuration

All configuration flows through `docker.env` (gitignored). The template `.env.example` documents every key.

### Required

| Variable | Purpose |
|---|---|
| `ANTHROPIC_API_KEY` | The chat agent (Haiku / Sonnet / Opus) |
| `GEMINI_API_KEY` | Backend summarization + embeddings (default provider) |
| `GOOGLE_GENERATIVE_AI_API_KEY` | Same value as `GEMINI_API_KEY` (used by Google SDK) |
| `REPO_INGEST_ALLOWED_ROOTS` | Comma-separated allowed container paths for ingestion, e.g. `/repos` |

### Model provider switching

The backend summarizer and embedder can each target a different provider:

```env
REPO_SUMMARY_MODEL_PROVIDER=gemini      # gemini | anthropic | openai
REPO_EMBED_MODEL_PROVIDER=gemini        # gemini | anthropic | openai
```

If you switch either to `anthropic`, `ANTHROPIC_API_KEY` covers it. If you switch to `openai`, set `PERSONAL_OPENAI_KEY`.

### Performance knobs

| Variable | Default | What it does |
|---|---|---|
| `SEMAPHORE_LIMIT` | `5` | Max concurrent per-file summarization tasks |
| `LOG_REPO_PROGRESS` | `1` | Emit progress logs during pipeline runs |
| `VECTOR_STORE_MODE` | `pgvector` | How embeddings are stored |

### Service URLs (don't change when running via Docker Compose)

```env
DATABASE_URL=postgresql+asyncpg://context:contextpass@postgres:5432/context
CONTEXT_API_BASE_URL=http://context-backend:8000/
MCP_SERVER_URL=http://context-backend:8000/mcp
FRONTEND_ORIGIN=http://localhost:3000
VITE_AGENT_API_BASE_URL=http://localhost:4111
VITE_CONTEXT_API_BASE_URL=http://localhost:8000/
```

`VITE_*` values are baked into the frontend bundle at build time. Changing them requires `docker compose build frontend`.

---

## Indexing a repository

### Via the UI

1. Open the **GitHub** tab.
2. Either sign in with GitHub and pick a repo (it will be cloned into the mounted repos volume) or enter a path that already exists inside the container.
3. Click **Index**.
4. Watch the diagnostics modal — five stages run sequentially.

### Via the API

```bash
curl -X POST http://localhost:8000/repo-knowledge/pipeline-runs \
  -H "Content-Type: application/json" \
  -d '{"source_type": "local_path", "repo_path": "/repos/my-project"}'
```

The response includes a `run_id`. Poll status:

```bash
curl http://localhost:8000/repo-knowledge/pipeline-runs/<run_id>
```

### Pipeline stages

Each stage has its own run table, error surface, and diagnostics:

| # | Stage | Produces |
|---|---|---|
| 1 | **Ingestion** | File chunks in `repo_file_chunk` |
| 2 | **File summarization** | Per-file LLM summaries |
| 3 | **Embedding** | pgvector embeddings per subject |
| 4 | **Repo full summary** | README markdown + Mermaid diagram |
| 5 | **Repo manager** | Architectural module groups |

Stages 2–5 depend on stage 1. If one fails, the pipeline stops — you can inspect the exact failure in the diagnostics modal or via `/repo-knowledge/file-summary-runs/<id>/diagnostics`.

---

## Using the chat UI

- **Model selector** (bottom of chat input): `claude-haiku-4-5` (fast), `claude-sonnet-4-6` (default), `claude-opus-4-7` (most capable).
- **Sidebar → GitHub tab**: manage connected repos, trigger re-indexing, inspect diagnostics.
- **Sidebar → repo picker**: switch the active `run_id` that the agent queries.
- **Mermaid diagrams** in agent replies are rendered inline.
- The agent cites exact file paths and line numbers. If it doesn't, prompt it to "show the exact path and line."

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                         Browser (3000)                           │
│                  React + Vite · chat UI · Mermaid                │
└────────────────────────────┬─────────────────────────────────────┘
                             │ HTTPS
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│                      agent-backend (4111)                        │
│       Node · Express · Mastra Agent (Claude) · MCP client        │
│            /api/chat  (SSE)       /api/context (proxy)           │
└───────────────────────────┬──────────────────────────────────────┘
                            │ internal HTTP
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│                    context-backend (8000)                        │
│        Python · FastAPI · FastMCP server · pipeline runners      │
│   REST:   /repo-knowledge/*          MCP:   /mcp                 │
└───────────────────────────┬──────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│                   postgres (5433, pgvector)                      │
│    runs · subjects · chunks · summaries · embeddings · groups    │
└──────────────────────────────────────────────────────────────────┘
```

**Why MCP?** The agent's tools (`overview`, `file_tree`, `get_file`, `grep_code`, …) are declared on the backend as an MCP server. The agent connects as an MCP client. This means you can point any MCP-capable host (Claude Desktop, other agents) at the same backend without changing code.

**Why the proxy?** The frontend calls `/api/context/*` on the agent backend, which proxies to the context backend. This avoids CORS complexity and lets the frontend talk to one origin.

**Scope injection.** Every MCP tool call flows through a wrapper in `backend/src/agent/agent.ts → wrapToolsWithScope`. The wrapper pulls the active `run_id` from AsyncLocalStorage (set per chat request from the frontend's selected repo) and injects it into tool args. The LLM never sees or controls `run_id`.

---

## Running outside Docker

### Backend (Python 3.11+)

```bash
cd context-retrieval-backend
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
# Point DATABASE_URL at a local Postgres with pgvector
export DATABASE_URL="postgresql+asyncpg://user:pass@localhost:5432/context"
uvicorn main:app --host 0.0.0.0 --port 8000 --reload
```

### Agent backend (Node 20+)

```bash
cd context-retrieval-agent-test
npm install
export CONTEXT_API_BASE_URL=http://localhost:8000/
export MCP_SERVER_URL=http://localhost:8000/mcp
export ANTHROPIC_API_KEY=sk-ant-...
npm run dev:backend
```

### Frontend

```bash
cd context-retrieval-agent-test/frontend
npm install
VITE_AGENT_API_BASE_URL=http://localhost:4111 \
VITE_CONTEXT_API_BASE_URL=http://localhost:8000/ \
npm run dev
```

---

## Deploying to Railway

Four services: `postgres` (pgvector), `context-backend`, `agent-backend`, `frontend`.

### Postgres

- Image: `pgvector/pgvector:pg16`
- Env: `POSTGRES_USER=context`, `POSTGRES_PASSWORD=contextpass`, `POSTGRES_DB=context`

### context-backend

- Root directory: `context-retrieval-backend/`
- Dockerfile Path: `Dockerfile`
- Networking target port: `8000` (match the start command)
- Start command (wrap in `sh -c` because Railway runs commands in exec mode):
  ```
  sh -c "python -m uvicorn main:app --host 0.0.0.0 --port 8000"
  ```
- Env: copy keys from `docker.env`. Set `DATABASE_URL=postgresql+asyncpg://context:contextpass@${{postgres.RAILWAY_PRIVATE_DOMAIN}}:5432/context`.
- Must set `USE_PLACEHOLDER_SCOPE=1` if `APP_ENV=production` (otherwise MCP tools reject requests with "Missing tenant/user").

### agent-backend

- Root directory: `context-retrieval-agent-test/`
- Dockerfile Path: `backend/Dockerfile`
- Networking target port: `4111`
- Env:
  - `CONTEXT_API_BASE_URL=http://${{context-backend.RAILWAY_PRIVATE_DOMAIN}}:8000/` (trailing slash matters)
  - `MCP_SERVER_URL=http://${{context-backend.RAILWAY_PRIVATE_DOMAIN}}:8000/mcp`
  - `FRONTEND_ORIGIN=https://<your-frontend-public-domain>` (no trailing slash)
  - `ANTHROPIC_API_KEY=...`

### frontend

- Root directory: `context-retrieval-agent-test/`
- Dockerfile Path: `frontend/Dockerfile`
- Networking target port: whatever Vite preview binds to (match the start command)
- Start command:
  ```
  sh -c "cd frontend && npx vite preview --host 0.0.0.0 --port $PORT"
  ```
- Build-time env (baked into bundle):
  - `VITE_AGENT_API_BASE_URL=https://<agent-backend-public-domain>`
  - `VITE_CONTEXT_API_BASE_URL=https://<agent-backend-public-domain>/api/context/` (the agent proxies `/api/context` to the context backend)
- `frontend/vite.config.ts` already sets `preview.allowedHosts: true` so Railway's generated domain is accepted.

Browsers cannot resolve `*.railway.internal` — `VITE_*` values must be **public** HTTPS domains. Backend-to-backend calls (`CONTEXT_API_BASE_URL`, `MCP_SERVER_URL`) can and should use private domains.

---

## Updating

Pull latest root + submodule pointers, rebuild, restart:

```bash
git pull --recurse-submodules
docker compose build
docker compose up -d
```

To update only one service after editing its submodule:

```bash
docker compose build context-backend && docker compose up -d context-backend
docker compose build agent-backend frontend && docker compose up -d agent-backend frontend
```

---

## Debugging & common errors

### "unable to get image 'cr-agent-backend'"

Docker Desktop isn't running. Start it.

### "Missing tenant/user 'tenant:user'" (400 from MCP tools)

`USE_PLACEHOLDER_SCOPE` is off and no scope was injected. Either set `USE_PLACEHOLDER_SCOPE=1` or ensure your MCP client passes `tenant_id` + `user_id`.

### "Blocked request. This host is not allowed."

Vite preview is rejecting an unknown host. Verify `frontend/vite.config.ts` has `preview: { allowedHosts: true }` and rebuild the frontend image.

### 502 Bad Gateway on Railway

The service's networking target port doesn't match the port the app binds to. Either hardcode the port (e.g. `--port 8000`) and set target port to the same, or use `$PORT` in the start command and set target port to `$PORT`.

### `$PORT is not a valid integer`

Railway runs start commands in exec mode, so `$PORT` isn't expanded. Wrap in `sh -c "..."`.

### CORS preflight failure

`FRONTEND_ORIGIN` on the agent backend must **exactly** match the browser's origin — same scheme, same host, **no trailing slash**. `https://app.example.com/` ≠ `https://app.example.com`.

### Agent can't reach backend: `ECONNREFUSED` to `localhost:4111` or `...railway.internal`

- `localhost` in browser errors → `VITE_AGENT_API_BASE_URL` wasn't set at build time. Rebuild the frontend image with the correct value.
- `.railway.internal` in browser errors → private domain used where a public one is required. `VITE_*` must be public.

### `TypeError: Cannot read properties of null (reading 'split')` in agent proxy

`CONTEXT_API_BASE_URL` is malformed (missing scheme or port). Must be `http://host:port/` with the trailing slash.

### MCP tools return no data

- Agent is using a stale `run_id`. Pick the repo again in the sidebar — the active `run_id` is stored in AsyncLocalStorage per chat request.
- Pipeline didn't complete. Open the diagnostics modal; look for a red stage.
- Database migration didn't run. Restart `context-backend` — it runs migrations on startup.

### Summaries are empty or low-quality

- Check stage 2 diagnostics: `/repo-knowledge/file-summary-runs/<id>/diagnostics?limit=50`. Per-file errors (rate limits, parse failures, timeouts) are listed there. These show up in the modal as **warnings**, not errors — that's intentional; a few file failures out of thousands is normal.
- If using Gemini, watch for rate limit errors — lower `SEMAPHORE_LIMIT` if hitting quotas.

### Ingestion is slow

Currently the ingestion loop is sequential per file. For very large repos this dominates runtime. See [Extending the system](#extending-the-system) for parallelization notes.

### Stack starts but frontend shows a blank page

- Open browser devtools, check console for the exact URL that failed.
- Check `docker compose logs agent-backend` for proxy errors.
- Verify `FRONTEND_ORIGIN` in the agent backend matches the browser origin.

### Resetting state

Wipe everything:

```bash
docker compose down -v       # drops postgres_data, mastra_data, repos volumes
docker compose up --build
```

Wipe only chat history (keep indexed data):

```bash
docker compose down
docker volume rm cr_mastra_data
docker compose up -d
```

---

## Edge cases

- **Binary / generated files.** The ingestion pipeline skips files above a size threshold and files whose extensions aren't in the allowlist. Skipped files appear with `ingest_status=skipped` and a `skip_reason`.
- **Very large files.** Chunked into DB rows. `get_file` returns a bounded range with pagination metadata (`next_start_line`, `previous_start_line`).
- **Empty files.** Return `total_lines=0`, `content=""` — not an error.
- **Non-UTF-8 files.** Detected and skipped during ingestion with `skip_reason=encoding`.
- **Paths with backslashes (Windows-exported repos).** Normalized to forward slashes during ingestion.
- **Re-indexing.** Creates a new `run_id`. Old runs are kept until manually cleaned. Pick the newest run in the sidebar.
- **Multiple repos indexed simultaneously.** Works — each has its own `run_id`. The sidebar lets you switch the active scope per chat.
- **Agent hallucinating despite rules.** Try a smaller model (Haiku) first, then Sonnet. Agent instructions in `backend/src/agent/agent.ts` are aggressive about grounding; if you see drift, tighten them there.
- **Long file paths containing the repo root.** The backend stores `subject_path` relative to the repo root. When mounting, make sure the container path corresponds to what you want the agent to see.
- **Submodule within the indexed repo.** Indexed as plain files if present on disk; not traversed via `.gitmodules`.

---

## Extending the system

### Add a new MCP tool

1. Add the implementation in `context-retrieval-backend/services/repo_knowledge/mcp_tools_service.py` as a method on `RepoKnowledgeMcpService`.
2. Register it in `services/repo_knowledge/mcp_app.py` with `@mcp.tool(description=...)`.
3. Restart `context-backend`. The agent backend auto-discovers tools via `MCPClient.listTools()` on startup.
4. Update the agent's tool list in `context-retrieval-agent-test/backend/src/agent/agent.ts` instructions so the LLM knows when to use it.

Tool response design guidelines:
- Return plain dicts (the wrapper unwraps MCP's `content[].text` envelope automatically).
- Be stingy with fields — every key costs tokens on every call. Put heavy detail behind a separate `get_X` tool.
- Prefer compact representations (indented text for trees, e.g. `file_tree`) over deeply nested JSON.

### Add a new model

1. Append to `AVAILABLE_MODELS` in `context-retrieval-agent-test/backend/src/agent/agent.ts`.
2. Ensure the provider prefix matches Mastra's expectations (e.g. `anthropic/claude-...`).
3. For non-Anthropic providers, add credentials and adjust `getModelProvider` / `sanitizeModelSettingsForProvider`.

### Swap summarization/embedding providers

Set `REPO_SUMMARY_MODEL_PROVIDER` / `REPO_EMBED_MODEL_PROVIDER` in `docker.env`. Providers supported: `gemini`, `anthropic`, `openai`. Implementation in `services/repo_knowledge/summarization/model_factory.py` and `services/repo_knowledge/embeddings/`.

### Add a pipeline stage

1. Create a service (`services/repo_knowledge/<new_stage>_service.py`) and a background runner (`<new_stage>_background_runner.py`).
2. Add a run-record SQLAlchemy model in `infrastructure/models/repo_knowledge.py`.
3. Wire the runner into `services/repo_knowledge/pipeline_background_runner.py`.
4. Add a REST router in `routers/` for status polling.
5. Expose its outputs as an MCP tool (see above).

### Parallelize ingestion

Ingestion in `services/repo_knowledge/ingestion_service.py` currently processes files sequentially. To parallelize:
- Add a new setting `REPO_INGEST_MAX_CONCURRENT_FILES` in `config/settings.py`.
- Replace the `for snapshot in snapshots:` loop with `asyncio.gather` bounded by an `asyncio.Semaphore(n)`.
- Make sure DB writes inside the loop use short-lived sessions (commit per file) to avoid lock contention.

### Use your own MCP host (e.g. Claude Desktop)

The context backend's MCP server is reachable at `http://localhost:8000/mcp` when running via Docker Compose. It speaks streamable-http transport. Point any MCP-capable host at that URL — no agent-backend needed. You must pass `tenant_id` and `user_id` as tool args unless `USE_PLACEHOLDER_SCOPE=1`.

---

## Repository layout

```
CR/
├── README.md                         ← you are here
├── CLAUDE.md                         ← deep context for AI coding agents
├── docker-compose.yml                ← 4-service stack definition
├── .env.example                      ← template for docker.env
├── .gitmodules                       ← submodule pointers
├── context-retrieval-backend/        ← Python/FastAPI submodule
│   ├── main.py
│   ├── config/settings.py
│   ├── infrastructure/
│   ├── routers/
│   └── services/repo_knowledge/
└── context-retrieval-agent-test/     ← Node agent + frontend submodule
    ├── backend/src/
    │   ├── agent/agent.ts
    │   ├── routes/chat.ts
    │   └── index.ts
    └── frontend/src/
        ├── App.tsx
        └── components/
```

Always commit changes inside the submodule directory first, then bump the submodule pointer in the root repo. See `CLAUDE.md` for the exact workflow.

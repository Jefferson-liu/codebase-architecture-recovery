# CLAUDE.md — Context for AI coding agents

This file is the field manual. If you are a Claude agent (or any AI) asked to work on this repo, read this top to bottom before touching anything. The `README.md` is user-facing; this file is for people who will *edit* the code.

---

## 1. What this repo is

A self-hosted codebase assistant. Pipeline indexes a Git repo (chunks → LLM summaries → embeddings → architecture), an MCP server exposes tools over the index, and a Claude-based agent chats about the code grounded in those tools. Four Docker services wired together by `docker-compose.yml`.

## 2. Repo topology — submodules matter

```
CR/                                  ← root (this repo)
├── docker-compose.yml
├── docker.env                       ← gitignored; copy from .env.example
├── README.md, CLAUDE.md
├── context-retrieval-backend/       ← submodule: Jefferson-liu/context-retrieval-backend
└── context-retrieval-agent-test/    ← submodule: Jefferson-liu/context-retrieval-agent-test
```

**Submodule rules — read this or you will cause pain:**

- Each submodule is a real git repo with its own history and default branch.
- `context-retrieval-backend` default branch is **`main`** on GitHub, but local work happens on **`master`**. Push with `git push origin master:main`.
- `context-retrieval-agent-test` default branch is `main` everywhere.
- The root `CR` repo records a **commit SHA pointer** per submodule. If you edit code in a submodule and commit there, the root repo sees a "modified: submodule" change. You must also commit the pointer bump in the root.

**Workflow for any code change:**

```bash
# 1. Edit inside the submodule
cd context-retrieval-backend
# ... edit files ...
git add <files> && git commit -m "..."
git push origin master:main        # (or just `git push` for the agent-test repo)

# 2. Bump the pointer in root
cd ..
git add context-retrieval-backend
git commit -m "Bump context-retrieval-backend submodule"
git push
```

Never edit files under `context-retrieval-*` from the root repo and commit only the pointer — the actual change won't exist upstream. Always commit *inside* the submodule first.

## 3. Architecture — 4 Docker services

```
postgres (5433, pgvector)
  └── context-backend (8000)     Python/FastAPI, FastMCP, pipeline runners
        └── agent-backend (4111) Node/Express/Mastra, MCP client, SSE chat
              └── frontend (3000) React/Vite
```

All services load env from `docker.env`. Inter-service URLs use Docker DNS names (`postgres`, `context-backend`, `agent-backend`). The frontend is a Vite preview build; `frontend/vite.config.ts` has `preview.allowedHosts: true` so it accepts arbitrary hosts (needed on Railway).

## 4. context-retrieval-backend (Python/FastAPI)

**Entry point:** `main.py` — builds the FastAPI app, mounts routers, starts background runners, mounts the MCP app at `/mcp`.

**Key paths:**

- `config/settings.py` — all env var defaults and Pydantic validation. Source of truth for what's configurable.
- `infrastructure/database.py` — SQLAlchemy async engine, `SessionLocal`.
- `infrastructure/models/repo_knowledge.py` — run-record models, subject/chunk/summary/embedding/group tables.
- `infrastructure/repositories/` — typed repo classes used by services.
- `routers/repo_knowledge_pipeline_router.py` — `POST /repo-knowledge/pipeline-runs`, `GET /repo-knowledge/pipeline-runs/{id}`.
- `routers/repo_knowledge_file_summary_router.py` — file-summary run status, **file-level diagnostics** (these surface in the UI modal).
- `services/repo_knowledge/` — the engine. Subfolders:
  - `ingestion_service.py` + `chunking_service.py` — read files on disk, chunk them, write `repo_file_chunk` rows. **Sequential per file today.**
  - `file_summary_service.py` + `file_summarizer.py` — per-file LLM summaries. ReAct agent, parallelism bounded by `SEMAPHORE_LIMIT`.
  - `embedding_service.py` + `embeddings/` — pgvector embeddings per subject.
  - `repo_full_summary_service.py` — README-style markdown + Mermaid diagram.
  - `repo_manager_service.py` + `grouping/` + `group_summarization/` — architectural module grouping.
  - `pipeline_background_runner.py` + `pipeline_service.py` — orchestrates the five stages.
  - `mcp_app.py` — FastMCP server (`@mcp.tool` decorators).
  - `mcp_tools_service.py` — implementations behind the tools.
  - `summarization/model_factory.py` — builds LangChain chat models per provider.
  - `sources/` — `local_path_source.py`, `git_clone_source.py`, etc. (how the pipeline acquires a repo).

**Pipeline stages** (each has its own run table, runner, diagnostics, REST router):

1. Ingestion — `repo_run` rows, writes chunks.
2. File summarization — `repo_file_summary_run`, per-file `repo_file_summary_diagnostic` rows.
3. Embedding — `repo_embedding_run`.
4. Repo full summary — `repo_full_summary_run`.
5. Repo manager — `repo_group_summary_run`, `repo_summary_group` + members.

Each stage references the previous via `source_run_id`. If stage N fails, N+1 does not run.

**Model provider config:**

- `REPO_SUMMARY_MODEL_PROVIDER` — `gemini` (default) | `anthropic` | `openai`
- `REPO_EMBED_MODEL_PROVIDER` — same options
- Keys: `GEMINI_API_KEY`, `ANTHROPIC_API_KEY`, `PERSONAL_OPENAI_KEY`
- `GOOGLE_GENERATIVE_AI_API_KEY` must mirror `GEMINI_API_KEY` (the Google SDK reads that name).

**Scope / multi-tenancy:**

Every DB row is scoped by `tenant_id` + `user_id`. In dev, `USE_PLACEHOLDER_SCOPE=1` auto-fills both with defaults. In production, MCP tool requests must pass `tenant_id` and `user_id` as args. The agent-backend passes these via `scope-store.ts` (AsyncLocalStorage).

## 5. context-retrieval-agent-test (Node/TypeScript)

**Structure:**

```
backend/src/
  agent/agent.ts          Mastra Agent setup, model cache, MCP tool wrapper
  agent/scope-store.ts    AsyncLocalStorage for run_id + tenant/user
  routes/chat.ts          POST /api/chat (SSE), thread management
  routes/threads.ts       Chat thread CRUD
  index.ts                Express app; proxies /api/context/* → context-backend
  config.ts               env loader
frontend/src/
  App.tsx                 main chat UI, Mermaid rendering, model selector
  components/             GitHubPanel, RepoBrowser, RunDiagnosticsModal, ContextBar
  hooks/useGitHub.ts      GitHub OAuth + indexing state
  types.ts                shared types
```

**Agent model selection.** Frontend sends `model` field in POST body. `getAgentForModel(model)` in `agent.ts` returns a cached `Mastra Agent` built with `anthropic/<modelName>`. Available: `claude-haiku-4-5`, `claude-sonnet-4-6` (default), `claude-opus-4-7`. Anthropic Claude 4 rejects `temperature`, so `prepareStep` strips it.

**MCP tool wiring.** On startup, `initContextAgent` connects the Mastra `MCPClient` to `config.mcpServerUrl` and calls `listTools()`. If the backend isn't up yet, it retries with backoff; `ensureMcpTools()` reconnects lazily at request time.

**Scope injection.** `wrapToolsWithScope` wraps every MCP tool's `execute` to:

1. Pull `run_id` from `scope-store` (set by the chat route from frontend input).
2. Merge it into tool args.
3. Call the MCP tool.
4. **Unwrap** the MCP `{content: [{type: 'text', text: '<json>'}]}` envelope into a plain object (via `unwrapMcpResult`). This avoids Mastra double-JSON-encoding the payload back to the LLM.

Keep that unwrap step. Without it tool responses arrive escaped and wrapped — roughly 2× token cost and ugly to read.

**Agent instructions rule.** The system prompt in `buildAgent()` forbids answering from training data. Every factual claim must cite a tool result. When tightening agent behavior, edit the instructions string there — do not rely on client-side prompting.

**Proxy.** Express proxies `/api/context/*` → `CONTEXT_API_BASE_URL`. The frontend uses this proxy so the browser only talks to one origin (avoids CORS). Keep `CONTEXT_API_BASE_URL` formatted exactly as `http://host:port/` with trailing slash — the proxy parses it via `URL()` and throws on malformed values.

## 6. Frontend notes

- Vite preview build; `vite.config.ts` has `preview: { allowedHosts: true }`. That `true` is required (not `'all'`) for Vite 5.4+.
- Runtime env comes from `VITE_AGENT_API_BASE_URL` and `VITE_CONTEXT_API_BASE_URL` baked at build time. Change either → `docker compose build frontend`.
- `RunDiagnosticsModal.tsx` renders pipeline stage details. File-level diagnostics default to severity `warning` (yellow), not `error` (red). Don't regress this — most "errors" are benign per-file blips.
- Mermaid is rendered client-side; if the agent returns malformed Mermaid, the block falls through as code.

## 7. Environment reference

Minimum working `docker.env`:

```env
ANTHROPIC_API_KEY=sk-ant-...
GEMINI_API_KEY=AIza...
GOOGLE_GENERATIVE_AI_API_KEY=AIza...
REPO_INGEST_ALLOWED_ROOTS=/repos
APP_ENV=development
REPO_SUMMARY_MODEL_PROVIDER=gemini
REPO_EMBED_MODEL_PROVIDER=gemini
DATABASE_URL=postgresql+asyncpg://context:contextpass@postgres:5432/context
CONTEXT_API_BASE_URL=http://context-backend:8000/
MCP_SERVER_URL=http://context-backend:8000/mcp
FRONTEND_ORIGIN=http://localhost:3000
VITE_AGENT_API_BASE_URL=http://localhost:4111
VITE_CONTEXT_API_BASE_URL=http://localhost:8000/
PORT=4111
```

`REPO_INGEST_ALLOWED_ROOTS` must include the container prefix of anything you want to index. Mount local paths under that prefix in `docker-compose.yml`.

## 8. Common tasks — cheat sheet

**Rebuild one service:**

```bash
docker compose build context-backend && docker compose up -d context-backend
docker compose build agent-backend frontend && docker compose up -d agent-backend frontend
```

**Logs:**

```bash
docker compose logs -f context-backend
docker compose logs -f agent-backend
```

**Trigger ingestion via API:**

```bash
curl -X POST http://localhost:8000/repo-knowledge/pipeline-runs \
  -H "Content-Type: application/json" \
  -d '{"source_type":"local_path","repo_path":"/repos/my-project"}'
```

**Check a run:**

```bash
curl http://localhost:8000/repo-knowledge/pipeline-runs/<run_id> | jq
```

**Wipe everything and start clean:**

```bash
docker compose down -v && docker compose up --build
```

## 9. Debugging — things that bite

| Symptom | Root cause | Fix |
|---|---|---|
| `unable to get image 'cr-agent-backend'` | Docker Desktop not running | Start it |
| Browser console: `ECONNREFUSED localhost:4111` on Railway | `VITE_AGENT_API_BASE_URL` wasn't set at build time | Set it, rebuild frontend |
| Browser console: URL with `.railway.internal` | Private domain baked into frontend | `VITE_*` must be public HTTPS |
| CORS preflight 400 | `FRONTEND_ORIGIN` has trailing slash | Strip it — must exactly match browser origin |
| 502 Bad Gateway (Railway) | Target port ≠ bind port | Align them or use `$PORT` + `sh -c` |
| `$PORT is not a valid integer` | Start command runs in exec mode | Wrap in `sh -c "..."` |
| Proxy 500 `Cannot read properties of null (reading 'split')` | Malformed `CONTEXT_API_BASE_URL` | Must be `http://host:port/` with trailing slash |
| 400 "Missing tenant/user" | `USE_PLACEHOLDER_SCOPE=0` with `APP_ENV=production` | Set `USE_PLACEHOLDER_SCOPE=1` or pass scope in args |
| "Blocked request. Host not allowed" (Vite) | `allowedHosts` not set | Set `preview.allowedHosts: true` in vite.config.ts |
| MCP tools return empty / wrong repo | Stale `run_id` in AsyncLocalStorage | Re-select repo in sidebar |
| Summaries empty or sparse | Stage 2 rate-limited or failed | Check `/repo-knowledge/file-summary-runs/<id>/diagnostics`, lower `SEMAPHORE_LIMIT` |
| Ingestion extremely slow | Sequential file loop | See "Extending" below |
| Agent hallucinates | Instructions drift under long contexts | Tighten agent prompt in `agent.ts` |
| Chat thread state weird | `mastra_data` volume corrupted | `docker volume rm cr_mastra_data` and restart |

**Diagnostic endpoints:**

- `GET /repo-knowledge/pipeline-runs/<id>` — pipeline-level state
- `GET /repo-knowledge/file-summary-runs/<id>` — stage 2 stats
- `GET /repo-knowledge/file-summary-runs/<id>/diagnostics?limit=50` — per-file issues
- `GET /repo-knowledge/file-summary-runs/<id>/usage/summary` — token/time stats
- `GET /repo-knowledge/repo-full-summary-runs/<id>`
- `GET /repo-knowledge/embedding-runs/<id>`
- `GET /repo-knowledge/repo-manager-runs/<id>`

## 10. Edge cases

- **Binary/large files** — skipped by ingestion with `ingest_status=skipped`, `skip_reason` set.
- **Non-UTF-8 encoding** — skipped with `skip_reason=encoding`.
- **Windows path separators** — normalized to `/` at ingestion time.
- **Empty files** — ingest as zero-line subjects; `get_file` returns `content=""`.
- **File above line limit** — `get_file` clamps the range and returns pagination metadata.
- **Re-indexing** — creates a new `run_id`, old runs stay until manually cleaned.
- **Multiple repos at once** — each has its own `run_id`; sidebar switches scope.
- **Agent keeps calling the same tool with the same args** — usually means the tool returned less than expected. Expand the tool response or guide the LLM with a more targeted tool.
- **Claude 4 `temperature` errors** — already handled by `prepareStep` in `agent.ts`; if you add another Anthropic Claude 4+ model, no change needed.

## 11. Extending the system

### Add a new MCP tool

1. `services/repo_knowledge/mcp_tools_service.py` — implement as a method on `RepoKnowledgeMcpService`. Return a plain dict, keep fields minimal, move heavy detail to a separate `get_X` tool.
2. `services/repo_knowledge/mcp_app.py` — register with `@mcp.tool(description=...)`. The description is seen by the LLM; be prescriptive about when to call it and what args mean.
3. Restart `context-backend`.
4. The agent backend auto-discovers via `MCPClient.listTools()` on startup (or `ensureMcpTools()` on next chat request).
5. Mention the tool in the agent instructions in `backend/src/agent/agent.ts` so the LLM actually uses it.

**Tool response guidelines — token cost compounds:**

- Every field returned is sent on every call. Drop fields that the LLM won't use.
- For list tools, return a compact row; put full detail on a sibling `get_X` tool.
- Prefer indented text over nested JSON for human-shaped data (trees, listings).
- The `wrapToolsWithScope` wrapper already unwraps the MCP `content[].text` envelope — return plain dicts from the Python side.

### Add a model

Append to `AVAILABLE_MODELS` in `backend/src/agent/agent.ts`. Mastra's model string format is `<provider>/<modelId>`, e.g. `anthropic/claude-sonnet-4-6`. Non-Anthropic providers need credentials + a check in `sanitizeModelSettingsForProvider`.

### Swap summarization/embedding provider

Flip `REPO_SUMMARY_MODEL_PROVIDER` or `REPO_EMBED_MODEL_PROVIDER` in `docker.env`. Supported: `gemini`, `anthropic`, `openai`. Implementation: `services/repo_knowledge/summarization/model_factory.py` and `services/repo_knowledge/embeddings/`.

### Add a pipeline stage

1. Service + background runner under `services/repo_knowledge/`.
2. Run-record model in `infrastructure/models/repo_knowledge.py` (`<stage>_run` + any artifact tables).
3. Wire it into `services/repo_knowledge/pipeline_background_runner.py` after its prerequisite stage.
4. Router in `routers/` for status/diagnostics endpoints.
5. Expose outputs as MCP tool(s).
6. Update `frontend/src/components/RunDiagnosticsModal.tsx` to display the new stage.

### Parallelize ingestion (pending, not done)

`services/repo_knowledge/ingestion_service.py` processes files sequentially. For 1000+ file repos this dominates runtime. Plan:

- New setting `REPO_INGEST_MAX_CONCURRENT_FILES` in `config/settings.py` (default e.g. 8).
- Replace the per-file `for` loop with `asyncio.gather` bounded by `asyncio.Semaphore(n)`.
- Use a short-lived session per file (commit per file) so parallel tasks don't fight over one transaction.
- Preserve deterministic ordering for the snapshot list (sort after).

### External MCP hosts

The context backend's MCP server is reachable at `http://localhost:8000/mcp` (streamable-http transport). Any MCP host (Claude Desktop, custom agent) can connect directly — no agent-backend required. The host must pass `tenant_id` and `user_id` as tool args unless `USE_PLACEHOLDER_SCOPE=1`.

## 12. Things that look like bugs but aren't

- **"File diagnostics" in the modal are yellow, not red.** Intentional. Most per-file issues (a Gemini timeout on one file out of 800) are benign. Only pipeline-level failures are red.
- **Agent sometimes calls `file_tree` before answering even when it seems unnecessary.** Intentional — the agent is instructed to ground everything in tools. Don't remove this without thinking about how the agent establishes orientation.
- **`list_modules` returns a slimmed-down row (no `representative_subject_ids`, `updated_at`, etc.).** Intentional — full detail is on `get_module`. See the MCP tool response guidelines above.
- **`file_tree` returns indented text in a string, not nested JSON.** Intentional — cuts token cost on deep repos. Downstream consumers should treat `tree` as human-readable text.

## 13. Historical pitfalls (read before repeating)

- **Do not `git push` from the root expecting submodule changes to ship.** Changes live in the submodule repo. Push there first, then bump the pointer in root.
- **Do not use `'all'` for Vite `allowedHosts`.** Vite 5.4+ wants the literal boolean `true` or a string array.
- **Do not use private `*.railway.internal` domains in `VITE_*` vars.** Browsers cannot resolve them. Use public HTTPS domains for anything baked into the frontend.
- **Do not add a trailing slash to `FRONTEND_ORIGIN`.** CORS origin comparison is exact.
- **Do not skip `sh -c` on Railway start commands.** Railway exec mode does not expand env vars or handle `&&` / `cd`.
- **Do not amend commits that already pushed.** Create new commits. The project contributors compare SHAs.
- **Do not return MCP tool results that the agent-backend wrapper won't unwrap.** Stick to `{content: [{type: 'text', text: '<json>'}]}` (FastMCP default) and keep `unwrapMcpResult` in place.

---

If you are asked to make a change and anything above is unclear, re-read the relevant file path listed here before guessing. The code is the source of truth — this file is the map.

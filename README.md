# Context Retrieval Agent

A self-hosted codebase assistant. Point it at a Git repository, let it index the code, then chat with an AI agent that answers questions grounded in your actual source files — not hallucinations.

## How it works

1. You connect a GitHub repo (or mount a local path) and trigger indexing
2. The backend runs a pipeline: file chunking → LLM summarization → embeddings → architecture analysis
3. The agent gets MCP tools that query this indexed knowledge (file tree, summaries, grep, file read)
4. You chat with the agent and it reasons over your actual code

## Stack

- **Backend** — Python/FastAPI, PostgreSQL + pgvector, FastMCP
- **Agent** — Node.js, Mastra, Claude (Haiku / Sonnet / Opus)
- **Frontend** — React/Vite

## Prerequisites

- [Docker Desktop](https://docs.docker.com/get-docker/)
- An Anthropic API key (for the agent)
- A Gemini API key (for backend summarization/embeddings — default provider)

## Setup

### 1. Clone with submodules

```bash
git clone --recurse-submodules https://github.com/Jefferson-liu/CR.git
cd CR
```

If you already cloned without `--recurse-submodules`:
```bash
git submodule update --init --recursive
```

### 2. Configure environment

```bash
cp .env.example docker.env
```

Edit `docker.env` and fill in your API keys:

```env
ANTHROPIC_API_KEY=sk-ant-...
GEMINI_API_KEY=AIza...
GOOGLE_GENERATIVE_AI_API_KEY=AIza...   # same value as GEMINI_API_KEY
```

### 3. Mount a repository for indexing

Open `docker-compose.yml` and find the `context-backend` volumes section. Replace the placeholder with a path to a local repo:

```yaml
context-backend:
  volumes:
    - /path/to/your/project:/repos/my-project:ro
```

Then set the allowed root in `docker.env`:

```env
REPO_INGEST_ALLOWED_ROOTS=/repos
```

### 4. Start the stack

```bash
docker compose up --build
```

Services will be available at:
- Frontend: http://localhost:3000
- Context API: http://localhost:8000
- Agent API: http://localhost:4111

### 5. Index your repository

Once the stack is running, open the UI at http://localhost:3000, go to the **GitHub** tab in the sidebar, and trigger indexing for your repo. Indexing runs in the background — you can watch progress in the logs:

```bash
docker compose logs -f context-backend
```

## Usage

1. Open http://localhost:3000
2. Select an indexed repo from the sidebar (GitHub tab)
3. Start chatting — the agent will use your codebase as context

The model selector in the chat input lets you switch between Claude Haiku (fast), Sonnet (default), and Opus (most capable).

## Updating

Pull the latest code and rebuild:

```bash
git pull --recurse-submodules
docker compose build
docker compose up -d
```

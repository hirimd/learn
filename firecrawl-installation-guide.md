# Firecrawl Installation Guide

This guide walks through installing, configuring, and running [Firecrawl](https://github.com/firecrawl/firecrawl) (formerly `mendableai/firecrawl`), the open source API service that turns websites into clean, LLM ready data (Markdown, structured JSON, etc.).

## Prerequisites

Install the following before you begin:

| Tool | Minimum Version | Purpose |
|---|---|---|
| Git | any recent version | Clone the repository |
| Docker | 24+ | Run the containerized stack (Method A) |
| Docker Compose | v2 (bundled with Docker Desktop) | Orchestrate multi-container services |
| Node.js | 18+ (20 LTS recommended) | Run the API/worker locally (Method B) |
| pnpm | 9+ | Package manager used by the monorepo |
| Redis | 7+ | Job queue backing store (only needed standalone for Method B; Docker Compose provides it automatically) |

Installation commands by platform:

macOS (Homebrew):
```bash
brew install git node pnpm redis
brew install --cask docker
```

Ubuntu/Debian:
```bash
sudo apt update
sudo apt install -y git curl
curl -fsSL https://get.docker.com | sudo sh
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo bash -
sudo apt install -y nodejs
npm install -g pnpm
```

Windows:
- Install [Git for Windows](https://git-scm.com/download/win)
- Install [Docker Desktop](https://www.docker.com/products/docker-desktop) (includes Docker Compose and WSL2 integration)
- Install [Node.js](https://nodejs.org/) via the official installer, then run `npm install -g pnpm` in PowerShell
- Run all commands below inside WSL2 (Ubuntu) or PowerShell; Docker Desktop handles the Linux container layer either way

Verify your setup:
```bash
git --version
docker --version
docker compose version
node --version
pnpm --version
```

## Step 1: Clone the Repository

```bash
git clone https://github.com/firecrawl/firecrawl.git
cd firecrawl
```

## Step 2: Environment Configuration

Firecrawl's configuration lives in an `.env` file consumed by the API and worker services. Copy the example file and edit it:

```bash
cp apps/api/.env.example apps/api/.env
```

Open `apps/api/.env` in your editor and review the key variables. For a self hosted, local only setup, the defaults below are typically enough to get started:

```bash
# Core service
PORT=3002
HOST=0.0.0.0

# Redis connection (matches the docker-compose service name)
REDIS_URL=redis://redis:6379
REDIS_RATE_LIMIT_URL=redis://redis:6379

# Disable hosted auth/database requirements for local/self-host use
USE_DB_AUTHENTICATION=false

# Headless browser service used for JS-rendered pages
PLAYWRIGHT_MICROSERVICE_URL=http://playwright-service:3000/scrape

# Optional: only needed if you want LLM-powered extraction features
OPENAI_API_KEY=
```

Notes:
- If you plan to run services outside Docker (Method B), change `redis://redis:6379` and `http://playwright-service:3000/scrape` to `localhost` equivalents, since there is no Docker network to resolve those service names.
- `OPENAI_API_KEY` (or another supported LLM provider key) is only required if you use extraction/summarization features that call an LLM; basic scraping and crawling work without it.
- Treat `.env` as secret; never commit it. It should already be covered by `.gitignore`.

## Step 3: Run Firecrawl (choose one method)

### Method A: Using Docker (Recommended)

This is the simplest path since it starts the API, worker, Redis, and the Playwright rendering service together with correct networking.

Build the images:
```bash
docker compose build
```

Start the stack in the background:
```bash
docker compose up -d
```

Check that all containers are healthy:
```bash
docker compose ps
```

Tail logs if something looks off:
```bash
docker compose logs -f
```

Stop the stack when you are done:
```bash
docker compose down
```

### Method B: Local Development Setup

Use this if you want to run and debug the TypeScript code directly (hot reload, breakpoints, etc.).

1. Start Redis (skip if you already have one running):
```bash
redis-server --daemonize yes
```
On Windows without WSL, run Redis via Docker instead: `docker run -d -p 6379:6379 redis:7-alpine`.

2. Install dependencies from the repo root:
```bash
pnpm install
```

3. Update `apps/api/.env` so Redis and the Playwright service point at `localhost` instead of Docker service names:
```bash
REDIS_URL=redis://localhost:6379
REDIS_RATE_LIMIT_URL=redis://localhost:6379
PLAYWRIGHT_MICROSERVICE_URL=http://localhost:3003/scrape
```

4. In one terminal, start the Playwright rendering service:
```bash
cd apps/playwright-service-ts
pnpm install
pnpm run start
```

5. In a second terminal, start the API server:
```bash
cd apps/api
pnpm run start
```

6. In a third terminal, start the background worker (handles queued crawl/scrape jobs):
```bash
cd apps/api
pnpm run workers
```

Keep all three processes running while you develop; the API depends on the worker to actually process jobs and on the Playwright service for JavaScript-rendered pages.

## Step 4: Verify the Installation

Once the stack is up (either method), confirm the API responds:

```bash
curl http://localhost:3002/test
```

You should get back a simple confirmation response (for example, `Hello, world!`), indicating the API process is reachable on port 3002.

Next, try an actual scrape request:

```bash
curl -s -X POST http://localhost:3002/v1/scrape \
  -H "Content-Type: application/json" \
  -d '{"url": "https://firecrawl.dev"}'
```

A successful response returns JSON containing `success: true` and a `data` object with the scraped Markdown/HTML content. If you disabled DB authentication (`USE_DB_AUTHENTICATION=false`), you do not need an `Authorization` header for local testing; otherwise include `-H "Authorization: Bearer <your_api_key>"`.

For a crawl (multi-page) job:
```bash
curl -s -X POST http://localhost:3002/v1/crawl \
  -H "Content-Type: application/json" \
  -d '{"url": "https://firecrawl.dev", "limit": 5}'
```
This returns a job ID; poll `GET http://localhost:3002/v1/crawl/<job_id>` to check status and retrieve results as pages finish.

## Troubleshooting (Common Issues)

**Port already in use (3002, 6379, or 3003)**
Find and stop the conflicting process, or change the port mapping:
```bash
lsof -i :3002
```
Then either kill the conflicting process or edit the `ports` section in `docker-compose.yaml` (e.g., map `3002:3002` to `3010:3002`) and update `PORT`/your test URLs to match.

**API container starts but scrape requests time out or fail**
Usually means the worker isn't running or can't reach Redis. Check:
```bash
docker compose logs api
docker compose logs worker
docker compose logs redis
```
Confirm `REDIS_URL` matches the Redis container/service name (Docker) or `localhost` (local dev).

**JavaScript-heavy pages return empty content**
The Playwright rendering service may not be reachable. Confirm `PLAYWRIGHT_MICROSERVICE_URL` is correct and the service container/process is running (`docker compose ps` or check the local terminal running `apps/playwright-service-ts`).

**"Missing API key" or auth errors**
If `USE_DB_AUTHENTICATION=true`, the API expects a valid key and a configured Supabase/database backend. For local self-hosting without a hosted auth layer, set `USE_DB_AUTHENTICATION=false` in `.env` and omit the `Authorization` header.

**LLM-based extraction features fail silently**
These require a valid provider key such as `OPENAI_API_KEY`. Set it in `apps/api/.env` and restart the API/worker so the new environment variable is picked up.

**Changes to `.env` don't seem to apply**
Environment variables are read at process start. After editing `.env`, restart the affected service:
```bash
docker compose restart api worker
```
or, for local dev, stop and re-run the relevant `pnpm run` command.

**Docker build is slow or fails on first run**
The initial build compiles the API, worker, and Playwright service images and can take several minutes, especially on the first run since base images and browser binaries need to download. Ensure you have a stable network connection and enough free disk space (several GB), then retry:
```bash
docker compose build --no-cache
```

# Firecrawl Installation Guide

This guide walks through installing, configuring, and running [Firecrawl](https://github.com/firecrawl/firecrawl), the open source API service that turns websites into clean, LLM ready data (Markdown, structured JSON, etc.).

Everything below was checked directly against a fresh clone of the repository (Docker Compose file, `SELF_HOST.md`, `CONTRIBUTING.md`, and `apps/api/package.json`) rather than written from memory, since the project's self hosting setup has changed significantly from earlier versions: it now runs RabbitMQ and a Postgres backed queue ("NuQ") alongside the API, worker, and Playwright rendering service.

## Prerequisites

| Tool | Version | Purpose |
|---|---|---|
| Git | any recent version | Clone the repository |
| Docker | 24+ | Run the containerized stack (Method A), and build local dependency containers for Method B |
| Docker Compose | v2 (bundled with Docker Desktop) | Orchestrate the multi-container stack |
| Node.js | 22 | Run the API/worker locally (Method B) |
| pnpm | exactly `11.4.0`, pinned in `apps/api/package.json` | Package manager; use Corepack rather than a globally installed pnpm to avoid version mismatches |
| Redis | 7+ | Job queue/rate limiting store; Docker Compose provides it automatically, Method B expects you to run it yourself |
| FoundationDB client library | matches the version pinned in `docker-compose.yaml` (`7.3.63` at the time of writing) | Only needed for Method B: `apps/api` depends directly on the `foundationdb` npm package, which compiles a native addon against local FoundationDB client headers |

Installation commands by platform:

macOS (Homebrew):
```bash
brew install git node redis
brew install --cask docker
corepack enable
```

Ubuntu/Debian:
```bash
sudo apt update
sudo apt install -y git curl redis-server
curl -fsSL https://get.docker.com | sudo sh
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo bash -
sudo apt install -y nodejs
corepack enable
```

Windows:
- Install [Git for Windows](https://git-scm.com/download/win)
- Install [Docker Desktop](https://www.docker.com/products/docker-desktop) (includes Docker Compose and WSL2 integration)
- Install [Node.js 22](https://nodejs.org/), then run `corepack enable` in PowerShell
- Run all commands below inside WSL2 (Ubuntu) or PowerShell; Docker Desktop handles the Linux container layer either way

Verify your setup:
```bash
git --version
docker --version
docker compose version
node --version
corepack --version
```

**Only needed for Method B** (local, non-Docker development): install the FoundationDB client library so `pnpm install` in `apps/api` can compile its native dependency. On Debian/Ubuntu:
```bash
curl -fsSLO https://github.com/apple/foundationdb/releases/download/7.3.63/foundationdb-clients_7.3.63-1_amd64.deb
sudo dpkg -i foundationdb-clients_7.3.63-1_amd64.deb
```
On macOS, use the matching `.pkg` from the same [FoundationDB releases page](https://github.com/apple/foundationdb/releases). Without this, `pnpm install` fails with `fatal error: foundationdb/fdb_c.h: No such file or directory`.

## Step 1: Clone the Repository

```bash
git clone https://github.com/firecrawl/firecrawl.git
cd firecrawl
```

For a self hosted deployment, `SELF_HOST.md` recommends checking out an exact release tag rather than `main`, since the API code and the Compose file's floating image tags can drift independently:
```bash
git fetch --tags
git checkout <release-tag>
```

## Step 2: Environment Configuration

Firecrawl uses **two separate environment files** for two separate workflows; do not copy one into the other:

- **Docker Compose (Method A)** reads a `.env` file at the **repository root**, which only overrides the small set of variables `docker-compose.yaml` actually references. Every one of those variables already has a sane default (`docker-compose.yaml` uses `${VAR:-default}` throughout), so a root `.env` is optional for a first run; create one only when you need to set something like an LLM API key:
```bash
cat > .env <<'EOF'
# Optional: only needed if you use LLM-powered extraction features
OPENAI_API_KEY=

# Optional: change the host port the API is published on (default 3002)
# PORT=3002
EOF
```
- **Local development (Method B)** reads `apps/api/.env`, copied from the example file:
```bash
cp apps/api/.env.example apps/api/.env
```
Key variables to check in `apps/api/.env`:
```bash
# For local dev, point these at localhost instead of the Docker service names
REDIS_URL=redis://localhost:6379
PLAYWRIGHT_MICROSERVICE_URL=http://localhost:3000/scrape

# Optional: only needed for LLM-powered extraction features
OPENAI_API_KEY=
```

`USE_DB_AUTHENTICATION` defaults to disabled either way, so a self-hosted instance does not require an API key for local testing until you deliberately configure a database backed auth layer.

## Step 3: Run Firecrawl (choose one method)

### Method A: Using Docker (Recommended)

`docker compose up` starts everything defined in `docker-compose.yaml`: the API (which internally launches its own workers), the Playwright rendering service, Redis, RabbitMQ, and a Postgres-backed queue ("nuq-postgres"). Only the API is published to the host, on port 3002 by default.

Build and start the stack:
```bash
docker compose up -d --build
```

Check container status:
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

Per `SELF_HOST.md`, keep in mind for anything beyond local testing: the default API has no authentication, no TLS, and the Compose file defines no persistent volumes for Postgres/Redis/RabbitMQ, so data does not survive a `docker compose down`.

### Method B: Local Development Setup

Use this to run and debug the TypeScript code directly. This does **not** use `docker-compose.yaml`; instead, Firecrawl's own "harness" script manages the Postgres and RabbitMQ containers for you (it shells out to `docker`/`podman`), while you run Redis and the Playwright service yourself.

1. Start Redis (skip if one is already running):
```bash
redis-server --daemonize yes
```

2. In one terminal, build and start the Playwright rendering service:
```bash
cd apps/playwright-service-ts
corepack pnpm install
corepack pnpm run build
corepack pnpm run start
```

3. In a second terminal, install and start the API (make sure `apps/api/.env` is configured as in Step 2, and the FoundationDB client library from the Prerequisites section is installed first):
```bash
cd apps/api
corepack pnpm install
corepack pnpm start
```
`pnpm start` builds the project, then runs the harness, which will build/launch its own `nuq-postgres` and `rabbitmq` containers automatically and start the API together with its workers. Watch the terminal output; the harness logs each service it brings up.

If you'd rather run components individually instead of through the harness (useful for debugging one piece at a time), `apps/api/package.json` exposes granular scripts such as `pnpm run workers`, `pnpm run extract-worker`, and `pnpm run index-worker`, but the harness-driven `pnpm start` is the supported first-run path.

## Step 4: Verify the Installation

Once the stack is up (either method), confirm the API responds:

```bash
curl http://localhost:3002/
```

A healthy instance returns JSON like:
```json
{"message":"Firecrawl API","documentation_url":"https://docs.firecrawl.dev"}
```

Next, try an actual scrape request:
```bash
curl -s -X POST http://localhost:3002/v1/scrape \
  -H "Content-Type: application/json" \
  -d '{"url": "https://firecrawl.dev"}'
```
A successful response returns JSON with `success: true` and a `data` object containing the scraped Markdown/HTML. With the default `USE_DB_AUTHENTICATION=false`, no `Authorization` header is required for local testing.

For a crawl (multi-page) job:
```bash
curl -s -X POST http://localhost:3002/v1/crawl \
  -H "Content-Type: application/json" \
  -d '{"url": "https://firecrawl.dev", "limit": 5}'
```
This returns a job ID; poll `GET http://localhost:3002/v1/crawl/<job_id>` to check status and retrieve results as pages finish.

## Troubleshooting (Common Issues)

**Port already in use (3002, 6379, 5672, or 5432)**
```bash
lsof -i :3002
```
Kill the conflicting process, or (Method A) set `PORT=<other-port>` in the root `.env` to change the host-side mapping, since `docker-compose.yaml` maps `${PORT:-3002}:${INTERNAL_PORT:-3002}`.

**`pnpm install` fails with `fatal error: foundationdb/fdb_c.h: No such file or directory`**
This is Method B only. `apps/api` has a plain (non-optional) dependency on the `foundationdb` npm package, which compiles a native addon against the FoundationDB C client headers. Install the client library from the Prerequisites section before running `pnpm install` again.

**`pnpm install` or `pnpm start` behaves unexpectedly / wrong pnpm version**
`apps/api/package.json` pins `"packageManager": "pnpm@11.4.0"`. Always invoke it via `corepack pnpm ...` (after `corepack enable`) rather than a separately installed global pnpm, so the exact pinned version is used.

**API container starts but scrape requests time out or fail**
Usually means a dependency service isn't reachable. Check:
```bash
docker compose logs api
docker compose logs redis
docker compose logs rabbitmq
docker compose logs nuq-postgres
```
For Method B, confirm Redis is actually running on `localhost:6379` and that the harness's log output shows the Postgres/RabbitMQ containers started successfully.

**JavaScript-heavy pages return empty content**
Confirm `PLAYWRIGHT_MICROSERVICE_URL` matches how you're running things (`http://playwright-service:3000/scrape` for Compose, `http://localhost:3000/scrape` for local dev) and that the Playwright service process/container is actually up.

**Docker image pulls fail with 403/Forbidden from a CDN like `production.cloudfront.docker.com`**
This means the network your Docker daemon runs on is blocking Docker Hub's blob storage CDN (common behind restrictive corporate proxies or locked-down sandboxes) — it is not a Firecrawl issue. Fix it at the network/proxy level (allow that host, or configure Docker to pull through an authorized mirror/registry) rather than retrying repeatedly.

**LLM-based extraction features fail silently**
These require a valid provider key such as `OPENAI_API_KEY`, set in the root `.env` (Method A) or `apps/api/.env` (Method B), followed by a restart of the relevant service.

**Changes to `.env` don't seem to apply**
Environment variables are read at process start.
```bash
docker compose restart api
```
or, for local dev, stop and re-run `corepack pnpm start`.

**Docker build is slow or fails on first run**
The initial build compiles the API and Playwright service images, which can take several minutes on first run. Ensure a stable connection and enough free disk space (several GB), then retry:
```bash
docker compose build --no-cache
```

# Hangar

A self-hosted deployment pipeline for containerised apps — built with TypeScript, Railpack, Caddy, and Docker.

Push a Git URL. Hangar clones it, builds it into a container image with Railpack, runs it, and fronts it with a live subdomain via Caddy. Build and deploy logs stream to the UI in real time over SSE.

---

## Demo

> 🎥 [Loom Walkthrough](#) — *(link coming)*

---

## Stack

| Layer | Technology |
|---|---|
| Frontend | Vite + React Router + TanStack Query |
| Backend | Hono (TypeScript) |
| Build | Railpack + BuildKit |
| Runtime | Docker |
| Ingress | Caddy |
| Database | SQLite (via better-sqlite3) |
| Logs | SSE (real-time streaming + persistence) |

---

## Architecture

```
User submits Git URL
        ↓
API creates deployment record → fires pipeline async
        ↓
clone.ts      — git clone repo to /tmp
build.ts      — Railpack prepares build plan → BuildKit builds image → loaded into Docker daemon
run.ts        — docker run with dynamic port binding
caddy.ts      — patches Caddy admin API to add reverse proxy route
        ↓
App live at http://{deploymentId}.localhost
```

Logs are written to SQLite at each pipeline stage and streamed to the frontend over SSE as they happen — not after the fact.

---

## Getting Started

### Prerequisites

- Docker + Docker Compose
- Git

That's it. Everything else runs inside Docker.

### Run

```bash
git clone https://github.com/AdedigbaOluwad1/hangar-mirror.git
cd hangar-mirror
docker compose up
```

Open [http://localhost](http://localhost).

### Environment Variables

All variables have sensible defaults and work out of the box with `docker compose up`. No external accounts required.

| Variable | Default | Description |
|---|---|---|
| `DATABASE_PATH` | `/data/hangar.db` | SQLite database path |
| `CADDY_ADMIN_URL` | `http://caddy:2019` | Caddy admin API |
| `BUILDKIT_HOST` | `tcp://buildkit:1234` | BuildKit daemon address |
| `DOCKER_HOST` | `unix:///var/run/docker.sock` | Docker socket |
| `NODE_ENV` | `development` | Node environment |

---

## How It Works

### Build

Hangar uses [Railpack](https://railpack.io) to analyse the repo and generate a build plan — no Dockerfiles needed. BuildKit executes the plan and produces a Docker image. The image is loaded directly into the Docker daemon via the Docker socket.

Railpack auto-detects the runtime (Node, Python, Go, etc.), package manager, install commands, and start command. It handles everything from dependency installation to the final image layer structure.

### Runtime

The built image is run as a Docker container with a dynamically assigned host port. Hangar tracks the port and container ID against the deployment record.

### Routing

Caddy is the single point of ingress. When a container starts, Hangar patches the Caddy admin API to add a new route matching `{deploymentId}.localhost` and reverse proxying to the container's port on the host.

Each deployment gets its own subdomain. Routes are added dynamically at runtime — no Caddy restarts required.

### Log Streaming

Every pipeline stage (`clone`, `build`, `deploy`, `system`) writes logs to SQLite and emits them over SSE simultaneously. The frontend subscribes to `/api/deployments/{id}/logs` as an `EventSource` and renders lines as they arrive — including mid-build output from Railpack and BuildKit.

Logs persist in SQLite so users can scroll back after the build completes.

---

## API

| Method | Path | Description |
|---|---|---|
| `GET` | `/deployments` | List all deployments |
| `POST` | `/deployments` | Create a deployment |
| `GET` | `/deployments/:id` | Get a deployment |
| `GET` | `/deployments/:id/logs` | Stream logs over SSE |

### Create a Deployment

```bash
curl -X POST http://localhost/api/deployments \
  -H "Content-Type: application/json" \
  -d '{
    "sourceType": "git",
    "sourceUrl": "https://github.com/render-examples/express-hello-world"
  }'
```

---

## Sample App

The [Render Express Hello World](https://github.com/render-examples/express-hello-world) works out of the box as a test deployment.

For a more complete test, deploy the included Next.js sample app: [PDF Editor Lite](https://github.com/AdedigbaOluwad1/pdf-editor-lite)

---

## Project Structure

```
apps/
  api/                  — Hono API server
    src/
      lib/
        emitter.ts      — SSE log emitter
        port.ts         — generates ports for containers
      pipeline/
        index.ts        — pipeline orchestrator
        clone.ts        — git clone
        build.ts        — Railpack + BuildKit image build
        run.ts          — docker run
        caddy.ts        — Caddy admin API patching
      routes/
        deployments.ts  — deployment CRUD
        logs.ts         — SSE
  web/                  — Vite + React Router frontend
caddy/
  Caddyfile             — Caddy config
packages/
  db/                   — better-sqlite3 queries (shared package)
docker-compose.yml
```

---

## Decisions

**Why Hono?**
Fast, lightweight, and has first-class TypeScript support. The middleware model maps cleanly to the pipeline stages. SSE support is built in.

**Why better-sqlite3?**
Zero infrastructure overhead for a take-home submission — no database service to spin up, no connection pool to configure. The synchronous API is actually a good fit here since the pipeline stages run sequentially and blocking I/O is acceptable. That said, SQLite is a file, not a database server, and it would be the first thing swapped out in production: concurrent writes from multiple deployments will cause lock contention, and it has no story for horizontal scaling. Postgres would be the right call for anything multi-tenant.

**Why BuildKit over plain `docker build`?**
BuildKit supports remote build daemons, layer caching, and parallel execution. Running it as a sidecar (`moby/buildkit`) means the API container doesn't need Docker build access — only socket access to load the finished image.

**Why subdomain routing over path routing?**
Subdomains (`{id}.localhost`) are cleaner and avoid path prefix stripping complexity in both Caddy and the app. Apps that use absolute paths or redirects work correctly without modification.

**Why Caddy?**
The admin API makes dynamic route injection trivial — no config reloads, no restarts. A single PATCH request adds a new upstream. Caddy also handles HTTPS automatically when a real domain is pointed at it.

---

## What I'd Do With More Time

**Swap SQLite for Postgres** — SQLite works fine for a single-node setup but Postgres is the right call for anything multi-tenant or highly concurrent.

**Add a job queue** — right now the pipeline fires directly from the POST handler. A queue (BullMQ + Redis) would decouple the API from the pipeline, give retry semantics for free, and make it easy to add per-user concurrency limits later.

**Add a local image registry** — right now built images live in the Docker daemon's local store. A local registry (`registry:2`) would let Nomad or any external orchestrator pull images by tag, and would make build cache reuse across deploys more reliable.

**Proper container lifecycle management** — stopping a deployment currently just kills the container. Zero-downtime redeploys (drain traffic → start new container → switch Caddy upstream → stop old container) would be the production pattern.

**Rollback** — image tags are stored per deployment. Redeploying a previous tag is a matter of re-running the `run` and `caddy` steps with the stored `imageTag`. The data model already supports it.

**Auth** — no auth at all right now. In production, deployments are scoped to users and env vars are stored encrypted per deployment. The queue would be per-user with rate limits.

**What I'd rip out** — the `host-gateway` extra_hosts hack in docker-compose. It works locally but it's fragile — the WSL IP changes on restart. In production this goes away entirely because Nomad handles container scheduling and Consul handles service discovery, so Caddy never needs to dial the host directly.

---

## What's Production Hangar?

The mirror repo is a scoped-down version of the full Hangar platform which runs the HashiCorp stack:

- **Nomad** for container orchestration (replaces raw `docker run`)
- **Consul** for service discovery (replaces manual port tracking)
- **Vault** for secrets management (replaces `.env` files)
- **Ansible** for server provisioning (one command from bare metal to running platform)
- **Terraform** for infrastructure provisioning (Hetzner + DigitalOcean)

The full stack is at [github.com/AdedigbaOluwad1/hangar](https://github.com/AdedigbaOluwad1/hangar).

---

## Time Spent

~8 hours across pipeline design, BuildKit integration, Caddy dynamic routing, SSE log streaming, and frontend wiring.

---

## Brimble Deploy + Feedback

> *(Coming — will update with deploy link and feedback)*

---

## Submission Checklist

- ✅ Runs end-to-end with `docker compose up`
- ✅ Live log streaming over SSE
- ✅ Railpack builds produce runnable images
- ✅ Caddy fronts every deployment
- ✅ Subdomain routing per deployment
- ✅ Logs persist and are scrollable after build
- ✅ README with architecture notes and decisions
- [ ] Loom walkthrough *(coming)*
- [ ] Brimble deploy + feedback *(coming)*
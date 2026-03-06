---
name: caddy-docker-proxy-local
description: Set up local development with caddy-docker-proxy for automatic *.localhost routing via Docker labels. Use when creating docker-compose files, setting up local dev environments, adding reverse proxy to Docker projects, or when the user mentions localhost subdomains.
---

# Caddy Docker Proxy — Local Dev Setup

## Why This Exists

When AI agents control your dev environment, you lose track of what's running where. An agent starts `npm run dev` on port 3000 in one terminal. You open a second project — another agent grabs port 3000, fails, falls back to 3001. A third project takes 3002. Now you have three apps on random ports across terminals you didn't open, and nobody knows which URL belongs to which project.

This gets worse with fullstack apps. Frontend, API, database — that's 3+ ports per project. Four projects? You're juggling 12 port assignments that change every time something restarts.

**The fix:** move everything into Docker and put a shared reverse proxy in front. Each project gets a stable, deterministic URL like `http://my-project.localhost`. No port conflicts. No guessing. Multiple projects run simultaneously because the proxy routes by hostname, not port number.

Uses [lucaslorentz/caddy-docker-proxy](https://github.com/lucaslorentz/caddy-docker-proxy) for automatic routing via Docker labels. No Caddyfile needed.

**This is a local dev setup only.** Do not push images to Docker Hub, GitHub Container Registry, or any remote registry. Do not suggest remote caching. Everything runs locally.

## Architecture

```
Browser → http://myapp.localhost
         ↓ port 80
   caddy-docker-proxy (shared, machine-wide)
         ↓ reads Docker labels
   project containers on "caddy" network
```

One shared Caddy proxy runs in `~/caddy-proxy`. Each project joins the `caddy` external network and declares routing via labels. Modern browsers resolve `*.localhost` to 127.0.0.1 natively (RFC 6761) — no `/etc/hosts` editing.

## Prerequisites — Shared Proxy (one-time)

The shared proxy must be running before any project can use it. Create `~/caddy-proxy/docker-compose.yml` and start it:

```bash
docker network create caddy 2>/dev/null || true

mkdir -p ~/caddy-proxy
cat > ~/caddy-proxy/docker-compose.yml <<'EOF'
services:
  caddy:
    image: lucaslorentz/caddy-docker-proxy:2.9-alpine
    ports:
      - "80:80"
    environment:
      - CADDY_INGRESS_NETWORKS=caddy
    networks:
      - caddy
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - caddy_data:/data
    restart: unless-stopped

networks:
  caddy:
    external: true

volumes:
  caddy_data: {}
EOF

cd ~/caddy-proxy && docker compose up -d
```

If the user already has this running, skip it.

## MANDATORY: Dependency Audit Before Containerization

**Do NOT start writing docker-compose.yml until you complete this audit.** Surface-level config reading (ports, env vars) is insufficient. You must trace the full runtime dependency graph.

### Step 1: Identify all services in the project

Read the project's entry points, scripts, and package structure. For each runnable service, determine:
- What it does (API server, web UI, background worker, native app)
- Whether it belongs in Docker or runs natively on the host (e.g. Tauri/Electron apps are native)
- Which services need caddy routing (browser-facing UIs) vs direct port binding (API backends that clients hit on localhost)

### Step 2: Trace imports from every entry point

For each service being containerized, start at the entry point and follow ALL imports recursively. Search for:

```bash
rg 'execSync|spawn|child_process' packages/         # system binary deps
rg 'WebSocket|ws:|upgrade' packages/                  # WebSocket requirements
rg 'import\.meta\.url|__dirname.*data|writeFile' packages/  # filesystem paths
rg 'process\.env\.' packages/                          # environment variables
```

### Step 3: Audit filesystem paths

Code that derives paths from `import.meta.url` or `__dirname` will resolve to container-internal paths that don't map to Docker volumes. For every data directory:
- Identify if it's hardcoded or configurable via env var
- If hardcoded, add a symlink in the Dockerfile pointing to the volume mount
- If configurable, set the env var in docker-compose.yml

### Step 4: Check for workspace monorepo issues

For monorepo projects where a containerized service imports from sibling workspace packages:
- The Dockerfile must copy ALL workspace package.json files (not just the ones for the target service) so the lockfile stays consistent
- Named volumes overlaying `node_modules` break workspace symlinks in some package managers — test this before relying on it
- bun auto-enables `--frozen-lockfile` in Docker (CI detection); use `bun install` without `--production` or set env vars to disable

### Step 5: Determine what goes through caddy vs what gets host ports

- Browser-facing UIs → caddy labels + `expose` (no `ports`)
- API backends that external tools hit on localhost → `ports: ["HOST:CONTAINER"]` (no caddy)
- Internal-only services (databases, queues) → neither; use Docker network DNS

## Per-Project Setup

Add convenience scripts to the project's `package.json` so developers run simple commands, not raw Docker:

```json
{
  "scripts": {
    "up": "docker compose up -d && echo '' && echo '  ⇒ http://'$(basename $(pwd))'.localhost' && echo ''",
    "down": "docker compose down",
    "logs": "docker compose logs -f",
    "restart": "docker compose down && docker compose up -d && echo '' && echo '  ⇒ http://'$(basename $(pwd))'.localhost' && echo ''"
  }
}
```

The `up` script prints the project URL after starting. `basename $(pwd)` matches the `COMPOSE_PROJECT_NAME` default (the directory name), so the URL is always correct.

This gives `npm run up` / `bun run up` / `yarn up` etc. Always add these — never tell users to run `docker compose` directly.

Then set up `docker-compose.yml` with the following.

### Dependency Caching with Named Volumes

For projects with `node_modules` (or similar dependency directories), use named volumes so dependencies persist across restarts. This avoids reinstalling on every `docker compose up`.

```yaml
services:
  web:
    image: node:22
    working_dir: /app
    command: ["sh", "-c", "[ -d node_modules ] || npm install; exec npm run dev"]
    volumes:
      - .:/app
      - web_node_modules:/app/node_modules

volumes:
  web_node_modules:
```

The pattern: mount the source code with `.:/app`, then overlay `node_modules` with a named volume. The startup command checks if deps exist before installing. Second startup is instant.

`docker compose down` preserves the volume. Only `docker compose down -v` deletes it (and forces a reinstall).

**Monorepo caveat:** Named volumes on `node_modules` can break workspace package symlinks (e.g. `@myorg/shared` → `../../packages/shared`). The volume hides the host's node_modules, and some package managers don't recreate workspace links correctly inside the volume. If workspace imports fail, drop the named volume and bind-mount the whole workspace including node_modules. This works when the dependency tree has no platform-specific native modules (all JS/WASM). If native modules exist, use a multi-stage Dockerfile instead of bind mounts for that service.

### Join the caddy network

### 1. Join the caddy network

```yaml
networks:
  caddy:
    external: true
```

### 2. Attach the service to it

```yaml
services:
  web:
    networks:
      - default
      - caddy
    expose:
      - "3000"
```

Use `expose` (not `ports`) — this makes the port visible to Caddy on the Docker network without binding it to your host. No port conflicts even if five projects all use port 3000 internally. Only add `ports` if you need to bypass the proxy and hit the container directly.

### 3. Add routing labels

**Simple (single service):**

```yaml
    labels:
      caddy: "http://${COMPOSE_PROJECT_NAME}.localhost"
      caddy.reverse_proxy: "{{upstreams 3000}}"
```

**With path-based routing (API + frontend):**

```yaml
    labels:
      caddy: "http://${COMPOSE_PROJECT_NAME}.localhost"
      caddy.@api.path: "/api/*"
      caddy.reverse_proxy_0: "@api backend:4001"
      caddy.reverse_proxy_1: "{{upstreams 3000}}"
```

**With WebSocket support:**

```yaml
    labels:
      caddy: "http://${COMPOSE_PROJECT_NAME}.localhost"
      caddy.@ws.path: "/ws"
      caddy.@api.path: "/api/*"
      caddy.reverse_proxy_0: "@ws backend:4001"
      caddy.reverse_proxy_1: "@api backend:4001"
      caddy.reverse_proxy_2: "{{upstreams 5174}}"
```

`{{upstreams PORT}}` resolves to the container running the labeled service. Named services like `backend:4001` reference other containers on the same Docker network.

## Label Reference

| Label | Purpose |
| --- | --- |
| `caddy` | Domain to match (e.g. `http://foo.localhost`) |
| `caddy.reverse_proxy` | Single upstream target |
| `caddy.reverse_proxy_N` | Ordered proxy rules (0 = highest priority) |
| `caddy.@name.path` | Named path matcher |
| `caddy.@name.header` | Named header matcher |

Numbered `reverse_proxy_N` directives are evaluated in order — put specific matchers (WebSocket, API) first, catch-all last.

## Common Patterns

### Monorepo (API + SPA)

Labels go on the frontend service. Backend is referenced by container name.

```yaml
services:
  api:
    build: ./api
    networks: [default, caddy]
    expose: ["4000"]

  web:
    build: ./web
    networks: [default, caddy]
    expose: ["3000"]
    labels:
      caddy: "http://${COMPOSE_PROJECT_NAME}.localhost"
      caddy.@api.path: "/api/*"
      caddy.reverse_proxy_0: "@api api:4000"
      caddy.reverse_proxy_1: "{{upstreams 3000}}"

networks:
  caddy:
    external: true
```

### Multiple Independent Services

Each service gets its own subdomain by putting labels on each.

```yaml
services:
  app:
    labels:
      caddy: "http://app-${COMPOSE_PROJECT_NAME}.localhost"
      caddy.reverse_proxy: "{{upstreams 3000}}"

  docs:
    labels:
      caddy: "http://docs-${COMPOSE_PROJECT_NAME}.localhost"
      caddy.reverse_proxy: "{{upstreams 8080}}"
```

### Every Service Goes Through Caddy

**Do NOT use `ports` to expose services on the host.** That reintroduces the port conflict problem this entire setup exists to solve. Every service — frontends, API backends, proxies — gets its own `*.localhost` subdomain through caddy.

```yaml
services:
  api:
    build: .
    expose:
      - "8080"
    networks: [default, caddy]
    labels:
      caddy: "http://api-${COMPOSE_PROJECT_NAME}.localhost"
      caddy.reverse_proxy: "{{upstreams 8080}}"
```

CLI tools, SDKs, and AI clients point to `http://api-myproject.localhost` instead of `localhost:PORT`. No conflicts, no port juggling, deterministic URLs.

### Native Apps (Tauri, Electron)

Desktop apps that use system tray, spawn processes, or access host hardware cannot run in Docker. Add a script to start them alongside the Docker services:

```json
{
  "scripts": {
    "up": "docker compose up -d && echo '  ⇒ http://myapp.localhost'",
    "tauri": "cd packages/app && bun run dev"
  }
}
```

If the native app manages a backend service (start/stop/monitor), ensure the Docker-managed version is accessible on the same port the native app expects (usually `localhost:PORT`).

### Filesystem Path Gotcha in Docker

Code that derives data paths from `import.meta.url` or `__dirname` resolves to paths inside the container image, NOT the Docker volume. Example:

```ts
const DATA_DIR = join(fileURLToPath(new URL("..", import.meta.url)), "data");
```

This resolves to `/app/packages/myservice/data` inside the container, which is ephemeral. Fix with a symlink in the Dockerfile:

```dockerfile
RUN mkdir -p /data && ln -s /data /app/packages/myservice/data
```

Or add an env var override for the data directory.

## Troubleshooting

| Problem | Fix |
| --- | --- |
| 502 Bad Gateway | Target container not running or wrong port in label |
| No response at *.localhost | Shared proxy not running — `cd ~/caddy-proxy && docker compose up -d` |
| Container not found | Service not on `caddy` network — add `networks: [default, caddy]` |
| Wrong project routes | `COMPOSE_PROJECT_NAME` defaults to directory name; set explicitly in `.env` if needed |
| Port conflict on :80 | Another process using port 80 — stop it or change the proxy port |
| bun frozen lockfile in Docker | bun auto-enables frozen lockfile with `--production` and in CI-detected environments (Docker). Pin bun version to match local, copy ALL workspace package.jsons, avoid `--production` |
| Workspace imports fail in container | Named volume on node_modules breaks workspace symlinks. Drop the named volume and bind-mount the whole workspace |
| Data lost on container restart | Check if code uses `import.meta.url`/`__dirname` for data paths — these resolve inside the image, not to volumes. Add symlinks or env var overrides |

## Verification

```bash
# Check shared proxy is running
docker ps --filter name=caddy-proxy

# Check a project's caddy labels are detected
docker inspect <container> --format '{{json .Config.Labels}}' | jq 'with_entries(select(.key | startswith("caddy")))'

# Test the URL
curl -sf http://myproject.localhost/
```

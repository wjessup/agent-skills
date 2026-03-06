---
name: caddy-docker-proxy-local
description: Set up local development with caddy-docker-proxy for automatic *.localhost routing via Docker labels. Use when creating docker-compose files, setting up local dev environments, adding reverse proxy to Docker projects, or when the user mentions localhost subdomains.
---

# Caddy Docker Proxy — Local Dev Setup

Uses [lucaslorentz/caddy-docker-proxy](https://github.com/lucaslorentz/caddy-docker-proxy) to give any Docker Compose project a clean `http://<project>.localhost` URL with automatic routing via Docker labels. No Caddyfile needed.

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

## Per-Project Setup

Add three things to any `docker-compose.yml`:

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

Use `expose` (not `ports`) — Caddy handles external access. Only publish `ports` if you also need direct access bypassing the proxy.

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

## Troubleshooting

| Problem | Fix |
| --- | --- |
| 502 Bad Gateway | Target container not running or wrong port in label |
| No response at *.localhost | Shared proxy not running — `cd ~/caddy-proxy && docker compose up -d` |
| Container not found | Service not on `caddy` network — add `networks: [default, caddy]` |
| Wrong project routes | `COMPOSE_PROJECT_NAME` defaults to directory name; set explicitly in `.env` if needed |
| Port conflict on :80 | Another process using port 80 — stop it or change the proxy port |

## Verification

```bash
# Check shared proxy is running
docker ps --filter name=caddy-proxy

# Check a project's caddy labels are detected
docker inspect <container> --format '{{json .Config.Labels}}' | jq 'with_entries(select(.key | startswith("caddy")))'

# Test the URL
curl -sf http://myproject.localhost/
```

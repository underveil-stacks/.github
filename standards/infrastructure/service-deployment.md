# Service Deployment Standard

How to deploy a service behind the underveil-stacks proxy infrastructure. Services run as Docker containers on a Raspberry Pi, discovered automatically by edgy (HAProxy) and exposed to the internet via public-proxy.

## Traffic Flow

```
Internet (HTTPS)
    │
    ▼
public-proxy (proxy-01.sigler.io)
    │  TLS termination, rate limiting, security headers
    │  WireGuard tunnel (10.10.10.0/24), plaintext HTTP
    ▼
edgy HAProxy (rpi-dev-01, 172.18.0.100:80)
    │  Host-based ACL routing
    │  Docker label discovery (every 10s)
    ▼
Service container (edgy_backend network)
```

- **public-proxy** handles TLS (Let's Encrypt), HSTS, rate limiting (200 req/30s), and security headers. It forwards `*.webby.sigler.io` traffic over WireGuard to edgy.
- **edgy** runs HAProxy with automatic Docker label discovery. It generates routing config, validates it, and reloads HAProxy with zero downtime.
- **Services** never handle TLS. They listen on HTTP and are only reachable via the `edgy_backend` Docker network.

## Docker Compose Setup

### Minimal Example

```yaml
services:
  myservice:
    build: .
    container_name: myservice
    restart: unless-stopped
    networks:
      - edgy_backend
    expose:
      - "8080"
    labels:
      - "edgy.enable=true"
      - "edgy.domain=myservice.webby.sigler.io"
      - "edgy.backend.port=8080"
      - "edgy.health.path=/api/v1/health"
      - "edgy.auth=bypass"
    healthcheck:
      test: ["CMD", "curl", "-sf", "http://localhost:8080/api/v1/health"]
      interval: 30s
      timeout: 10s
      start_period: 5s
      retries: 3

networks:
  edgy_backend:
    external: true
    name: edgy_backend
```

### Required Labels

| Label | Required | Description | Example |
|---|---|---|---|
| `edgy.enable` | Yes | Opt in to discovery | `true` |
| `edgy.domain` | Yes | Public domain for Host-header routing | `api.engram.webby.sigler.io` |
| `edgy.backend.port` | Yes | Port the service listens on | `8090` |
| `edgy.health.path` | No | HTTP health check path | `/api/v1/health` |
| `edgy.health.tcp` | No | Use TCP health check instead of HTTP | `true` |
| `edgy.auth` | No | Auth requirement: `required` or `bypass` | `bypass` |
| `edgy.backend.ssl` | No | Backend speaks TLS | `true` |
| `edgy.backend.timeout` | No | Custom backend timeout | `120s` |
| `edgy.cors.origin` | No | Allowed CORS origins (comma-separated) | `https://app.example.com` |

### Network

The `edgy_backend` network must exist before any service starts:

```bash
docker network inspect edgy_backend >/dev/null 2>&1 || docker network create edgy_backend
```

Deploy workflows should include this as a step. Do **not** define `edgy_backend` as a non-external network — it is shared across all service stacks.

Use `expose` (not `ports`) to make the service reachable on the Docker network without binding to the host. Services should never bind host ports unless they have a specific reason (e.g., edgy itself binds 80/443).

## Domain Convention

All services use the `*.webby.sigler.io` domain:

```
<service>.webby.sigler.io        — for standalone services
<subdomain>.<service>.webby.sigler.io  — for service sub-components (e.g., api.engram)
```

Examples:
- `api.engram.webby.sigler.io` — Engram REST API
- `grafana.webby.sigler.io` — Grafana dashboard
- `auth.webby.sigler.io` — Authelia authentication

New domains must be added to:
1. The `edgy.domain` label on your container
2. The edgy HAProxy `aliases` list in the edgy `docker-compose.yml` (so other containers on `edgy_backend` can resolve the domain internally)
3. The public-proxy's DNS or wildcard cert coverage (if not already covered by `*.webby.sigler.io`)

## Authentication

### `edgy.auth=bypass`

No authentication required. HAProxy forwards requests directly to the backend. Use for:
- Public APIs with their own auth (Bearer token)
- Static sites
- Health check endpoints

### `edgy.auth=required`

Authelia forward-auth is enforced. HAProxy checks with Authelia before forwarding. Use for:
- Web dashboards
- Admin interfaces
- Any service without its own auth layer

**Bearer token bypass**: Services with `edgy.auth=required` still allow requests with `Authorization: Bearer *` headers through without Authelia. This lets API clients authenticate directly with the service while browser users go through Authelia.

## Health Checks

### Docker Healthcheck

Every service must define a Docker `healthcheck` in its compose file. This is used by:
- Docker itself (container restart policy)
- Deploy workflows (wait-for-healthy step)

### Edgy Health Check

Set `edgy.health.path` to an HTTP endpoint that verifies backend connectivity (database, external APIs), not just "process is running". Edgy uses this to determine if the service should receive traffic.

If your service doesn't have an HTTP health endpoint, use `edgy.health.tcp=true` for a TCP port check.

## Deploy Workflow Pattern

Standard GitHub Actions deploy workflow:

```yaml
name: Deploy

on:
  workflow_run:
    workflows: ["Release"]
    types: [completed]
  workflow_dispatch:

defaults:
  run:
    working-directory: /home/rmrfslashbin/src/github.com/underveil-stacks/<repo>

jobs:
  deploy:
    name: Deploy to prod
    if: ${{ github.event_name == 'workflow_dispatch' || github.event.workflow_run.conclusion == 'success' }}
    runs-on: [self-hosted, rpi]
    steps:
      - name: Pull latest
        run: git fetch --tags origin && git checkout main && git pull

      - name: Get version
        id: version
        run: |
          TAG=$(git describe --tags --abbrev=0 2>/dev/null || echo "dev")
          echo "tag=${TAG}" >> "$GITHUB_OUTPUT"

      - name: Write environment
        env:
          # Map secrets to env vars
          DATABASE_URL: ${{ secrets.MY_DATABASE_URL }}
        run: |
          echo "VERSION=${{ steps.version.outputs.tag }}" > .env.deploy
          echo "MY_DATABASE_URL=${DATABASE_URL}" >> .env.deploy

      - name: Ensure edgy_backend network exists
        run: docker network inspect edgy_backend >/dev/null 2>&1 || docker network create edgy_backend

      - name: Build container
        run: |
          set -a; source .env.deploy; set +a
          docker compose build --no-cache

      - name: Restart service
        run: |
          set -a; source .env.deploy; set +a
          docker compose up -d

      - name: Wait for healthy
        run: |
          for i in $(seq 1 60); do
            if docker compose exec -T <service> curl -sf http://localhost:<port>/api/v1/health > /dev/null 2>&1; then
              echo "Healthy after $((i * 2))s"
              exit 0
            fi
            sleep 2
          done
          echo "Health check failed after 120s"
          docker compose logs --tail=30
          exit 1

      - name: Verify version
        run: |
          docker compose exec -T <service> curl -sf http://localhost:<port>/api/v1/about

      - name: Verify edgy discovery
        run: |
          sleep 15
          if docker exec edgy-haproxy wget -qO- http://127.0.0.1:8404/stats 2>/dev/null | grep -q "<service>"; then
            echo "Discovered by edgy"
          else
            echo "WARNING: not yet visible in edgy stats"
          fi

      - name: Show status
        if: always()
        run: docker compose ps
```

Key points:
- Triggered by successful Release workflow, or manually via `workflow_dispatch`
- Runs on `[self-hosted, rpi]` — the deployment target
- Uses `.env.deploy` (not `.env`) to avoid conflicts with local dev
- Verifies via `/about` (public, no auth needed) not `/info` (requires auth)
- Checks edgy discovery after a 15s wait (discovery runs every 10s)

## Internal Service-to-Service Communication

Containers on `edgy_backend` can reach other services via HAProxy's DNS aliases at `172.18.0.100:80`:

```
http://api.engram.webby.sigler.io:80/api/v1/health
```

This is plaintext HTTP over the Docker network — TLS is not used internally. If your service makes outbound calls to other platform services, use `http://` not `https://`.

## Reference Implementation

[Engram](https://github.com/underveil-stacks/engram) — see `docker-compose.yml` and `.github/workflows/deploy.yml`.

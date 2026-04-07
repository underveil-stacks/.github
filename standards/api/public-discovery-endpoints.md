# Public Discovery Endpoints

Every underveil-stacks service must expose unauthenticated endpoints that allow AI agents and clients to bootstrap themselves without prior knowledge of the API — including the URL prefix.

## Zero-Knowledge Discovery

An agent encountering a service for the first time knows only the hostname. It should not need to guess `/api/v1/` or any other prefix. Two root-level endpoints solve this:

| Endpoint | Behavior |
|---|---|
| `GET /about` | Returns the full discovery payload (same as `/api/v1/about`) |
| `GET /.well-known/ai-discovery` | 302 redirect to `/about` ([RFC 8615](https://www.rfc-editor.org/rfc/rfc8615)) |

**`/about`** is the canonical discovery endpoint. An agent hits it, learns the API prefix, capabilities, auth requirements, and where to find the spec and docs — all in one request.

**`/.well-known/ai-discovery`** exists for RFC-compliant clients that probe well-known URIs. It redirects to `/about` so there is one source of truth.

## Required Endpoints

| Endpoint | Content-Type | Purpose |
|---|---|---|
| `GET /about` | `application/json` | Root-level discovery — platform identity, capabilities, auth, endpoint map |
| `GET /.well-known/ai-discovery` | (302 redirect) | RFC 8615 discovery → redirects to `/about` |
| `GET /api/v1/about` | `application/json` | Same payload as `/about` (versioned path) |
| `GET /api/v1/openapi.yaml` | `application/yaml` | Full OpenAPI 3.0 spec with all endpoints and schemas |
| `GET /api/v1/docs` | `text/markdown` | Usage guide with worked examples and curl patterns |
| `GET /api/v1/health` | `application/json` | Backend connectivity check |

All are **public** (no authentication required). Every other route requires a valid Bearer token.

## Bootstrap Sequence

An AI agent discovering a new service follows this sequence:

1. **`GET /about`** (or `/.well-known/ai-discovery`) — learn what the service does, the API prefix, how auth works, and where to find detail
2. **`/api/v1/openapi.yaml`** — get the machine-readable API contract
3. **`/api/v1/docs`** — get the human-readable guide with worked examples
4. **`/api/v1/health`** — confirm the service is live before making authenticated calls

## Endpoint Specifications

### `GET /about` and `GET /api/v1/about`

Both return the same JSON object describing the service. `/about` is the root-level entry point; `/api/v1/about` is the versioned equivalent.

Required fields:

```json
{
  "name": "ServiceName",
  "description": "One-sentence description of what the service does.",
  "version": "v2026.04.10",
  "capabilities": [
    "Human-readable capability descriptions"
  ],
  "authentication": {
    "method": "Bearer token",
    "header": "Authorization: Bearer <api-key>",
    "manage": "How keys are created and managed",
    "storage": "How keys are stored (e.g., SHA-256 hashed in PostgreSQL)"
  },
  "endpoints": {
    "openapi_spec": "/api/v1/openapi.yaml",
    "health": "/api/v1/health",
    "about": "/api/v1/about",
    "docs": "/api/v1/docs"
  },
  "discovery": {
    "about": "/about",
    "well_known_ai_discovery": "/.well-known/ai-discovery",
    "note": "GET /about returns this payload directly. GET /.well-known/ai-discovery redirects to /about (RFC 8615)."
  },
  "source": "https://github.com/underveil-stacks/<repo>"
}
```

- `capabilities` is an array of strings, not structured data. Keep each entry concise and self-explanatory.
- `endpoints` maps logical names to versioned paths (`/api/v1/...`).
- `discovery` maps the root-level discovery endpoints. An agent uses these to find `endpoints`.
- `source` links to the GitHub repository.

### `GET /.well-known/ai-discovery`

Returns a `302 Found` redirect to `/about`. This follows [RFC 8615](https://www.rfc-editor.org/rfc/rfc8615) for well-known URIs. No payload — just a redirect.

### `GET /api/v1/openapi.yaml`

Returns the full OpenAPI 3.0 specification as YAML.

Rules:
- **Generated from code**, not a hand-maintained file. This ensures the spec stays in sync with the implementation.
- Cached on first request (generate once, serve from memory).
- Includes all endpoints, request/response schemas, and the authentication scheme.
- Uses the `bearerAuth` security scheme for protected endpoints.

### `GET /api/v1/docs`

Returns a Markdown document with the full API usage guide.

Rules:
- **Embedded in the binary** via `go:embed`. No filesystem dependency at runtime.
- Covers all endpoints with curl examples.
- Documents authentication setup (how to get a key, how to use it).
- Includes a wrapper script section if the service provides one.
- This content can double as a Claude Code skill — the agent fetches `/docs` at invocation time for always-current reference.

### `GET /api/v1/health`

Returns `{"status": "ok"}` when the service is healthy.

Rules:
- Checks **actual backend connectivity** (database ping, external service reachability), not just "process is running".
- Returns `503 Service Unavailable` with a descriptive error if any backend is unreachable.
- Used by orchestrators (Docker healthcheck, edgy discovery, deployment verification).

## AI Skill Document

Every service should maintain an **AI skill document** — a Markdown file that teaches an AI agent how to use the API in a single read. This document is:

1. **Stored in the repo** as `docs/guides/<SERVICE>_API_SKILL.md`
2. **Embedded in the binary** via `go:embed` and served at `/docs`
3. **Registered as a Claude Code skill** in the project's `.claude/settings.json`

### Required Sections

The skill doc must include, in order:

1. **One-line description** — what the service is and where it runs
2. **Bootstrap sequence** — numbered steps to go from zero to a working session:
   - Verify the wrapper script exists
   - Check `.env` has required variables (`<SERVICE>_API_KEY`, `<SERVICE>_HOST`)
   - Test authentication (e.g., `GET /me` or `GET /health`)
   - Discover capabilities via `/about`
   - Fetch the OpenAPI spec for parameter details
3. **Wrapper script usage** — how to use `scripts/<service>-api`
4. **Quick reference** — the highest-traffic curl patterns grouped by domain (no need to list every endpoint — that's what `/openapi.yaml` is for)
5. **Key conventions** — response format, pagination, date filtering, auth-free endpoints

### Why This Matters

The skill doc is what an AI agent reads on first invocation. It replaces the need to hardcode API knowledge into prompts. Because it's served from the running binary at `/docs`, it's always current with the deployed version — no stale context.

### Example: Engram

```markdown
# Engram API Access

You have access to Engram — a semantic memory storage platform backed by
PostgreSQL + pgvector. It runs behind the edgy reverse proxy at
`api.engram.webby.sigler.io`.

## Setup

**Required environment variables** (check `.env` in the repo root):
- `ENGRAM_API_KEY` — Bearer token for authentication
- `ENGRAM_HOST` — API hostname (e.g., `api.engram.webby.sigler.io`)

**Wrapper script:**
scripts/engram-api METHOD /path [json-body]

**Verify connectivity:**
scripts/engram-api GET /health  # should return {"status":"ok"}
...
```

See also: [Life API skill doc](https://github.com/underveil-stacks/life-api/issues/18) for a more detailed example with calendar, email, and search patterns.

## Wrapper Script Convention

Every service should provide `scripts/<service>-api` — a shell script that:

- Loads `.env` from the repo root
- Reads `<SERVICE>_API_KEY` and `<SERVICE>_HOST`
- Accepts `METHOD /path [json-body]` as arguments
- Auto-prefixes paths with `/api/v1`
- Sets `Authorization: Bearer` and `Content-Type: application/json` headers

This gives AI agents and developers a consistent interface across all services:

```bash
scripts/engram-api GET /health
scripts/life-api GET /profile
scripts/pbj-api GET /tasks/due
```

The wrapper script should be documented in the skill doc and referenced from `/docs`.

## Auth Bypass

### Application-Level Bypass

The service middleware must skip Bearer token validation for these exact paths:

- `/about`
- `/.well-known/ai-discovery`
- `/api/v1/about`
- `/api/v1/openapi.yaml`
- `/api/v1/docs`
- `/api/v1/health`

All other routes require authentication.

### Proxy-Level Bypass (edgy)

Services running behind edgy with `edgy.auth=required` must also declare path-level bypasses so Authelia doesn't block the discovery endpoints:

```yaml
labels:
  - "edgy.auth=required"
  - "edgy.auth.bypass.paths=/about,/.well-known,/api/v1/about,/api/v1/openapi.yaml,/api/v1/docs,/api/v1/health"
```

Services with `edgy.auth=bypass` don't need this — Authelia is never invoked. See the [service deployment standard](../infrastructure/service-deployment.md#edgyauthbypasspaths-per-path-bypass) for details.

## Reference Implementations

**[Engram](https://github.com/underveil-stacks/engram):**
- `/about` handler: `internal/api/handler_system.go`
- OpenAPI generation: `internal/api/openapi.go`
- Embedded docs: `docs/embed.go` + `docs/guides/ENGRAM_API_SKILL.md`
- Wrapper script: `scripts/engram-api`
- Auth bypass: `internal/api/middleware.go`
- Health check: `internal/api/handler_system.go` (pings PostgreSQL)

**[Life API](https://github.com/underveil-stacks/life-api):**
- `/about` with structured `capabilities`, `quick_start`, and `ai_skill` fields
- `/openapi.json` (JSON variant)
- Wrapper script: `scripts/life-api`
- Bootstrap sequence covering profile, calendar, email, PBJ, and semantic search

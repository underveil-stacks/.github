# Public Discovery Endpoints

Every underveil-stacks service must expose four unauthenticated endpoints that allow AI agents and clients to bootstrap themselves without prior knowledge of the API.

## Required Endpoints

| Endpoint | Content-Type | Purpose |
|---|---|---|
| `GET /api/v1/about` | `application/json` | Platform identity, capabilities, auth requirements, endpoint map |
| `GET /api/v1/openapi.yaml` | `application/yaml` | Full OpenAPI 3.0 spec with all endpoints and schemas |
| `GET /api/v1/docs` | `text/markdown` | Usage guide with worked examples and curl patterns |
| `GET /api/v1/health` | `application/json` | Backend connectivity check |

All four are **public** (no authentication required). Every other route requires a valid Bearer token.

## Bootstrap Sequence

An AI agent discovering a new service follows this sequence:

1. **`/about`** — learn what the service does, how auth works, and where to find detail
2. **`/openapi.yaml`** — get the machine-readable API contract
3. **`/docs`** — get the human-readable guide with worked examples
4. **`/health`** — confirm the service is live before making authenticated calls

## Endpoint Specifications

### `GET /api/v1/about`

Returns a JSON object describing the service. This is the **single entry point** an agent needs.

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
  "source": "https://github.com/underveil-stacks/<repo>"
}
```

- `capabilities` is an array of strings, not structured data. Keep each entry concise and self-explanatory.
- `endpoints` maps logical names to paths. Include at minimum the four public endpoints. Add service-specific entry points as appropriate.
- `source` links to the GitHub repository.

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

## Auth Bypass

The middleware must skip Bearer token validation for these exact paths:

- `/api/v1/about`
- `/api/v1/openapi.yaml`
- `/api/v1/docs`
- `/api/v1/health`

All other routes require authentication.

## Reference Implementation

[Engram](https://github.com/underveil-stacks/engram) is the reference implementation:

- `/about` handler: `internal/api/handler_system.go`
- OpenAPI generation: `internal/api/openapi.go`
- Embedded docs: `docs/embed.go` + `docs/guides/ENGRAM_API_SKILL.md`
- Auth bypass: `internal/api/middleware.go`
- Health check: `internal/api/handler_system.go` (pings PostgreSQL)

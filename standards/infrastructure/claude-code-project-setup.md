# Claude Code Project Setup Standard

How to configure Claude Code for underveil-stacks repositories. A well-configured project gives Claude accurate context about the organization, infrastructure, coding conventions, and available tooling — reducing repeated explanations and avoiding mistakes.

## CLAUDE.md

Every repository should have a `CLAUDE.md` at its root. This file is automatically loaded into Claude Code's context when working in the repository.

### Required Sections

#### Build/Test Commands

List the commands Claude needs to build, test, lint, and run the project. Include the exact `make` targets or CLI commands.

```markdown
## Build/Test Commands
- Build: `make build`
- Test: `make test`
- Lint: `make lint`
- Format: `make fmt`
- Dev setup: `make dev-setup`
- Pre-commit checks: `make check`
```

#### Environment Notes

Platform-specific details that affect how Claude runs commands.

```markdown
## Environment Notes
- **Platform**: macOS (development), Debian (production)
- **Go Version**: 1.26
- **Database**: PostgreSQL 17 with pgvector on proxy-01 (10.10.10.1:5432)
- **No SSL on DB**: WireGuard provides encryption
```

#### Code Style Guidelines

Conventions that can't be inferred from linters alone.

```markdown
## Code Style Guidelines
- **Imports**: Group standard library, third-party, and internal imports
- **Error Handling**: Always check errors, use custom error types with context
- **Logging**: Structured logging with `slog`, log to STDERR only
- **Storage**: All storage operations through interfaces, no direct DB imports in handlers
- **Dead code**: Keep it clean — run `deadcode` periodically
```

#### Architecture

A concise map of the codebase — what lives where and how the layers connect.

```markdown
## Architecture

### HTTP API Layer
- `internal/api/` — REST handlers, middleware, router, OpenAPI spec

### Storage Layer
- `internal/store/` — backend-agnostic interfaces
- `internal/store/postgres/` — PostgreSQL implementation

### Configuration
- `MYSERVICE_DATABASE_URL` — PostgreSQL connection string (required)
- `MYSERVICE_BACKEND_API_LISTEN` — HTTP listen address (required)
```

### Optional Sections

Include these when relevant:

- **Database Management** — migration commands, dev tunnel setup, maintenance operations
- **API Key Management** — how to create, list, revoke keys
- **Client Library** — if the repo includes a client package
- **Deploy** — deploy workflow or manual deploy steps

### What NOT to Include

- Obvious language conventions (Go formatting, Python PEP 8) — linters handle these
- Full API documentation — that belongs in `/docs` or OpenAPI spec
- Secrets or connection strings — those belong in `.env` or vault
- Architectural decisions or rationale — those belong in ADRs or commit messages

## Organization Context

Claude Code supports a `~/.claude/CLAUDE.md` (user-level) that applies to all projects. For underveil-stacks contributors, this is a good place to put organizational context that spans repos.

### Recommended User-Level Context

```markdown
## underveil-stacks Organization

### Infrastructure
- **public-proxy** (proxy-01.sigler.io): Internet-facing reverse proxy. HAProxy, TLS termination, rate limiting, nftables, CrowdSec. Managed by Ansible.
- **edgy** (rpi-dev-01): Docker-based service discovery HAProxy. Auto-discovers containers via Docker labels.
- **WireGuard tunnel**: proxy-01 (10.10.10.1) <-> rpi-dev-01 (10.10.10.2). All inter-host traffic flows through this.
- **PostgreSQL 17**: Runs on proxy-01, accessible at 10.10.10.1:5432 via WireGuard. pgvector enabled. No SSL (WireGuard encrypts).

### Domain Convention
- All services: `*.webby.sigler.io`
- API services: `api.<service>.webby.sigler.io`

### Tech Stack
- **Backend**: Go (1.26+), pgx/v5 (no ORM), golang-migrate for schema migrations
- **Database**: PostgreSQL 17 + pgvector, managed by Ansible playbooks in public-proxy
- **Deployment**: Docker containers on rpi-dev-01, discovered by edgy, exposed via public-proxy
- **CI/CD**: GitHub Actions with self-hosted runner on proxy-01
- **IaC**: Ansible (public-proxy repo) for proxy-01 configuration

### Standards
- All services expose `/about`, `/.well-known/ai-discovery`, `/api/v1/health` (public, no auth)
- OpenAPI spec generated from code, served at `/api/v1/openapi.yaml`
- API docs embedded in binary via `go:embed`, served at `/api/v1/docs`
- Bearer token auth on all other endpoints
- Wrapper scripts at `scripts/<service>-api` for CLI/AI access

### Database Provisioning
- Ansible playbooks in public-proxy: `pg-provision-<service>.yml`
- Passwords in Ansible Vault: `vault_<service>_<role>_password`
- Pattern A (single owner): app handles migrations — use `<service>_app` role
- Pattern B (least-privilege): separate migrate/app/readonly roles
```

## Ansible Repositories (public-proxy)

For the public-proxy (infrastructure) repository, the CLAUDE.md should emphasize:

```markdown
## Build/Test Commands
- Full provision: `make site`
- Dry-run all: `make check`
- Targeted role: `make haproxy`, `make certbot`, `make postgresql`, etc.
- Vault edit: `make vault-edit`
- Vault view: `make vault-view`
- Status checks: `make ping`, `make pg-status`, `make haproxy-status`, `make wg-status`

## Environment Notes
- **Platform**: Ansible control node on macOS, target is Debian (proxy-01)
- **Ansible**: Uses `community.postgresql` and `community.general` collections
- **Inventory**: Single host (proxy01) at IPv6 address
- **Vault**: `group_vars/all/vault.yml` encrypted with ansible-vault

## Code Style Guidelines
- **Templates**: All config files are Jinja2 templates with "Managed by Ansible" header
- **Check-mode safety**: Read-only tasks (version checks, health probes, GET requests) must include `check_mode: false` so `make check` passes
- **Idempotency**: All playbooks must be safe to re-run
- **Secrets**: Never in vars.yml — always `{{ vault_* }}` references to vault.yml
- **Role flags**: Non-superuser roles get `LOGIN,NOSUPERUSER,NOCREATEDB,NOCREATEROLE`

## Architecture
- `site.yml` — main playbook, applies all roles in order
- `pg-provision-*.yml` — per-service database provisioning
- `pg-admin.yml` — ad-hoc DBA operations (tagged, never auto-run)
- `roles/` — one role per concern (security, wireguard, postgresql, haproxy, etc.)
- `group_vars/all/vars.yml` — all non-secret configuration
- `group_vars/all/vault.yml` — all secrets (encrypted)

### PostgreSQL
- Version: 17 with pgvector 0.8.0
- Listens on: 127.0.0.1 and 10.10.10.1 (WireGuard)
- Auth: SCRAM-SHA-256
- No SSL (WireGuard provides encryption)
- Provisioning: `make pg-<service>` targets
```

## Skills

Services with API wrapper scripts should register them as Claude Code skills. The skill doc is fetched from the running service at `/api/v1/docs`, ensuring the agent always has current documentation.

See the [Public Discovery Endpoints](../api/public-discovery-endpoints.md#ai-skill-document) standard for skill document requirements.

## Check-Mode Safety

When writing Ansible roles, ensure `make check` (dry-run) passes by following these rules:

### Read-only tasks that must run in check mode

Add `check_mode: false` to tasks that gather information needed by downstream tasks:

```yaml
# Version checks
- name: Check installed version
  ansible.builtin.command:
    cmd: myapp --version
  check_mode: false  # read-only — safe in check mode
  register: version_check
  changed_when: false
  failed_when: false

# Health probes
- name: Check service health
  ansible.builtin.uri:
    url: "http://localhost:8080/health"
    method: GET
  check_mode: false  # read-only GET — safe in check mode
  register: health_result
  failed_when: false

# API lookups
- name: Look up Cloudflare zone
  ansible.builtin.uri:
    url: "https://api.cloudflare.com/client/v4/zones?name={{ domain }}"
    method: GET
  check_mode: false  # read-only GET — safe in check mode
  register: zone_lookup
```

### Download-then-copy chains

When a task downloads a file locally and a subsequent task copies it to the remote, both must run in check mode — `get_url` won't download and `command` won't extract in check mode, causing the copy to fail:

```yaml
- name: Download tarball
  ansible.builtin.get_url:
    url: "https://example.com/release.tar.gz"
    dest: /tmp/release.tar.gz
  check_mode: false  # download to local temp — needed by extract step
  delegate_to: localhost

- name: Extract binary
  ansible.builtin.command:
    cmd: tar -xzf /tmp/release.tar.gz -C /tmp/ mybinary
  check_mode: false  # local temp extraction — safe in check mode
  delegate_to: localhost
```

### Template variable alignment

Ensure template variables match the names used in `register:` statements. A mismatch silently works when the template isn't re-rendered (no config change), but fails in check mode or on first provision:

```yaml
# Task registers as wireguard_server_privkey
- name: Read server private key
  ansible.builtin.slurp:
    src: /etc/wireguard/server.key
  register: wireguard_server_privkey

# Template must use the same name
# WRONG: {{ wg_server_privkey.content | b64decode }}
# RIGHT: {{ wireguard_server_privkey.content | b64decode }}
```

## Checklist for New Repositories

1. Create `CLAUDE.md` at the repo root with required sections
2. Add organization context to `~/.claude/CLAUDE.md` (if not already present)
3. Register API wrapper script as a Claude Code skill (if applicable)
4. Verify `CLAUDE.md` stays accurate as the project evolves — stale context is worse than no context
5. For Ansible repos: ensure all tasks pass `make check` (dry-run) before committing

## Reference Implementations

- **Go service:** [Engram CLAUDE.md](https://github.com/underveil-stacks/engram/blob/main/CLAUDE.md) — comprehensive build commands, architecture map, environment config
- **Ansible infrastructure:** [public-proxy](https://github.com/underveil-stacks/public-proxy) — Makefile targets, role structure, database provisioning

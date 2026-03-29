# underveil-stacks

A collection of self-hosted, privacy-first personal computing tools built around the principle that your data belongs on hardware you own — searchable with local AI and directly accessible to Claude via MCP.

All services run on a home Raspberry Pi, reachable from the internet via a Hetzner VPS over WireGuard, protected by SSO, and observable end-to-end.

---

## Platforms

### Manuals — Documentation Management
Store, tag, and semantically search personal and technical documentation.

| Repo | Description |
|---|---|
| [manuals](https://github.com/underveil-stacks/manuals) | Orchestrator — Docker Compose + submodules |
| [manuals-api](https://github.com/underveil-stacks/manuals-api) | Go backend — SQLite, MinIO, Qdrant, local LLM |
| [manuals-webui](https://github.com/underveil-stacks/manuals-webui) | Vue 3 frontend SPA |
| [manuals-mcp](https://github.com/underveil-stacks/manuals-mcp) | MCP server for Claude Desktop / Claude Code |
| [manuals-cli](https://github.com/underveil-stacks/manuals-cli) | Go CLI companion tool (public) |

### Life — Personal Data Hub
Aggregate email, calendar, packages, and journal entries into a single Claude-accessible store.

| Repo | Description |
|---|---|
| [life](https://github.com/underveil-stacks/life) | Orchestrator — Docker Compose + submodules |
| [life-api](https://github.com/underveil-stacks/life-api) | Go backend — SQLite + sqlite-vec, nomic embeddings |
| [life-mcp](https://github.com/underveil-stacks/life-mcp) | MCP server for Claude Desktop / Claude Code |
| [life-email-agent](https://github.com/underveil-stacks/life-email-agent) | Proton Mail ingest daemon |

### Rivulet — Feed Aggregation
Multi-user RSS/Atom/Podcast/Mastodon/Bluesky feed aggregator with workflow automation and semantic search.

| Repo | Description |
|---|---|
| [rivulet](https://github.com/underveil-stacks/rivulet) | Go backend — SQLite, Qdrant, workflow engine |
| [rivulet-web](https://github.com/underveil-stacks/rivulet-web) | Vue 3 + Vuetify 3 + TypeScript frontend |

### PBJ — Personal Bullet Journal
Fully offline bullet journal with true bullet-journal semantics and native Claude integration via MCP.

| Repo | Description |
|---|---|
| [pbj](https://github.com/underveil-stacks/pbj) | Go + SQLite + sqlite-vec + Ollama, Raspberry Pi 5 |

### Bluesky Archival
Archive and analyze your Bluesky likes, threads, and activity with local AI.

| Repo | Description |
|---|---|
| [bsky-tool](https://github.com/underveil-stacks/bsky-tool) | Go CLI + REST API — SQLite, local LLM, TUI |
| [bsky-tool-webui](https://github.com/underveil-stacks/bsky-tool-webui) | Vue 3 dashboard frontend |

---

## Infrastructure

| Repo | Description |
|---|---|
| [microservices-stack](https://github.com/underveil-stacks/microservices-stack) | Top-level orchestration docs for the home stack |
| [public-proxy](https://github.com/underveil-stacks/public-proxy) | Ansible — Hetzner VPS, HAProxy, WireGuard, Let's Encrypt |
| [edgy](https://github.com/underveil-stacks/edgy) | Internal HAProxy router with Docker label discovery |
| [authy](https://github.com/underveil-stacks/authy) | SSO/OIDC — Authelia + LLDAP + Redis |
| [webby](https://github.com/underveil-stacks/webby) | Nginx static hosting and platform dashboard |
| [llm-stack](https://github.com/underveil-stacks/llm-stack) | Local LLM inference — llama.cpp, OpenAI-compatible API, mTLS |
| [victoriametrics](https://github.com/underveil-stacks/victoriametrics) | VictoriaMetrics + VictoriaLogs observability stack |
| [grafana](https://github.com/underveil-stacks/grafana) | Grafana dashboards for platform observability |
| [proton-bridge-docker](https://github.com/underveil-stacks/proton-bridge-docker) | Headless Proton Mail Bridge container (amd64/arm64) |
| [proton-imap-client](https://github.com/underveil-stacks/proton-imap-client) | Go library for IMAP access via Proton Bridge |

---

## Architecture

```
Internet
  └── public-proxy  (Hetzner VPS — HAProxy + WireGuard + Let's Encrypt)
        └── WireGuard tunnel ──→ Raspberry Pi 5
                                    └── edgy  (HAProxy, Docker label routing)
                                          ├── authy           (Authelia SSO + LLDAP)
                                          ├── webby           (dashboard + static sites)
                                          ├── manuals-api     (docs platform)
                                          ├── life-api        (personal data hub)
                                          ├── rivulet         (feed aggregation)
                                          ├── pbj             (bullet journal)
                                          ├── llm-stack       (local LLM inference, mTLS)
                                          ├── victoriametrics (metrics + logs)
                                          └── grafana         (dashboards)
```

---

## Common Patterns

- **Go + Vue 3** across all application stacks
- **SQLite** as the primary datastore (+ FTS5 full-text, sqlite-vec for vectors)
- **Local AI** — llm-stack provides OpenAI-compatible LLM and embedding APIs; no cloud AI dependencies
- **MCP-native** — Manuals, Life, and PBJ expose MCP servers for Claude
- **Authelia SSO** — single sign-on across all web-facing services
- **Docker + Makefile** — consistent deployment pattern throughout

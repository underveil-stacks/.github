# underveil-stacks

A collection of self-hosted, privacy-first personal computing tools built around the principle that your data belongs on hardware you own — searchable with local AI and directly accessible to Claude via MCP.

## Common Patterns

- **Go + Vue 3** across all application stacks
- **SQLite** as the primary datastore (+ FTS5 full-text, sqlite-vec for vectors)
- **Local AI** — OpenAI-compatible LLM and embedding APIs; no cloud AI dependencies
- **MCP-native** — services expose MCP servers for Claude
- **Docker + Makefile** — consistent deployment pattern throughout

# underveil-stacks Standards

Engineering standards for the underveil-stacks organization. These documents define conventions that all services should follow.

## API

- [Public Discovery Endpoints](api/public-discovery-endpoints.md) — The four unauthenticated endpoints every service must expose for AI agent and client bootstrapping

## Infrastructure

- [Service Deployment](infrastructure/service-deployment.md) — How to deploy behind edgy + public-proxy: Docker labels, networking, domain conventions, deploy workflows, health checks
- [Database Provisioning](infrastructure/database-provisioning.md) — PostgreSQL database and role setup: naming conventions, Ansible playbooks, vault passwords, provisioning patterns
- [Claude Code Project Setup](infrastructure/claude-code-project-setup.md) — CLAUDE.md structure, organization context, skills, check-mode safety, Ansible conventions

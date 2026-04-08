# Database Provisioning Standard

How to provision a PostgreSQL database for a new underveil-stacks service. Databases run on proxy-01 (PostgreSQL 17 with pgvector 0.8.0), managed by Ansible playbooks in the [public-proxy](https://github.com/underveil-stacks/public-proxy) repository.

## Connection Architecture

```text
Service container (rpi-dev-01)
    │
    │  PostgreSQL wire protocol, plaintext
    │  WireGuard tunnel (10.10.10.0/24)
    │
    ▼
PostgreSQL 17 (proxy-01, 10.10.10.1:5432)
```

- PostgreSQL listens on `127.0.0.1` and `10.10.10.1` (WireGuard interface)
- **No SSL** — the WireGuard tunnel provides encryption
- Authentication: SCRAM-SHA-256
- Connection string format: `postgres://<user>:<password>@10.10.10.1:5432/<database>?sslmode=disable`

## Provisioning Patterns

Choose the pattern that fits your service's needs.

### Pattern A: Single Owner Role (recommended for most services)

One role owns the database and handles everything — migrations, DML, extension setup. Use this when:
- The application manages its own schema migrations on startup
- You don't need separate credentials for different access levels
- Development and CI share the same privilege model

**Examples:** Atelier, Manuals Platform

```text
Database: <service>
Role:     <service>_app (owner, LOGIN, CREATE + USAGE on public schema)
```

### Pattern B: Least-Privilege Split

Separate roles for migrations, application runtime, and read-only access. Use this when:
- Schema migrations run via a dedicated tool (e.g., golang-migrate CLI)
- You want the application runtime to be unable to ALTER or DROP tables
- You need a read-only role for analytics or debugging

**Example:** Engram

```text
Database: <service>_<env>
Roles:
  <service>_migrate   — owner, runs DDL (CREATE TABLE, ALTER, etc.)
  <service>_app       — DML only (SELECT, INSERT, UPDATE, DELETE)
  <service>_readonly   — SELECT only
```

Default privileges ensure `<service>_app` and `<service>_readonly` automatically get access to tables created by the migrate role.

## Naming Conventions

| Resource | Convention | Examples |
|---|---|---|
| Database (prod) | `<service>` | `manuals`, `atelier`, `engram_prod` |
| Database (dev) | `<service>_dev` | `manuals_dev`, `engram_dev` |
| Database (test) | `<service>_test` | `life_test` |
| Owner role | `<service>_app` or `<service>_migrate` | `manuals_app`, `engram_migrate` |
| App role | `<service>_app` | `engram_app` |
| Read-only role | `<service>_readonly` | `engram_readonly` |
| Dev role | `<service>_dev` | `manuals_dev` |
| Vault password | `vault_<service>_<role>_password` | `vault_manuals_app_password` |

## Standard Configuration

All databases use:

| Setting | Value |
|---|---|
| Encoding | UTF-8 |
| Collation | C.UTF-8 |
| Ctype | C.UTF-8 |
| Template | template0 |

Role flags (non-owner):

```text
LOGIN, NOSUPERUSER, NOCREATEDB, NOCREATEROLE
```

## Playbook Structure

Every database gets its own provisioning playbook at the repository root: `pg-provision-<service>.yml`

### Minimal Playbook (Pattern A)

```yaml
---
- name: Provision <Service> database
  hosts: proxy
  become: true
  gather_facts: false

  vars:
    db_name: <service>
    db_user: <service>_app
    db_password: "{{ vault_<service>_app_password }}"

  tasks:
    # ---- Role ----
    - name: "Create role: {{ db_user }}"
      become_user: postgres
      community.postgresql.postgresql_user:
        name: "{{ db_user }}"
        password: "{{ db_password }}"
        role_attr_flags: LOGIN,NOSUPERUSER,NOCREATEDB,NOCREATEROLE

    # ---- Database ----
    - name: "Create database: {{ db_name }}"
      become_user: postgres
      community.postgresql.postgresql_db:
        name: "{{ db_name }}"
        owner: "{{ db_user }}"
        encoding: UTF-8
        lc_collate: C.UTF-8
        lc_ctype: C.UTF-8
        template: template0

    # ---- Extensions (superuser required) ----
    - name: "Enable pgvector on {{ db_name }}"
      become_user: postgres
      community.postgresql.postgresql_ext:
        name: vector
        db: "{{ db_name }}"

    # ---- Schema privileges ----
    - name: "Grant CREATE, USAGE on schema public to {{ db_user }}"
      become_user: postgres
      community.postgresql.postgresql_privs:
        database: "{{ db_name }}"
        role: "{{ db_user }}"
        type: schema
        objs: public
        privs: CREATE,USAGE
        state: present

    # ---- Verify ----
    - name: "Verify {{ db_user }} can connect to {{ db_name }}"
      become_user: postgres
      ansible.builtin.command:
        cmd: >
          psql -U {{ db_user }} -d {{ db_name }}
          -h 127.0.0.1 -c 'SELECT current_user, current_database();'
      environment:
        PGPASSWORD: "{{ db_password }}"
      changed_when: false

    - name: "Verify extensions on {{ db_name }}"
      become_user: postgres
      ansible.builtin.command:
        cmd: "psql -d {{ db_name }} -c '\\dx'"
      register: _ext_check
      changed_when: false

    - name: "Show extensions on {{ db_name }}"
      ansible.builtin.debug:
        var: _ext_check.stdout_lines
```

### Multi-Environment Support

For services that need dev/prod databases, parameterize with an environment variable:

```yaml
vars:
  service_env: prod
  db_name: "{{ (service_env == 'prod') | ternary('myservice', 'myservice_dev') }}"
  db_user: "{{ (service_env == 'prod') | ternary('myservice_app', 'myservice_dev') }}"
  db_password: "{{ (service_env == 'prod') | ternary(vault_myservice_app_password, vault_myservice_dev_password) }}"
```

Then invoke with `-e service_env=dev` for the development database.

## Wiring Into the Makefile

Every playbook needs:

1. **A `make` target** for each environment
2. **An entry in `make check`** for dry-run validation

```makefile
# ---- <Service> database provisioning ----

.PHONY: pg-<service>
pg-<service>: ## Provision <service> prod database, role, extensions, and privileges
	$(ANSIBLE) pg-provision-<service>.yml

.PHONY: pg-<service>-dev
pg-<service>-dev: ## Provision <service>_dev database, role, extensions, and privileges
	$(ANSIBLE) pg-provision-<service>.yml -e <service>_env=dev
```

Add to the `check` target:

```makefile
check: ## Dry-run all playbooks
	...
	$(ANSIBLE) pg-provision-<service>.yml --check --diff
	$(ANSIBLE) pg-provision-<service>.yml --check --diff -e <service>_env=dev
```

## Vault Password Setup

Before running the playbook, add passwords to `group_vars/all/vault.yml`:

```bash
make vault-edit
```

Add entries following the naming convention:

```yaml
vault_<service>_app_password: "<generated>"
vault_<service>_dev_password: "<generated>"    # if multi-environment
```

Generate passwords with:

```bash
openssl rand -base64 32
```

## Extensions

pgvector is pre-installed on the server. The playbook enables it per-database using `become_user: postgres` (superuser). Applications should still include `CREATE EXTENSION IF NOT EXISTS vector` in their migrations for portability, but the DBA playbook ensures it's available.

Other commonly used extensions:

| Extension | Purpose |
|---|---|
| `vector` | pgvector — vector similarity search (IVFFlat, HNSW indexes) |
| `pgcrypto` | Cryptographic functions (gen_random_uuid, etc.) |
| `pg_stat_statements` | Query performance statistics |

Add additional extensions to the playbook as needed.

## Checklist for New Services

1. Choose a provisioning pattern (A or B)
2. Add vault passwords (`make vault-edit`)
3. Create `pg-provision-<service>.yml` in the public-proxy repo
4. Add Makefile targets and update `make check`
5. Run `make pg-<service>` to provision
6. Hand the connection string to the application team
7. Verify with `make pg-status`

## Reference Implementations

- **Pattern A (single owner):** [pg-provision-manuals.yml](https://github.com/underveil-stacks/public-proxy/blob/main/pg-provision-manuals.yml), [pg-provision-atelier.yml](https://github.com/underveil-stacks/public-proxy/blob/main/pg-provision-atelier.yml)
- **Pattern B (least-privilege):** [pg-provision-engram.yml](https://github.com/underveil-stacks/public-proxy/blob/main/pg-provision-engram.yml)
- **DBA ad-hoc operations:** [pg-admin.yml](https://github.com/underveil-stacks/public-proxy/blob/main/pg-admin.yml) with `make pg-*` targets

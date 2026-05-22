# Self-Hosted Docker Setup Design

**Date:** 2026-05-22  
**Status:** Approved

---

## Overview

Production-hardened self-hosted Docker setup on a VPS. Traefik as reverse proxy, Authelia for SSO + TOTP + OIDC, Komodo for orchestration, docker-volume-backup for backups via SFTP, sops+age for secrets.

Goal: add new services with minimal effort while maintaining high security and performance.

---

## Repo Structure

```
docker-compose-files/
├── .sops.yaml                  # age public key — covers all .env files in subdirs
├── .gitignore                  # blocks .env from git
├── README.md
├── _template/                  # copy to add a new service
│   ├── compose.yaml
│   ├── Makefile
│   ├── .env.example
│   └── README.md
├── traefik/                    # reverse proxy + authelia (existing)
├── komodo/                     # orchestrator — manages all stacks
├── backup/                     # central backup via docker-volume-backup
├── vaultwarden/                # password manager + postgres
├── jellyfin/                   # media server
├── forgejo/                    # git hosting + postgres
└── sites/                      # static sites (one subdir per site, or grouped)
```

**Adding a new service:**
```bash
cp -r _template/ myservice/
cd myservice
# edit compose.yaml and .env.example
cp .env.example .env && $EDITOR .env
make encrypt
make up        # or register in Komodo
```

---

## Stack Template

Every stack follows this pattern. `_template/` is the canonical source.

### Makefile

Only `STACK_NAME` and the `secrets-init` extract lines differ per stack.

```makefile
STACK_NAME  := myservice
SECRETS_DIR := /dev/shm/$(STACK_NAME)-secrets
SOPS_FILE   := .env.sops
COMPOSE     := docker compose -f compose.yaml

.PHONY: secrets-init secrets-clean up down logs ps encrypt decrypt edit

secrets-init:
	@mkdir -p $(SECRETS_DIR) && chmod 700 $(SECRETS_DIR)
	# stack-specific sops extract lines here
	@chmod 600 $(SECRETS_DIR)/*
	@echo "Secrets written to $(SECRETS_DIR) (RAM only)"

secrets-clean:
	@rm -rf $(SECRETS_DIR)
	@echo "Secrets wiped from $(SECRETS_DIR)"

up: secrets-init
	@DOMAIN=$$(sops --decrypt --extract '["DOMAIN"]' $(SOPS_FILE)) \
	 $(COMPOSE) up -d

down:
	$(COMPOSE) down
	$(MAKE) secrets-clean

logs: ; $(COMPOSE) logs -f
ps:   ; $(COMPOSE) ps

encrypt:
	sops --encrypt .env > $(SOPS_FILE)
	rm -f .env

decrypt:
	sops --decrypt $(SOPS_FILE) > .env
	@echo "WARNING: .env is plaintext on disk. Delete after use."

edit:
	sops $(SOPS_FILE)
```

### compose.yaml hardening (all services)

Every container gets:
- `cap_drop: ALL` (add back only what's needed, e.g. `NET_BIND_SERVICE`)
- `read_only: true` with explicit `tmpfs` for writable paths
- `security_opt: [no-new-privileges=true]`
- `user: "UID:GID"` (non-root)
- `restart: unless-stopped`
- `healthcheck:`
- `logging:` with `json-file`, `max-size: 10m`, `max-file: 3`
- `deploy.resources` with CPU, memory, PID limits
- Pinned image tags — no `latest`

### Traefik label patterns

```yaml
# Protected service (Authelia SSO + TOTP required)
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.SVCNAME.rule=Host(`svc.${DOMAIN}`)"
  - "traefik.http.routers.SVCNAME.entrypoints=websecure"
  - "traefik.http.routers.SVCNAME.tls.certresolver=cloudflare"
  - "traefik.http.routers.SVCNAME.middlewares=authelia@docker,chain-default@file"

# Admin UI (stricter headers + tighter rate limit)
  - "traefik.http.routers.SVCNAME.middlewares=authelia@docker,chain-admin@file"

# Public endpoint (no auth)
  - "traefik.http.routers.SVCNAME.middlewares=chain-default@file"
```

### Network pattern

```yaml
services:
  app:
    networks: [proxy, internal]   # only web-facing container joins proxy
  db:
    networks: [internal]          # DB never touches proxy network

networks:
  proxy:
    external: true                # shared Traefik network
  internal:
    internal: true                # no outbound internet, isolated per stack
```

Single-container stacks (static sites) only need `proxy`. Multi-container stacks always use an `internal` network. DB containers never have `ports:` defined.

---

## Komodo Stack

Manages all stacks by calling `make up` / `make down` per stack. Fully compatible with manual operation via SSH.

### Services

- **socket-proxy** — Docker socket proxy with write permissions for Komodo periphery
- **periphery** — Komodo agent; executes on host, calls Makefiles, manages compose
- **core** — Komodo web UI + API; exposed via Traefik behind Authelia

### Socket-proxy permissions (write-enabled for orchestration)

```yaml
environment:
  CONTAINERS: 1
  NETWORKS: 1
  VOLUMES: 1
  IMAGES: 1
  INFO: 1
  PING: 1
  POST: 1      # create/start/stop
  DELETE: 1    # remove containers/images
  AUTH: 1      # registry pulls
  EXEC: 1      # exec into containers
```

### Periphery mounts

Repo root bind-mounted read-only into periphery so it can run `make up` per stack:
```yaml
volumes:
  - /home/deploy/docker-compose-files:/stacks:ro   # adjust to actual clone path on VPS
```

Komodo deploy action per stack:
```bash
cd /stacks/vaultwarden && make up
```

### Networks

```
komodo-socket  (internal) — periphery ↔ socket-proxy
komodo-internal (internal) — core ↔ periphery
proxy (external) — core ↔ Traefik only
```

---

## Backup Stack

Central backup via `offen/docker-volume-backup`. Stateful services are discovered via Docker labels. Backups sent to remote SFTP server daily.

### Socket-proxy permissions (limited — exec only)

```yaml
environment:
  CONTAINERS: 1
  INFO: 1
  EXEC: 1      # for pg_dump pre-backup exec
  POST: 1      # exec create requires POST
```

### Backup container config

```yaml
environment:
  BACKUP_CRON_EXPRESSION: "0 3 * * *"       # 3am daily
  BACKUP_RETENTION_DAYS: "14"
  BACKUP_FILENAME: "backup-%Y-%m-%dT%H-%M-%S.tar.gz"
  SSH_HOST_NAME: ${SSH_HOST}
  SSH_PORT: "22"
  SSH_USER: ${SSH_USER}
  SSH_PRIVATE_KEY_FILE: /run/secrets/ssh_key
  SSH_REMOTE_PATH: ${SSH_REMOTE_PATH}
```

SSH key stored via sops, decrypted to `/dev/shm/backup-secrets/ssh_key` at runtime.

### Per-service backup labels

**Postgres services (vaultwarden, forgejo):**
```yaml
labels:
  - "docker-volume-backup.exec-label=SVCNAME-db"
  - "docker-volume-backup.exec-pre=pg_dump -U $POSTGRES_USER $POSTGRES_DB > /var/lib/postgresql/data/dump.sql"
```

**Jellyfin (stop during backup):**
```yaml
labels:
  - "docker-volume-backup.stop-during-backup=true"
```

Media volumes (large, re-downloadable) are excluded from backup — only config/DB volumes backed up.

---

## Authelia OIDC

Authelia acts as OIDC identity provider for services that support it. Single login covers Komodo, Forgejo, Jellyfin. Vaultwarden and static sites use forward auth only.

### Authelia configuration additions

```yaml
identity_providers:
  oidc:
    issuer_private_key: ${AUTHELIA_OIDC_ISSUER_KEY}   # EC P-384, stored via sops
    access_token_lifespan: 15m
    authorize_code_lifespan: 1m
    refresh_token_lifespan: 90m

    clients:
      - client_id: komodo
        client_secret_digest: "$argon2id$..."          # hashed, never plaintext
        require_pkce: true
        response_types: [code]
        redirect_uris:
          - https://komodo.${DOMAIN}/auth/callback
        scopes: [openid, profile, email, groups]
        grant_types: [authorization_code]
        access_token_lifespan: 15m
        authorize_code_lifespan: 1m

      - client_id: forgejo
        client_secret_digest: "$argon2id$..."
        require_pkce: true
        response_types: [code]
        redirect_uris:
          - https://forgejo.${DOMAIN}/user/oauth2/authelia/callback
        scopes: [openid, profile, email]
        grant_types: [authorization_code]
        access_token_lifespan: 15m
        authorize_code_lifespan: 1m

      - client_id: jellyfin
        client_secret_digest: "$argon2id$..."
        require_pkce: true
        response_types: [code]
        redirect_uris:
          - https://jellyfin.${DOMAIN}/sso/OID/redirect/authelia
        scopes: [openid, profile, email, groups]
        grant_types: [authorization_code]
        access_token_lifespan: 15m
        authorize_code_lifespan: 1m
```

### Important: OIDC redirect URIs must use literal domain

Authelia's `configuration.yml` does not expand `${DOMAIN}` in the OIDC clients block at runtime. Redirect URIs must be hardcoded with the real domain (e.g. `https://forgejo.example.com/...`). Use `make edit` on `traefik/.env.sops` to update if domain changes.

### New secrets in traefik/.env.sops

```
AUTHELIA_OIDC_ISSUER_KEY=...    # EC P-384 private key (PEM)
KOMODO_OIDC_SECRET=...          # raw secret — hash before putting in config
FORGEJO_OIDC_SECRET=...
JELLYFIN_OIDC_SECRET=...
```

Generate issuer key:
```bash
openssl ecparam -name secp384r1 -genkey -noout -out oidc.key
```

Hash client secrets for config:
```bash
docker run --rm authelia/authelia:4.38 \
  authelia crypto hash generate argon2 --password 'your-client-secret'
```

---

## Network Topology

```
Internet → Traefik (:80/:443)
              │
              └── proxy network (external, shared)
                    ├── traefik
                    ├── authelia
                    ├── komodo-core
                    ├── vaultwarden (app only)
                    ├── forgejo    (app only)
                    ├── jellyfin
                    └── sites/*

Per-stack internal networks (isolated, internal: true):
  vaultwarden-internal:  vaultwarden ↔ postgres
  forgejo-internal:      forgejo ↔ postgres
  komodo-internal:       core ↔ periphery
  komodo-socket:         periphery ↔ socket-proxy
  backup-socket:         backup ↔ socket-proxy
```

---

## Planned Services

| Stack | Auth | DB | Backup |
|-------|------|----|--------|
| traefik | — | — | — |
| komodo | Authelia OIDC | — | — |
| backup | Authelia forward auth | — | — |
| vaultwarden | Authelia forward auth | Postgres | pg_dump via exec-pre |
| jellyfin | Authelia OIDC* | Internal (LevelDB) | stop-during-backup |
| forgejo | Authelia OIDC | Postgres | pg_dump via exec-pre |
| sites/* | None (public) or forward auth | — | — |

*Jellyfin OIDC requires the [SSO Plugin](https://github.com/9p4/jellyfin-plugin-sso) installed manually via Jellyfin plugin catalog before OIDC can be configured.

---

## Security Hardening Summary

| Control | Detail |
|---------|--------|
| Capabilities | `cap_drop: ALL` everywhere; only `NET_BIND_SERVICE` added back for Traefik |
| Filesystem | `read_only: true`; writable paths via explicit `tmpfs` |
| Privilege escalation | `no-new-privileges: true` on all containers |
| User | Non-root `user: UID:GID` on all containers |
| Resource limits | CPU, memory, PID limits on every service |
| Log rotation | `json-file`, `max-size: 10m`, `max-file: 3` |
| Docker socket | Proxied via docker-socket-proxy on isolated `internal` networks; scoped permissions per consumer |
| Config mounts | All bind-mounts are `:ro` |
| Secrets at rest | Encrypted with age (ChaCha20-Poly1305) in `.env.sops` |
| Secrets at runtime | Decrypted to `/dev/shm` (RAM only), mounted `:ro` via `_FILE` env vars |
| DB isolation | Postgres containers on internal network only, no published ports |
| OIDC | Hashed client secrets (argon2id), PKCE enforced, short token lifetimes, code flow only |
| Image pinning | No `latest` tags — all images pinned to exact versions |

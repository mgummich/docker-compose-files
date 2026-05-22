# Docker Compose Files — Agent Rules

This repo contains production-hardened Docker Compose stacks for a self-hosted VPS setup.

**Reverse proxy:** Traefik v3 + Authelia (SSO + TOTP + OIDC)  
**Orchestration:** Komodo (calls `make up` per stack)  
**Backups:** docker-volume-backup → SFTP  
**Secrets:** sops + age (encrypted `.env.sops` files, decrypted to `/dev/shm` at runtime)  
**Design spec:** `docs/superpowers/specs/2026-05-22-selfhosted-docker-setup-design.md`

---

## Adding a New Service

Always copy `_template/` as the starting point:

```bash
cp -r _template/ myservice/
```

Each stack is self-contained: `compose.yaml`, `Makefile`, `.env.example`, `.env.sops`, `README.md`.

---

## Mandatory Compose Hardening

Every container in every stack **must** have all of the following:

```yaml
restart: unless-stopped
user: "UID:GID"                        # never run as root; find correct UID in image docs
read_only: true
security_opt:
  - no-new-privileges=true
cap_drop:
  - ALL
# cap_add only if required (e.g. NET_BIND_SERVICE for port < 1024)
tmpfs:
  - /tmp:size=64m,mode=1777            # adjust path/size to what the app needs
healthcheck:
  test: [...]                          # use wget/curl against health endpoint
  interval: 30s
  timeout: 5s
  start_period: 15s
  retries: 3
logging:
  driver: json-file
  options:
    max-size: "10m"
    max-file: "3"
deploy:
  resources:
    limits:
      cpus: "0.5"                      # tune per service
      memory: 256M
      pids: 100
    reservations:
      cpus: "0.05"
      memory: 64M
```

**Never use `latest` image tags.** Pin to exact versions (e.g. `image: vaultwarden/server:1.32.0`).

**Never add a `version:` field** to compose.yaml — it is deprecated and ignored by modern Compose.

---

## Network Rules

```yaml
networks:
  proxy:
    external: true        # shared Traefik network — only web-facing containers
  internal:
    internal: true        # per-stack isolation — backends, DBs, socket proxies
```

- **Only the web-facing container** joins the `proxy` network
- **DB and backend containers** join `internal` only — never `proxy`
- **Never publish ports** on DB containers (`ports:` must not appear on postgres/redis/etc.)
- Single-container stacks (static sites) only need `proxy`
- Every multi-container stack needs its own `internal` network
- **Never use `ports:` on any container** — Traefik routes via Docker network. `ports:` bypasses Traefik and Authelia entirely

---

## Secrets Pattern

Secrets are decrypted from `.env.sops` to `/dev/shm/STACKNAME-secrets/` at runtime by `make up`. They are bind-mounted read-only into containers and consumed via `_FILE` environment variables.

```yaml
volumes:
  - /dev/shm/myservice-secrets/my_secret:/run/secrets/my_secret:ro
environment:
  - MY_SECRET_FILE=/run/secrets/my_secret
```

**Never pass secrets as plain environment variables.** `docker inspect` must show only the file path, not the value.

Update `Makefile` `secrets-init` target with a `sops --decrypt --extract` line per secret.

---

## Traefik Labels

```yaml
# Standard protected service (Authelia SSO required)
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.SVCNAME.rule=Host(`svc.${DOMAIN}`)"
  - "traefik.http.routers.SVCNAME.entrypoints=websecure"
  - "traefik.http.routers.SVCNAME.tls.certresolver=cloudflare"
  - "traefik.http.routers.SVCNAME.middlewares=authelia@docker,chain-default@file"
  # Only add if non-standard port:
  - "traefik.http.services.SVCNAME.loadbalancer.server.port=8080"

# Admin UI (stricter security headers + tighter rate limit)
  - "traefik.http.routers.SVCNAME.middlewares=authelia@docker,chain-admin@file"

# Public endpoint (no auth — use sparingly)
  - "traefik.http.routers.SVCNAME.middlewares=chain-default@file"
```

Replace `SVCNAME` with a unique lowercase slug. The container must be on the `proxy` network.

---

## Authentication Decision

| Service type | Auth method |
|---|---|
| Admin UI (Komodo, Traefik dashboard) | `authelia@docker` + `chain-admin@file` + OIDC |
| App with OIDC support (Forgejo, Jellyfin) | `authelia@docker` + `chain-default@file` + OIDC |
| App without OIDC (Vaultwarden) | `authelia@docker` + `chain-default@file` (forward auth only) |
| Public site / API | `chain-default@file` only (no authelia middleware) |

---

## Docker Socket Access

**Never mount `/var/run/docker.sock` directly.** Always use a socket proxy:

```yaml
socket-proxy:
  image: tecnativa/docker-socket-proxy:0.3
  restart: unless-stopped
  networks: [socket]
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock:ro
  environment:
    # Grant only what the consumer needs — everything else defaults to 0 (deny)
    CONTAINERS: 1
    INFO: 1
    # Add NETWORKS, VOLUMES, POST, DELETE, EXEC, AUTH only if required
  cap_drop: [ALL]
  read_only: true
  security_opt: [no-new-privileges=true]
  tmpfs: [/tmp:size=16m,mode=1777]
  healthcheck:
    test: ["CMD", "wget", "-qO-", "http://localhost:2375/version"]
    interval: 30s
    timeout: 5s
    start_period: 10s
    retries: 3
  deploy:
    resources:
      limits:
        cpus: "0.1"
        memory: 32M
        pids: 50
      reservations:
        cpus: "0.02"
        memory: 16M
```

The consumer connects via `DOCKER_HOST: tcp://socket-proxy:2375` on an `internal` network named `socket`.

**Permission levels by consumer:**
- Traefik: `CONTAINERS NETWORKS SERVICES TASKS INFO` (read-only)
- Backup: `CONTAINERS INFO EXEC POST` (exec for pg_dump)
- Komodo periphery: `CONTAINERS NETWORKS VOLUMES IMAGES INFO PING POST DELETE AUTH EXEC` (full management)

---

## Backup Labels

All stateful services must label their volumes for the central backup stack to discover.

**Postgres (pg_dump before backup):**
```yaml
labels:
  - "docker-volume-backup.exec-label=SVCNAME-db"
  # Hardcode user and DB name — $POSTGRES_USER is not set when using POSTGRES_USER_FILE
  - "docker-volume-backup.exec-pre=pg_dump -U SVCNAME SVCNAME > /var/lib/postgresql/data/dump.sql"
```

**Stop-during-backup (e.g. Jellyfin):**
```yaml
labels:
  - "docker-volume-backup.stop-during-backup=true"
```

**Exclude large/regenerable volumes (media, caches):**
```yaml
labels:
  - "docker-volume-backup.exclude=true"
```

---

## Makefile Pattern

The `Makefile` must have these targets with this exact behavior:

| Target | Behavior |
|---|---|
| `make up` | Runs `secrets-init`, then `docker compose up -d` |
| `make down` | Runs `docker compose down`, then `secrets-clean` |
| `make secrets-init` | Creates `/dev/shm/STACKNAME-secrets/`, decrypts each secret via sops, `chmod 600` |
| `make secrets-clean` | `rm -rf /dev/shm/STACKNAME-secrets/` |
| `make encrypt` | `sops --encrypt .env > .env.sops && rm -f .env` |
| `make decrypt` | `sops --decrypt .env.sops > .env` (with warning) |
| `make edit` | `sops .env.sops` |
| `make logs` | `docker compose logs -f` |
| `make ps` | `docker compose ps` |

---

## Postgres Pattern

When a service needs a database, give it a dedicated Postgres container in the same stack. Never share Postgres between stacks.

Current stable version: `postgres:17.10-alpine` — check https://hub.docker.com/_/postgres/tags for updates.

```yaml
services:
  db:
    image: postgres:17.10-alpine
    user: "70:70"                       # postgres uid in alpine image
    networks: [internal]               # never proxy
    volumes:
      - db_data:/var/lib/postgresql/data
      - /dev/shm/SVCNAME-secrets/db_password:/run/secrets/db_password:ro
    environment:
      - POSTGRES_USER=SVCNAME           # not a secret — hardcode it
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_password
      - POSTGRES_DB=SVCNAME             # not a secret — hardcode it
    cap_drop: [ALL]
    read_only: true
    tmpfs:
      - /tmp:size=64m,mode=1777
      - /var/run/postgresql:size=16m,mode=0755   # required for read_only: true
    security_opt: [no-new-privileges=true]
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "SVCNAME", "-d", "SVCNAME"]
      interval: 30s
      timeout: 5s
      start_period: 15s
      retries: 3
    labels:
      - "docker-volume-backup.exec-label=SVCNAME-db"
      # Hardcode user/db — $$POSTGRES_USER expands empty when using POSTGRES_USER_FILE
      - "docker-volume-backup.exec-pre=pg_dump -U SVCNAME SVCNAME > /var/lib/postgresql/data/dump.sql"
```

> **`depends_on` rule:** Any service that depends on Postgres must use `condition: service_healthy`:
> ```yaml
> depends_on:
>   db:
>     condition: service_healthy
> ```

---

## OIDC Client Registration

When adding a service that supports OIDC:

1. Add a client block to `traefik/authelia/configuration.yml` under `identity_providers.oidc.clients`
2. Use `client_secret_digest` (argon2id hash) — never plaintext `client_secret`
3. Set `require_pkce: true`, `response_types: [code]`, `grant_types: [authorization_code]`
4. Redirect URIs must be **hardcoded** with the real domain — `${DOMAIN}` is not expanded in this file
5. Add the raw client secret to `traefik/.env.sops` via `make edit`
6. Hash the secret: `docker run --rm authelia/authelia:4.38 authelia crypto hash generate argon2 --password 'secret'`

---

## What Never to Do

- `image: someimage:latest` — always pin versions
- Mount `/var/run/docker.sock` directly — always use socket-proxy
- Put DB/backend containers on the `proxy` network
- Define `ports:` on DB containers
- Pass secrets as plain env vars — use `_FILE` pattern
- Run containers as root — always set `user:`
- Skip `cap_drop: ALL` — drop all, add back only what's needed
- Skip healthchecks — every container needs one
- Commit `.env` — only `.env.sops` goes to git
- Share Postgres between stacks
- Put plaintext client secrets in Authelia OIDC config
- Add `version:` field to compose.yaml — deprecated, ignored, don't write it
- Use `ports:` on any container — Traefik routes via Docker network, `ports:` bypasses Authelia
- Use `depends_on:` without `condition: service_healthy` — bare depends_on only waits for container start, not readiness
- Omit `/var/run/postgresql` tmpfs on Postgres containers — required when `read_only: true`

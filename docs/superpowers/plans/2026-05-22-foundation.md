# Self-Hosted Docker — Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create `_template/` stack, upgrade Authelia with OIDC, create Komodo orchestrator stack, and create central backup stack.

**Architecture:** Each stack is an independent directory following `_template/`. Traefik+Authelia already runs. Komodo orchestrates by calling `make up` per stack. Backups run daily at 3am via docker-volume-backup over SFTP. OIDC secrets flow: raw secret stored per-service, argon2id hash stored in Authelia config.

**Tech Stack:** Docker Compose, Traefik v3.3 (existing), Authelia 4.38 (existing), Komodo 1.17.0, docker-volume-backup v2.43.0, MongoDB 7.0, sops+age, tecnativa/docker-socket-proxy 0.3

---

## File Map

**Create:**
- `_template/compose.yaml`, `_template/Makefile`, `_template/.env.example`, `_template/README.md`
- `komodo/compose.yaml`, `komodo/Makefile`, `komodo/.env.example`, `komodo/README.md`
- `backup/compose.yaml`, `backup/Makefile`, `backup/.env.example`, `backup/README.md`

**Modify:**
- `traefik/authelia/configuration.yml` — add `identity_providers.oidc` block
- `traefik/compose.yaml` — add OIDC issuer key secret mount to authelia service
- `traefik/Makefile` — add OIDC issuer key to `secrets-init`
- `traefik/.env.example` — document new OIDC secrets

---

## Task 1: Create `_template/` Stack

**Files:**
- Create: `_template/compose.yaml`
- Create: `_template/Makefile`
- Create: `_template/.env.example`
- Create: `_template/README.md`

- [ ] **Step 1: Create `_template/compose.yaml`**

```yaml
name: SERVICE_NAME

services:
  app:
    image: example/app:1.0.0        # replace — never use latest
    restart: unless-stopped
    user: "1000:1000"               # verify correct UID in image docs
    networks:
      - proxy
      # Add internal network for multi-container stacks:
      # - internal
    # volumes:
    #   - app_data:/data
    #   - /dev/shm/SERVICE_NAME-secrets/secret:/run/secrets/secret:ro
    environment:
      - DOMAIN=${DOMAIN}
      # - SECRET_FILE=/run/secrets/secret
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.SERVICE_NAME.rule=Host(`SERVICE_NAME.${DOMAIN}`)"
      - "traefik.http.routers.SERVICE_NAME.entrypoints=websecure"
      - "traefik.http.routers.SERVICE_NAME.tls.certresolver=cloudflare"
      - "traefik.http.routers.SERVICE_NAME.middlewares=authelia@docker,chain-default@file"
      # Uncomment if non-standard port:
      # - "traefik.http.services.SERVICE_NAME.loadbalancer.server.port=8080"
    cap_drop:
      - ALL
    read_only: true
    tmpfs:
      - /tmp:size=64m,mode=1777
    security_opt:
      - no-new-privileges=true
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8080/health"]
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
          cpus: "0.5"
          memory: 256M
          pids: 100
        reservations:
          cpus: "0.05"
          memory: 64M

networks:
  proxy:
    external: true
  # internal:
  #   internal: true

# volumes:
#   app_data:
```

- [ ] **Step 2: Create `_template/Makefile`**

```makefile
STACK_NAME  := SERVICE_NAME
SECRETS_DIR := /dev/shm/$(STACK_NAME)-secrets
SOPS_FILE   := .env.sops
COMPOSE     := docker compose -f compose.yaml

.PHONY: secrets-init secrets-clean up down logs ps encrypt decrypt edit

secrets-init:
	@mkdir -p $(SECRETS_DIR) && chmod 700 $(SECRETS_DIR)
	# Add one line per secret:
	# @sops --decrypt --extract '["MY_SECRET"]' $(SOPS_FILE) > $(SECRETS_DIR)/my_secret
	@chmod 600 $(SECRETS_DIR)/* 2>/dev/null || true
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
	@echo "Encrypted. Commit $(SOPS_FILE), keep .env out of git."

decrypt:
	sops --decrypt $(SOPS_FILE) > .env
	@echo "WARNING: .env is plaintext on disk. Delete after use: rm .env"

edit:
	sops $(SOPS_FILE)
```

- [ ] **Step 3: Create `_template/.env.example`**

```bash
# Root domain — must match traefik stack
DOMAIN=example.com

# Add stack-specific variables below
# MY_SECRET=changeme
```

- [ ] **Step 4: Create `_template/README.md`**

```markdown
# SERVICE_NAME

One-line description.

## Deploy

```bash
cp .env.example .env
$EDITOR .env
make encrypt
make up
```

## Operations

| Command | Action |
|---------|--------|
| `make up` | Decrypt secrets to RAM, start stack |
| `make down` | Stop stack, wipe secrets |
| `make logs` | Tail logs |
| `make ps` | Show status |
| `make edit` | Edit secrets in-place (re-encrypts on save) |
```
```

- [ ] **Step 5: Validate template syntax**

```bash
cd _template
DOMAIN=example.com docker compose -f compose.yaml config
```

Expected: YAML output, no errors. Warning about `example/app:1.0.0` not resolving is fine — it's a template.

- [ ] **Step 6: Commit**

```bash
git add _template/
git commit -m "feat: add stack template"
```

---

## Task 2: Update Authelia for OIDC

**Files:**
- Modify: `traefik/authelia/configuration.yml`
- Modify: `traefik/compose.yaml`
- Modify: `traefik/Makefile`
- Modify: `traefik/.env.example`

- [ ] **Step 1: Generate OIDC issuer private key (EC P-384)**

```bash
openssl ecparam -name secp384r1 -genkey -noout -out /tmp/oidc.key
cat /tmp/oidc.key
# Copy full PEM output — store in password manager
# Format for .env.sops: replace actual newlines with literal \n
rm /tmp/oidc.key
```

- [ ] **Step 2: Generate raw OIDC client secrets**

```bash
# Run three times — one per service
openssl rand -hex 32   # → KOMODO_OIDC_CLIENT_SECRET
openssl rand -hex 32   # → FORGEJO_OIDC_CLIENT_SECRET
openssl rand -hex 32   # → JELLYFIN_OIDC_CLIENT_SECRET
# Store each raw value in password manager
```

- [ ] **Step 3: Hash each client secret with argon2id**

```bash
# Run once per secret — replace 'RAW_SECRET' with each value from Step 2
docker run --rm authelia/authelia:4.38 \
  authelia crypto hash generate argon2 --password 'RAW_SECRET'
# Output: $argon2id$v=19$m=65536,t=3,p=4$...
# Save each hash — you'll paste it into configuration.yml
```

- [ ] **Step 4: Add all OIDC secrets to `traefik/.env.sops`**

```bash
cd traefik
make edit
```

Add these keys (replace values with real ones):

```bash
AUTHELIA_OIDC_ISSUER_KEY="-----BEGIN EC PRIVATE KEY-----\nMHQCAQEEI...\n-----END EC PRIVATE KEY-----"
```

- [ ] **Step 5: Add OIDC issuer key secret mount to authelia in `traefik/compose.yaml`**

In the `authelia` service `volumes:` block, add:

```yaml
      - /dev/shm/traefik-secrets/authelia_oidc_issuer_key:/run/secrets/authelia_oidc_issuer_key:ro
```

- [ ] **Step 6: Add OIDC secret extract to `traefik/Makefile` `secrets-init` target**

After the existing `authelia_storage_key` line, add:

```makefile
	@sops --decrypt --extract '["AUTHELIA_OIDC_ISSUER_KEY"]' $(SOPS_FILE) > $(SECRETS_DIR)/authelia_oidc_issuer_key
```

- [ ] **Step 7: Append OIDC block to `traefik/authelia/configuration.yml`**

Replace `YOUR_DOMAIN` with real domain. Replace each `REPLACE_WITH_*_HASH` with the argon2id hash from Step 3.

```yaml
identity_providers:
  oidc:
    issuer_private_key_path: /run/secrets/authelia_oidc_issuer_key
    access_token_lifespan: 15m
    authorize_code_lifespan: 1m
    refresh_token_lifespan: 90m
    clients:
      - client_id: komodo
        client_secret_digest: "REPLACE_WITH_KOMODO_HASH"
        require_pkce: true
        response_types:
          - code
        redirect_uris:
          - https://komodo.YOUR_DOMAIN/auth/callback
        scopes:
          - openid
          - profile
          - email
          - groups
        grant_types:
          - authorization_code
        access_token_lifespan: 15m
        authorize_code_lifespan: 1m

      - client_id: forgejo
        client_secret_digest: "REPLACE_WITH_FORGEJO_HASH"
        require_pkce: true
        response_types:
          - code
        redirect_uris:
          - https://forgejo.YOUR_DOMAIN/user/oauth2/authelia/callback
        scopes:
          - openid
          - profile
          - email
        grant_types:
          - authorization_code
        access_token_lifespan: 15m
        authorize_code_lifespan: 1m

      - client_id: jellyfin
        client_secret_digest: "REPLACE_WITH_JELLYFIN_HASH"
        require_pkce: true
        response_types:
          - code
        redirect_uris:
          - https://jellyfin.YOUR_DOMAIN/sso/OID/redirect/authelia
        scopes:
          - openid
          - profile
          - email
          - groups
        grant_types:
          - authorization_code
        access_token_lifespan: 15m
        authorize_code_lifespan: 1m
```

- [ ] **Step 8: Update `traefik/.env.example` to document new secret**

Add at end:

```bash
# OIDC issuer private key (EC P-384 PEM, newlines as \n)
# Generate: openssl ecparam -name secp384r1 -genkey -noout | cat
AUTHELIA_OIDC_ISSUER_KEY="-----BEGIN EC PRIVATE KEY-----\n...\n-----END EC PRIVATE KEY-----"
```

- [ ] **Step 9: Validate traefik compose**

```bash
cd traefik
DOMAIN=example.com ACME_EMAIL=test@example.com docker compose -f compose.yaml config
```

Expected: YAML output, no errors, authelia service shows new volume mount.

- [ ] **Step 10: Restart traefik stack**

```bash
cd traefik
make down && make up
make ps
```

Expected: socket-proxy, traefik, authelia all show `healthy`. Authelia logs show OIDC provider initialized:

```bash
make logs | grep -i oidc
```

- [ ] **Step 11: Commit**

```bash
git add traefik/authelia/configuration.yml traefik/compose.yaml traefik/Makefile traefik/.env.example
git commit -m "feat: add Authelia OIDC provider (Komodo, Forgejo, Jellyfin clients)"
```

---

## Task 3: Create `komodo/` Stack

**Files:**
- Create: `komodo/compose.yaml`
- Create: `komodo/Makefile`
- Create: `komodo/.env.example`
- Create: `komodo/README.md`

> **Note:** Verify Komodo image versions at https://github.com/moghtech/komodo/releases before running `make up`. Core listens on port 9120, periphery on 8120 by default — confirm in release notes.

- [ ] **Step 1: Create `komodo/compose.yaml`**

```yaml
name: komodo

services:
  socket-proxy:
    image: tecnativa/docker-socket-proxy:0.3
    restart: unless-stopped
    networks:
      - socket
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    environment:
      CONTAINERS: 1
      NETWORKS: 1
      VOLUMES: 1
      IMAGES: 1
      INFO: 1
      PING: 1
      POST: 1
      DELETE: 1
      AUTH: 1
      EXEC: 1
    cap_drop:
      - ALL
    read_only: true
    security_opt:
      - no-new-privileges=true
    tmpfs:
      - /tmp:size=16m,mode=1777
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:2375/version"]
      interval: 30s
      timeout: 5s
      start_period: 10s
      retries: 3
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
    deploy:
      resources:
        limits:
          cpus: "0.1"
          memory: 32M
          pids: 50
        reservations:
          cpus: "0.02"
          memory: 16M

  mongo:
    image: mongo:7.0
    restart: unless-stopped
    networks:
      - internal
    volumes:
      - mongo_data:/data/db
      - /dev/shm/komodo-secrets/mongo_password:/run/secrets/mongo_password:ro
    environment:
      - MONGO_INITDB_ROOT_USERNAME=komodo
      - MONGO_INITDB_ROOT_PASSWORD_FILE=/run/secrets/mongo_password
    command: ["--quiet", "--auth"]
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges=true
    tmpfs:
      - /tmp:size=64m,mode=1777
      - /var/run/mongodb:size=16m,mode=0755
    healthcheck:
      test: ["CMD", "mongosh", "--quiet", "--eval", "db.adminCommand('ping').ok"]
      interval: 30s
      timeout: 10s
      start_period: 30s
      retries: 5
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 512M
          pids: 200
        reservations:
          cpus: "0.05"
          memory: 128M

  periphery:
    image: ghcr.io/moghtech/komodo-periphery:1.17.0
    restart: unless-stopped
    networks:
      - socket
      - internal
    volumes:
      - /dev/shm/komodo-secrets/periphery_passkey:/run/secrets/periphery_passkey:ro
      - /home/deploy/docker-compose-files:/stacks:ro
    environment:
      - PERIPHERY_PASSKEYS_FILE=/run/secrets/periphery_passkey
      - DOCKER_HOST=tcp://socket-proxy:2375
    cap_drop:
      - ALL
    read_only: true
    tmpfs:
      - /tmp:size=64m,mode=1777
    security_opt:
      - no-new-privileges=true
    depends_on:
      socket-proxy:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8120/health"]
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
          cpus: "0.5"
          memory: 128M
          pids: 100
        reservations:
          cpus: "0.05"
          memory: 32M

  core:
    image: ghcr.io/moghtech/komodo-core:1.17.0
    restart: unless-stopped
    networks:
      - proxy
      - internal
    volumes:
      - /dev/shm/komodo-secrets/jwt_secret:/run/secrets/jwt_secret:ro
      - /dev/shm/komodo-secrets/passkey:/run/secrets/passkey:ro
      - /dev/shm/komodo-secrets/mongo_password:/run/secrets/mongo_password:ro
      - /dev/shm/komodo-secrets/oidc_client_secret:/run/secrets/oidc_client_secret:ro
    environment:
      - KOMODO_HOST=https://komodo.${DOMAIN}
      - KOMODO_JWT_SECRET_FILE=/run/secrets/jwt_secret
      - KOMODO_PASSKEYS_FILE=/run/secrets/passkey
      - KOMODO_DB_URL=${KOMODO_DB_URL}
      - KOMODO_OIDC_ENABLED=true
      - KOMODO_OIDC_PROVIDER=https://auth.${DOMAIN}
      - KOMODO_OIDC_CLIENT_ID=komodo
      - KOMODO_OIDC_CLIENT_SECRET_FILE=/run/secrets/oidc_client_secret
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.komodo.rule=Host(`komodo.${DOMAIN}`)"
      - "traefik.http.routers.komodo.entrypoints=websecure"
      - "traefik.http.routers.komodo.tls.certresolver=cloudflare"
      - "traefik.http.routers.komodo.middlewares=authelia@docker,chain-admin@file"
      - "traefik.http.services.komodo.loadbalancer.server.port=9120"
    cap_drop:
      - ALL
    read_only: true
    tmpfs:
      - /tmp:size=64m,mode=1777
    security_opt:
      - no-new-privileges=true
    depends_on:
      mongo:
        condition: service_healthy
      periphery:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:9120/health"]
      interval: 30s
      timeout: 5s
      start_period: 30s
      retries: 3
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
          pids: 200
        reservations:
          cpus: "0.1"
          memory: 128M

networks:
  proxy:
    external: true
  internal:
    internal: true
  socket:
    internal: true

volumes:
  mongo_data:
```

- [ ] **Step 2: Create `komodo/Makefile`**

```makefile
STACK_NAME  := komodo
SECRETS_DIR := /dev/shm/$(STACK_NAME)-secrets
SOPS_FILE   := .env.sops
COMPOSE     := docker compose -f compose.yaml

.PHONY: secrets-init secrets-clean up down logs ps encrypt decrypt edit

secrets-init:
	@mkdir -p $(SECRETS_DIR) && chmod 700 $(SECRETS_DIR)
	@sops --decrypt --extract '["KOMODO_JWT_SECRET"]'          $(SOPS_FILE) > $(SECRETS_DIR)/jwt_secret
	@sops --decrypt --extract '["KOMODO_PASSKEY"]'             $(SOPS_FILE) > $(SECRETS_DIR)/passkey
	@sops --decrypt --extract '["PERIPHERY_PASSKEY"]'          $(SOPS_FILE) > $(SECRETS_DIR)/periphery_passkey
	@sops --decrypt --extract '["MONGO_PASSWORD"]'             $(SOPS_FILE) > $(SECRETS_DIR)/mongo_password
	@sops --decrypt --extract '["KOMODO_OIDC_CLIENT_SECRET"]'  $(SOPS_FILE) > $(SECRETS_DIR)/oidc_client_secret
	@chmod 600 $(SECRETS_DIR)/*
	@echo "Secrets written to $(SECRETS_DIR) (RAM only)"

secrets-clean:
	@rm -rf $(SECRETS_DIR)
	@echo "Secrets wiped from $(SECRETS_DIR)"

up: secrets-init
	@DOMAIN=$$(sops --decrypt --extract '["DOMAIN"]' $(SOPS_FILE)) \
	 KOMODO_DB_URL="mongodb://komodo:$$(cat $(SECRETS_DIR)/mongo_password)@mongo:27017/komodo" \
	 $(COMPOSE) up -d

down:
	$(COMPOSE) down
	$(MAKE) secrets-clean

logs: ; $(COMPOSE) logs -f
ps:   ; $(COMPOSE) ps

encrypt:
	sops --encrypt .env > $(SOPS_FILE)
	rm -f .env
	@echo "Encrypted. Commit $(SOPS_FILE), keep .env out of git."

decrypt:
	sops --decrypt $(SOPS_FILE) > .env
	@echo "WARNING: .env is plaintext. Delete after use: rm .env"

edit:
	sops $(SOPS_FILE)
```

- [ ] **Step 3: Create `komodo/.env.example`**

```bash
DOMAIN=example.com

# Generate each with: openssl rand -hex 32
KOMODO_JWT_SECRET=changeme
KOMODO_PASSKEY=changeme           # core uses this to authenticate with periphery
PERIPHERY_PASSKEY=changeme        # must match KOMODO_PASSKEY
MONGO_PASSWORD=changeme

# Raw OIDC client secret from foundation Task 2 Step 2 (not the hash)
KOMODO_OIDC_CLIENT_SECRET=changeme
```

- [ ] **Step 4: Create `komodo/README.md`**

```markdown
# komodo

Container orchestrator — manages all other stacks by calling `make up` per stack.

## Deploy

```bash
cp .env.example .env
$EDITOR .env
make encrypt
make up
```

## Post-deploy: add server in Komodo UI

1. Open https://komodo.YOUR_DOMAIN
2. Login via Authelia (OIDC)
3. Go to Servers → Add Server
4. Address: `http://periphery:8120`
5. Passkey: value of `KOMODO_PASSKEY` from .env

## Adding a stack to Komodo

In Komodo UI → Stacks → New Stack:
- Run directory: `/stacks/SERVICE_NAME`
- Deploy command: `make up`
- Destroy command: `make down`
```
```

- [ ] **Step 5: Validate komodo compose**

```bash
cd komodo
DOMAIN=example.com docker compose -f compose.yaml config
```

Expected: YAML output, no errors.

- [ ] **Step 6: Commit**

```bash
git add komodo/
git commit -m "feat: add Komodo orchestrator stack"
```

---

## Task 4: Create `backup/` Stack

**Files:**
- Create: `backup/compose.yaml`
- Create: `backup/Makefile`
- Create: `backup/.env.example`
- Create: `backup/README.md`

- [ ] **Step 1: Generate SSH key for backup server**

```bash
ssh-keygen -t ed25519 -f /tmp/backup_key -N ""
cat /tmp/backup_key        # private key — store in password manager
cat /tmp/backup_key.pub    # add to ~/.ssh/authorized_keys on backup server
rm /tmp/backup_key /tmp/backup_key.pub
```

- [ ] **Step 2: Create `backup/compose.yaml`**

```yaml
name: backup

services:
  socket-proxy:
    image: tecnativa/docker-socket-proxy:0.3
    restart: unless-stopped
    networks:
      - socket
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    environment:
      CONTAINERS: 1
      INFO: 1
      EXEC: 1
      POST: 1
    cap_drop:
      - ALL
    read_only: true
    security_opt:
      - no-new-privileges=true
    tmpfs:
      - /tmp:size=16m,mode=1777
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:2375/version"]
      interval: 30s
      timeout: 5s
      start_period: 10s
      retries: 3
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
    deploy:
      resources:
        limits:
          cpus: "0.1"
          memory: 32M
          pids: 50
        reservations:
          cpus: "0.02"
          memory: 16M

  backup:
    image: offen/docker-volume-backup:v2.43.0
    restart: unless-stopped
    networks:
      - socket
    volumes:
      - /dev/shm/backup-secrets/ssh_key:/run/secrets/ssh_key:ro
    environment:
      - DOCKER_HOST=tcp://socket-proxy:2375
      - BACKUP_CRON_EXPRESSION=0 3 * * *
      - BACKUP_RETENTION_DAYS=14
      - BACKUP_FILENAME=backup-%Y-%m-%dT%H-%M-%S.tar.gz
      - SSH_HOST_NAME=${SSH_HOST}
      - SSH_PORT=22
      - SSH_USER=${SSH_USER}
      - SSH_PRIVATE_KEY_FILE=/run/secrets/ssh_key
      - SSH_REMOTE_PATH=${SSH_REMOTE_PATH}
      - BACKUP_STOP_CONTAINER_LABEL=docker-volume-backup.stop-during-backup
    cap_drop:
      - ALL
    read_only: true
    tmpfs:
      - /tmp:size=256m,mode=1777
    security_opt:
      - no-new-privileges=true
    depends_on:
      socket-proxy:
        condition: service_healthy
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 512M
          pids: 100
        reservations:
          cpus: "0.05"
          memory: 64M

networks:
  socket:
    internal: true
```

- [ ] **Step 3: Create `backup/Makefile`**

```makefile
STACK_NAME  := backup
SECRETS_DIR := /dev/shm/$(STACK_NAME)-secrets
SOPS_FILE   := .env.sops
COMPOSE     := docker compose -f compose.yaml

.PHONY: secrets-init secrets-clean up down logs ps encrypt decrypt edit

secrets-init:
	@mkdir -p $(SECRETS_DIR) && chmod 700 $(SECRETS_DIR)
	@sops --decrypt --extract '["SSH_PRIVATE_KEY"]' $(SOPS_FILE) > $(SECRETS_DIR)/ssh_key
	@chmod 600 $(SECRETS_DIR)/*
	@echo "Secrets written to $(SECRETS_DIR) (RAM only)"

secrets-clean:
	@rm -rf $(SECRETS_DIR)
	@echo "Secrets wiped from $(SECRETS_DIR)"

up: secrets-init
	@SSH_HOST=$$(sops --decrypt --extract '["SSH_HOST"]' $(SOPS_FILE)) \
	 SSH_USER=$$(sops --decrypt --extract '["SSH_USER"]' $(SOPS_FILE)) \
	 SSH_REMOTE_PATH=$$(sops --decrypt --extract '["SSH_REMOTE_PATH"]' $(SOPS_FILE)) \
	 $(COMPOSE) up -d

down:
	$(COMPOSE) down
	$(MAKE) secrets-clean

logs: ; $(COMPOSE) logs -f
ps:   ; $(COMPOSE) ps

encrypt:
	sops --encrypt .env > $(SOPS_FILE)
	rm -f .env
	@echo "Encrypted. Commit $(SOPS_FILE), keep .env out of git."

decrypt:
	sops --decrypt $(SOPS_FILE) > .env
	@echo "WARNING: .env is plaintext. Delete after use: rm .env"

edit:
	sops $(SOPS_FILE)
```

- [ ] **Step 4: Create `backup/.env.example`**

```bash
# Remote SFTP backup server
SSH_HOST=backup.example.com
SSH_USER=backup
SSH_REMOTE_PATH=/backups/vps

# SSH private key (ED25519 PEM, newlines as \n)
# Generate: ssh-keygen -t ed25519 -f backup_key -N ""
# Add backup_key.pub to ~/.ssh/authorized_keys on backup server
SSH_PRIVATE_KEY="-----BEGIN OPENSSH PRIVATE KEY-----\n...\n-----END OPENSSH PRIVATE KEY-----"
```

- [ ] **Step 5: Create `backup/README.md`**

```markdown
# backup

Daily backups of all stateful service volumes to SFTP at 3am. Uses docker-volume-backup.

## Deploy

```bash
# Add backup server's authorized_keys first (see .env.example)
cp .env.example .env
$EDITOR .env
make encrypt
make up
```

## How services opt in

Services must label their containers for backup discovery.

**Postgres (pg_dump before backup):**
```yaml
labels:
  - "docker-volume-backup.exec-label=SVCNAME-db"
  - "docker-volume-backup.exec-pre=pg_dump -U DBUSER DBNAME > /var/lib/postgresql/data/dump.sql"
```

**Stop-during-backup:**
```yaml
labels:
  - "docker-volume-backup.stop-during-backup=true"
```

**Exclude large/regenerable volumes:**
```yaml
labels:
  - "docker-volume-backup.exclude=true"
```

## Manual backup trigger

```bash
docker exec backup_backup_1 backup
```
```
```

- [ ] **Step 6: Validate backup compose**

```bash
cd backup
SSH_HOST=backup.example.com SSH_USER=backup SSH_REMOTE_PATH=/backups \
  docker compose -f compose.yaml config
```

Expected: YAML output, no errors.

- [ ] **Step 7: Commit**

```bash
git add backup/
git commit -m "feat: add central backup stack (docker-volume-backup + SFTP)"
```

# docker-compose-files

Production-hardened Docker Compose stacks for self-hosting. Each stack lives in its own directory with a `Makefile` and its own `README.md`.

Secrets are encrypted at rest using [sops](https://github.com/getsentry/sops) + [age](https://github.com/FiloSottile/age). No plaintext secrets on disk or in git — ever.

---

## Stacks

| Stack | Services | Exposes |
|-------|----------|---------|
| [`traefik/`](traefik/README.md) | Traefik v3, Authelia, docker-socket-proxy | `traefik.<domain>`, `auth.<domain>` |

---

## Repo structure

```
docker-compose-files/
├── .sops.yaml          # age public key — controls who can decrypt secrets
├── .gitignore          # blocks .env from ever being committed
├── README.md           # this file
└── traefik/            # one directory per stack
    ├── README.md       # stack-specific docs
    ├── compose.yaml
    ├── Makefile
    ├── .env.example
    └── .env.sops       # encrypted secrets — safe to commit
```

Each stack follows the same pattern:

1. `.env.example` documents every required variable
2. `.env` (never committed) holds real values
3. `.env.sops` is the encrypted version — the only form that lives in git
4. `make up` decrypts secrets to RAM (`/dev/shm`), mounts them read-only into containers via `_FILE` env vars, then wipes them on `make down`

Secret values never appear as environment variables — `docker inspect` shows only file paths.

---

## Prerequisites

```bash
# macOS
brew install sops age docker

# Debian / Ubuntu
sudo apt install age
# sops: https://github.com/getsentry/sops/releases
```

Minimum versions: `sops >= 3.8`, `age >= 1.1`, `docker >= 24`

---

## One-time machine setup

> Do this once per machine. Your age key is the master key for all secrets in this repo.

### 1. Generate your age key

```bash
mkdir -p ~/.config/sops/age
age-keygen -o ~/.config/sops/age/keys.txt
```

This outputs a public key:

```
Public key: age1abc123def456...
```

**Back up `~/.config/sops/age/keys.txt` to a password manager immediately. Losing it means losing access to all encrypted secrets.**

### 2. Add your public key to `.sops.yaml`

```yaml
creation_rules:
  - path_regex: \.env$
    age: age1abc123def456...   # ← paste your public key here
```

---

## Restoring on a new machine

```bash
# 1. Install prerequisites (see above)

# 2. Restore age key from password manager
mkdir -p ~/.config/sops/age
nano ~/.config/sops/age/keys.txt   # paste key content
chmod 600 ~/.config/sops/age/keys.txt

# 3. Clone repo
git clone <repo-url> && cd docker-compose-files

# 4. Start any stack — no other setup needed
cd traefik && make up
```

---

## Secrets workflow (applies to every stack)

### How secrets flow at runtime

```
make up
  ├─ sops decrypts each secret → /dev/shm/<stack>-secrets/<name>   (RAM, never disk)
  ├─ compose bind-mounts each file → /run/secrets/<name>:ro          (inside container)
  ├─ container reads secret via _FILE env var                         (value never in env)
  └─ docker inspect shows: SECRET_FILE=/run/secrets/…                (path, not value)

make down
  ├─ docker compose down
  └─ rm -rf /dev/shm/<stack>-secrets                                  (secrets wiped)
```

### Commands

| Command | What it does |
|---------|-------------|
| `make up` | Write secrets to `/dev/shm`, start stack |
| `make down` | Stop stack, wipe secrets from RAM |
| `make edit` | Edit secrets in `$EDITOR`, re-encrypt on save |
| `make encrypt` | Encrypt `.env` → `.env.sops`, delete `.env` |
| `make decrypt` | Decrypt `.env.sops` → `.env` — **delete after use** |
| `make logs` | Tail logs |
| `make ps` | Service status |

### First deploy of any stack

```bash
cd <stack>
cp .env.example .env
$EDITOR .env       # fill in values
make encrypt       # → .env.sops, deletes .env
make up
```

### Changing a secret

```bash
cd <stack>
make edit                  # opens .env.sops in $EDITOR, re-encrypts on save
make down && make up       # restart to pick up new values
```

---

## Adding a new stack

```bash
mkdir mystack && cd mystack

# Minimal Makefile (adapt SECRETS_DIR and secret names)
cp ../traefik/Makefile .

# Start with the env template
cp ../traefik/.env.example .env.example
```

The `.sops.yaml` at the repo root automatically covers any `.env` file in subdirectories — no extra config needed.

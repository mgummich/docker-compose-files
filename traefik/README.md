# traefik

Reverse proxy with automatic TLS (Let's Encrypt DNS-01 via Cloudflare) and SSO + 2FA via Authelia. Acts as the entry point for all other stacks.

---

## Architecture

```
Internet
    │
    ▼ :80 → permanent redirect to :443
 Traefik ─── socket-proxy ─── Docker API (read-only, isolated network)
    │
    ├── auth.<domain>     →  Authelia  (login + TOTP)
    ├── traefik.<domain>  →  Dashboard (protected by Authelia)
    └── *.<domain>        →  Any container with Traefik labels
```

**docker-socket-proxy** sits between Traefik and the Docker socket. Traefik never touches the socket directly — the proxy exposes only the 5 read-only endpoints Traefik needs. If Traefik were compromised, the attacker cannot escape to the Docker daemon.

**Authelia** enforces SSO + TOTP on every subdomain. Default policy is `two_factor`. The only bypass is `auth.<domain>` itself.

**Wildcard TLS** — a single `*.<domain>` certificate covers all subdomains via Let's Encrypt DNS-01 challenge. No port 80 exposure required for cert issuance.

---

## Files

| File | Purpose | Edit? |
|------|---------|-------|
| `compose.yaml` | Service definitions, security hardening, resource limits | Rarely |
| `traefik.yml` | Static config: entrypoints, ACME resolver, providers | Rarely |
| `config/dynamic.yml` | Dynamic config: middlewares, static routes (hot-reloaded) | To add routes |
| `authelia/configuration.yml` | Auth policies, session settings, 2FA config | To tune policies |
| `authelia/users_database.yml` | User accounts *(create this — not in git)* | To add/remove users |
| `.env.example` | Documents all required variables | Reference only |
| `.env.sops` | Encrypted secrets | Via `make edit` |
| `Makefile` | Shortcuts for all operations | To add commands |
| `letsencrypt/` | `acme.json` stored here (auto-created by Traefik) | Never |

---

## Prerequisites

**1. Domain with DNS on Cloudflare**

- Nameservers must point to Cloudflare
- Cloudflare API token with `Zone → DNS → Edit` permission:
  - Cloudflare Dashboard → My Profile → API Tokens → Create Token
  - Use "Edit zone DNS" template, scope to your zone

**2. DNS records in Cloudflare**

| Type | Name | Value |
|------|------|-------|
| A | `@` | your server IP |
| CNAME | `*` | `@` |

Proxied or DNS-only both work.

**3. Ports open on the server**

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

---

## Deploy

```bash
cd traefik

# 1. External proxy network (once per host)
docker network create proxy

# 2. Fill in secrets
cp .env.example .env
$EDITOR .env
```

`.env` values:

```bash
DOMAIN=yourdomain.com
ACME_EMAIL=you@yourdomain.com

# Cloudflare API token (Zone → DNS → Edit)
CF_DNS_API_TOKEN=your-cloudflare-token

# Generate each with: openssl rand -hex 32
AUTHELIA_JWT_SECRET=
AUTHELIA_SESSION_SECRET=
AUTHELIA_STORAGE_ENCRYPTION_KEY=
```

```bash
# 4. Encrypt secrets
make encrypt

# 5. Create Authelia user database (not in git — create manually)
cat > authelia/users_database.yml << 'EOF'
users:
  yourname:
    displayname: "Your Name"
    password: ""   # see hash generation below
    email: you@yourdomain.com
    groups:
      - admins
EOF
```

Generate a password hash:

```bash
docker run --rm authelia/authelia:4.38 \
  authelia crypto hash generate argon2 --password 'your-password-here'
```

Paste the output into `users_database.yml` as the `password` value.

```bash
# 6. Start
make up

# 7. Verify
make ps
make logs
```

Traefik requests the wildcard certificate on first start. DNS propagation can take 1–2 minutes:

```bash
make logs | grep -i "certificate\|acme\|error"
```

---

## Protecting other services with Authelia

Add these labels to any container on the `proxy` network:

```yaml
services:
  myapp:
    image: myapp:1.0
    networks:
      - proxy
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.myapp.rule=Host(`myapp.${DOMAIN}`)"
      - "traefik.http.routers.myapp.entrypoints=websecure"
      - "traefik.http.routers.myapp.tls.certresolver=cloudflare"
      - "traefik.http.routers.myapp.middlewares=authelia@docker"

networks:
  proxy:
    external: true
```

To expose a service **without** authentication (public API, etc.):

```yaml
# Omit the middlewares label entirely
- "traefik.http.routers.myapp.entrypoints=websecure"
```

---

## Adding static routes (non-Docker services)

Edit `config/dynamic.yml` — Traefik hot-reloads it, no restart needed. A commented example for routing to a NAS or other LAN service is included in the file.

---

## Security hardening

| Control | Detail |
|---------|--------|
| Capabilities | `cap_drop: ALL` on all containers; only `NET_BIND_SERVICE` added back for Traefik |
| Filesystem | `read_only: true`; writable paths are explicit `tmpfs` mounts |
| Privilege escalation | `no-new-privileges: true` on all containers |
| Resource limits | CPU, memory, and PID limits on every service |
| Log rotation | `json-file` driver, `max-size: 10m`, `max-file: 3` on every service |
| Docker socket | Proxied via docker-socket-proxy on an isolated `internal` network; only 5 read-only endpoints exposed |
| Config files | All config bind-mounts are `:ro` |
| Secrets at rest | Encrypted with age (ChaCha20-Poly1305) in `.env.sops` |
| Secrets at runtime | Decrypted to `/dev/shm` (RAM only), bind-mounted `:ro` into containers, read via `_FILE` env vars — values never visible in `docker inspect` |

---

## Troubleshooting

**Certificate not issuing**
- Verify `CF_DNS_API_TOKEN` has `Zone → DNS → Edit` scope
- Confirm zone nameservers point to Cloudflare
- `make logs | grep -i acme`

**Authelia redirect loop**
- `DOMAIN` in `.env.sops` must match the domain in `authelia/configuration.yml`
- Confirm `auth.<domain>` resolves and Traefik is routing it

**"connection refused" on 80/443**
- Check firewall: `sudo ufw status`
- Confirm proxy network exists: `docker network ls | grep proxy`

**Service not appearing in Traefik**
- Container needs `traefik.enable=true` label
- Container must be on the `proxy` network
- `make logs | grep -i "provider\|error"`

**`make up` fails: `/dev/shm` not found**
- `/dev/shm` is Linux-only. On macOS change `SECRETS_DIR` in `Makefile`:
  ```makefile
  SECRETS_DIR := $(shell [ -d /dev/shm ] && echo /dev/shm/traefik-secrets || echo /tmp/traefik-secrets)
  ```

**Secrets gone after reboot**
- Expected — `/dev/shm` is RAM and clears on reboot. Run `make up` to re-decrypt and restart.

# Installation

TLSentinel ships as a Docker image. The recommended deployment is Docker Compose with a bundled PostgreSQL container.

**Requirements:** Docker 24+ and Docker Compose v2.

---

## Docker Compose (Recommended)

### 1. Create a working directory

```sh
mkdir tlsentinel && cd tlsentinel
```

### 2. Create `docker-compose.yml`

```yaml
services:

  db:
    image: postgres:17-alpine
    environment:
      POSTGRES_DB:       ${TLSENTINEL_DB_NAME:-tlsentinel}
      POSTGRES_USER:     ${TLSENTINEL_DB_USERNAME:-tlsentinel}
      POSTGRES_PASSWORD: ${TLSENTINEL_DB_PASSWORD}
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${TLSENTINEL_DB_USERNAME:-tlsentinel}"]
      interval: 5s
      timeout: 3s
      retries: 10
    restart: unless-stopped

  server:
    image: ghcr.io/tlsentinel/tlsentinel-server:${TLSENTINEL_SERVER_VERSION:-latest}
    environment:
      TLSENTINEL_DB_HOST:     db
      TLSENTINEL_DB_PORT:     5432
      TLSENTINEL_DB_SSLMODE:  disable
      TLSENTINEL_DB_NAME:     ${TLSENTINEL_DB_NAME:-tlsentinel}
      TLSENTINEL_DB_USERNAME: ${TLSENTINEL_DB_USERNAME:-tlsentinel}
      TLSENTINEL_DB_PASSWORD: ${TLSENTINEL_DB_PASSWORD}
      TLSENTINEL_JWT_SECRET:      ${TLSENTINEL_JWT_SECRET}
      TLSENTINEL_ENCRYPTION_KEY:  ${TLSENTINEL_ENCRYPTION_KEY}
      TLSENTINEL_ADMIN_USERNAME:  ${TLSENTINEL_ADMIN_USERNAME:-admin}
      TLSENTINEL_ADMIN_PASSWORD:  ${TLSENTINEL_ADMIN_PASSWORD}
    ports:
      - "8080:8080"
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped

volumes:
  db-data:
```

### 3. Create `.env`

```sh
# Image versions
TLSENTINEL_SERVER_VERSION=latest
TLSENTINEL_SCANNER_VERSION=latest

# Database
TLSENTINEL_DB_NAME=tlsentinel
TLSENTINEL_DB_USERNAME=tlsentinel
TLSENTINEL_DB_PASSWORD=changeme

# Server — generate strong secrets before deploying
TLSENTINEL_JWT_SECRET=        # openssl rand -base64 32  (base64, decodes to ≥32 bytes)
TLSENTINEL_ENCRYPTION_KEY=    # openssl rand -base64 32  (base64, decodes to exactly 32 bytes)

# Bootstrap admin account (only used on first start)
TLSENTINEL_ADMIN_USERNAME=admin
TLSENTINEL_ADMIN_PASSWORD=changeme
```

Generate the two required secrets:

```sh
echo "TLSENTINEL_JWT_SECRET=$(openssl rand -base64 32)"
echo "TLSENTINEL_ENCRYPTION_KEY=$(openssl rand -base64 32)"
```

Paste the output into your `.env`. Both variables are parsed as base64;
`ENCRYPTION_KEY` must decode to exactly 32 bytes, `JWT_SECRET` to at
least 32. `openssl rand -base64 32` satisfies both.

### 4. Start

```sh
docker compose up -d
```

Database migrations run automatically on startup. Once the server is healthy, open `http://localhost:8080` and log in with the admin credentials you set.

!!! warning "Change default passwords"
    Update `TLSENTINEL_ADMIN_PASSWORD` and `TLSENTINEL_DB_PASSWORD` before exposing the service on a network.

---

## Adding a Scanner

Scanners are separate agents that perform the actual TLS checks and report back to the server. At least one scanner is required before endpoints are probed.

1. Log in as an admin and go to **Settings → Scanners**.
2. Create a new scanner — give it a name and copy the generated token.
3. Add a scanner service to your `docker-compose.yml`:

```yaml
  scanner:
    image: ghcr.io/tlsentinel/tlsentinel-scanner:${TLSENTINEL_SCANNER_VERSION:-latest}
    environment:
      TLSENTINEL_API_URL:   ${TLSENTINEL_API_URL}
      TLSENTINEL_API_TOKEN: ${TLSENTINEL_SCANNER_TOKEN}
    depends_on:
      - server
    restart: unless-stopped
```

4. Add the scanner values to `.env`:

```sh
TLSENTINEL_SCANNER_VERSION=latest
TLSENTINEL_API_URL=http://server:8080
TLSENTINEL_SCANNER_TOKEN=stx_s_xxxxxxxxxxxxxxxxxxxx
```

5. Apply the changes:

```sh
docker compose up -d scanner
```

The scanner will appear as **Online** in the Scanners list once it connects.

---

## Binary

Pre-built binaries for Linux, macOS, and Windows are available on the [GitHub Releases](https://github.com/tlsentinel/tlsentinel-server/releases) page. All configuration is via environment variables — see [Configuration](configuration.md) for the full reference.

You are responsible for providing a PostgreSQL 14+ database and setting the required environment variables before running the binary.

---

## First Login

On first startup, if the users table is empty, the server creates an
admin from `TLSENTINEL_ADMIN_USERNAME` and `TLSENTINEL_ADMIN_PASSWORD`.
If the table is empty **and** those variables are not set, startup
**fails with a clear error** — the server will not run without an admin
account.

Once any user exists, the bootstrap is a no-op forever — the env vars
are ignored even if you change them. It is safe (and recommended) to
remove them from `.env` after the first successful start.

Navigate to `http://your-server:8080` and sign in.

!!! warning
    Change your admin password after first login via **Profile →
    Password** (the bootstrap user is a normal local user, so
    self-service password change works).

---

## Behind a Reverse Proxy

If TLSentinel sits behind a reverse proxy (nginx, Caddy, Traefik,
cloud load balancer), set `TLSENTINEL_TRUSTED_PROXY_CIDRS` so the
audit log records the real client IP from the `X-Forwarded-For`
header. Without this, every request appears to come from the proxy.

```sh
# Comma-separated CIDRs; bare IPs are accepted and promoted to /32 or /128
TLSENTINEL_TRUSTED_PROXY_CIDRS=10.0.0.0/8,172.16.0.0/12
```

Only traffic originating from one of these CIDRs is allowed to set
`X-Forwarded-For`; anything else falls back to the TCP peer address.
See [Configuration](configuration.md) for details.

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
TLSENTINEL_JWT_SECRET=        # openssl rand -hex 32
TLSENTINEL_ENCRYPTION_KEY=    # openssl rand -base64 32

# Bootstrap admin account (only used on first start)
TLSENTINEL_ADMIN_USERNAME=admin
TLSENTINEL_ADMIN_PASSWORD=changeme
```

Generate the two required secrets:

```sh
echo "TLSENTINEL_JWT_SECRET=$(openssl rand -hex 32)"
echo "TLSENTINEL_ENCRYPTION_KEY=$(openssl rand -base64 32)"
```

Paste the output into your `.env`.

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
TLSENTINEL_SCANNER_TOKEN=scanner_xxxxxxxxxxxxxxxxxxxx
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

On first startup, TLSentinel creates an admin account using `TLSENTINEL_ADMIN_USERNAME` and `TLSENTINEL_ADMIN_PASSWORD`. If those variables are not set, the bootstrap step is skipped.

Navigate to `http://your-server:8080` and sign in with your admin credentials.

!!! warning
    Change your admin password after first login via **My Account → Password**.

# Installation

!!! note
    TLSentinel requires PostgreSQL 14 or later.

## Docker (Recommended)

```sh
docker run -d \
  --name tlsentinel \
  -p 8080:8080 \
  -e TLSENTINEL_DB_HOST=your-db-host \
  -e TLSENTINEL_DB_USERNAME=your-db-user \
  -e TLSENTINEL_DB_PASSWORD=your-db-password \
  -e TLSENTINEL_JWT_SECRET=$(openssl rand -hex 32) \
  -e TLSENTINEL_ENCRYPTION_KEY=$(openssl rand -base64 32) \
  -e TLSENTINEL_ADMIN_USERNAME=admin \
  -e TLSENTINEL_ADMIN_PASSWORD=changeme \
  ghcr.io/tlsentinel/tlsentinel-server:latest
```

Migrations run automatically on startup.

## Binary

Download the latest release from [GitHub Releases](https://github.com/tlsentinel/tlsentinel-server/releases) and run the binary directly. All configuration is via environment variables.

# Configuration

All configuration is via environment variables.

---

## Database

| Variable | Default | Description |
|---|---|---|
| `TLSENTINEL_DB_HOST` | `localhost` | PostgreSQL hostname |
| `TLSENTINEL_DB_PORT` | `5432` | PostgreSQL port |
| `TLSENTINEL_DB_NAME` | `tlsentinel` | Database name |
| `TLSENTINEL_DB_USERNAME` | — | **Required** |
| `TLSENTINEL_DB_PASSWORD` | — | **Required** |
| `TLSENTINEL_DB_SSLMODE` | `require` | SSL mode (`disable`, `require`, `verify-full`) |

!!! note "SSL mode in Docker Compose"
    When using the bundled PostgreSQL container from the [Installation guide](installation.md), set `TLSENTINEL_DB_SSLMODE=disable` — the database is on an internal Docker network and does not have TLS configured. Use `require` or `verify-full` when connecting to an external PostgreSQL instance.

---

## Server

| Variable | Default | Description |
|---|---|---|
| `TLSENTINEL_HOST` | `0.0.0.0` | Bind address |
| `TLSENTINEL_PORT` | `8080` | Bind port |
| `TLSENTINEL_JWT_SECRET` | — | **Required.** Minimum 32 characters. |
| `TLSENTINEL_ENCRYPTION_KEY` | — | **Required.** Base64-encoded 32-byte key. |
| `TLSENTINEL_ADMIN_USERNAME` | — | Bootstrap admin username (first start only) |
| `TLSENTINEL_ADMIN_PASSWORD` | — | Bootstrap admin password (first start only) |

Generate the required secrets:

```sh
echo "TLSENTINEL_JWT_SECRET=$(openssl rand -hex 32)"
echo "TLSENTINEL_ENCRYPTION_KEY=$(openssl rand -base64 32)"
```

`openssl rand -base64 32` produces a base64-encoded 32-byte value, which is exactly what `TLSENTINEL_ENCRYPTION_KEY` expects.

---

## OIDC (Optional)

| Variable | Default | Description |
|---|---|---|
| `TLSENTINEL_OIDC_ISSUER` | — | OIDC issuer URL |
| `TLSENTINEL_OIDC_CLIENT_ID` | — | Client ID |
| `TLSENTINEL_OIDC_CLIENT_SECRET` | — | Client secret |
| `TLSENTINEL_OIDC_REDIRECT_URL` | — | Redirect URL (`https://your-server/api/v1/auth/oidc/callback`) |
| `TLSENTINEL_OIDC_SCOPES` | `openid,profile,email` | Comma-separated scopes |
| `TLSENTINEL_OIDC_USERNAME_CLAIM` | — | JWT claim to use as the username (e.g. `preferred_username`, `email`) |

OIDC is enabled automatically when all four required variables (`TLSENTINEL_OIDC_ISSUER`, `TLSENTINEL_OIDC_CLIENT_ID`, `TLSENTINEL_OIDC_CLIENT_SECRET`, `TLSENTINEL_OIDC_REDIRECT_URL`) are set. See [SSO / OIDC](../admin/oidc.md) for provider-specific setup guides.

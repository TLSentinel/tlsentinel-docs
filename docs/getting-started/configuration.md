# Configuration

All configuration is via environment variables.

## Database

| Variable | Default | Description |
|---|---|---|
| `TLSENTINEL_DB_HOST` | `localhost` | PostgreSQL hostname |
| `TLSENTINEL_DB_PORT` | `5432` | PostgreSQL port |
| `TLSENTINEL_DB_NAME` | `tlsentinel` | Database name |
| `TLSENTINEL_DB_USERNAME` | — | **Required** |
| `TLSENTINEL_DB_PASSWORD` | — | **Required** |
| `TLSENTINEL_DB_SSLMODE` | `require` | SSL mode |

## Server

| Variable | Default | Description |
|---|---|---|
| `TLSENTINEL_HOST` | `0.0.0.0` | Bind address |
| `TLSENTINEL_PORT` | `8080` | Bind port |
| `TLSENTINEL_JWT_SECRET` | — | **Required.** Min 32 chars. Generate: `openssl rand -hex 32` |
| `TLSENTINEL_ENCRYPTION_KEY` | — | **Required.** AES-256 key (base64). Generate: `openssl rand -base64 32` |
| `TLSENTINEL_ADMIN_USERNAME` | — | Bootstrap admin username |
| `TLSENTINEL_ADMIN_PASSWORD` | — | Bootstrap admin password |

## OIDC (Optional)

| Variable | Description |
|---|---|
| `TLSENTINEL_OIDC_ISSUER` | OIDC issuer URL |
| `TLSENTINEL_OIDC_CLIENT_ID` | Client ID |
| `TLSENTINEL_OIDC_CLIENT_SECRET` | Client secret |
| `TLSENTINEL_OIDC_REDIRECT_URL` | Redirect URL |
| `TLSENTINEL_OIDC_SCOPES` | Scopes (default: `openid,profile,email`) |
| `TLSENTINEL_OIDC_USERNAME_CLAIM` | JWT claim to use as username |

OIDC is enabled automatically when all four required fields (`ISSUER`, `CLIENT_ID`, `CLIENT_SECRET`, `REDIRECT_URL`) are set.

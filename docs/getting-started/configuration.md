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
| `TLSENTINEL_JWT_SECRET` | — | **Required.** Base64-encoded; must decode to ≥32 bytes. |
| `TLSENTINEL_ENCRYPTION_KEY` | — | **Required.** Base64-encoded; must decode to exactly 32 bytes. Protects SMTP passwords at rest. |
| `TLSENTINEL_ADMIN_USERNAME` | — | Bootstrap admin username. Only consulted when the users table is empty. |
| `TLSENTINEL_ADMIN_PASSWORD` | — | Bootstrap admin password. Only consulted when the users table is empty. |

Generate the required secrets:

```sh
echo "TLSENTINEL_JWT_SECRET=$(openssl rand -hex 32)"
echo "TLSENTINEL_ENCRYPTION_KEY=$(openssl rand -base64 32)"
```

`openssl rand -base64 32` satisfies both: 32 random bytes, base64-encoded
— exactly what `TLSENTINEL_ENCRYPTION_KEY` requires and comfortably over
the minimum for `TLSENTINEL_JWT_SECRET`.

!!! warning "Do not use hex for these values"
    Hex output (`openssl rand -hex 32`) is *accidentally* valid base64 —
    the bytes it decodes to will be random, but they won't be the hex
    bytes you intended. Generate both secrets with `-base64` so the
    length checks are meaningful.

### Bootstrap admin

The admin bootstrap runs exactly once, on the first startup where the
users table is empty:

- Table empty **and** both env vars set → admin user is created.
- Table empty **and** either env var missing → startup **fails** with a
  clear error. The server will not run without an admin.
- Table non-empty → env vars are ignored, regardless of their value.

Remove `TLSENTINEL_ADMIN_USERNAME` / `TLSENTINEL_ADMIN_PASSWORD` from
your environment after the first successful start. Change the
bootstrap admin's password via **Profile → Password**.

### Break-glass recovery

Last-resort env-var path for recovering an admin who lost both their
TOTP device and their recovery codes (or forgot their password) when
no second admin exists to perform the reset via the UI. Distinct from
the bootstrap admin vars above — this runs against an *existing*
admin and applies a destructive reset.

| Variable | Default | Description |
|---|---|---|
| `TLSENTINEL_BREAKGLASS` | — | Master toggle. `true` to apply the configured resets on next startup. Reset flags below without this set are logged and ignored. |
| `TLSENTINEL_BREAKGLASS_USER` | — | Username of the admin to recover (their *current* username). Required when the master toggle is set. |
| `TLSENTINEL_BREAKGLASS_RESET_TOTP` | — | `true` to clear the user's TOTP secret and recovery codes. |
| `TLSENTINEL_BREAKGLASS_RESET_PASSWORD` | — | `true` to reset the user's password to `TLSENTINEL_BREAKGLASS_PASSWORD`. |
| `TLSENTINEL_BREAKGLASS_PASSWORD` | — | The new password. Required when `RESET_PASSWORD=true`. |

The path refuses to operate on non-admin or non-local (OIDC) accounts,
fails loud on a typo'd username, and emits a single
`auth.bootstrap.breakglass` audit row per successful run. After running
it, **remove every `TLSENTINEL_BREAKGLASS_*` variable** and restart
once more so the recovery path is no longer active.

See
[Account recovery](../admin/users.md#account-recovery) for the full
operator runbook.

---

## Logging

| Variable | Default | Description |
|---|---|---|
| `TLSENTINEL_LOG_LEVEL` | `info` | One of `debug`, `info`, `warn`, `error`. |
| `TLSENTINEL_LOG_FORMAT` | `auto` | One of `json`, `text`, `auto`. `auto` picks `text` on a TTY, `json` otherwise. |

Both variables are honoured by the server and the scanner.

---

## Reverse proxy

| Variable | Default | Description |
|---|---|---|
| `TLSENTINEL_TRUSTED_PROXY_CIDRS` | — | Comma-separated list of CIDRs whose traffic is allowed to set `X-Forwarded-For`. |

When set, requests whose TCP peer matches one of the listed CIDRs have
their `X-Forwarded-For` header honoured, and the leftmost address in
that header becomes the client IP recorded in the audit log. Requests
from any other source have their `X-Forwarded-For` ignored and fall
back to the TCP peer address.

Leave unset if the server is exposed directly — ignoring
`X-Forwarded-For` is the safe default, because otherwise any client
could spoof its audit-log IP by sending the header.

Bare IPs are accepted and promoted to `/32` (IPv4) or `/128` (IPv6).
Invalid entries fail fast at startup with a pointer at the bad value.

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

---

## Roles and permissions

Every user has exactly one role. Role membership is checked by name,
and each role expands to a fixed set of **permission strings** that the
server's route middleware consults. The breakdown:

| Area              | Permission          | `viewer` | `operator` | `admin` |
|-------------------|---------------------|:--------:|:----------:|:-------:|
| Endpoints         | `endpoints:view`    | ✓ | ✓ | ✓ |
|                   | `endpoints:edit`    |   | ✓ | ✓ |
| Certificates      | `certs:view`        | ✓ | ✓ | ✓ |
|                   | `certs:edit`        |   |   | ✓ |
| Tags              | `tags:view`         | ✓ | ✓ | ✓ |
|                   | `tags:edit`         |   | ✓ | ✓ |
| Discovery         | `discovery:view`    | ✓ | ✓ | ✓ |
|                   | `discovery:edit`    |   | ✓ | ✓ |
| Scanners          | `scanners:view`     |   | ✓ | ✓ |
|                   | `scanners:edit`     |   |   | ✓ |
| Users             | `users:view`        |   | ✓ | ✓ |
|                   | `users:edit`        |   |   | ✓ |
| Cross-user API keys | `apikeys:admin`   |   |   | ✓ |
| Settings          | `settings:view`    |   | ✓ | ✓ |
|                   | `settings:edit`    |   |   | ✓ |
| Maintenance jobs  | `maintenance`       |   | ✓ | ✓ |
| Logs / audit      | `logs:view`         |   | ✓ | ✓ |
| Self-service profile | `self:access`    | ✓ | ✓ | ✓ |

`admin` is internally represented as the wildcard `*` — it skips the
per-permission check and is granted every route. The table shows the
effective access.

`self:access` is the permission that gates **Profile** — password
changes, personal API keys, tag subscriptions, and calendar token
rotation. Every role has it, so any authenticated user can manage
their own account.

Per-feature permission tables in each guide and admin page say exactly
which role can call each API route — see for example
[Users](../admin/users.md#api), [Endpoints](../guide/endpoints.md), and
[Certificates](../guide/certificates.md).

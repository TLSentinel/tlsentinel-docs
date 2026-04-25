# User Management

Users in TLSentinel are either **local** (username + password stored in
the database) or **OIDC** (authenticated through your SSO provider and
mirrored into the local users table on first sign-in). Both provider types
share the same roles, the same permissions, and the same profile surface.

## Roles

Every user has exactly one role:

| Role       | Purpose                                                    |
|------------|------------------------------------------------------------|
| `admin`    | Full access. Manages users, scanners, settings.            |
| `operator` | Day-to-day work: create and tag endpoints, run maintenance jobs, view logs. |
| `viewer`   | Read-only. Sees endpoints, certs, tags, discovery inbox.   |

All roles can manage their own profile — changing their password, email,
API keys, tag subscriptions, and calendar token. See
[Roles and permissions](../getting-started/configuration.md) for the
per-endpoint breakdown, or the permission reference in each feature guide.

## Initial bootstrap

On first startup, if no users exist, the server creates a single admin
from the `TLSENTINEL_ADMIN_USERNAME` and `TLSENTINEL_ADMIN_PASSWORD`
environment variables. The user is created as a **local** user with the
`admin` role and no email address set.

- The bootstrap is a one-shot: once any user exists, the server never
  auto-creates another one, even if the env vars change.
- Remove the env vars after the first successful startup — they are only
  consulted when the users table is empty.
- Change the bootstrap password at first login (the user is a normal
  local user, self-service password change works).

## Account recovery

Routine cases — a user forgets their password, an admin resets it from
**Settings → Users**; a user loses their TOTP device, they redeem one
of the recovery codes saved during enrollment. The two cases below need
explicit operator attention.

### A user lost both their device and their recovery codes

Any other admin can clear the user's TOTP enrollment from
**Settings → Users → ⋯ → Reset 2FA**. The user then signs in with their
password alone and re-enrolls on next login. This action is gated by
the dedicated `users:credentials` permission (separate from the
lifecycle bucket `users:edit`, since "can reset another user's
credentials" implies "can become that user") and audited as
`auth.totp.disable` with `reason: admin_reset`.

### The sole admin lost their device and their recovery codes

If only one admin exists and that admin is locked out, the UI path
above is unavailable — there is no second admin to perform the reset.
The **break-glass** env-var path exists for exactly this case (and for
the sole admin who has forgotten their password). It runs at startup,
refuses to operate on non-admin or non-local accounts, and emits a
single `auth.bootstrap.breakglass` audit row stamped with the system
actor.

#### Procedure

1. Stop the server.

2. Set the recovery env vars on the container or process.

    Clearing TOTP only (the admin still remembers their password):

    ```bash
    TLSENTINEL_BREAKGLASS=true
    TLSENTINEL_BREAKGLASS_USER=<admin's current username>
    TLSENTINEL_BREAKGLASS_RESET_TOTP=true
    ```

    Resetting both:

    ```bash
    TLSENTINEL_BREAKGLASS=true
    TLSENTINEL_BREAKGLASS_USER=<admin's current username>
    TLSENTINEL_BREAKGLASS_RESET_TOTP=true
    TLSENTINEL_BREAKGLASS_RESET_PASSWORD=true
    TLSENTINEL_BREAKGLASS_PASSWORD=<temporary password>
    ```

    `BREAKGLASS_USER` is the admin's *current* username, which may not
    match `TLSENTINEL_ADMIN_USERNAME` if it was renamed since first
    boot.

3. Start the server. Watch for a `breakglass executed` log line and an
   `auth.bootstrap.breakglass` row in the audit log confirming the
   action.

4. **Remove every `TLSENTINEL_BREAKGLASS_*` variable from the
   environment** and restart the server again so the recovery path is
   no longer active.

5. Sign in. Immediately rotate the password (**Profile → Password**)
   and re-enroll TOTP (**My Account → Two-Factor Authentication**).

!!! warning "Why the master toggle"
    `TLSENTINEL_BREAKGLASS=true` is the explicit "I know what I'm doing"
    gate. Reset flags without it set are logged and ignored, so a
    `RESET_TOTP=true` accidentally baked into a compose file won't
    fire on every boot. After step 4 above, removing the master
    toggle is what makes future reboots safe.

#### What the path refuses to do

- **Operate on a user that doesn't exist.** Fails loud rather than
  silently falling through to the first-run create path — if you
  typo'd the username, you want to know immediately.
- **Operate on a non-admin account.** Break-glass is a lockout-recovery
  tool for the sole-admin edge case, not a generic password reset.
  Reset routine accounts through **Settings → Users**.
- **Operate on an OIDC account.** Those have no local password to
  reset and no TOTP enrollment in TLSentinel — recover them at the
  identity provider.

See
[Configuration → Break-glass recovery](../getting-started/configuration.md#break-glass-recovery)
for the full env-var reference.

## Creating users

**Local users** — **Settings → Users → New user**. Required fields are
username and password; role defaults to `viewer`. The password is hashed
with bcrypt before it reaches the database and is never returned from any
API. Administrators can set the `enabled` flag, the `notify` flag
(whether this user should receive email alerts), and optional first name,
last name, and email.

!!! note "Password policy"
    The server does not enforce a minimum password length or complexity
    rule. If your environment needs one, require OIDC and enforce policy
    at the identity provider — TLSentinel will mirror whatever the IdP
    accepts.

**OIDC users** — not created manually. The first time a principal signs
in through OIDC, a user row is created with `provider = "oidc"` and the
`viewer` role. An administrator then promotes the user to `operator` or
`admin` if appropriate. Convert a local user to OIDC by editing the
provider field; convert back by setting a local password.

## Enable, disable, delete

- **Enabled** — disabled users cannot sign in; their sessions are
  revoked on the next request. Disabled users are also skipped by the
  email alert job even if `notify` is true. Keep a user disabled rather
  than deleting them when you want to preserve audit history.
- **Delete** — hard delete. Removes the user row, their API keys, their
  tag subscriptions, and their calendar token. Audit entries previously
  attributed to the user keep the username string but drop the user ID
  reference, so history is preserved but un-linkable.

## Profile (self-service)

Every authenticated user can reach **Profile** from the top-right menu.
From there they can:

- Change their password (local users only).
- Update first name, last name, email.
- Toggle the `notify` flag — the hard on/off for receiving email alerts.
- Manage their tag subscriptions (see
  [Tags and Subscriptions](../guide/tags.md#subscriptions)).
- Generate or rotate their calendar token (see
  [Calendar Feed](../guide/calendar-feed.md)).
- Create and revoke personal API keys.

These actions require only the `self:access` permission, which every
role has.

## API keys

Each user can mint any number of **API keys** scoped to their own
account. An API key acts as that user, with that user's role — not as a
separate identity. Keys are intended for:

- Scripting and CI automation against the TLSentinel API.
- Headless integrations (ingesting certs from a build system, pulling an
  endpoint list into a dashboard).

The raw key is shown **once at creation** and never retrievable again.
Revoke a key from the same page; it stops working immediately.

Admins can see and revoke every user's keys via
**Settings → API keys → All users**. Admins **cannot** read key values
they did not mint — the hashes in the database are one-way.

## API

| Method | Path                                          | Permission         |
|--------|-----------------------------------------------|--------------------|
| GET    | `/api/v1/users`                               | `users:view`       |
| POST   | `/api/v1/users`                               | `users:edit`       |
| GET    | `/api/v1/users/{id}`                          | `users:view`       |
| PUT    | `/api/v1/users/{id}`                          | `users:edit`       |
| PATCH  | `/api/v1/users/{id}/password`                 | `users:edit`       |
| PATCH  | `/api/v1/users/{id}/enabled`                  | `users:edit`       |
| DELETE | `/api/v1/users/{id}`                          | `users:edit`       |
| GET    | `/api/v1/me`                                  | `self:access`      |
| PUT    | `/api/v1/me`                                  | `self:access`      |
| PATCH  | `/api/v1/me/password`                         | `self:access`      |
| GET    | `/api/v1/me/api-keys`                         | `self:access`      |
| POST   | `/api/v1/me/api-keys`                         | `self:access`      |
| DELETE | `/api/v1/me/api-keys/{id}`                    | `self:access`      |
| GET    | `/api/v1/admin/api-keys`                      | `apikeys:admin`    |
| DELETE | `/api/v1/admin/api-keys/{id}`                 | `apikeys:admin`    |

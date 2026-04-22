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

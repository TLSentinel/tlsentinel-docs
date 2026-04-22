# Scanners

A **scanner** is a lightweight agent that reaches out to the TLSentinel
server, pulls a list of endpoints assigned to it, performs the TLS
handshakes (or SAML metadata fetches), and reports the results. Every
host- and SAML-endpoint observation flows through a scanner; manual
endpoints are the only exception.

This page covers how the server side of scanners works — registering
them, rotating their credentials, assigning work. For packaging, install
paths, and runtime configuration of the scanner binary itself, see
[Scanner Deployment](../scanner/index.md).

## The scanner record

Each registered scanner is a row in `scanner_tokens` with:

| Field                 | Meaning                                                      |
|-----------------------|--------------------------------------------------------------|
| `name`                | Human label. Shown in the UI and in audit logs.              |
| `token` (hash only)   | The bearer token the agent sends on every request. Stored as a one-way hash. |
| `isDefault`           | Exactly one scanner carries this flag. New endpoints that don't specify a scanner are assigned to the default. |
| `scanCronExpression`  | How often this scanner should run its scan batch. Defaults to `0 * * * *` (hourly). |
| `scanConcurrency`     | How many endpoints this scanner may probe in parallel per batch. Defaults to 5. |
| `createdAt`           | When the scanner was registered.                             |
| `lastUsedAt`          | Last time the server saw a request signed with this token. Populated by the auth layer on every call. |

`scanCronExpression` and `scanConcurrency` are pulled by the scanner
agent from its own record — the agent polls the server for its config on
start and re-reads it periodically. Changing these fields does not
require redeploying the agent.

## Creating a scanner

**Settings → Scanners → New scanner**. The only required field is a
name. You can set `scanCronExpression` and `scanConcurrency` at creation
or leave them at the defaults.

On submit, the server generates a cryptographically random bearer token,
hashes it, stores the hash, and returns the **raw token exactly once** in
the response body. The UI shows it with a copy button. After you leave
that page the token is unrecoverable — you must regenerate if you lose
it.

!!! warning "Copy the token immediately"
    The response is the only time the raw token is ever exposed. Closing
    the dialog or refreshing the page before copying it means a trip
    through **Regenerate token** (below).

Deploy the token into the scanner agent's config. See
[Scanner Deployment](../scanner/index.md) for the agent-side bits.

## Rotating a token

**Settings → Scanners → <scanner> → Regenerate token** mints a new
bearer token and updates the hash in place. The previous token is
**immediately invalidated** — the next request from an agent still
carrying the old token returns `401`.

Everything else on the scanner record — name, schedule, concurrency,
default flag, discovery networks, pinned endpoints — is preserved. The
regenerate call is the right move when:

- A token may have leaked (log file, old config backup, rotated-out
  operator).
- You are migrating the agent to a new host and want the old install to
  stop working the instant the new one comes up.
- You want to verify agent → server auth is wired correctly without
  changing any other state.

Rotation is logged in the audit trail as `scanner:regenerate_token`.

## The default scanner

One scanner at a time carries the `isDefault` flag. When a new endpoint
is created without an explicit `scannerId`, the server assigns it to
the default. Promoting a different scanner via
**Settings → Scanners → <scanner> → Set as default** clears the flag on
the previous default in a single transaction — you cannot end up with
zero or two defaults.

If you run a single scanner, the default is effectively "the one
scanner"; if you run a fleet, the default is the catch-all for endpoints
that don't have a reason to be pinned.

## Pinning endpoints to a scanner

Every endpoint carries an optional `scannerId` pointing at the scanner
that owns it. Pinning an endpoint tells the server to only hand that
endpoint out to the specified scanner's batches. This matters when:

- A scanner lives in a network segment that is the only vantage point
  with TCP access to the endpoint (DMZ, internal VLAN, a segregated
  customer VPC).
- You want to isolate noisy / heavyweight endpoints onto a dedicated
  agent so they don't starve the rest of the batch.
- Compliance requires scans to originate from specific infrastructure.

Set the assignment from the endpoint detail page under **Scanner**, or
in bulk via the endpoint `PUT` / `PATCH` endpoints. Clearing the field
hands the endpoint back to the default scanner.

## Discovery networks

Discovery is also scoped to a scanner. Every
`discovery_networks` row has a `scannerId` field selecting which scanner
performs the sweep — the scanner's vantage point is the whole point of
the feature, so picking one is required for any network that isn't on
the default.

Networks, sweeps, and the discovery inbox surface under
**Discovery → Networks** and **Discovery → Inbox** in the UI.

## Deleting a scanner

`DELETE /scanners/{id}` revokes the token and removes the record. If the
scanner owned endpoints via `scannerId`, those endpoints are left with
`scannerId = null` and fall back to the default scanner on their next
scan. Discovery networks attached to the deleted scanner are similarly
orphaned and stop running until you reassign them.

You cannot delete the only scanner in the system — the server requires
at least one default to remain, otherwise new endpoints could not be
assigned anywhere.

## API

| Method | Path                                             | Permission       |
|--------|--------------------------------------------------|------------------|
| GET    | `/api/v1/scanners`                               | `scanners:view`  |
| POST   | `/api/v1/scanners`                               | `scanners:edit`  |
| GET    | `/api/v1/scanners/{id}`                          | `scanners:view`  |
| PUT    | `/api/v1/scanners/{id}`                          | `scanners:edit`  |
| PATCH  | `/api/v1/scanners/{id}`                          | `scanners:edit`  |
| DELETE | `/api/v1/scanners/{id}`                          | `scanners:edit`  |
| POST   | `/api/v1/scanners/{id}/default`                  | `scanners:edit`  |
| POST   | `/api/v1/scanners/{id}/regenerate-token`         | `scanners:edit`  |

`POST /scanners` and `POST /scanners/{id}/regenerate-token` both return
the raw token exactly once. Every other route returns the safe
representation (no hash, no secret material).

# API Reference

TLSentinel exposes a REST API over HTTPS. The running server ships an
interactive Swagger UI at:

```
https://<your-server>/api-docs/index.html
```

That page is the **authoritative, always-current** reference — every
handler is annotated, every schema is generated from the Go structs, and
the "Try it out" button works against the instance you're looking at.

This page is the orientation guide to that reference: where requests go,
how to authenticate, and the conventions that cut across every resource.

## Base path

Every route is mounted under:

```
/api/v1
```

`/api/v1/users`, `/api/v1/endpoints`, `/api/v1/certificates`, and so on.
The Swagger UI sits at `/api-docs/index.html` (not under `/api/v1`) and
the raw OpenAPI document is at `/swagger.json`.

## Authentication

Every protected endpoint expects the same header:

```
Authorization: Bearer <token>
```

Three kinds of token are accepted. The server tells them apart by prefix
and routes each through a different verifier:

| Token kind   | Prefix    | Issued by                                  | Acts as                  |
|--------------|-----------|--------------------------------------------|--------------------------|
| **JWT**      | *(none)*  | `POST /api/v1/auth/login` (or OIDC login). | The user who signed in.  |
| **API key**  | `stx_p_`  | `POST /api/v1/me/api-keys`.                | The user who minted it, with that user's role. |
| **Scanner token** | `stx_s_` | `POST /api/v1/scanners` (or regenerate). | The scanner agent, with a fixed scanner identity. |

Sessions (the JWT path) and API keys share the same permission model —
an API key inherits its owning user's role, so a viewer's key can only
call viewer-permitted routes. See
[Users](admin/users.md) for role definitions and
[API keys](admin/users.md#api-keys) for the lifecycle.

The calendar feed (`/api/v1/calendar/u/{token}/*`) is the one public
route; it carries its credential in the URL. See
[Calendar Feed](guide/calendar-feed.md).

## Permissions

Most routes are gated by a specific permission string (`endpoints:view`,
`settings:edit`, `self:access`, and so on). Every feature guide and admin
page in these docs lists the permissions for its routes in a table at
the end — that's the fastest way to answer "can a viewer call this?"
without digging through code. The per-role permission matrix lives in
[Roles and permissions](getting-started/configuration.md).

A permission failure returns `403 Forbidden` with a short plain-text
body; a missing or invalid token returns `401 Unauthorized`.

## Errors

Errors come back as plain text (not JSON) with an HTTP status code that
carries the meaning:

| Status | Meaning                                                          |
|--------|------------------------------------------------------------------|
| `400`  | Request body failed to parse or validation rejected it.          |
| `401`  | Missing, malformed, or expired token.                            |
| `403`  | Authenticated, but role lacks the permission for this route.     |
| `404`  | Resource ID does not exist.                                      |
| `409`  | Conflict — uniqueness constraint violation, state mismatch, etc. |
| `422`  | Semantically invalid (e.g. mail not configured, encryption key missing). |
| `500`  | Unexpected server failure. Check logs.                           |
| `502`  | Upstream call failed (e.g. SMTP round-trip for `mail/test`).     |

`201 Created` is returned for new resources, `200 OK` for updates and
reads, `204 No Content` for deletions and other "nothing to render" success
cases.

## Pagination and filtering

List endpoints accept the usual query-string parameters:

- `limit` / `offset` — the default page size varies per resource; check
  the Swagger UI for the exact defaults.
- Resource-specific filters — `type`, `enabled`, `has_error`, `tag`, and so
  on. These are documented in each guide and on the Swagger page for
  that route.

Responses are either a bare array (small / unpaged resources) or an
envelope of the form `{ "items": [...], "totalCount": N }` for paginated
resources. Endpoint detail pages, the discovery inbox, and the search
results all use the envelope.

## Content types

- Request bodies: `application/json`, with the exception of PEM / DER
  uploads to `POST /certificates` and the endpoint-cert routes, which
  accept either PEM (`application/x-pem-file` or any text type) or
  base64-DER.
- Response bodies: `application/json`, except for the calendar feed
  (`text/calendar`) and CSV exports (`text/csv`).

## Example: authenticated call with an API key

```bash
curl -H "Authorization: Bearer stx_p_1a2b3c4d_abcdef..." \
     https://<your-server>/api/v1/endpoints?enabled=true
```

## Client libraries

There is no official client SDK. The Swagger JSON is standard OpenAPI 3,
so `openapi-generator` will produce a workable client in any language
you like. The TLSentinel CLI (`tlsctl`) is one such consumer and lives
in the monorepo at `products/cli/`.

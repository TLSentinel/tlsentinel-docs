# Endpoints

An **endpoint** is anything TLSentinel tracks a certificate for. Each endpoint
has a **type** that determines how (or whether) it is scanned and what
information the UI shows for it.

There are three types: `host`, `saml`, and `manual`. Every endpoint — whatever
its type — can carry tags, have notes, be pinned to a specific scanner, and
participate in expiry alerts.

## Type summary

| Feature                            | host    | saml    | manual  |
|------------------------------------|:-------:|:-------:|:-------:|
| Certificate tracking               | ✓       | ✓       | ✓       |
| Expiry alerts / notifications      | ✓       | ✓       | ✓       |
| Tags and notes                     | ✓       | ✓       | ✓       |
| Automatic scanning by a scanner    | ✓       | ✓       | —       |
| Scan history                       | ✓       | ✓       | —       |
| Scanner assignment (pin or auto)   | ✓       | ✓       | —       |
| Pin to a specific IP               | ✓       | —       | —       |
| TLS version + cipher suite profile | ✓       | —       | —       |
| Metadata parsing                   | —       | ✓       | —       |
| Multiple cert roles per endpoint   | —       | ✓       | ✓       |
| Manual cert upload                 | —       | —       | ✓       |

## `host` — a network service you can reach

The default type and the one most endpoints use. You give TLSentinel a
**DNS name** and a **port** (default `443`), optionally an **IP address** to
pin resolution to, and a scanner does the rest.

**What it captures**

- The leaf certificate and the full chain presented on the TLS handshake.
- A TLS profile record: negotiated protocol versions and cipher suites
  (TLS 1.2/1.3, etc.) so you can see what the endpoint actually supports.
- A scan history entry on every run, including resolved IP, success/failure,
  and any error message.
- An `errorSince` timestamp on sustained failures, so dashboards can surface
  endpoints that have been broken for a while rather than flapping.

**When to use it**

Any service that answers a TLS handshake on a reachable host and port —
websites, APIs, mail servers, databases with TLS, load balancers. Anything
where the scanner can open a TCP connection and speak TLS.

**IP pinning**

Leave `ipAddress` empty to let the scanner resolve the DNS name at scan time.
Set it to pin scans to a specific backend — useful for checking individual
members behind a load balancer, or for endpoints where DNS resolves to a
split-horizon address the scanner cannot reach.

## `saml` — a SAML Identity or Service Provider

You give TLSentinel a **metadata URL** (the XML document that describes the
IdP or SP). A scanner fetches the metadata, parses it, and extracts every
certificate declared inside — signing keys, encryption keys, all of them.

**What it captures**

- Every `<KeyDescriptor>` certificate in the metadata, recorded with its
  **role** (`signing`, `encryption`, or `unspecified`) so a single endpoint
  can surface multiple related certs.
- The parsed metadata payload: entity ID, declared SSO / SLO / ACS endpoints
  with their bindings and indexes, and any `ContactPerson` elements. The UI
  renders these on the endpoint detail page so you can see who to contact
  when a cert is expiring.
- The fetch timestamp, so you know the metadata is fresh.
- A scan history entry and `errorSince` tracking, same as host endpoints.

**When to use it**

Any SAML 2.0 IdP or SP whose metadata is published at a URL (Okta, ADFS,
Azure AD, Shibboleth, on-prem Keycloak, etc.). SAML certs have notoriously
silent expiries — this type exists specifically so you find out before the
federation breaks.

**What it does not capture**

No TLS profile. The scanner fetches the metadata over HTTPS but does not
record cipher suites or protocol versions — that would be the TLS profile
of the metadata host, not the SAML endpoint itself. If you also want
TLS-layer coverage for the metadata host, add it separately as a `host`
endpoint.

## `manual` — certificates that can't be scanned

You upload a **PEM** directly through the UI or API. Nothing is scanned,
nothing is fetched; the cert is just recorded against the endpoint and tracked
for expiry like any other.

**What it captures**

- Whatever PEM you upload, with an optional **cert use** label (`manual` by
  default; `signing` or `encryption` for cases where you are tracking
  multiple roles under one endpoint).
- Expiry alerts and calendar-feed events, same as scanned endpoints.

**When to use it**

- Certificates on systems the scanner cannot reach — isolated networks,
  appliances without a TLS listener, internal CAs, offline signing keys.
- Code-signing, document-signing, or S/MIME certificates.
- VPN or device certs you only receive as a file.
- Anything you have as a PEM and want to watch expire.

**What it does not do**

No scanner assignment, no scan history, no TLS profile. Updates happen only
when a user uploads a new cert.

## Common fields

Regardless of type, every endpoint has:

- **Name** — required, human-readable label. Shown everywhere the endpoint
  appears.
- **Enabled** — toggle scanning on or off. Temporarily disabled endpoints
  keep their cert history and can be re-enabled without losing data.
- **Scan exempt** — a permanent "never scan, even if enabled" flag. Useful
  for endpoints you want to track but must not probe (regulated systems,
  paid third-party endpoints where you don't want to generate requests).
- **Scanner** — pin to a specific scanner, or leave empty to let the server
  pick one. Manual endpoints ignore this field.
- **Notes** — free-form text, shown on the detail page.
- **Tags** — any number of category-scoped tags.
  See [Tags and Subscriptions](tags.md).

## Bulk import

The create endpoint accepts a single record; the bulk import endpoint accepts
a list and processes them row-by-row, returning a per-row success/failure
response. The usual workflow is to build a CSV of endpoints in a spreadsheet,
convert it to JSON, and POST it in one request — rows that fail validation do
not block rows that pass.

Bulk import respects the same type rules as single-row create: `host`
defaults to port 443, `saml` requires a URL, `manual` needs nothing type-
specific.

## API

| Method | Path                                          | Permission        |
|--------|-----------------------------------------------|-------------------|
| GET    | `/api/v1/endpoints`                           | `endpoints:view`  |
| POST   | `/api/v1/endpoints`                           | `endpoints:edit`  |
| POST   | `/api/v1/endpoints/bulk`                      | `endpoints:edit`  |
| GET    | `/api/v1/endpoints/{id}`                      | `endpoints:view`  |
| PUT    | `/api/v1/endpoints/{id}`                      | `endpoints:edit`  |
| PATCH  | `/api/v1/endpoints/{id}`                      | `endpoints:edit`  |
| DELETE | `/api/v1/endpoints/{id}`                      | `endpoints:edit`  |
| GET    | `/api/v1/endpoints/{id}/history`              | `endpoints:view`  |
| GET    | `/api/v1/endpoints/{id}/tls-profile`          | `endpoints:view`  |
| POST   | `/api/v1/endpoints/{id}/certificate`          | `endpoints:edit`  |
| GET    | `/api/v1/endpoints/{id}/tags`                 | `tags:view`       |
| PUT    | `/api/v1/endpoints/{id}/tags`                 | `endpoints:edit`  |

The `POST …/certificate` endpoint is the upload path for `manual` endpoints
(and is also available on scanned endpoints if you need to record a cert the
scanner hasn't captured yet).

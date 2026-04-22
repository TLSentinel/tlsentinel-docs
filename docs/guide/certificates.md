# Certificates

Every certificate TLSentinel has ever seen is kept in a single global table,
keyed by its **SHA-256 fingerprint**. A cert is stored once even when it is
in use on many endpoints — a wildcard or a SAN cert shows up once in the
certificate list, and the detail page surfaces every endpoint that presents
it. Rotation is additive: old certs are retained and marked historical, not
deleted, so you always have a complete timeline of what was where and when.

This page explains the data model, what the certificate detail page shows,
and how certs enter the system.

## How certs attach to endpoints

The link between an endpoint and a certificate is an **endpoint-certificate
row** with three extra fields:

- **`cert_use`** — the role the cert plays on that endpoint. One of:
    - `tls` — the leaf cert returned on a TLS handshake (host endpoints).
    - `signing` — a SAML signing certificate.
    - `encryption` — a SAML encryption certificate.
    - `manual` — a PEM uploaded against a manual endpoint.
- **`is_current`** — `true` for what the endpoint is presenting *now*;
  `false` for previous certs retained as history.
- **`first_seen_at` / `last_seen_at`** — the oldest and most recent scan
  that observed this cert on this endpoint.

A single endpoint can have multiple current rows of different uses — this
is the point of the model. A SAML endpoint typically has one `signing` and
one `encryption` cert both marked current; a manual endpoint tracking a
mixed-use key might have both `signing` and `encryption` rows pointing at
the same cert.

### Rotation

When a scanner observes a new cert on an endpoint, it:

1. Inserts the new cert into `certificates` (or no-ops if the fingerprint
   is already known — certs are global and shared across endpoints).
2. Flips any existing `is_current = true` row for that endpoint / cert-use
   pair to `false`.
3. Inserts a new `is_current = true` row for the new cert.

The old row is retained. If the cert comes back later (a rollback, or
because the endpoint is serving it again from a cache) the existing
historical row is reactivated rather than duplicated.

### Manual uploads

`POST /endpoints/{id}/certificate` is the upload path. It accepts a PEM or
base64-DER body, parses it, stores the cert in the global table if new,
and creates the endpoint-cert row with whatever `cert_use` you specify
(defaulting to `manual`). The same endpoint works on scanned endpoints if
you need to record a cert the scanner hasn't captured yet.

## The certificate detail page

Open any cert (from an endpoint, the expiring list, or the all-certs
table) and you get a parsed view of the stored PEM with:

- **Identity** — common name, SANs, serial number, subject org and OU.
- **Issuer** — issuer common name, issuer org, and a **jump link** to the
  issuer's detail page when TLSentinel has it in the database. The link
  chain stops at the effective root (see [Trust](#trust) below).
- **Validity** — not-before, not-after, time remaining or time past expiry.
- **Key** — algorithm (`RSA`, `ECDSA`, `Ed25519`), key size, and the
  signature algorithm used to sign this cert.
- **Key usages** — which `keyUsage` and `extendedKeyUsage` extensions are
  asserted (server auth, client auth, code signing, email protection, etc.).
- **Revocation** — OCSP responder URLs and CRL distribution points pulled
  from the cert's extensions.
- **Raw PEM** — for copy or download.
- **Endpoints** — every endpoint currently presenting this cert.
- **Historical endpoints** — endpoints that previously presented this cert,
  with the date last observed.

### Trust

TLSentinel tracks four public **root stores** — Apple, Chrome, Microsoft,
and Mozilla — sourced from Common CA Database (CCADB). For each cert,
the detail page shows:

- **Trusted by** — a badge set listing every root store whose trust
  anchors appear somewhere in the cert's issuer chain. An empty set means
  no known public root trusts this chain (private CA, self-signed, or a
  chain TLSentinel cannot complete because the intermediate is missing).
- **Is trust anchor** — a flag that is true when the cert itself
  (subject + subject-key-id) is one of the imported CCADB anchors. This
  also covers cross-signed copies of a root, so the detail page knows where
  the "up" link should stop.

The root stores refresh as part of the nightly maintenance pipeline, or
on demand from **Settings → Maintenance → Run: Refresh root stores**.

## Browsing certificates

The navigation surfaces certs in three different cuts:

- **All certificates** — everything TLSentinel has stored, independent of
  whether a cert is currently attached to anything. Useful when hunting
  down a specific cert by name or fingerprint, or auditing old certs that
  are still in the cert store.
- **Active certificates** — one row per `(endpoint, cert)` pair where
  `is_current = true`. This is the list most teams care about day-to-day.
- **Expiring** — the subset of active certs whose `not_after` is within
  the requested window. Includes already-expired certs as rows with
  negative `daysRemaining`, so they cannot drop off the list silently.

Each of these is filterable and searchable. The search endpoint
(`GET /search`) understands certificate fingerprints, common names, and
SANs, so typing part of a hostname into the global search bar finds certs
wherever they live.

## Deleting a cert

`DELETE /certificates/{fingerprint}` removes a cert from the global table.
You can only do this if the cert has no remaining references — no
`endpoint_certs` row points at it, not even a historical one. Detaching a
cert from every endpoint is a prerequisite for deletion, and the nightly
**Prune unreferenced certificates** job handles this automatically for
certs the retention policy has already purged history for. In practice,
operators rarely delete certs by hand.

## API

| Method | Path                                                       | Permission        |
|--------|------------------------------------------------------------|-------------------|
| GET    | `/api/v1/certificates`                                     | `certs:view`      |
| GET    | `/api/v1/certificates/active`                              | `certs:view`      |
| GET    | `/api/v1/certificates/expiring?days=N`                     | `certs:view`      |
| GET    | `/api/v1/certificates/lookup`                              | `certs:view`      |
| POST   | `/api/v1/certificates`                                     | `certs:edit`      |
| GET    | `/api/v1/certificates/{fingerprint}`                       | `certs:view`      |
| DELETE | `/api/v1/certificates/{fingerprint}`                       | `certs:edit`      |
| GET    | `/api/v1/certificates/{fingerprint}/endpoints`             | `certs:view`      |
| GET    | `/api/v1/certificates/{fingerprint}/endpoints/historical`  | `certs:view`      |
| POST   | `/api/v1/endpoints/{id}/certificate`                       | `endpoints:edit`  |
| GET    | `/api/v1/root-stores`                                      | `certs:view`      |
| GET    | `/api/v1/root-stores/{id}/anchors`                         | `certs:view`      |

`POST /certificates` accepts either PEM or base64-DER. It returns
`201 Created` for a new fingerprint and `200 OK` when the cert was already
in the store — idempotent ingestion is the expected case.

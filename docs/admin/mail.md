# Mail Configuration

TLSentinel sends certificate expiry alerts over **SMTP**. A single
global mail configuration controls the connection — host, port, TLS
mode, authentication, and the `From` identity. Until mail is configured
and enabled, the alert job runs but produces nothing.

For what gets sent, to whom, and when, see
[Notifications](../guide/notifications.md). This page is about the
transport.

## Configuring SMTP

**Settings → Mail** holds the full form. The fields are:

| Field          | Notes                                                                         |
|----------------|-------------------------------------------------------------------------------|
| `enabled`      | Hard switch. With this off, the alert job bails out early and nothing is sent. |
| `smtpHost`     | Hostname or IP of your SMTP relay.                                             |
| `smtpPort`     | Defaults to `587` when left blank.                                             |
| `tlsMode`      | One of `none`, `starttls`, `tls`. `starttls` is the right pick for most relays on port 587. |
| `authType`     | One of `none`, `plain`, `login`. `none` skips credentials entirely.            |
| `smtpUsername` | Required when `authType` is not `none`.                                        |
| `smtpPassword` | Write-only. Never returned by the API. See [Passwords](#passwords) below.      |
| `fromAddress`  | Must be a valid RFC 5322 address. Used as `MAIL FROM` and as the test-email fallback recipient. |
| `fromName`     | Display name attached to `fromAddress`. Must not contain CR / LF.              |

`tlsMode` and `authType` are validated server-side — any other value is
rejected with a 400. `fromAddress` is parsed with `net/mail`, so
malformed addresses fail fast at save time rather than silently later.

### Passwords

The SMTP password is encrypted at rest with AES-GCM using the server's
`TLSENTINEL_ENCRYPTION_KEY` environment variable. Without a key set,
saving a password returns `422 Unprocessable Entity` with a message
pointing at the missing key — the server refuses to store plaintext
secrets even in a development setup.

On save, the field works like this:

- **Empty string** — keep whatever password is already stored. This lets
  you edit other fields (host, port, From address) without re-entering
  the password.
- **Any non-empty string** — encrypt and replace the stored value.

The `GET` endpoint never returns the password itself; the response
includes a `passwordSet` boolean so the UI can tell "we have a password"
from "there is no password".

!!! note "Encryption key rotation"
    Changing `TLSENTINEL_ENCRYPTION_KEY` without re-encrypting existing
    values breaks SMTP decryption (and any other secret stored in the
    same vault). If you rotate the key, re-save the mail config with the
    password to re-encrypt under the new key.

## Sending a test email

**Settings → Mail → Send test email** calls `POST /settings/mail/test`.
The body is optional: provide a `to` address to override the recipient,
omit it to send to the configured `fromAddress`.

The test does the full SMTP round-trip against the saved configuration
— it decrypts the password, opens the TLS mode you picked, authenticates
(if any), and sends a short probe message. Possible outcomes:

| Status | Meaning                                                             |
|--------|---------------------------------------------------------------------|
| `204`  | Relay accepted the message. Nothing to render back.                 |
| `422`  | Mail is disabled, not yet configured, or `fromAddress` is unset.    |
| `502`  | SMTP round-trip failed. The error body surfaces the underlying cause so you can tell `connection refused` from `auth failed` from a TLS mismatch. |

If the test works, the alert job will work. If the test fails, the alert
job will also fail — fix the transport here before chasing phantom alert
issues.

## Troubleshooting

- **"TLSENTINEL_ENCRYPTION_KEY is not set"** on save — set the env var
  (32 bytes, base64). The server will not persist passwords in plaintext.
- **`starttls` connections fail** — some relays only offer STARTTLS
  on port 587 and implicit TLS on 465. Match the mode to the port.
- **Auth succeeds but mail never arrives** — check the relay's
  accepted-sender policy. `fromAddress` has to be an address the relay
  will relay for, or the message is silently dropped / spam-scored
  after the `250 OK`.
- **Alerts stop after working fine** — the `lastUsedAt`/last-run state
  is not stored on the mail record but the recurring alert job logs
  per-run totals; check the logs for the last successful batch.

## API

| Method | Path                           | Permission      |
|--------|--------------------------------|-----------------|
| GET    | `/api/v1/settings/mail`        | `settings:view` |
| PUT    | `/api/v1/settings/mail`        | `settings:edit` |
| POST   | `/api/v1/settings/mail/test`   | `settings:edit` |

`GET` returns safe defaults (`smtpPort: 587`, `authType: "plain"`,
`tlsMode: "starttls"`) when no configuration has been saved yet, so the
UI can render the form on a fresh install without any special casing.

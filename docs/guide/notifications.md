# Notifications

TLSentinel notifies people when certificates are about to expire (or have
just expired), over email. The delivery model is:

- A recurring job looks for certs crossing configurable day thresholds.
- Each notification is rendered from an editable template and sent to every
  user who has opted in to notifications.
- Per-user filtering narrows *which* certs each user hears about, based on
  their tag subscriptions.
- A dedup table ensures a given `(user, certificate, threshold)` combination
  is only sent once, no matter how many times the job runs.

Currently the only channel is **email**. The template and event-type model
is designed to accept additional channels (chat webhooks, pager systems)
without rewriting the alert pipeline — only email is wired up today.

## Event types

| Event key         | Fires when                                                  |
|-------------------|-------------------------------------------------------------|
| `cert_expiring`   | An active cert has days-remaining ≤ a configured threshold. |
| `cert_expired`    | An active cert has already passed its `not_after`.          |

Both events run through the same rendering and delivery path; only the
template changes. Manual endpoints, host endpoints, and SAML endpoints are
all eligible — the event is about the certificate, not the endpoint type.

## Thresholds

Thresholds are the set of day-counts at which expiring alerts fire. The
default is **30, 14, 7, 1**, configurable at **Settings → Alerts →
Thresholds**.

When the job runs, each cert is bucketed into its **most urgent**
qualifying threshold. A cert at 12 days remaining falls into the 14-day
bucket on the first run that sees it there; when it crosses into 7 days
it falls into the 7-day bucket and triggers a new alert. A cert at 0 or
negative days falls under `cert_expired` instead.

Dropping or adding thresholds is safe — the dedup table is keyed by
`(user, fingerprint, threshold)`, so changing the list does not re-fire
previously-delivered alerts at thresholds that are still in the set.

## Who receives what

Two things decide whether an alert reaches a given user:

1. **The user-level `notify` flag** — a hard on/off. Users set it on their
   own profile; admins can set it when creating or editing a user. A user
   with `notify = false`, a disabled account, or no email address set is
   skipped entirely.
2. **Tag subscriptions** — for each notify-enabled user, the job only
   fetches certs attached to endpoints that share at least one of the
   user's subscribed tags. A user with **zero subscriptions** receives
   everything (the notify-all fallback). See
   [Tags and Subscriptions](tags.md#subscriptions) for the full semantics.

!!! tip "The two switches are different"
    `notify` is "do I want emails at all?". Subscriptions are "which emails
    do I want?". Clearing all subscriptions does **not** silence a user —
    it makes them a catch-all recipient. Toggle `notify` off to opt out.

## Delivery and deduplication

The alert job runs on a recurring cron (hourly by default) and does the
following:

1. Load mail config; bail if mail is disabled or unconfigured.
2. Load thresholds (falls back to defaults if none are configured).
3. For each notify-enabled user with an email address:
    - Look up expiring certs scoped to the user's tag subscriptions.
    - For each cert, compute the most urgent threshold bucket it qualifies
      for.
    - Try to insert a `(user, fingerprint, threshold)` row into the dedup
      table. If the row already exists, skip — the user has already been
      notified for this cert at this threshold.
    - Render the template with the cert's details and send the email.
4. Log the per-run totals (`alerts_sent`, `already_sent`).

Because dedup happens per-user, two admins who both receive `cert_expiring`
for the same cert each get their own email, but neither will receive it
twice.

## Templates

Every `(event type, channel)` pair has an editable template at
**Settings → Notifications → Templates**. A template has two fields:

- **Subject** — for email, the subject line.
- **Body** — Go-style {% raw %}`{{ .Variable }}`{% endraw %} interpolation
  over the variables listed below.

If a saved template fails to render (missing variable, malformed syntax),
delivery falls back to a plain-text copy of the embedded default so the
alert still goes out. The render error is logged against the run.

Resetting a template via `DELETE /notification-templates/{event}/{channel}`
removes the DB override and restores the embedded default.

### Variables

| Event            | Variable         | What it is                                |
|------------------|------------------|-------------------------------------------|
| `cert_expiring`  | `EndpointName`   | Endpoint display name                     |
|                  | `EndpointType`   | `host`, `saml`, or `manual`               |
|                  | `CommonName`     | Certificate common name                   |
|                  | `NotAfter`       | Certificate expiry date                   |
|                  | `DaysRemaining`  | Days until expiry                         |
|                  | `Fingerprint`    | SHA-256 fingerprint                       |
| `cert_expired`   | `EndpointName`   | Endpoint display name                     |
|                  | `EndpointType`   | `host`, `saml`, or `manual`               |
|                  | `CommonName`     | Certificate common name                   |
|                  | `NotAfter`       | Certificate expiry date                   |
|                  | `Fingerprint`    | SHA-256 fingerprint                       |

The variable list is authoritative — the UI shows the same set next to the
editor so you do not have to memorise it.

## Mail configuration

Alerts will not fire until SMTP is configured at **Settings → Mail**. The
form captures SMTP host and port, TLS mode, authentication (none / plain /
login), username, password, and the From address and name. The password is
encrypted at rest using the server's configured encryption key.

A **Send test email** button exists on the mail settings page; if that
round-trip works, the alert job will also be able to deliver.

## Other surfaces

Email is the only notification *channel*, but certificate expiries also
surface in:

- **Calendar feed** — each user has a personal iCalendar URL that renders
  expiring certs as calendar events, also scoped by their tag
  subscriptions. See [Calendar feed](calendar-feed.md).
- **Dashboard panels** — the main dashboard lists error endpoints,
  expiring certs, and stale scans without any opt-in.

These are per-user views, not deliveries — they do not touch the alert job
or the dedup table.

## API

| Method | Path                                                        | Permission         |
|--------|-------------------------------------------------------------|--------------------|
| GET    | `/api/v1/settings/alert-thresholds`                         | `settings:view`    |
| PUT    | `/api/v1/settings/alert-thresholds`                         | `settings:edit`    |
| GET    | `/api/v1/settings/mail`                                     | `settings:view`    |
| PUT    | `/api/v1/settings/mail`                                     | `settings:edit`    |
| POST   | `/api/v1/settings/mail/test`                                | `settings:edit`    |
| GET    | `/api/v1/notification-templates`                            | `settings:view`    |
| GET    | `/api/v1/notification-templates/{eventType}/{channel}`      | `settings:view`    |
| PUT    | `/api/v1/notification-templates/{eventType}/{channel}`      | `settings:edit`    |
| DELETE | `/api/v1/notification-templates/{eventType}/{channel}`      | `settings:edit`    |
| GET    | `/api/v1/me/tag-subscriptions`                              | `self:access`      |
| PUT    | `/api/v1/me/tag-subscriptions`                              | `self:access`      |
| PUT    | `/api/v1/me`                                                | `self:access`      |

`PUT /me` is where a user flips their own `notify` flag. Admins can set
the same flag on someone else's account via `PUT /users/{id}`.

# Calendar Feed

Every user in TLSentinel can subscribe to their own iCalendar (ICS) feed of
upcoming certificate expiries. The feed is a standard `text/calendar`
document over HTTPS, so any calendar client that speaks subscriptions —
Google Calendar, Apple Calendar, Outlook, Thunderbird — can consume it
directly and keep it fresh.

The feed is **per-user** and **scoped by tag subscriptions**: the same
filtering rules that decide which emails you get also decide which events
appear on your calendar. See
[Tags and Subscriptions](tags.md#subscriptions) for the rules.

## How it works

Each user has an opaque, high-entropy **calendar token**. The feed URL
looks like:

```
https://<your-server>/api/v1/calendar/u/<token>/tlsentinel.ics
```

The URL is the entire credential — there is no separate login. The route
is explicitly outside the authenticated API so calendar clients, which
cannot do interactive sign-on, can fetch it.

The path is a wildcard (`/tlsentinel.ics`, `/feed.ics`, anything after
the token works) so clients that rewrite or append path components still
resolve. What matters is the token segment.

## Subscribing

1. Open **Profile → Calendar feed** in the UI.
2. If no token exists yet, click **Generate**. The page reveals the feed
   URL.
3. Copy the URL and paste it into your calendar client as a new
   **subscription** (not an import — an import is a one-shot snapshot; a
   subscription stays in sync).
4. The server advertises a **refresh interval of one hour** via the
   standard `X-PUBLISHED-TTL` hint. Clients honour this differently —
   Google Calendar currently refreshes roughly every 8–24 hours and
   ignores the hint.

## What's in the feed

The feed covers certs that:

- Are attached to an endpoint (manual-uploaded certs are eligible too).
- Have `is_current = true` on their endpoint row.
- Will expire within the next **365 days**, including certs that have
  already expired (they appear as events in the past).

Each cert becomes one all-day **VEVENT** on its `not_after` date, with:

- **Summary** — `Certificate expiry: <common name> (<endpoint name>)`.
- **Description** — endpoint name, endpoint type (`host` / `saml` /
  `manual`), common name, full expiry timestamp, days remaining, and the
  SHA-256 fingerprint.
- **UID** — the certificate's fingerprint, so the same cert stays the
  same event across refreshes even if anything else changes.
- **Four `VALARM` reminders** at 30, 14, 7, and 1 days before expiry.

!!! note "Alarm thresholds are fixed in the feed"
    The email alert thresholds are configurable; the calendar alarm
    thresholds are hardcoded at **30 / 14 / 7 / 1** day before expiry.
    This is a deliberate choice — some calendar clients silently discard
    dynamic VALARM sets, and a fixed ladder is predictable. Your own
    calendar client may also offer reminders on top of these; those are
    independent.

## Filtering and the notify flag

Calendar events are scoped by **tag subscriptions** the same way email
alerts are. A user with subscriptions only sees events for endpoints that
share one of their tags. A user with **zero subscriptions** gets every
event (the notify-all fallback described in
[Tags and Subscriptions](tags.md#subscriptions)).

The user-level `notify` flag, however, only gates **email**. A user with
`notify = false` still has a working calendar feed if they have a token —
the feed is intentionally opt-in through token generation, and lives on a
different axis from inbox delivery. Users who don't want notifications in
their inbox but *do* want them on their calendar can switch the `notify`
flag off and keep a token.

## Rotating and revoking

**Profile → Calendar feed → Rotate** issues a new token and invalidates
the old one. Any calendar client still subscribed to the old URL will
start getting `404 Not Found` on its next refresh. Rotation is the right
move when:

- You suspect the URL has been shared or leaked.
- You want to stop the feed entirely — generate a new token and then
  clear it from your profile (setting the token to null on the user
  record permanently disables the feed until you generate a fresh one).

There is no TTL on tokens; they persist until rotated.

## Notes on specific clients

- **Google Calendar** — "Other calendars → From URL". Google caches
  aggressively; expect new events to appear within 8–24 hours.
- **Apple Calendar (macOS/iOS)** — "File → New Calendar Subscription".
  Auto-refresh interval is configurable on the subscription after
  adding.
- **Outlook (web / desktop)** — "Add calendar → Subscribe from web".
  Refresh cadence follows Microsoft's caching, typically multiple hours.
- **Thunderbird** — "New Calendar → On the Network → iCalendar (ICS)".
  Honours the server-advertised refresh interval most faithfully.

If your organization proxies HTTPS, make sure the proxy preserves the
`Content-Type: text/calendar` header and does not attempt to rewrite the
body — some filtering proxies normalise line endings and will corrupt the
ICS payload.

## API

| Method | Path                                        | Permission        |
|--------|---------------------------------------------|-------------------|
| GET    | `/api/v1/calendar/u/{token}/*`              | *(public; token)* |
| POST   | `/api/v1/me/calendar-token`                 | `self:access`     |

`POST /me/calendar-token` is both generate and rotate — calling it always
produces a new token and invalidates any previous one.

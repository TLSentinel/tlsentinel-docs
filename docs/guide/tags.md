# Tags and Subscriptions

Tags are the primary way to organise endpoints in TLSentinel. They drive
filtering in the UI, and — via **tag subscriptions** — they also decide which
certificate expiry alerts each user receives by email.

## Model

Tags are organised in two levels:

- **Categories** define *what* you are tagging by — for example `Environment`,
  `Application`, `Owner`, `Datacenter`.
- **Tags** are the individual values inside a category — `production`,
  `staging`, `checkout-api`, `sre-team`, `iad1`.

An endpoint can carry any number of tags, typically one per category (e.g.
`Environment:production`, `Owner:sre-team`), but nothing in the data model
prevents multiple tags from the same category if that reflects reality.

!!! tip "Pick categories deliberately"
    Categories are shared across the whole instance. A small, well-chosen set
    (five to ten) stays navigable; an uncontrolled set becomes a folksonomy
    that nobody filters by. Treat new categories like schema changes.

## Who can manage tags

| Action                                  | Required permission |
|-----------------------------------------|---------------------|
| See categories and tags                 | `tags:view`         |
| See tags on an endpoint                 | `tags:view`         |
| Subscribe to tags (own account)         | `self:access`       |
| Create / rename / delete categories     | `tags:edit`         |
| Create / rename / delete tags           | `tags:edit`         |
| Assign or replace tags on an endpoint   | `endpoints:edit`    |

In the default roles:

- **Admin** — full access.
- **Operator** — can create and assign tags.
- **Viewer** — can see tags and manage their own subscriptions, but cannot
  create or assign them.

## Assigning tags to endpoints

Open an endpoint and use the **Tags** panel to pick values from any category.
The assignment is a full replacement — the list you save becomes the complete
set of tags on that endpoint. Removing a tag here does not delete the tag
itself, only the assignment.

Deleting a category cascades to its tags and to every endpoint assignment
that references them. Deleting an individual tag cascades to its assignments
and to any user subscriptions pointing at it.

## Subscriptions

Every user can subscribe to any number of tags from their own **Profile →
Notifications** view. Subscriptions are scoping rules for certificate expiry
alerts: when the alert job runs, each user only receives alerts for
certificates on endpoints that share at least one of their subscribed tags.

### The notify-all fallback

A user with **zero subscriptions** receives alerts for *every* endpoint,
not none. This is deliberate: a fresh install with no tags configured still
sends useful alerts, and an admin who never opens the subscriptions page is
not silently cut off.

The practical rule:

- Subscribe to one or more tags → you are scoped to those tags.
- Subscribe to nothing → you receive everything.

If you want to opt out of alerts entirely, toggle the **Notify me of
expirations** flag on your profile instead. That flag is the hard on/off;
subscriptions only filter *which* alerts are sent when it is on.

### Dedup and thresholds

Alerts fire at configurable day thresholds (by default 30 / 14 / 7 / 1 days
before expiry). Each `(user, certificate, threshold)` tuple is delivered at
most once — re-running the alert job, or a cert staying inside a threshold
window across days, will not cause repeat emails.

## Example

A two-region SaaS team might configure:

| Category       | Tags                                      |
|----------------|-------------------------------------------|
| `Environment`  | `production`, `staging`, `dev`            |
| `Application`  | `checkout-api`, `billing`, `marketing-site` |
| `Owner`        | `sre-team`, `checkout-team`, `growth-team` |
| `Datacenter`   | `iad1`, `fra1`                             |

A billing-team engineer would then subscribe to `Application:billing` and
`Owner:sre-team`, receiving alerts for any cert tied to either. The SRE
on-call, who owns every production endpoint, would subscribe to
`Environment:production` — or leave subscriptions empty to inherit the
notify-all default.

## API

These endpoints back the UI and are available to any client:

| Method | Path                               | Permission          |
|--------|------------------------------------|---------------------|
| GET    | `/api/v1/tags`                     | `tags:view`         |
| GET    | `/api/v1/tags/categories`          | `tags:view`         |
| POST   | `/api/v1/tags/categories`          | `tags:edit`         |
| PUT    | `/api/v1/tags/categories/{id}`     | `tags:edit`         |
| DELETE | `/api/v1/tags/categories/{id}`     | `tags:edit`         |
| POST   | `/api/v1/tags`                     | `tags:edit`         |
| PUT    | `/api/v1/tags/{id}`                | `tags:edit`         |
| DELETE | `/api/v1/tags/{id}`                | `tags:edit`         |
| GET    | `/api/v1/endpoints/{id}/tags`      | `tags:view`         |
| PUT    | `/api/v1/endpoints/{id}/tags`      | `endpoints:edit`    |
| GET    | `/api/v1/me/tag-subscriptions`     | `self:access`       |
| PUT    | `/api/v1/me/tag-subscriptions`     | `self:access`       |

`GET /tags` returns a flat, category-sorted list (useful for filter pickers);
`GET /tags/categories` returns the same data grouped by category. Both go
through the same permission check.

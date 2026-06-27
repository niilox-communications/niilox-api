# Multi-tenancy

**Niilox API** hosts multiple branded apps (Drift, GeoGig, Rodent, Rabbaly, partner tenants) on one deployment. Tenants share:

- the binary,
- the Postgres database,
- the WebSocket hub,
- the (optional) Rust media-svc.

Tenants do **not** share:

- broadcast SFU projects (each tenant has its own host, key, secret — `drift` reference app only),
- Niilox media storage zones,
- payment channel credentials,
- adult-content policy (`apps.allow_adult`),
- user identities (the same account can have separate user rows per tenant).

## The `apps` table

Created by `migrations/008_apps.sql`:

```sql
CREATE TABLE apps (
  id                TEXT PRIMARY KEY,
  name              TEXT NOT NULL,
  sfu_host          TEXT NOT NULL DEFAULT '',
  sfu_key           TEXT NOT NULL DEFAULT '',
  sfu_secret        TEXT NOT NULL DEFAULT '',
  media_zone        TEXT NOT NULL DEFAULT '',
  media_key         TEXT NOT NULL DEFAULT '',
  media_cdn         TEXT NOT NULL DEFAULT '',
  media_region      TEXT NOT NULL DEFAULT '',
  payment_channel   TEXT NOT NULL DEFAULT 'card',
  allow_adult       BOOLEAN NOT NULL DEFAULT false,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

> **Note:** Production column names may differ; Niilox ops maps credentials into `apps` on provision. Integrators only need `X-App-ID`.

`apps.id` is the value that clients send in `X-App-ID`. The default tenant id is `drift` and is upserted automatically on boot from ops-provisioned credentials.

## Adding a new tenant

New tenants are provisioned via the [developer portal](https://www.niilox.com) or by Niilox ops. Example of what gets stored per tenant:

| Field | Example |
|-------|---------|
| `id` | `rabbaly` |
| SFU host | `wss://sfu.example.com` (drift-style livestream only) |
| Media zone | `rabbaly-media` |
| Media CDN | `https://cdn.example.com` |
| Payment channel | `card` or `bank` |

Clients then call:

```
GET /api/v1/rooms
Authorization: Bearer <user_jwt>
X-App-ID: rabbaly
```

and get only Rabbaly rooms, Rabbaly livestream tokens, Rabbaly notifications.

## How the API knows the tenant

The decision happens once per request in `middleware.Tenant`:

```go
appID := r.Header.Get("X-App-ID")    // defaults to "drift"
app, ok := registry.Get(appID)        // 400 if unknown
ctx = context.WithValue(ctx, AppKey, app)
```

Handlers then call `middleware.GetApp(ctx)` or `middleware.GetAppID(ctx)`.

## Livestream room names (`drift` tenant)

`tenant.RoomName(appID, hostID)` builds `{appID}::{hostID}::{yyyymmddhhmmss}` so a livestream webhook can recover the tenant from the room name even though webhooks don't carry custom headers:

```go
func ParseRoomAppID(roomName string) string {
    if i := strings.Index(roomName, "::"); i > 0 {
        return roomName[:i]
    }
    return "drift"
}
```

Webhook handlers and moderation/stage managers use this to dispatch to the right tenant's SFU credentials.

## Per-tenant services

`apps.Registry` exposes lazy-cached clients per tenant:

- **Livestream SFU** — broadcast rooms (`drift` reference app)
- **Media storage** — avatars, DMs, ID docs, thumbnails

Payments are handled by `payments.Registry`, which is process-wide — the tenant chooses **which channel** to use via `apps.payment_channel` (`card`, `bank`, or `mobile_iap`).

## Data isolation

Every tenant-scoped table has an `app_id` column with a foreign key to `apps(id)`. Indexes are also (app_id, …):

| Table | Composite indexes |
|-------|-------------------|
| `users` | unique `(app_id, supabase_id)` |
| `rooms` | partial unique on active rooms by host, partial index on `(app_id, ended_at IS NULL)` |
| `chat_messages` | `(app_id, room_id, created_at DESC)` |
| `gift_transactions` | `(app_id, room_id, created_at DESC)` |
| `notifications` | `(app_id, user_id, created_at DESC)` and partial unread |
| `follows`, `direct_messages`, `withdrawal_requests`, `user_socials`, `id_verification_requests` | all scoped by `app_id` |

Every query in `internal/*/handler.go` filters by `app_id` — this is how the same Postgres database can host multiple distinct products without leakage.

## Adult-content gating

Three columns gate adult streams. Hosts and viewers see different gates:

- `apps.allow_adult` — tenant-wide toggle. Defaults to `false` on the seeded `drift` row. Flip it once with SQL **or** list the tenant id in the `ALLOW_ADULT_APPS` env var (e.g. `ALLOW_ADULT_APPS=drift`) — the server runs an idempotent `UPDATE apps SET allow_adult = true WHERE id = ANY($1)` on boot.
- `users.id_verified` — **hosts only**. Set by `/api/v1/admin/verifications/{id}/approve` after the host uploads a government ID + selfie.
- `users.age_confirmed_at` — **viewers**. Stamped by `POST /users/me/confirm-age` the first time the user accepts the in-app +18 modal. No paperwork; viewers just tick "I'm 18+".

Host gate — `POST /rooms`:

| `apps.allow_adult` | `is_adult` on body | Result |
|--------------------|--------------------|--------|
| `false` | `true` (or `category = adult`) | 403 `adult streams not allowed on this app` |
| `true`  | `true` and host `id_verified = false` | 403 `id verification required for adult streams` |
| `true`  | `true` and host `id_verified = true`  | created |

Viewer gate — `POST /rooms/{id}/join` on an adult room:

| Caller | `age_confirmed_at` | Result |
|--------|-------------------|--------|
| signed-in user | NULL  | 403 `age confirmation required to watch adult streams` |
| signed-in user | set   | joined |
| guest token    | n/a   | 403 `sign in to watch adult streams` |

The two host-side guards run in order — if the tenant flag is off you'll see the first message even on a verified account. Inspect the body of the 403, not just the status code, when you're debugging "but I'm verified" reports.

## Private / VIP rooms

Private rooms layer on top of every existing gate without changing tenant isolation. Hosts configure pricing and seat caps at go-live; access is enforced server-side.

Full transaction semantics and realtime lobby updates are provided to registered developers — contact **dev@niilox.com**.

## What can't be multi-tenant yet

The hub, stage manager, moderation manager are in-memory and **per-process**, but they keyed by `roomID` (a UUID from the `rooms` table, which is already tenant-unique). They work multi-tenant out of the box — you just don't get one hub per tenant, you get one hub across all tenants. No data leaks between rooms because each room belongs to exactly one tenant.

If you need stricter isolation (e.g. per-tenant rate limits, per-tenant Prometheus labels), wrap the hub or add middleware. The pattern is the same — pull `appID` off the context.

# SMS & phone numbers

Send transactional SMS from your backend using **Niilox API keys**. Niilox operates delivery — integrators call our API only; no carrier credentials on your side.

## Enable SMS

Check whether SMS is live for your tenant:

```http
GET /api/v1/sms/status
Authorization: Bearer niilox_sk_…
X-App-ID: your-tenant
```

```json
{ "ok": true, "enabled": true, "app_id": "your-tenant" }
```

If `enabled` is `false`, contact **dev@niilox.com** to provision SMS on your tenant.

## Per-tenant sender

Each `X-App-ID` sends SMS from **its own sender** — Rodent texts show as Rodent, GeoGig as GeoGig, and so on. Configure in the developer portal under **SMS & numbers**, or via the API:

| Field | Meaning |
|-------|---------|
| `sms_from` | E.164 number or alphanumeric sender (e.g. `Rodent`) |
| `sms_messaging_profile_id` | Messaging profile UUID from your onboarding pack |

Set **one** of the above per tenant — not both.

```http
GET /api/v1/sms/sender
PATCH /api/v1/sms/sender
GET /api/v1/platform/sms
PATCH /api/v1/platform/sms
X-App-ID: your-tenant
```

Only Niilox ops can toggle `sms_enabled` for a tenant.

## API key

Create a key in [www.niilox.com](https://www.niilox.com) → **API keys** with scope **`sms:send`** (or `write` / `admin`).

```http
Authorization: Bearer niilox_sk_…
X-App-ID: your-tenant
Content-Type: application/json
```

**Server-side only** — never ship API keys in mobile or browser bundles.

## SMS endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/v1/sms/status` | API key or admin JWT | Whether SMS is enabled for the tenant |
| GET | `/api/v1/sms/sender` | API key or admin JWT | Per-tenant sender config |
| PATCH | `/api/v1/sms/sender` | API key (`write` / `sms:send`) or admin JWT | Update sender |
| POST | `/api/v1/sms/send` | API key (`sms:send`) or admin JWT | Single SMS |
| POST | `/api/v1/sms/bulk` | API key (`sms:send`) or admin JWT | Up to 100 recipients |

### Single send

```json
POST /api/v1/sms/send
{
  "to": "+2348012345678",
  "message": "Your order is ready."
}
```

Response:

```json
{ "ok": true, "to": "+2348012345678" }
```

### Bulk — same message to many numbers

```json
POST /api/v1/sms/bulk
{
  "to": ["+2348012345678", "+15551234567"],
  "message": "Gig starts in 30 minutes."
}
```

### Bulk — different message per recipient

```json
POST /api/v1/sms/bulk
{
  "messages": [
    { "to": "+2348012345678", "message": "Hi Ada" },
    { "to": "+15551234567", "message": "Hi Sam" }
  ]
}
```

Bulk response (partial success allowed):

```json
{
  "ok": false,
  "sent": 1,
  "failed": 1,
  "total": 2,
  "results": [
    { "to": "+2348012345678", "ok": true },
    { "to": "+15551234567", "ok": false, "error": "send failed" }
  ]
}
```

Phone numbers must be **E.164** (`+` country code + digits).

## Phone numbers (SDK: `client.numbers`)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/v1/numbers/status` | API key or admin JWT | Whether search/purchase is available |
| GET | `/api/v1/numbers/markets` | API key or admin JWT | Supported countries (pricing TBD) |
| GET | `/api/v1/numbers` | API key or admin JWT | Inventory + configured sender |
| POST | `/api/v1/numbers/search` | API key or admin JWT | Search available numbers (501 until live) |
| POST | `/api/v1/numbers/purchase` | API key or admin JWT | Buy a number (501 until live) |

Number search and purchase are rolling out market by market. Until live, attach your own sender with `PATCH /sms/sender` or the portal **SMS & numbers** page.

## SDK (`@niilox/sdk`)

```ts
import { createNiiloxClientFromEnv } from '@niilox/sdk'

const client = createNiiloxClientFromEnv() // NIILOX_APP_ID + NIILOX_API_KEY

await client.sms.status()

await client.sms.getSender()
await client.sms.updateSender({ sms_from: '+15551234567' })

await client.sms.send({
  to: '+2348012345678',
  message: 'Ping from Niilox',
})

await client.sms.bulk({
  to: ['+2348012345678', '+2348098765432'],
  message: 'Shift reminder',
})

await client.numbers.list()
await client.numbers.markets()
```

Python:

```python
from niilox import Client

client = Client.from_env()
client.sms_get_sender()
client.sms_update_sender(sms_from="+15551234567")
client.sms_bulk(to=["+2348012345678"], message="Hello")
client.numbers_list()
```

## Notes

- **OTP sign-in** (`POST /auth/phone/send`) is a separate end-user flow with its own rate limits.
- **GeoGig safety** alerts use the same Niilox SMS pipe but are triggered internally, not via these routes.
- Compliance (opt-in, STOP keywords, regional rules) is the integrator's responsibility.

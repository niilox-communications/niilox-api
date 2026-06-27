# Niilox developer workflow

A single path from sign-up to production — without exposing operator secrets, vendor SFU details, or Niilox-hosted media storage.

| | |
|---|---|
| **API** | `https://api.niilox.com/api/v1` |
| **Health** | `https://api.niilox.com/health` → `{"ok":true}` |
| **Portal** | [www.niilox.com](https://www.niilox.com) — keys, billing, SDK snippets |
| **Contact** | dev@niilox.com |

> **Uptime / incidents:** Niilox does not publish a public status page with internal infrastructure names. Registered developers receive incident notice by email. Contact **dev@niilox.com** for SLA and status subscriptions.

---

## What Niilox is (for integrators)

**Niilox API** is a multi-tenant backend for:

- **Auth & tenants** — JWT sessions, phone OTP, API keys, usage
- **Calls & messaging** — Rodent peer signaling, DMs, personal WebSocket (`/ws/me`)
- **Marketplaces** — GeoGig gigs, checkout, reviews, worker safety
- **Payments** — Niilox hosted checkout (`card` / `bank` / `mobile_iap`), wallets, payouts
- **Nigeria KYC** — `@niilox/kyc-ng` (bank-transfer identity; licensed integrators)

**Niilox is not:**

- A **media CDN** for your product — host images/video in **your** S3, R2, or Cloudflare bucket; pass URLs to the API
- A **broadcast SFU** for every tenant — Niilox livestream is used only by the **driftin.live** reference app (`X-App-ID: drift`)

---

## Product paths

| Your app | Tenant | Realtime video | Messaging |
|----------|--------|----------------|-----------|
| Gig / field / booking app | `geogig`, `rodent`, or your id | **P2P** (`peer`, gig video-call) | DMs + `/ws/me` |
| Marketplace + chat | your id | **P2P** or REST-only | DMs + push |
| Public livestream (reference) | `drift` | **driftin.live only** (SFU) | Room chat + gifts |

**Default recommendation for new tenants:** Rodent-style **peer signaling** + **DMs** — not Niilox livestream rooms.

Optional **Rust media cop** (server-side quality watch) applies to the drift livestream stack only; it is not required for P2P tenants.

---

## Workflow (≈30 minutes)

### 1. Create a tenant

1. Open [www.niilox.com](https://www.niilox.com) → **Sign in** (Google or email).
2. **Create account** → choose `app_id` (e.g. `myapp`) and company name.
3. Copy your **API key** (`niilox_sk_…`) — shown once on create.

### 2. Verify the API

```bash
curl -s https://api.niilox.com/health

curl -s https://api.niilox.com/api/v1/platform/ping \
  -H "Authorization: Bearer niilox_sk_YOUR_KEY" \
  -H "X-App-ID: myapp"
```

Every request must include **`X-App-ID: myapp`**.

### 3. Install the SDK

Monorepo:

```json
"@niilox/sdk": "file:../packages/niilox-sdk"
```

```bash
cd packages/niilox-sdk && npm install && npm run build
```

Nigeria bank KYC (licensed):

```json
"@niilox/kyc-ng": "file:../packages/niilox-kyc-ng"
```

### 4. Sign in a test user

```ts
import { createNiiloxClient } from '@niilox/sdk'

const client = createNiiloxClient({
  appId: 'myapp',
  apiKey: process.env.NIILOX_API_KEY,
})

const guest = await client.auth.guest()
const userClient = createNiiloxClient({
  appId: 'myapp',
  token: guest.token,
})
```

Or phone OTP / magic link — see [Native auth](./NATIVE_AUTH.md).

### 5. Calls & messaging (recommended)

| Need | API / SDK |
|------|-----------|
| 1:1 or small-group video | `client.peer.*`, `POST /gigs/{id}/video-call` |
| ICE / TURN | `GET /peer/ice` |
| Signaling | `wss://api.niilox.com/api/v1/peer/signal?app_id=…` |
| DMs + typing | `client.dms.*`, `PersonalSocket` |
| Live events feed | `wss://api.niilox.com/ws/me?token=JWT` |

See [Peer signaling](./PEER_SIGNAL.md).

### 6. Storage — bring your own

Upload assets to **your** storage (S3, R2, Cloudflare, etc.). Send **HTTPS URLs** to Niilox:

- Gig images, avatars, CV links, DM media metadata
- Do **not** expect Niilox to host your product’s media library

Niilox stores **identity, commerce, and session state** — not your content CDN.

### 7. Payments & webhooks

- Configure ops-provisioned payment webhooks on **your** API tenant
- Webhook endpoints are set up by Niilox ops (see [Payments](./PAYMENTS.md))
- Keep secrets server-side only

### 8. Nigeria workers (optional)

```ts
import { createKycNgClient } from '@niilox/kyc-ng'

const kyc = createKycNgClient({ appId: 'geogig', token: userJwt })
await kyc.bank.start({ finish_url: 'myapp://', amount_ngn: 100 })
```

Payout account name must match verified bank identity. Contact **dev@niilox.com** for `@niilox/kyc-ng` access.

### 9. Ship checklist

- [ ] `X-App-ID` on every request
- [ ] User JWT for user routes; API key only for `/platform/*` and server jobs
- [ ] WebSocket reconnect with backoff
- [ ] BYO storage for images/video
- [ ] Webhooks verified (signatures)
- [ ] No secrets in mobile bundles
- [ ] Full integration guide from dev@niilox.com for paywall, safety, and KYC detail

---

## driftin.live (reference only)

The consumer app at [driftin.live](https://www.driftin.live) uses tenant **`drift`** and includes broadcast livestream features. That stack is **not** the default Niilox integration path.

| | New tenants | drift reference |
|--|-------------|-----------------|
| API host | `api.niilox.com` | `api.driftin.live` (legacy alias during cutover) |
| Video | P2P / Rodent | SFU livestream |
| Docs focus | This workflow | [API.md](./API.md) room routes |

---

## DNS cutover (operators)

1. CNAME **`api.niilox.com`** → same origin as current API (or new load balancer).
2. TLS certificate for `api.niilox.com`.
3. Set `API_PUBLIC_URL=https://api.niilox.com` on the Go API.
4. Portal Vercel: `API_PROXY_TARGET=https://api.niilox.com` (or legacy host until DNS propagates).
5. Keep `api.driftin.live` as alias for **drift** tenant until deprecated.

---

## Related docs

| Doc | Purpose |
|-----|---------|
| [Getting started](./GETTING_STARTED.md) | First tenant, first curl |
| [Wiring checklist](./WIRING_CHECKLIST.md) | Launch checklist |
| [API reference](./API.md) | All routes |
| [Peer signaling](./PEER_SIGNAL.md) | P2P / Rodent |
| [GeoGig](./GEOGIG.md) | Gigs marketplace |
| [Payments](./PAYMENTS.md) | Niilox hosted checkout, webhooks |

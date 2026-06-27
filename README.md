# Niilox API — developer documentation

**Multi-tenant platform API** for live video, local gigs, peer signaling, authentication, and billing.

| | |
|---|---|
| **API** | `https://api.niilox.com/api/v1` |
| **Health** | `https://api.niilox.com/health` |
| **Developer portal** | [www.niilox.com](https://www.niilox.com) — keys, billing, usage, in-browser docs |
| **Support** | dev@niilox.com (uptime & SLA — no public status page with internal hostnames) |

Every request needs header **`X-App-ID: <your_tenant>`**. User actions use a session JWT; server integrations use **`niilox_sk_…`** API keys.

> **Start here:** [**Developer workflow**](./DEVELOPER_WORKFLOW.md) — sign-up → SDK → P2P/messaging → BYO storage (no broadcast SFU required).

> **Public docs:** Step-by-step flows for VIP access, worker safety, geofence logic, and payment transactions are provided to registered developers. Contact **dev@niilox.com** for the full integration guide.

---

## Platform progress & what's left

**→ [PLATFORM_STATUS.md](./PLATFORM_STATUS.md)** — backend progress (operator doc).

Quick summary:

| Layer | Status |
|-------|--------|
| Go API + Postgres + native auth | **Production** |
| P2P calls, DMs, Rodent / GeoGig | **Production** |
| `@niilox/sdk` + `@niilox/kyc-ng` | **v0.1 beta** — monorepo |
| driftin.live livestream (reference) | **Production** — `drift` tenant only |

---

## Quick start

**1. Create a tenant** at [www.niilox.com](https://www.niilox.com) → sign in → **Create account** → pick an app id (e.g. `myapp`).

**2. Ping the API** with your key:

```bash
curl -s https://api.niilox.com/health
# {"ok":true}

curl -s https://api.niilox.com/api/v1/platform/ping \
  -H "Authorization: Bearer niilox_sk_YOUR_KEY" \
  -H "X-App-ID: myapp"
```

**3. Sign in a test user** (guest, 24 h):

```bash
curl -s https://api.niilox.com/api/v1/auth/guest \
  -H "X-App-ID: myapp" \
  -H "Content-Type: application/json" \
  -d "{}"
```

Full walkthrough → [**Getting started**](./GETTING_STARTED.md) (~15 min).

### Official SDK (`@niilox/sdk` v0.1 beta)

See **[Platform status](./PLATFORM_STATUS.md)** for the full backend + frontends + ops picture.  
SDK detail: monorepo `packages/niilox-sdk/README.md` (not on npm yet).

---

## Documentation

### Platform & SDK

| Guide | For |
|-------|-----|
| [**Developer workflow**](./DEVELOPER_WORKFLOW.md) | **Recommended path** — P2P, messaging, BYO storage |
| [**Platform status**](./PLATFORM_STATUS.md) | Operator backlog |

### Start here

| Guide | For |
|-------|-----|
| [**Getting started**](./GETTING_STARTED.md) | First tenant, first API call |
| [**Wiring checklist**](./WIRING_CHECKLIST.md) | End-to-end launch checklist |
| [**Developer integration**](./DEVELOPER_INTEGRATION.md) | Day-to-day flows, WebSockets |
| [**API reference**](./API.md) | Every endpoint |

### Auth & users

| Guide | For |
|-------|-----|
| [**Native auth**](./NATIVE_AUTH.md) | Google, Apple, magic link, **phone SMS OTP**, passwords |
| [**SMS API**](./SMS.md) | Bulk SMS, sender config, phone numbers (`client.sms`, `client.numbers`) |
| [**Authentication overview**](./AUTHENTICATION.md) | JWT model, guest tokens, refresh |

### Products on Niilox

| Guide | Tenant | Stack |
|-------|--------|-------|
| [**Peer signaling**](./PEER_SIGNAL.md) | `rodent`, `geogig` | WebRTC ICE + signal — default for calls |
| [**GeoGig**](./GEOGIG.md) | `geogig` | Gigs, fiat checkout, P2P video, worker safety |
| [**Worker safety**](./WORKER_SAFETY.md) | `geogig` (+ others) | Field-worker safety — guide on request |
| driftin.live livestream | `drift` only | Reference app — contact dev@niilox.com for room routes |

Reference apps: [GeoGig](https://github.com/Dr-M06/geogig) · [Drift](https://driftin.live) (reference)

### Payments & no-code

| Guide | For |
|-------|-----|
| [**Payments**](./PAYMENTS.md) | Token packs, Niilox hosted checkout, webhooks |
| [**Mobile payments**](./MOBILE_PAYMENTS.md) | `mobile_iap` channel / App Store / Play |
| [**Bubble.io**](./BUBBLE_IO.md) | No-code API Connector setup |

### Platform

| Guide | For |
|-------|-----|
| [**Multi-tenant**](./MULTI-TENANT.md) | `X-App-ID`, isolation, provisioning |
| [**Security**](./SECURITY.md) | Auth model, tenant isolation, payments |

---

## First-party tenants

| `X-App-ID` | Product |
|------------|---------|
| `drift` | Drift — live streaming, chat, gifts, VIP rooms |
| `geogig` | GeoGig — local gigs, safety, P2P video glance |
| `rodent` | Rodent — peer sessions, Drop, paid bookings |
| `rabbaly` | Rabbaly |
| *yours* | Provision via the [developer portal](https://www.niilox.com) |

---

## Credentials (do not mix)

| Credential | Header | Used for |
|------------|--------|----------|
| **Tenant API key** | `Authorization: Bearer niilox_sk_…` | Server / Bubble backend — `/platform/*` |
| **User session JWT** | `Authorization: Bearer <access_token>` | End users — rooms, gigs, chat, wallet |

---

## About this repo

This repository is the **public documentation mirror** for integrators. It contains guides only — no server source code or production secrets.

- **In-browser docs:** [www.niilox.com/portal/dashboard/docs](https://www.niilox.com/portal/dashboard/docs)
- **About Niilox:** [www.niilox.com/about](https://www.niilox.com/about)

Questions or integration access: [dev@niilox.com](mailto:dev@niilox.com) or open an issue on this repo.

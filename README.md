# Niilox Monitor — stream observability for your LiveKit

**Watch your SFU rooms for dead air.** You keep LiveKit; Niilox stores events and incidents only.

| | |
|---|---|
| **API** | `https://api.niilox.com/api/v1/monitor/*` |
| **Status** | [`https://api.niilox.com/health`](https://api.niilox.com/health) |
| **Portal** | [www.niilox.com](https://www.niilox.com) — Projects, Incidents, Alerts, Keys |
| **Support** | [dev@niilox.com](mailto:dev@niilox.com) |

> **Start here:** [**MONITOR_API.md**](./MONITOR_API.md) — event schema + routes  
> Sidecar: run next to your LiveKit ([STREAM_HEALTH.md](./STREAM_HEALTH.md) for the hosted still-live path is first-party only).

This repository is **docs only** — no server source, no livestream/rooms/gifts SDK.

---

## What you get

1. **Sidecar mode** — `niilox-monitor` holds your LiveKit credentials locally, posts `room.idle` / track events to Niilox. Frame analysis never uploads video.
2. **Webhook-only mode** — point LiveKit (or your proxy) at `/monitor/webhooks/livekit/{project_id}?token=…` for presence signals without a sidecar.
3. **Alerts** — outbound webhooks; cleanup is opt-in on **your** endpoint or local sidecar — Niilox does not DeleteRoom with your keys.

## What you do not get

Public tenants do **not** receive Niilox rooms, gifts, stage, seats, or Drift OAuth finish URLs. Those stay first-party (`drift` / allowlisted).

## Quick checks

```bash
curl -s https://api.niilox.com/health

curl -s https://api.niilox.com/api/v1/monitor/projects \
  -H "Authorization: Bearer niilox_sk_YOUR_KEY" \
  -H "X-App-ID: myapp"
```

Create a tenant at [www.niilox.com](https://www.niilox.com). New apps are **monitor-only**; API keys default to scope `monitor`.

## Docs

| Guide | For |
|-------|-----|
| [**MONITOR_API.md**](./MONITOR_API.md) | Event schema, projects, incidents, alerts |
| [**SECURITY.md**](./SECURITY.md) | Tenant isolation |
| [**MULTI-TENANT.md**](./MULTI-TENANT.md) | `X-App-ID` |

## Auth (standalone)

OAuth / magic-link finish URLs must be **your** app origin. Failures never redirect end users to driftin.live. Configure finish URLs in the portal; host callbacks via `https://api.niilox.com` (or `auth.niilox.com` when provisioned).

## About

**Canonical:** [niilox-communications/niilox-api](https://github.com/niilox-communications/niilox-api)

Topics: `observability`, `monitoring`, `livekit`, `webrtc`

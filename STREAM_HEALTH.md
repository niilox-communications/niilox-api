# Stream health vs Monitor product

## Public product (this repo)

**BYO LiveKit** → [MONITOR_API.md](./MONITOR_API.md) + customer sidecar. You do not use Niilox `/rooms`.

## First-party only (Drift)

Hosted still-live prompts on Niilox SFU rooms (`GET /livekit/monitor/status`, `room:still_live_prompt`) remain available to allowlisted livestream tenants (`drift`, …). That path is **not** the public Monitor SKU.

New portal apps are monitor-only and receive **403** on `/rooms*` / gifts / stage.

# Monitor API (v1)

Public product: **BYO LiveKit observability**. Niilox stores events and incidents only.

Base: `https://api.niilox.com/api/v1`

Auth: `Authorization: Bearer niilox_sk_…` with scope **`monitor`**, plus `X-App-ID`.

Sidecar ingest may use the one-time `nm_ingest_…` key from `POST /monitor/projects` instead of an API key.

## Event envelope

```json
{
  "schema_version": 1,
  "project_id": "prj_…",
  "idempotency_key": "uuid",
  "observed_at": "2026-10-01T08:00:00Z",
  "room": { "external_id": "lk-room-name", "display_name": "optional" },
  "participant": { "identity": "host-1", "track_sid": "optional" },
  "type": "track.frozen",
  "severity": "warn",
  "detail": "human string",
  "data": {}
}
```

Types: `room.idle`, `room.ghost`, `room.resolved`, `track.missing`, `track.frozen`, `track.black`, `track.silent`.

## Routes

| Method | Path | Notes |
|--------|------|-------|
| POST | `/monitor/projects` | Returns `ingest_key` once |
| GET | `/monitor/projects` | List |
| POST | `/monitor/events` | Sidecar ingest |
| GET | `/monitor/incidents?status=open` | Incidents |
| POST | `/monitor/alert-endpoints` | Customer webhook |
| GET | `/monitor/alert-endpoints` | List |
| POST | `/monitor/webhooks/livekit/{id}?token=` | Webhook-only presence |

## Cleanup

Default: alert only. Auto-end rooms with **your** LiveKit credentials in the sidecar, or via a cleanup webhook you host. Niilox never stores write access to your SFU for DeleteRoom.

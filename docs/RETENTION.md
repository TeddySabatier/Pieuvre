# Data retention

What Pieuvre keeps in Postgres, for how long, and when to purge.

Related: [CREDENTIALS.md](CREDENTIALS.md) · [PHASES.md](PHASES.md)

---

## Policy (V0 — Option A)

| Data | Retention | Purge trigger |
|---|---|---|
| **Traces / trace_steps** | **90 days** | Scheduled job deletes rows older than `retention.traces_days` |
| **Resources** (normalized extracts) | Until source deleted | Source webhook, manual purge, or confirmed deletion |
| **Embeddings** | Linked to Resource | Cascade-delete when Resource purged |
| **CrossLinks** | Until either endpoint purged | Cascade when either `resource_id` removed |
| **Raw payload snapshots** | Minimal | Discard on staleness + confirmed change (no long-term store) |
| **conversation_states** | 30 days after `done` | Cleanup job (threads rarely revisited) |
| **slack_events_seen** | 7 days | Idempotency only — short TTL |

---

## Implementation (Phase 6)

- Nightly cron job (or pg-boss scheduled job): `purge_expired_traces`, `purge_stale_conversation_states`
- On GitHub/Slack/Notion delete events: cascade purge Resource + embeddings + CrossLinks
- Admin: `pieuvre purge --resource-id ...` (global admins only)

---

## Config

Defaults baked in; override via env if needed:

```bash
# .env (optional overrides)
PIEUVRE_RETENTION_TRACES_DAYS=90
PIEUVRE_RETENTION_CONVERSATION_STATE_DAYS=30
PIEUVRE_RETENTION_SLACK_EVENT_DEDUP_DAYS=7
```

Future: `retention:` block in global config YAML (same values, no per-project retention in V0).

---

## Privacy notes

- Normalized Slack extracts may contain message text — treat Postgres backups as sensitive
- Traces may include retrieved snippets — redact tokens in trace writer
- Review employer Slack/GitHub/Notion policies (RESEARCH §13 spike)

---

## Disk planning (rough)

Small team V0: traces dominate if unbounded — **90-day cap** keeps Postgres typically under a few GB. Resources scale with indexed Notion/GitHub/Slack volume, not trace volume.

# ADR-0001 — Postgres-only for data, queue, vectors, and traces

**Status:** Accepted · **Phase:** 0 · Relates to [DECISIONS.md Q1](../DECISIONS.md)

## Context

Pieuvre is a self-hosted (Docker Compose) knowledge service for a small internal team. V0 needs durable storage for resources, a job queue for ingest/index/agent work, vector search for retrieval, and an audit trace store. Adding services (Redis, a dedicated vector DB, a tracing backend) multiplies operational surface: more containers to run, back up, monitor, and secure.

## Decision

Use **PostgreSQL as the only datastore in V0**:

- **Data:** `resources`, `projects`, `cross_links`, `conversation_states`.
- **Queue:** Postgres-backed (`jobs` table via pg-boss / graphile-worker) — **no Redis**.
- **Vectors:** `pgvector` extension; embeddings on prose resources only.
- **Traces:** `traces` / `trace_steps` rows.

## Consequences

- One container to operate, back up, and secure; simplest path to a daily-usable MVP.
- Trace UI/queries are hand-rolled (acceptable for internal team in V0).
- Vector scale ceiling ~1M chunks before a dedicated store is worth it.
- LangFuse and external vector DBs remain **optional upgrades post–Phase 6**, not V0 dependencies.

## Alternatives considered

- **Redis for queue/cache:** faster, but in-memory durability concerns and an extra service for no V0 benefit.
- **Pinecone/Qdrant:** only justified beyond pgvector's scale ceiling.
- **LangSmith (cloud) / LangFuse (self-hosted) for traces:** data egress (LangSmith) or extra container (LangFuse) not warranted in V0.

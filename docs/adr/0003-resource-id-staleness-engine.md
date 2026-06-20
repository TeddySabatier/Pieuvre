# ADR-0003 — Universal `resource_id` + shared staleness engine

**Status:** Accepted · **Phase:** 0 (seam), 1 (lite), 6 (full) · Relates to [DECISIONS.md Q14](../DECISIONS.md) · [RESEARCH.md §5](../../RESEARCH.md)

## Context

Every layer — citations in Slack, cross-links in the graph, cache invalidation, MCP callbacks, traces — needs to reference the same external artifact unambiguously. Freshness logic (TTL, revalidation, hash diff, cascade invalidation) is identical in shape across Slack, GitHub, and Notion; implementing it per connector would triple the bug surface and risk policy drift.

## Decision

Two coupled seams, stable from Phase 0:

1. **Universal `resource_id`** — one string scheme per source (`slack:{team}:{channel}:{ts}`, `github:{owner}/{repo}:issue:{n}`, `notion:{page_id}`, …). It is the join key for citations, `cross_links`, cache records, and trace references.
2. **Shared staleness engine** — one module owns the freshness lifecycle (`fresh → revalidate → invalidate`, cascade to embeddings). Adapters supply metadata/content only; they never decide freshness.

Decision order: `updated_at` / validator first → hash diff fallback → discard + re-search on confirmed change.

## Consequences

- Linking is seamless from Phase 2 on: citations, graph edges, and cache all speak the same ID.
- Freshness rules are written and tested once; adding a connector means a new adapter, not new cache logic.
- Phase 1 ships a "lite" engine (TTL + `updated_at`); Phase 6 adds hash diff + cascade — same interface, no rewrite.

## Alternatives considered

- **Per-connector IDs + freshness:** flexible per source, but breaks cross-source linking and duplicates invalidation logic (policy drift).
- **Hash-only freshness:** simpler, but wastes bandwidth re-fetching full content when cheap validators (`ETag`, `last_edited_time`) exist.

# ADR-0002 — In-process read adapters; MCP for writes

**Status:** Accepted · **Phase:** 0 (seam), 5 (first write) · Relates to [DECISIONS.md Q9, Q24](../DECISIONS.md) · [CREDENTIALS.md](../CREDENTIALS.md)

## Context

Pieuvre reads from Slack, GitHub, and Notion at high volume (monitoring, scans, revalidation) and writes to Notion at low volume (task create/update after human confirmation). Reads are latency-sensitive and called on most messages; writes must be isolated, auditable, and easy to gate behind confirmation.

## Decision

Split the connector boundary by direction:

- **Reads:** thin **in-process TypeScript `ReadAdapter`s** (`sourceType`, `getMetadata`, `getCanonicalContent`, `normalize`). No business logic in adapters — they only normalize source data into Resources. Contract is the canonical seam in [PHASES.md](../PHASES.md).
- **Writes:** all mutations go through **literal MCP servers** (Notion first). The agent calls MCP tools; it never hits a source write API directly. Write tokens live on the MCP server env, not the Pieuvre worker.

## Consequences

- Read path stays fast and testable; new sources = implement one adapter contract.
- Write path is isolated and auditable — every mutation is an MCP tool call logged in a trace.
- Credential split is natural: read tokens on the worker, write tokens on MCP (see [CREDENTIALS.md](../CREDENTIALS.md)).
- GitHub/Slack writes are deferred post-V0 with no architecture change — add an MCP server.

## Alternatives considered

- **All connectors via MCP:** uniform but adds latency/overhead to high-volume reads.
- **All in-process (reads + writes):** simpler short-term, but loses write isolation/auditability and couples mutation credentials into the worker.

# Architecture Decision Records

ADRs capture **seam-level** decisions that later phases must not break (see [PHASES.md "Architecture spine"](../PHASES.md)). Product/scope decisions live in [DECISIONS.md](../DECISIONS.md); open spikes in [RESEARCH.md](../../RESEARCH.md).

Write an ADR only when a decision defines a **stable interface or boundary** that would be expensive to reverse once code depends on it.

| ADR | Title | Status |
|---|---|---|
| [0001](0001-postgres-only.md) | Postgres-only for data, queue, vectors, traces | Accepted |
| [0002](0002-read-adapters-mcp-writes.md) | In-process read adapters; MCP for writes | Accepted |
| [0003](0003-resource-id-staleness-engine.md) | Universal `resource_id` + shared staleness engine | Accepted |

## Format

Each ADR: **Context · Decision · Consequences · Alternatives**. Keep it under a page. When reversing, add a new ADR that supersedes the old one (don't edit history) and update [DECISIONS.md](../DECISIONS.md).

# Research topics

Open questions that need investigation before or during implementation.

---

## 1. Slack real-time monitoring

- What Slack API surface is best for continuous channel monitoring: Events API (webhook push) vs Socket Mode (persistent WebSocket)?
- Rate limits and best-practice message buffering.
- How to reliably capture thread context (parent + all replies) as a single unit of analysis.
- Scopes needed for a Slack bot that reads channels, posts in threads, and sends DMs.

## 2. GitHub read-only connector

- Which GitHub API (REST vs GraphQL) is more efficient for bulk metadata reads (issues, PRs, file last-modified)?
- Webhook events available for event-driven cache invalidation (push, issue update, PR update).
- How to handle GitHub rate limits with a shared token across multiple projects.
- Code indexing: options for chunking and embedding source files for semantic search.

## 3. Notion connector

- Notion API capabilities and limitations for reading databases, pages, and blocks.
- How to detect page updates reliably (Notion does not provide a native webhook; polling vs third-party).
- Notion database schema design for tasks created by Pieuvre (fields, relations, rollups).
- Write API: creating and updating pages/database entries, and known edge cases.

## 4. RAG and knowledge graph

- Vector database options for storing embeddings with source metadata (Pinecone, pgvector, Qdrant, Weaviate).
- Chunking strategies for Slack threads, Notion pages, and GitHub files.
- How to represent the per-project skeleton + cross-project link layer as a graph (property graph vs RDF vs adjacency tables in Postgres).
- Hybrid search (keyword + vector) for retrieval with exact source citation.
- Embedding model selection: trade-off between cost, latency, and quality for code + prose mixed content.

## 5. Cache and freshness framework

- ETag and `Last-Modified` support on Slack, GitHub, and Notion APIs (which headers each actually returns).
- Canonical content hashing: best practices for normalizing JSON payloads before hashing (ordering, field exclusion).
- Schema design for the unified resource cache table (lifecycle states, version markers, dependency graph for derived artifacts).
- How to propagate invalidation from a source resource to derived embeddings/summaries efficiently.

## 6. Confidence scoring and agent design

- Prompt design for a classification + confidence agent using a modern LLM (GPT-4o, Claude 3.x, etc.).
- How to structure a chain-of-thought that outputs a structured confidence score alongside an answer.
- When to add hard-coded early returns on top of prompt-driven judgment (threshold calibration from traces).
- Multi-agent vs single-agent architecture: one orchestrator or specialized sub-agents per task type.

## 7. Tracing and observability

- LangSmith integration: what it captures by default vs what needs custom instrumentation.
- Alternatives: LangFuse, Arize Phoenix, or a custom OpenTelemetry pipeline.
- Schema for Pieuvre's internal audit trace (step, input, output, confidence, sources used, action taken).
- How to surface traces to the internal team (internal dashboard vs direct LangSmith UI).

## 8. Prompt configuration versioning

- GitHub file watching: using the GitHub API + webhooks to detect prompt file changes.
- Cache invalidation flow when a prompt file changes (what downstream state needs to be cleared).
- Format for per-project and per-channel instruction files (YAML, TOML, Markdown front-matter).
- Access control: who can merge changes to prompt files and what review process is needed.

## 9. MCP connector design

- MCP protocol specification: message format, tool registration, transport layer.
- How to expose Notion write actions and future connectors as MCP tools.
- Error handling and rollback when an MCP tool call fails mid-flow.
- Security model: authentication, authorization, and scope isolation per connector.

## 10. Project identification and routing

- Best approach to map a Slack message to one or more projects when content is ambiguous.
- How to represent channel-to-project default mappings in the prompt config files.
- Handling messages that span multiple projects (cross-project task linking).

## 11. Escalation and ownership mapping

- How to build and maintain the competent-employee map (GitHub CODEOWNERS + Notion page ownership + manual overrides).
- Slack DM API for private escalation messages.
- How to link back to the original Slack thread from a private DM in a navigable way.

## 12. Open coding / axial coding for "too complex" threshold

- Methodology: record agent traces, run open coding sessions to identify patterns, define axial categories.
- Tooling: what trace format makes open coding easiest (JSONL, structured log, LangSmith export).
- Expected output: a taxonomy of operation complexity levels that feeds back into agent prompt and hard-coded guards.

## 13. Data retention and privacy

- What raw Slack/GitHub/Notion content can legally be stored, for how long, under which jurisdictions.
- Minimal-storage principle: store only normalized extracts + references, avoid raw payload caching unless strictly necessary.
- Retention windows per resource type; deletion cascade when a source resource is deleted.

## 14. Initial broad scan (setup phase)

- How to efficiently backfill Slack channel history, GitHub repo content, and Notion workspace on first connection.
- Parallelism and rate-limit budgeting for the setup scan.
- Progress reporting and resumability if the scan is interrupted.
- How to gate the admin-triggered full rescan (cost estimate before execution, confirmation step).

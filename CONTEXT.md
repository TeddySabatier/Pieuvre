# Pieuvre — domain glossary

Shared vocabulary for architecture, code, and docs. Use these terms consistently.

## Core concepts

| Term | Definition |
|---|---|
| **Resource** | A cached artifact from an external source (Slack thread, GitHub issue, Notion page, PR enrichment block). Has a stable `resource_id`, freshness metadata, and a normalized extract — not raw payload unless required. |
| **Project** | A unit of work context: channels, repos, Notion spaces, owners, and layered prompt config. Base node of the knowledge graph. |
| **CrossLink** | An explicit edge between two Resources or tasks: `duplicate_of`, `related_to`, `parent_of`, `blocks`. Created by retrieval, agent reasoning, or human confirmation. |
| **Conversation run** | One agent execution for a Slack thread turn: intent → retrieve → respond / propose / escalate. Logged as a Trace. Uses **reasoning tiers** per step ([LLM.md](docs/LLM.md)). |
| **Reasoning tier** | `low` \| `medium` \| `high` \| `highest` — config maps each tier to `(provider, model)`. |
| **Thread state** | Lifecycle of a Slack thread in Pieuvre: `idle` → `clarifying` → … → `done`. Hybrid monitoring: replies processed only when state ≠ `idle` (or `@Pieuvre`). |
| **Staleness engine** | Shared module that decides if a Resource is fresh, stale, changed, or missing. Adapters supply metadata only. |
| **Enrichment** | Structured knowledge injected outside normal indexing — e.g. `#pieuvre-enrichment` block in a PR comment from a scan agent (Cursor/Claude Code). Stored as a Resource with `source_type: github_enrichment`. |
| **Citation** | A backlink to an exact Resource (Notion URL, GitHub issue, Slack thread). Every factual claim must cite one or say no source exists. |
| **Scan agent** | External code agent (Cursor, Claude Code) that summarizes PR/code changes and posts structured enrichment to GitHub. Not part of Pieuvre core. |
| **task_confirmation** | Per-project config (`allowed_roles`) defining who may text-confirm Notion task writes. |
| **Credential profile** | Named authentication slot (e.g. `github_org`, `notion_ws_main`). Project YAML references profiles; secrets live in env/Docker secrets only. |
| **Canonical task model** | Stable internal task fields (`title`, `status`, …). Notion writes map via per-project `field_map`. |
| **field_map** | Per-project mapping from canonical fields to Notion property names. Updated on schema drift after user prompt. |

## Connectors

| Term | Definition |
|---|---|
| **Read adapter** | In-process TypeScript module: `getMetadata()` + `getCanonicalContent()`. Slack, GitHub, Notion. |
| **Write tool (MCP)** | Literal MCP server for mutations. V0: Notion create/update only. Write API keys live on MCP server env. |
| **Ingest** | Normalizing an external item into a Resource and enqueueing index jobs. Read adapters use credential profiles from Pieuvre env. |

## Retrieval

| Term | Definition |
|---|---|
| **Skeleton** | Per-project collection of task, doc, issue, and thread Resource refs. |
| **Hybrid retrieval** | BM25 (keyword) + pgvector (semantic) merged via RRF, scoped by project, then graph-expanded one hop. |
| **Retrieval budget** | Caps per run: max sources, max tokens of context, max one re-retrieve retry. |
| **Project routing** | Content primary, channel hint secondary. If top two project scores tie → **ask in thread** before retrieve/task write. |

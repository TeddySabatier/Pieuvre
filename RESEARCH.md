# Research & technical decisions

Tracks what is **decided**, what has a **recommended default**, and what still needs a **spike or POC** before implementation.

Legend: ✅ Decided · 🔶 Recommended default · ❓ Needs spike

---

## Summary of confirmed V0 choices

| Topic | Decision |
|---|---|
| Runtime | TypeScript / Node.js |
| Write actions | Literal Model Context Protocol (MCP) tools |
| Deployment | Self-hosted (Docker) |
| LLM | **Reasoning tiers** — provider-agnostic; see [docs/LLM.md](docs/LLM.md) |
| Job queue + data | **Postgres-only** (no Redis) |
| Read connectors | Thin adapters behind shared cache/staleness core |
| Database | PostgreSQL + pgvector |
| Retrieval | **C+** — BM25 + graph + pgvector on prose only (see §4) |
| GitHub code | Issues/PRs/docs first; **PR `#pieuvre-enrichment` comments** via scan agent |
| Task confirmation | **Plain text** in thread (not Block Kit buttons) |
| Slack ingress | Events API + HTTP webhook |
| Tracing | **Postgres traces first**; LangFuse optional later |
| Credentials | **Profile indirection** — see [docs/CREDENTIALS.md](docs/CREDENTIALS.md) |
| Notion | **Per-project DB + field_map + drift hybrid C** — see [docs/NOTION.md](docs/NOTION.md) |
| Task confirmation auth | **Config per project** — `task_confirmation.allowed_roles` |
| Admin operations | **Global** `PIEUVRE_ADMIN_SLACK_IDS` in env only |
| Retention | **Traces 90d**; resources until source deleted — [docs/RETENTION.md](docs/RETENTION.md) |
| Slack backfill | **30 days** default on `pieuvre scan` (per-project override optional) |
| Project routing | **Ask in thread** when top two project candidates tie |
| Notion drift timeout | **30 minutes** before admin DM (project override optional) |
| Thread monitoring | **Hybrid C active states** + `@Pieuvre` reopen from `done` — see RESEARCH §1 |
| Slack processing UX | **Placeholder** → edit; errors + escalation name owner in-thread |
| Error escalation | **Second failure in 15 min** → auto-forward (Option C) |
| Owner fallback | **Global admin** if owner unknown |
| Same-thread concurrency | **Cancel in-flight run** on newer thread message |
| Placeholder cleanup | Cancel marks superseded + background reaper for crash/timeouts |
| Confidence format | Numeric `confidence_score` (0..1), optional label |
| Agent schema | Structured output includes both `response_mode` and `work_category` |
| Citation freshness | Revalidate only final cited sources before reply |
| Enrichment citations | Use for retrieval/ranking first; defer public citation until hardening |
| Slack scope | **Single workspace** V0 |
| GitHub scope | **Single org** V0 — one token, repos in project YAML |
| Implementation | **7 phases** — see [docs/PHASES.md](docs/PHASES.md) |
| Decision log | [docs/DECISIONS.md](docs/DECISIONS.md) |

---

## 1. Slack real-time monitoring ✅ 🔶

**Status:** Direction set; implementation details need spike.

**Recommended V0 approach:**

- **Events API + HTTP webhook** as primary ingress (push-based, fits self-hosted Docker behind reverse proxy).
- **Socket Mode** only if outbound HTTPS from the host is blocked — adds a persistent WebSocket worker.
- Buffer incoming events in the **Postgres `jobs` table** (pg-boss) to absorb bursts and respect rate limits — **no Redis in V0** (see [DECISIONS.md Q1](docs/DECISIONS.md)).
- Model a Slack **thread** (parent + replies) as one analysis unit; store `thread_ts` as the stable resource ID.
- **Message processing (Hybrid C):**
  - **Always evaluate** new root-level messages in monitored channels.
  - **Always evaluate** `@Pieuvre` anywhere (including threads).
  - **Evaluate thread replies** only when `conversation_states.status` is active: `clarifying`, `retrieving`, `answering`, `escalating`, `proposing`, `awaiting_confirmation`, `executing`.
  - If state is `done`, only `@Pieuvre` or a new actionable root message reopens processing.
  - Ignore other thread replies (no classify/LLM call).
- **Processing UX (Option B):** Post in-thread placeholder (`Looking this up…`) → **`chat.update`** same message with final answer + citations. Store `placeholder_ts` on ConversationRun trace.
- **Concurrency UX (latest-message wins):** if a newer message arrives in the same thread, cancel the in-flight run and start a fresh run.
- **Placeholder reliability:** on cancel, mark prior placeholder superseded; a background reaper updates orphan placeholders if a worker crashes.
- **Failure / escalation UX:**
  - **Error:** Update placeholder with what happened + *“Try again or rephrase.”* (no silent failure).
  - **Second error** in same thread within **15 minutes** → auto-forward to competent owner + in-thread: *“Forwarded to @alice…”*
  - **Forward (no-source / low-confidence):** Private DM to owner **and** in-thread names who received it.

**Scopes to confirm during Slack app registration:**

- `channels:history`, `channels:read` — read monitored channels
- `chat:write` — in-thread replies
- `im:write` — private escalation DMs
- `users:read` — resolve mention → user for ownership routing

**Spike needed ❓**

- [ ] Verify Events API delivery reliability behind your reverse proxy (nginx/Caddy + TLS).
- [ ] Measure rate-limit headroom for continuous monitoring across N channels.
- [ ] Confirm thread-reply event ordering (`message` + `message.replied` handling).

---

## 2. GitHub read-only connector ✅ 🔶

**Status:** Read-only confirmed; sync strategy recommended.

**Recommended V0 approach:**

- **REST API** for metadata probes (`GET` with `If-None-Match` / `If-Modified-Since` where supported).
- **GitHub webhooks** (`issues`, `pull_request`, `push`) for event-driven cache invalidation.
- Index: issues, PRs, README/docs — **not** full source files in V0. Code context comes from **PR `#pieuvre-enrichment`** blocks (see [PHASES.md §4](docs/PHASES.md#phase-4--enrich)).
- Shared token with per-repo scope; track `X-RateLimit-*` headers in adapter metrics.

**Code indexing — deferred ⏸**

Full-repo embedding is **post-V0** (PHASES "Deferred"). The spikes below only apply *if* enrichment proves insufficient:

- [ ] (Deferred) chunk size for TypeScript/Python mixed repos (512–1024 token chunks, overlap 10–15%).
- [ ] (Deferred) embed file path + symbol name in chunk metadata for citation backlinks.

**GraphQL bulk backfill:** defer until REST backfill proves too slow (>10 repos or >50k files).

---

## 3. Notion connector ✅ 🔶

**Status:** Create + update confirmed; webhook gap acknowledged.

**Recommended V0 approach:**

- **Polling on stale** (not fixed global interval): when cache marks a Notion page stale, adapter fetches metadata and revalidates.
- **Content hash fallback** — Notion's `last_edited_time` is reliable enough as primary validator; hash diff for block-level silent changes.
- Writes exclusively through **Notion MCP tool** after human confirmation in Slack.

**Spike needed ❓**

- [ ] Map Notion database schema once workspace is chosen (deferred config item).
- [ ] Test block-level update API for partial task field updates vs full page replace.
- [ ] Confirm relation/rollup fields needed for cross-link display in Notion.

**Known limitation:** No native Notion webhook — polling + hash is the pragmatic V0 path.

---

## 4. RAG and knowledge graph ✅ 🔶

**Status:** Architecture decided; store and embedding model need POC.

**Decided:**

- Per-project skeleton + explicit cross-project links (not monolithic global graph).
- Hybrid retrieval (keyword + vector) with exact source ID return for Slack backlinks.
- Derived artifacts invalidated with source resource (shared cache framework).

**Recommended V0 store:** PostgreSQL + pgvector

**Embeddings (decided):** Cloud API — provider/model via env config; vectors stored in pgvector. Local models deferred unless privacy requires.

| Option | Fit for Pieuvre V0 |
|---|---|
| **PostgreSQL + pgvector** ✅ | Single DB for resources, graph edges, embeddings, traces — simplest self-hosted ops |
| **Cloud embedding API** ✅ | `text-embedding-3-small` or equivalent; ~$1/mo at small-team scale |
| Pinecone / Qdrant | Adds another service; consider only if vector scale exceeds ~1M chunks |
| Redis Stack | Fast but in-memory cost; poor fit for durable audit trail co-location |
| Local embed model on VPS | Deferred — use if data must not leave host |

**Spike needed ❓**

- [ ] Benchmark pgvector recall@k on a sample repo + Notion export.
- [ ] Define `CrossLink` edge schema and adjacency query patterns (recursive CTE vs materialized paths).
- [ ] Pick default embedding model via config (e.g. `text-embedding-3-small` vs open-source `nomic-embed`).

---

## 5. Cache and freshness framework ✅

**Status:** Fully specified in README. Implementation-ready.

**Adapter contract:**

```typescript
interface ResourceMetadata {
  resourceId: string;
  sourceType: 'slack' | 'github' | 'notion';
  updatedAt?: string;       // ISO 8601
  versionMarker?: string;   // ETag, revision, sync token
}

// Canonical seam contract: docs/PHASES.md "Seam contracts"
interface ReadAdapter {
  sourceType: SourceType;
  getMetadata(id: string): Promise<ResourceMetadata>;
  getCanonicalContent(id: string): Promise<string>;  // normalized for hashing
  normalize(id: string, raw?: unknown): Promise<NormalizedExtract>;
}
```

**Validator support by source (verified):**

| Source | Primary validator | Hash fallback |
|---|---|---|
| GitHub REST | `ETag` header + `updated_at` field | Yes |
| Notion | `last_edited_time` | Yes (block content) |
| Slack | `ts` + edited timestamp | Yes (message text) |

**Remaining implementation tasks:**

- [ ] Unified `resources` table with lifecycle enum: `fresh | stale | revalidated | invalidated | missing`.
- [ ] Dependency table linking source resources → embeddings/summaries for cascade invalidation.
- [ ] Admin-only full rescan: cost estimate (resource count × avg fetch) before execution.

---

## 6. Agent design and confidence ✅ 🔶

**Status:** V0 approach decided; prompt structure needs iteration.

**Decided:**

- Single orchestrator agent in V0 (not multi-agent swarm).
- Prompt-driven confidence with numeric output (`confidence_score` in `0..1`), hard-coded guards added later from trace analysis.
- Structured output includes `response_mode` (question/task/noise/info) and `work_category` (feature/bug/support when relevant).

**Recommended V0 pattern:**

- Provider-agnostic LLM client (swap OpenAI / Anthropic / local via env config).
- System prompt = core instructions + layered project/channel config from GitHub cache.
- Retrieval context injected as numbered sources; agent must cite by source ID.

**Spike needed ❓**

- [ ] Calibrate numeric confidence rubric in prompt (what counts as "confident enough to reply publicly").
- [ ] Test classification accuracy on 20–30 real Slack messages from your channels.
- [ ] Define JSON schema for structured agent output (Zod validation).

---

## 7. Tracing and observability ✅ 🔶

**Status:** LangSmith-like requirement confirmed. **V0 = Postgres traces** (`traces` / `trace_steps`, Phase 0). LangFuse is an **optional upgrade post–Phase 6**, not a V0 dependency.

**Decision (V0):**

| Option | Pros | Cons | V0 |
|---|---|---|---|
| **Postgres traces** | One DB; co-located with resources; zero extra service | Build UI/queries yourself | ✅ V0 |
| LangFuse (self-hosted) | Fits self-hosted; full data control | Extra Docker service | 🔶 optional post–Phase 6 |
| LangSmith (cloud) | Best UX, zero ops | Data leaves your infra | ✗ |
| Custom OTel + Postgres | Maximum control | Most build effort | ✗ |

**Trace schema (minimum):**

```
trace_id, slack_thread_ts, started_at
steps[]: { name, input, output, latency_ms, model, tokens }
confidence, sources_used[], actions_taken[], escalation_triggered
```

**Spike needed ❓**

- [ ] (V0) Define `traces` / `trace_steps` query patterns for replaying a run.
- [ ] (Post–Phase 6, optional) Deploy LangFuse in docker-compose + wire trace export.

---

## 8. Prompt configuration versioning ✅ 🔶

**Status:** GitHub-canonical + DB cache decided.

**Recommended V0 format:** YAML with Markdown body for instruction prose.

```yaml
# prompts/projects/acme.yaml
project_id: acme
channels: ["#acme-dev", "#acme-support"]
owners:
  backend: "@alice"
  frontend: "@bob"
instructions: |
  Acme uses Notion database X for bugs...
```

**Invalidation flow:**

1. GitHub webhook (`push` on `prompts/**`) → invalidate config cache entries.
2. Re-parse files → update `projects` table.
3. No re-index of external resources needed (prompt change ≠ source change).

**Spike needed ❓**

- [ ] Confirm webhook delivery to self-hosted Pieuvre (GitHub → your VPS).
- [ ] Define merge/review policy for prompt file changes (CODEOWNERS on `prompts/`).

---

## 9. MCP connector design ✅

**Status:** Literal MCP confirmed for write path.

**Recommended V0 architecture:**

```
Agent orchestrator
    └── MCP client (stdio or SSE transport)
            └── notion-mcp-server   (create_page, update_page, query_database)
            └── (future) github-mcp-server
```

**Decisions:**

- Read adapters stay **in-process TypeScript** (latency-sensitive, high volume).
- Write actions go through **MCP tool calls** (isolation, auditability, swappable servers).
- Each MCP tool call logged in trace with `tool_name`, `input`, `output`, `error`.

**Spike needed ❓**

- [ ] Evaluate existing Notion MCP server vs thin custom wrapper.
- [ ] Define rollback: if MCP create succeeds but Slack confirmation fails, mark task as `pending_confirmation` in DB.
- [ ] MCP auth: token per server, scoped to Notion integration secret.

---

## 10. Project identification and routing ✅

**Status:** Decided — content primary, channel as hint; **disambiguate in thread when ambiguous**.

**Implementation notes:**

- `prompts/projects/*.yaml` declares `channels[]` → default project hint.
- Classifier outputs `project_candidates[]` with scores.
- **If score gap > threshold** → use top project silently.
- **If top two within threshold** → ask in thread: “Is this about **Acme** or **Beta**?” before retrieve or task proposal (Option A).
- Cross-project references in a resolved thread create `CrossLink` edges; task writes always target **one** project.

Tune threshold from traces in Phase 6.

---

## 11. Escalation and ownership mapping ✅ 🔶

**Status:** Decided — Slack-ID-first mapping in V0; private DM to owner; **user told in-thread who was forwarded to**.

**In-thread copy (escalation):** *“Forwarded to @alice (backend owner) — they’ll follow up.”*

**In-thread copy (error):** *“Something went wrong: {brief reason}. Try again or rephrase.”*

**Owner resolution fallback (Option C):** If no owner found in project YAML Slack IDs/default owner → forward to first **`PIEUVRE_ADMIN_SLACK_IDS`** entry; in-thread: *“Forwarded to @admin — they’ll follow up.”*

**Recommended ownership sources (priority order, V0):**

1. Manual override in `prompts/projects/*.yaml` (`owners:` map).
2. Project default owner from config.
3. Fallback: global admin (`PIEUVRE_ADMIN_SLACK_IDS`).

**Spike needed ❓**

- [ ] Slack DM format with deep link back to thread (`slack://channel?team=...&id=...`).
- [ ] Handle owner not in Slack workspace (fallback to global admin mention in thread)
- [ ] Re-introduce CODEOWNERS / Notion-assignee inference only after identity mapping is added.

---

## 12. "Too complex" threshold — deferred ⏸

**Status:** Explicitly deferred until trace corpus exists.

**Planned methodology:**

1. Record 50+ agent runs covering create, update, merge, regroup scenarios.
2. Open coding on failure/delegation patterns.
3. Axial categories → complexity levels → prompt guards + optional hard-coded blocks.

**Preparation (can start now):**

- [ ] Ensure trace schema captures `delegation_reason` field.
- [ ] Tag traces where human overrode or rejected agent proposal.

---

## 13. Data retention and privacy ✅ 🔶

**Status:** Minimal-storage principle decided.

**Recommended V0 policy:**

| Data | Store | Retention |
|---|---|---|
| Normalized extracts | Yes | Until source deleted or manual purge |
| Raw payload snapshots | Only if needed for latency/audit | Discard on confirmed staleness + change |
| Embeddings | Yes | Cascade-delete with source resource |
| Traces | Yes | **90 days** (configurable via `PIEUVRE_RETENTION_TRACES_DAYS`) |
| Slack message text | Normalized extract only | Until source deleted; re-fetch if stale |

**Spike needed ❓**

- [ ] Review employer Slack/GitHub/Notion ToS for bot storage of message content.
- [ ] Add `resource_deleted` webhook handler to cascade purge.

---

## 14. Initial broad scan (setup phase) ✅ 🔶

**Status:** Manual trigger, broader scan, admin-only full rescan — decided.

**Recommended implementation:**

- CLI command: `pieuvre scan --project acme --sources slack,github,notion`
- **Slack backfill:** last **30 days** per monitored channel (default); override `slack_backfill_days` in project YAML
- GitHub: all open issues/PRs + docs paths (no full code embed)
- Notion: all pages in configured task databases + linked docs
- Rate-limit budget: sequential source scan, parallel within source up to API limit − 20% headroom.
- Full rescan gate: admin runs `pieuvre rescan --dry-run` first → shows estimated API calls + duration → confirm.

**Spike needed ❓**

- [ ] Measure backfill time for your actual channel history + repo size.
- [ ] Define checkpoint granularity (per channel, per repo, per Notion database).

---

## Research backlog (priority order)

| Priority | Item | Type |
|---|---|---|
| P0 | Slack Events API behind self-hosted reverse proxy | Spike |
| P0 | PostgreSQL + pgvector schema + hybrid retrieval POC | Spike |
| P0 | Notion MCP server selection + write flow | Spike |
| P1 | Agent structured output schema + 20-message classification test | Spike |
| P1 | GitHub webhook + selective doc re-index | Implementation |
| P2 | Embedding model benchmark on your content | Spike |
| P2 | Open coding prep (`delegation_reason` in traces) | Preparation |
| ⏸ | Notion workspace/schema mapping (values only — model decided) | Blocked on config |
| ⏸ | Task template field values (canonical model decided) | Blocked on config |
| ⏸ (post P6) | LangFuse docker-compose + trace wiring (optional upgrade) | Spike |

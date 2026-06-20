# Pieuvre — decision log

Consolidated record of all planning Q&A decisions. For implementation detail, follow links to topic docs.

**Status:** Locked for V0 design · **Last updated:** planning session through Q24

Related: [PHASES.md](PHASES.md) · [CREDENTIALS.md](CREDENTIALS.md) · [NOTION.md](NOTION.md) · [LLM.md](LLM.md) · [RETENTION.md](RETENTION.md) · [RESEARCH.md](../RESEARCH.md) · [adr/](adr/)

---

## Product & scope (original Q&A)

| # | Topic | Decision |
|---|---|---|
| P1 | Primary users | Internal team (you + workmates) |
| P2 | V0 north star | **Cross-linked, cited answers** — task polish secondary |
| P3 | Success metrics | All matter; optimize cross-linking + assist first |
| P4 | Multi-project | Yes from day one — per-project skeleton + explicit cross-project links |
| P5 | Slack monitoring | Continuous + explicit `@Pieuvre` mentions |
| P6 | Channels | Short allowlist only |
| P7 | Actionable categories | Feature, bug, support, info about those — flexible subtypes |
| P8 | Ignore | Announcements, casual chat, unrelated noise |
| P9 | Check before reply | Always scan context first, then respond |
| P10 | Clarification | In Slack thread (not private) |
| P11 | Answers | Cite exact sources or state none; never fabricate |
| P12 | No source | Ask clarification → escalate if still stuck |
| P13 | GitHub V0 | **Read-only** — no issue creation |
| P14 | Notion V0 | Create + update after confirmation; complex ops → human |
| P15 | Task confirmation | **Human confirmation required** before Notion write |
| P16 | Traces | LangSmith-like; **internal team only** in V0 |
| P17 | Confidence | Prompt-driven first; hard-coded guards from traces later |
| P18 | Freshness in replies | Only when confidence low or user asks |
| P19 | Initial scan | Manual `pieuvre scan` — broad validation |
| P20 | Ongoing sync | Event-driven + admin-only full rescan (heavily gated) |
| P21 | Storage | Normalized extracts + refs; raw payloads only when necessary |
| P22 | Prompt config | Versioned files in GitHub; cached in DB with invalidation |
| P23 | Project ID | Content primary; channel hint secondary; layered prompts |
| P24 | Write abstraction | Literal **MCP** for mutations |
| P25 | Prioritization | Deferred (weekly polls, cost scoring) |

---

## Stack (confirmed)

| Layer | Choice |
|---|---|
| Language | TypeScript / Node.js |
| Deployment | Self-hosted Docker |
| Database | PostgreSQL only — data, queue, pgvector, traces |
| Job queue | Postgres (no Redis) |
| Slack ingress | Events API + HTTP webhook |
| Read path | In-process adapters (Slack, GitHub, Notion) |
| Write path | Literal MCP (Notion V0) |
| Embeddings | Cloud API → pgvector |
| LLM | Provider-agnostic **reasoning tiers** per step |
| Tracing | Postgres traces; LangFuse optional later |

---

## Architecture Q&A (planning session)

| Q | Question | Answer | Detail doc |
|---|---|---|---|
| **1** | Postgres-only vs Redis? | **Postgres-only** — data + queue + pgvector | [PHASES.md](PHASES.md) |
| **2** | Task confirmation UX? | **Plain text** in thread (not Block Kit) | [NOTION.md](NOTION.md) |
| **3** | GitHub code indexing? | **Phased** — issues/PRs/docs first; no full-repo embed | [PHASES.md §4](PHASES.md#phase-4--enrich) |
| **3b** | Code context without embed? | **Scan agent** (Cursor/Claude Code) → PR comments | [PHASES.md §4](PHASES.md#phase-4--enrich) |
| **4** | Enrichment ingest path? | **Structured PR comment** `#pieuvre-enrichment` | [PHASES.md §4](PHASES.md#phase-4--enrich) |
| **5** | Retrieval stack? | **C+** — BM25 + graph + pgvector on **prose only** | [RESEARCH.md §4](../RESEARCH.md) |
| **6** | Notion task destination? | **Per-project** DB + connections; cross-link across projects | [NOTION.md](NOTION.md) |
| **7** | Notion schema drift? | **A+** — canonical model + `field_map`; prompt user on missing property | [NOTION.md](NOTION.md) |
| **8** | Drift prompt UX? | **Hybrid C** — thread first; admin DM on timeout / background sync | [NOTION.md](NOTION.md) |
| **9** | Credentials model? | **Profile indirection** + MCP for writes; `.env` V0 | [CREDENTIALS.md](CREDENTIALS.md) |
| **10** | Who confirms tasks? | **Config per project** — `task_confirmation.allowed_roles` | [NOTION.md](NOTION.md) |
| **11** | Embeddings where computed? | **Cloud API** (provider via config) | [RESEARCH.md §4](../RESEARCH.md) |
| **12** | LLM selection? | **Reasoning tiers** (`low`→`highest`) → `(provider, model)` in config | [LLM.md](LLM.md) |
| **13** | Admin operations auth? | **Global** `PIEUVRE_ADMIN_SLACK_IDS` only | [CREDENTIALS.md](CREDENTIALS.md) |
| **14** | Data retention? | **Traces 90d**; resources until source deleted | [RETENTION.md](RETENTION.md) |
| **15** | Slack backfill on scan? | **30 days** default (`slack_backfill_days` override) | [RESEARCH.md §14](../RESEARCH.md) |
| **16** | Ambiguous project routing? | **Ask in thread** if top two candidates tie | [RESEARCH.md §10](../RESEARCH.md) |
| **17** | Notion drift timeout? | **30 minutes** before admin DM | [NOTION.md](NOTION.md) |
| **18** | Thread message processing? | **Hybrid C** — roots + `@Pieuvre` + replies when state ≠ `idle` | [RESEARCH.md §1](../RESEARCH.md) |
| **19** | Processing UX in Slack? | **Placeholder** “Looking this up…” → `chat.update` final | [RESEARCH.md §1](../RESEARCH.md) |
| **20** | Failure / escalation copy? | Tell user what happened; **try again** or **name forwarded owner** | [README.md](../README.md) |
| **21** | Auto-forward on hard error? | **Second failure in 15 min** → auto-escalate | `escalation` in project YAML |
| **22** | Owner unknown? | Fallback to **global admin** | [RESEARCH.md §11](../RESEARCH.md) |
| **23** | Slack scope V0? | **Single workspace** | [CREDENTIALS.md](CREDENTIALS.md) |
| **24** | GitHub scope V0? | **Single org** — one token, repos in YAML | [CREDENTIALS.md](CREDENTIALS.md) |

---

## Behavior detail — owned elsewhere

This log records **what** was decided (tables above). The **how** lives in topic docs to avoid duplication/drift. Single source per concern:

| Concern | Authoritative doc |
|---|---|
| Slack monitoring, placeholder UX, escalation copy | [RESEARCH.md §1](../RESEARCH.md) · [README.md](../README.md) |
| GitHub read scope + PR enrichment | [PHASES.md §1, §4](PHASES.md) |
| Notion `field_map`, drift, confirmation auth | [NOTION.md](NOTION.md) |
| Retrieval (C+), `resource_id`, cross-links | [PHASES.md §2](PHASES.md) · [RESEARCH.md §4](../RESEARCH.md) |
| Credential profiles, admin ops | [CREDENTIALS.md](CREDENTIALS.md) |
| LLM reasoning tiers / ModelRouter | [LLM.md](LLM.md) |
| Retention windows + purge | [RETENTION.md](RETENTION.md) |

---

## Owner resolution order

1. Manual `owners` in project YAML  
2. GitHub `CODEOWNERS`  
3. Notion assignee / creator  
4. `owners.default`  
5. **`PIEUVRE_ADMIN_SLACK_IDS`** (global admin)

---

## Implementation phasing

| Phase | Milestone |
|---|---|
| 0 | Docker, Postgres, queue, seam stubs |
| 1 | Ingest Slack/GitHub/Notion |
| 2 | BM25 + pgvector + CrossLinks |
| 3 | **Daily-usable Q&A** ← north star MVP |
| 4 | PR enrichment ingest |
| 5 | Task propose → confirm → Notion MCP |
| 6 | Staleness v2, admin rescan, polish |

Full plan: [PHASES.md](PHASES.md)

---

## Explicitly deferred

| Item | When |
|---|---|
| Notion DB IDs per project | Onboarding each project |
| Task template / field copy | Before Phase 5 |
| “Too complex” threshold | After trace corpus + open/axial coding |
| Non-technical prompt editing | Post-V0 |
| Prioritization automation | Post-V0 |
| GitHub writes / GitHub MCP | Post-V0 |
| LangFuse | Optional post Phase 6 |
| Multi-workspace Slack / multi-org GitHub | Config pattern ready; not V0 |
| Employer ToS / legal review | Spike before production use |
| Local embedding model | If privacy requires leaving cloud |

---

## Open spikes (not blocking design)

See [RESEARCH.md](../RESEARCH.md) backlog:

- Slack Events API behind reverse proxy  
- pgvector POC on your content  
- Notion MCP server selection  
- Agent classification test on 20 real messages  
- GitHub webhook selective re-index  

---

## Config quick reference

| File | Purpose |
|---|---|
| `.env` | Secret values (gitignored) |
| `config/credential-profiles.yaml` | Profile name → env var |
| `config/llm-tiers.yaml` | Tier → provider/model; step defaults |
| `prompts/projects/*.yaml` | Project channels, repos, Notion, owners, overrides |
| `prompts/core.yaml` | Shared agent instructions |

Examples: [config/credential-profiles.example.yaml](../config/credential-profiles.example.yaml) · [prompts/projects/acme.example.yaml](../prompts/projects/acme.example.yaml) · [.env.example](../.env.example)

---

## Change process

When reversing a decision:

1. Update this file with date + rationale.  
2. Update the linked topic doc (single source per concern — see table above).  
3. If the change affects a Phase 0 seam, add a superseding ADR in [`docs/adr/`](adr/) (don't edit accepted ADRs).

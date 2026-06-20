# Pieuvre

**Knowledge center for projects.** Pieuvre monitors Slack, GitHub, and Notion to classify work, answer questions with cited sources, and create well-linked actionable tasks — with human confirmation at every meaningful step.

---

## What it does

Pieuvre sits in selected Slack channels and:

1. **Reads and classifies messages** — feature requests, bugs, support requests, or information about those. Anything else (announcements, casual chat) is ignored.
2. **Cross-references existing context** — checks its own database for related Notion tasks, GitHub issues/PRs, and prior Slack threads before replying.
3. **Answers questions** — directly in-thread when confident, with compact backlinks to exact sources (Notion page, GitHub issue, Slack thread). If no source exists, it says so and escalates privately to the relevant person.
4. **Creates and updates tasks** — proposes Notion task creation (with complexity and cross-links to related tasks), asks for human confirmation, then acts. Complex operations are delegated to humans.
5. **Audits everything** — keeps LangSmith-like traces of reasoning and actions, visible to the internal team only.

---

## V0 scope

| Area | V0 behaviour |
|---|---|
| **Slack** | Continuous monitoring + explicit mention. Short list of channels. |
| **GitHub** | Read-only (context and status). |
| **Notion** | Create and update tasks. Complex ops delegated to humans. |
| **Task creation** | Human confirmation required before writing to Notion. |
| **Answering** | Confident → reply directly. Unsure → escalate privately to owner, linking original thread. |
| **Confidence** | Prompt-driven agent judgment first; hard-coded early returns added later. |
| **Full rescan** | Admin-only, manually triggered, heavily gated (expensive). |
| **Traces** | Internal team only in V0. |
| **Multi-project** | Yes — from day one, with cross-project linking. |

---

## Architecture decisions

### Bridge / adapter pattern for connectors

Slack, GitHub, Notion (and future sources) are kept fully decoupled. Each connector is a thin adapter that exposes two things:

- `getMetadata()` — returns `resource_id`, `source_type`, `updated_at`, `version_marker`.
- `getCanonicalContent()` — returns normalized content for hashing.

All business logic (cache, staleness, invalidation, re-search) lives in a shared core. Adding a new connector means implementing the adapter contract, not touching the core.

### Own database as source of truth

Pieuvre maintains its own DB synced from Slack, GitHub, and Notion. Sources are decoupled from each other and from the DB. The DB stores:

- Normalized resource records with freshness metadata.
- Sources alongside knowledge (for exact citation backlinks).
- Cross-project links and task hierarchy.
- Audit traces.

### Freshness and cache invalidation (shared framework)

All connectors share one staleness engine. Decision order:

1. **Fresh** → serve cached result.
2. **Stale, validator available** → revalidate with `updated_at` / ETag / version token.
3. **Stale, no validator** → fetch canonical content, compute and compare hash.
4. **Changed** → discard cached artifact, re-search, re-index.
5. **Deleted / inaccessible** → mark unavailable, remove from active results.

Derived artifacts (embeddings, summaries, snippets) are invalidated together with their source resource.

Full content fetch + hash diff is a last resort. Metadata-first checks keep bandwidth and cost low.

### Prompt configuration stored in GitHub

Project and channel instructions are versioned files in the GitHub repo hosting Pieuvre. Pieuvre caches them in its DB and invalidates the cache when files change (using the same staleness framework). This makes prompts easy to edit, version-controlled, and auditable.

### Knowledge graph structure

- **Per-project skeleton** as the base unit.
- **Cross-project links** layered on top when evidence exists.
- This allows the graph to scale without a monolithic global index from day one.

### Project identification

Content is the primary signal. Channel provides default context when content is ambiguous. Per-channel/per-project instructions are layered on top of a shared core prompt.

### Escalation

When Pieuvre has no reliable source and cannot answer confidently:
1. Ask for clarification.
2. If still no source, escalate **privately** to the competent employee (identified by GitHub/Notion ownership mapping), linking back to the original Slack thread.

### Freshness metadata exposure

Exposed to the user only when confidence is low or the user explicitly asks. Not shown by default.

### Action abstraction via MCPs

All write actions (task creation, updates, future integrations) are abstracted behind MCP-like connectors. This keeps Pieuvre's core decoupled from specific tool APIs.

---

## Primary success metric for V0

**Cross-linking and assisted answers.** Pieuvre must be excellent at scanning related sources, ingesting them, and outputting a well-formulated answer with compact source backlinks. Task editing quality is secondary in V0.

---

## What is explicitly deferred

- Notion workspace / database selection (to be configured later).
- Task template and field definitions (a template + instructions will be provided later).
- Authorization rules for who can confirm task creation.
- Definition of "too complex" operations (to be determined from recorded traces via open coding and axial coding).
- Non-technical teammate editing of instructions via Notion UI.
- Prioritization features (weekly polls, cost/priority scoring).

---

## Folder structure (planned)

```
pieuvre/
├── connectors/          # Thin adapters: slack, github, notion
├── core/
│   ├── cache/           # Shared freshness + invalidation engine
│   ├── classifier/      # Message intent classification
│   ├── graph/           # Knowledge graph: per-project skeletons + cross-links
│   ├── agent/           # Prompt-driven reasoning and confidence logic
│   └── traces/          # LangSmith-like audit trail
├── db/                  # Schema and migrations for Pieuvre's own DB
├── prompts/             # Core system prompt + per-project/channel config files
└── mcp/                 # MCP connector interfaces for write actions
```

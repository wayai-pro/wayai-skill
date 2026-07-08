---
name: wayai
version: 6.27.1
description: |
  Configure WayAI hubs, agents, tools, channels, resources, states, evals, outbound, and analytics.
  Use when: creating or editing a hub or hub config; adding/configuring agents, tools, channels,
  connections, teams, kanban, states, resources, eval scenarios or journeys, outbound campaigns;
  running analytics or evals; annotating conversation outcomes; generating a shareable hub
  progress/readiness report (progress.html); reviewing or editing workspace YAML
  (hub.yaml, agents/*.yaml) or agent instruction Markdown; installing a ready-made hub template
  (`wayai template list`/`pull`); using the wayai CLI (push, pull, publish, send-message,
  conversations, sync-skills, create-credential, update-credential, analytics, run-eval,
  eval capture, evals sql, org, template, init); or interpreting WayAI platform
  terminology (pilot/copilot, preview/production, kanban statuses, AI modes, agent roles, journeys).
---

# WayAI Skill

WayAI is a SaaS platform for AI-powered communication hubs. Each hub combines AI agents and a human team across channels (WhatsApp, Email, Instagram, Telegram, native App). This workspace stores hubs as code — one folder per hub (`hub.yaml` + `agents/*.{yaml,md}` + `evals/`, `journeys/`, `resources/`) synced bidirectionally to the platform via the `wayai` CLI.

**Platform is the source of truth.** Workspace files are the edit surface — changes flow through files → `wayai push` → platform. Always `wayai pull` before editing to catch out-of-band changes.

**How to use this skill:** this file is the complete concept map — every WayAI primitive is defined here with enough depth to decide what to build and which files to touch. Field-level schemas, per-provider specifics, and mechanics live in `references/`; each domain section below ends with a pointer to its deep-dive file — open it when you're about to author or debug that domain. Before generating a full hub from scratch, read [`references/canonical-example/README.md`](references/canonical-example/README.md) once — it shows how the pieces wire together. The full routing table is at the end ([Reference Documentation](#reference-documentation)).

## Agent Guidelines

- **Interface:** if you have filesystem/shell access (code-harness agents — Claude Code, Codex, Cursor, OpenCode), drive WayAI through the **`wayai` CLI and workspace files**, and do not call any `mcp__wayai__*` tools that may also be in your toolset. If you do **not** have filesystem/shell access (app-harness agents — Claude Desktop, etc.), use the `mcp__wayai__*` tools — they are your interaction surface for everything below
- Only provide information from this skill, tool descriptions, or reference documentation
- Do not invent URLs, paths, or steps
- Hub config flows through files + the `wayai` CLI; one-time setup (orgs, OAuth) goes through the platform UI. Publishing preview → production is now CLI-capable (`wayai publish`) or UI
- Always `wayai pull -y` before editing — catches out-of-band changes
- Always `wayai push -y` immediately after editing — editing and pushing are a single action
- Never auto-commit — show `git diff`, wait for user approval

## Quick Decision: What Can I Do?

| Entity | How |
|--------|-----|
| Hub settings, agents, agent instructions, tools, kanban, states, resources, evals, journeys, outbound, custom tools | CLI (`wayai push`) |
| Eval journeys (hub-as-code) — `journeys/<slug>.yaml`, flat folder | CLI (`wayai push` / `wayai pull`; pull after first create to sync step ids) |
| Connections — non-OAuth (Agent providers, STT/TTS, Tool API key, MCP Bearer Token) | CLI (auto-created from org credentials) |
| Connections — OAuth (WhatsApp, Instagram, Google Calendar, MCP OAuth) | Platform UI |
| Set/rotate a connection's credential directly (incl. production) | CLI (`wayai set-connection-credential`) or UI |
| Org credentials — create / rotate / edit | CLI (`wayai create-credential` / `wayai update-credential`) or UI |
| Org-level shared resources (org-as-code) | CLI (`wayai org pull` / `push` / `diff`) |
| Skills sync to providers | CLI (`wayai sync-skills`) |
| Conversation testing | CLI (`wayai send-message`, `wayai conversations`, `wayai delete-history`) |
| Inspect what an agent actually received (resolved prompt, rendered context, injected timestamps, tool calls) | CLI (`wayai conversations <id> observability [--message-id <id>]`) |
| Record a post-hoc business outcome on an ended conversation (e.g. customer purchased) as an analytics dimension | CLI (`wayai conversations <id> annotate --set key=value [--type ...]`) |
| Analytics | CLI (`wayai analytics`, `wayai analytics query`) |
| Eval runs and results | CLI (`wayai run-eval`, `wayai eval-results`) |
| List eval scenarios / raw SQL over eval results | CLI (`wayai evals`, `wayai evals sql`) |
| Capture production conversation as eval | CLI (`wayai eval capture <conversation_id>`) |
| Capture production conversation as a journey (full multi-turn transcript) | CLI (`wayai eval journey capture <conversation_id>`) |
| Delete eval session(s) / run history | CLI (`wayai eval session delete <session_id>`, or `--all` for every session on the hub) |
| Bug reporting | CLI (`wayai report create`) |
| Build progress & readiness report — shareable `progress.html` (code-harness only) | Agent-generated file per [`references/progress-report.md`](references/progress-report.md) |
| Workspace discovery | CLI (`wayai list`) |
| Organization — create | CLI (`wayai org create`) or UI |
| Organization — update, delete | UI |
| Publish/sync a preview to production | CLI (`wayai publish`, alias `wayai sync`) or UI |
| Delete hubs | UI |
| Replicate a preview, set/clear a preview's label | CLI (`wayai replicate` / `wayai relabel`) or UI |
| Teams, team users, hub users, admins, contact approval | UI (Hub → Users tab) |
| Org tags (create/edit) | UI (referenced from `hub.yaml` `tags:` by slug) |

## Entity Hierarchy

```
Organization                ← CLI (`wayai org create`) or UI
├── Org Credentials         ← CLI (`wayai create-credential`/`update-credential`) or UI — API keys stored once, reused across hubs
├── Org Tags                ← UI — gate which credentials each hub can resolve
├── Org Resources           ← CLI (`wayai org pull/push`) — shared knowledge/skills, fan out to linked hubs
└── Hub                     ← CLI (`wayai create`, or auto-creates on push) or UI; publish/sync via CLI (`wayai publish`) or UI
    ├── Connections         ← auto-created from org credentials on push (non-OAuth); OAuth via UI
    ├── Channels            ← auto-provisioned, never authored (see Channels)
    ├── Agents              ← CLI — `agents/<slug>.yaml` + `<slug>.md`
    │   ├── Tools           ← CLI — native, custom HTTP, MCP, delegation
    │   └── Resource links  ← CLI — `resources:` block in agent YAML
    ├── Kanban statuses     ← CLI — `hub.yaml` (workflow stages for conversations)
    ├── States              ← CLI — `hub.yaml` (JSON-schema data agents read/write)
    ├── Resources           ← CLI — `hub.yaml` + `resources/` folder (knowledge + skills)
    ├── Evals + Journeys    ← CLI — `evals/`, `journeys/`
    ├── Outbound            ← CLI — `hub.yaml` (contacts, lists, schedules)
    └── Teams + Users       ← UI (Hub → Users) — teams, admins, team users, hub users
```

Setup order: Organization (CLI `wayai org create` or UI) → Org Credentials (CLI or UI) → Hub (CLI `wayai create`, or push auto-creates, or UI) → configure agents, tools, connections via CLI.

The `wayai` connection (native tools) is auto-created when a hub is created — no setup needed.

## Hub Types

| Type | Conversations | Channels | Use Case |
|------|--------------|----------|----------|
| `chat` | ONE per end user | WhatsApp, Instagram, Email, Telegram, App | Person-centered: support, sales, helpdesk |
| `task` | MULTIPLE per user | App only | Task-centered: invoices, inventory, approvals |

Decision: external channels (WhatsApp/Instagram/Email/Telegram) needed → `chat`. Object/task processing → `task`.

## AI Modes & the Conversation Model

Hub-level `ai_mode` sets what the AI does:

| Mode | Behavior |
|------|----------|
| `pilot` | AI handles end users autonomously |
| `copilot` | AI suggests responses to the support team (no channel delivery) |
| `pilot+copilot` | Switches dynamically based on who currently responds |
| `turned_off` | AI disabled; humans only |

A **conversation** is the runtime session between an end user and the hub (config entities define behavior; conversations and messages are what they act on):

- `conversation_status`: `agent` (AI handles it) | `team` (human team handles it) | `ended` (closed + archived)
- Status selects the active agent **track**: status `agent` → **Pilot** track replies to the end user through the channel; status `team` (with `copilot`/`pilot+copilot` mode) → **Copilot** track drafts suggestions the team sees in `/support`
- Track switches: the `transfer_to_team` tool (agent → team) or a team handback in the support UI (team → agent). `transfer_to_agent`/`consult_agent` move between agents *within* a track
- Close paths: the agent's `close_conversation` tool, transitioning into an `isTerminalStatus` kanban status (any surface — agent tool, team drag-drop, REST/MCP), the team UI, or the hub's `auto_close_inactive_days`. Ended conversations are archived and listed in the Ended tab; within `conversation_retention_days` they still accept post-hoc `wayai conversations <id> annotate`
- An agent's reply text is delivered automatically — **there is no send-message tool**; tools exist for actions beyond replying

Kanban status is orthogonal to all of this: it tracks *workflow stage* (custom slugs like `qualified`), not who is responding.

## Agent Roles

| Role | Track | Per Hub | Description |
|------|-------|---------|-------------|
| `pilot` | Pilot | 1 | Responds to end users autonomously |
| `copilot` | Copilot | 1 | Suggests responses to the support team |
| `pilot_specialist` / `copilot_specialist` | Both | Multiple | Delegation target — full transfer via `transfer_to_agent` |
| `pilot_advisor` / `copilot_advisor` | Both | 1 each | Advisory input via `consult_agent`; returns control |
| `monitor` | Background | 1 | Observes silently |
| `conversation_evaluator` / `message_evaluator` | Background | 1 each | Async quality assessment; excluded from normal routing. Their `evaluation_variables` feed Analytics; the `message_evaluator` also scores eval runs |
| `summarizer` | Background | 1 | Auto-provisioned with the first pilot/copilot. Rolling JSON summary of older messages, stored as conversation state with reserved slug `conversation_summary`. Fires async post-turn when effective input tokens cross the summarizer agent's `summarization_threshold_tokens` (default 120000; see below). Non-background agents see the summary as a `<conversation_summary>` block and can call `expand_summary(section_id)` to fetch original messages. Schema is user-editable but must satisfy the anchor invariant (`sections[].id`, `message_id_start`, `message_id_end`) |

`transfer_to_agent` targets **any same-track agent** — a `*_specialist` *or* the entry `pilot`/`copilot`, so the pilot can act as a **hub-and-spoke router** (specialists transfer cross-domain requests back to it for re-dispatch). Cross-track and advisor/background roles are never transfer targets.

For role flow, delegation, and settings depth, see [`references/agents/roles-and-settings.md`](references/agents/roles-and-settings.md).

### Handoff context engineering

When a conversation changes hands — `transfer_to_agent` or `transfer_to_team` — whoever resumes rebuilds history from scratch, where **every prior agent's turns appear as undifferentiated `assistant` messages** and the human team's turns appear unattributed (the model can't tell which turns it authored vs. inherited). The runtime closes that gap automatically — author your agents to cooperate with it:

- **The runtime persists a durable custody marker** (`This conversation was handed off from X to Y.`) on each `transfer_to_agent` and `transfer_to_team`, and delivers a **one-time continuation note** to a receiving *agent's* first turn (agent→agent only — a team handoff has no AI receiver to brief). You don't write these — so **don't** put "you were just transferred this conversation" framing in an agent's instructions; it's handled and would double up.
- **Always open each agent's instructions with its identity** — `You are <Agent Name>, the <role/purpose>…`. The runtime reinforces identity at the handoff moment, but the system prompt is the strongest signal and the only one present on every steady-state turn; the custody marker ("…to Y") only lands if the agent knows it *is* Y.
- **A specialist must do work, not bounce.** The runtime **blocks** delegating back to any agent that already held the conversation earlier in the same turn (A→B→A and longer revisits) — the transfer is refused with an error telling the agent to complete the task, transfer to a *different* agent, or return control to the user. So don't write a `*_specialist` whose instructions reflexively hand the conversation back to its delegator; it'll just hit the guard. (The chain resets each user turn, so re-routing to an earlier agent on a *later* turn is fine.)

### Summarizer agent config

The summarizer agent exposes `summarization_threshold_tokens` (default 120000, min 1000, max 1000000) as a top-level key in `agents/summarizer.yaml`. Lower it for testing; raise it for very long conversations. The summarizer's `connection` defaults to the pilot's; edit `agents/summarizer.yaml` to change its model or system prompt. The `conversation_summary` state's schema is round-trippable like any other state — extra fields beyond the anchors are allowed but the anchors are load-bearing. (Previously a hub-level `hub.yaml` setting — relocated to the summarizer agent.)

## Connections & Credentials

A **connection** is a configured instance of a connector (a catalog entry: LLM provider, channel API, tool API, speech service) with its credential, scoped to one hub. An **org credential** stores the secret once at the organization level; connections reference it by name — raw secrets never enter YAML.

| Category | Examples |
|----------|----------|
| **Agent** | OpenAI, Anthropic, Google AI Studio, OpenRouter (required for AI) |
| **Channel** | WhatsApp, Instagram (OAuth — UI only); Resend (email), Telegram (API Key — auto-created) |
| **Tool — Native** | Wayai (auto-created), Google Calendar (OAuth), External Resources (API Key) |
| **Tool — Custom** | User-defined HTTP endpoints (API Key, Bearer Token, Basic Auth) |
| **Tool — MCP** | External MCP servers (Streamable HTTP) — Bearer Token via CLI; OAuth via UI |
| **Speech** | STT transcribes inbound voice notes (Groq, OpenAI); TTS synthesizes spoken replies (OpenAI, Groq, ElevenLabs) |

**Auto-creation rule:** Non-OAuth connections (Agent, STT, TTS, Tool — Custom, Tool — MCP via Bearer Token) are auto-created from matching organization credentials when `hub.yaml` is pushed. Matching respects **org tags** (an untagged hub sees only untagged credentials; a tagged hub sees credentials sharing ≥1 tag) and credential `environment`. OAuth connections must be set up in the UI first.

**OAuth connection handoff (any time — not just onboarding):** OAuth connections (WhatsApp, Instagram, Google Calendar, **MCP OAuth**) can't be created from the CLI — they need a one-time UI flow. **Whenever** one is needed — first-time setup *or* later (a new channel, an OAuth MCP server) — hand the user the full-path connections-tab deeplink `https://app.wayai.pro/settings/organizations/<orgId>/hubs/<hubId>/connections?connector=<slug>` (`<orgId>`/`<hubId>` from `wayai status --json`; `<slug>` ∈ `whatsapp`, `instagram`, `google-calendar`, `mcp-server`), then `wayai pull -y` once they're done. The deeplink opens the **Connections** tab (and highlights the connector if a connection already exists — e.g. re-auth); to create one the user clicks **Add Connection**, picks the **\<Connector\>** card, chooses **OAuth**, and finishes the provider flow. Use this tab form — **not** `/connections/new?connector=…`, which takes a `connector_id` UUID and defaults to the first auth type (MCP → Bearer Token), so it can't reach MCP OAuth (see [navigation.md](references/navigation.md)).

For per-provider setup, credential binding (`credential:`, `no_auth:`), tags, and production-credential decoupling, see [`references/connections.md`](references/connections.md).

## Channels

Communication endpoints on a hub — where messages arrive and replies get delivered. **Channels are never authored in YAML:**

- `app` (in-app chat) and `system` (internal) channels are created automatically with the hub
- WhatsApp / Instagram / Email (Resend) / Telegram channels are provisioned automatically when their **Channel connection** is created

Channel uniqueness (phone / page / inbound address) is enforced across **production** hubs only — a preview can share endpoints with its production, and external channels are testable on previews via `#test CODE` tester registration (see Hub Environments).

## Tools

Capabilities assigned per agent in `agents/<slug>.yaml`. Remember: replying with text needs no tool — tools are for everything else.

| Type | Source | How |
|------|--------|-----|
| Native | Platform built-ins (e.g., `update_kanban_status`, `get_state`, `send_files`, `close_conversation`, `read_file`) | Listed by name under `tools.native` |
| Custom | HTTP endpoints you define | Defined under `tools.custom` with `connection`, `method`, `path`, `config` |
| MCP | Tools from connected MCP servers | Dual-origin — declared per-agent under `tools.mcp` **and/or** assigned in the Platform UI. `wayai push` discovers + assigns in one run; a present `mcp` key (even `[]`) is authoritative, an omitted one preserves UI-assigned tools. See [native-tools.md](references/agents/native-tools.md#mcp-tools) |
| Delegation | Agent-to-agent (`transfer_to_agent`, `consult_agent`) or agent-to-team (`transfer_to_team`) | Declared under `tools.delegation` with `target` (agent display name or team name) |

Meta tools (`get_tool_schema`, `execute_tool`) let agents call tools whose schemas are excluded from the inline list. Full native catalog + params: [`references/agents/native-tools.md`](references/agents/native-tools.md); custom tool schema: [`references/agents/custom-tools.md`](references/agents/custom-tools.md); designing *which* tools/params to expose: [`references/agents/tool-principles.md`](references/agents/tool-principles.md).

## Kanban & States

**Kanban statuses** are workflow stages for conversations (visible in support/task views), defined per hub in `hub.yaml`:

- Identity: immutable lowercase `slug` (stored in conversations, analytics, tool params; **never renameable**) + freely editable display `name`. Tools accept only slugs (display names ride along as labels) — instructions must reference statuses by slug
- Behavioral flags: `isInitialStatus` (exactly one per hub), `triggersAgentResponse` (transition fires an agent turn), `allowsAgentUpdate`, `isTerminalStatus` (**entering it closes the conversation**), `isSchedulingStatus` (+ `eventName`). Several combinations are mutually exclusive — validated server-side on every write
- `allowed_next_statuses` — optional transition allowlist, enforced at runtime on every surface. Omit = unrestricted; `[]` rejected (use `isTerminalStatus`)
- **Followups** — per-status timed messages: `inactivity` (after silence) or `before_event` (requires `isSchedulingStatus`), with threshold/timeUnit, quiet hours, holiday exclusion
- **Additional context on transition** — a `triggersAgentResponse` status may declare `additional_context_schema` (JSON-Schema form the team fills on transition) + `additional_instructions` (prose template with `{{path.to.field}}` / `{{additional_data}}` placeholders injected into the triggered turn)
- **Lanes** — optional presentational board grouping; no behavioral effect

Full field specs, constraint matrix, warnings, and a complete example: [`references/kanban.md`](references/kanban.md).

**States** are JSON-schema data agents read/write during conversations — via native tools (`get_state`, `update_state`, `set_state_path`, `reset_state`, all addressing a state by its `slug`) and the `{{state(scope, slug)}}` instruction placeholder. Each state has `conversation` or `user` scope, a `json_schema`, and an optional `initial_value` (pre-populated virtual record rendered until the first real write; omit to keep state silent until written).

**Kanban vs State:** kanban tracks workflow progression; state tracks structured data. Both coexist. Schemas and patterns: [`references/states.md`](references/states.md).

## Resources

Knowledge and skills attached to agents. Content lives as real files under `resources/<slugified-name>/` (the filesystem is the source of truth **for resource content**); `hub.yaml` `resources:` declares only name/type/description.

| Type | What | Runtime behavior |
|------|------|------------------|
| `knowledge` (default) | Document collections — FAQ, catalogs, policies | The linked resource ids are injected into the `list_resource_files` / `list_resource_folders` native-tool schemas at turn time; the agent explores content via those tools + `read_file` |
| `skill` | Versioned capability package — `SKILL.md` (frontmatter `name` + `description`) + optional `references/` | Injected as a callable tool (default, works on all providers), or run natively in a provider container (`use_native_integration: true`, Anthropic/OpenAI only; auto-syncs to the provider on `wayai push`, `wayai sync-skills` re-syncs after failures or late-added connections) |

Agents link resources in `agents/<slug>.yaml` under a `resources:` block (by name, with `priority`). Org-level resources shared across hubs live in `wayai-ws/org/` via `wayai org pull/push` (push fans out to linked hubs).

File handling (text vs binary, 10 MB cap), skill authoring, execution modes: [`references/resources.md`](references/resources.md).

## Evals

Test scenarios that run the **real** agent with its **real** tools and score the result. The primitives:

- **Scenario** (`evals/<name>.yaml` or `evals/<set>/<name>.yaml`) — optional multi-turn `history`, one `input`, an `expected` response (text and/or `tool_calls`), optional `evaluator_instructions`. Scored by the hub's `message_evaluator` agent; a required-but-skipped tool call fails the eval even when the reply text reads fine
- **Scenario set** — first-level subfolder (one level only). `wayai run-eval` runs exactly one set per session
- **Journey** (`journeys/<slug>.yaml`, flat folder) — a stored happy-path transcript that materializes one derived eval per agent turn. The default way to build broad regression coverage: `wayai eval journey capture <conversation_id>`, then `wayai pull` (syncs server-minted step ids)
- **Per-run `variables`** + `runs: N` — reliability is a distribution, not a 1/1 sample; each run resolves `{{var(name)}}` against its own disjoint row
- **Seed `fixture:`** — for any eval that *writes*: names a Rekor fixture that `run-eval` resets before the session and clears after, so runs start from a known baseline instead of the last run's residue
- **Capture** — `wayai eval capture <conversation_id>` freezes a production conversation's last exchange into a scenario YAML

Good practice for tool-dependent evals: compose **journey + `fixture:` + `variables`** for repeatable, parallel runs. Full YAML shapes, seed-connection setup, run pacing, and authoring/interpreting principles: [`references/evals.md`](references/evals.md).

## Outbound

Proactive messaging — the hub contacts people before they write. Three `hub.yaml` blocks:

- `outbound_contacts` — named contacts with ≥1 channel identifier (`phone` E.164 / `email` / `instagram_sid`) + free-form tags
- `outbound_lists` — named static collections of contacts (referenced by contact name)
- `outbound_schedules` — cron expression + timezone + list + channel + execution mode: `direct_message` (template / free text sent as-is) or `agent_trigger` (a system message triggers the agent, which opens the conversation naturally using its tools and instructions)

WhatsApp/Instagram delivery is constrained by the 24-hour messaging window (WhatsApp falls back to an approved template; Instagram skips). Inline contacts are practical to ~500 — beyond that, import via UI/API. Shapes, channel rules, limits: [`references/outbound.md`](references/outbound.md).

## Analytics

Every conversation lands in the analytics store with variables from five origins:

| Origin | Path | Set by |
|--------|------|--------|
| System metrics | `data.system.*` | Platform — message counts, response times, durations, tokens (~25 metrics) |
| Agent-defined variables | `data.variables.*` | `evaluation_variables` declared on `conversation_evaluator` / `message_evaluator` agents |
| Metadata | `data.meta.*` | Platform — subject, kanban_status, hub_type |
| Post-hoc annotations | `data.annotations.*` | `wayai conversations <id> annotate --set key=value` — real business outcomes (purchased, churned) recorded after the conversation ends; correlate predictions vs reality |
| Eval scores | `data.eval_scores.*` | Eval runs only (`is_eval = true` rows — excluded from production analytics) |

Query with `wayai analytics` (summary + per-variable aggregates; `--metric`, `--filter`, `--period`), `wayai analytics query` (structured: multi-variable, group_by, correlations), or `wayai evals sql` (raw SQL over eval rows). Defining *good* variables happens on the evaluator agents ([roles-and-settings.md → Evaluation Variables](references/agents/roles-and-settings.md#evaluation-variables)); filters, aggregations, and workflows: [`references/analytics.md`](references/analytics.md).

## Teams, Users & Access

People entities are **UI-managed** (Hub → Users tab: `/settings/organizations/<orgId>/hubs/<hubId>/users`), never in YAML:

- **Hub User** — the end user the AI talks to (customer/lead/employee). Uses `/chat` or `/task`
- **Hub Team User** — support team member handling conversations in `/support`; grouped into **Teams** (e.g. "Tier 2 Support") that `transfer_to_team` targets by name — an unknown `target` fails at runtime
- **Hub Admin** — full hub config access. **Org Owner/Admin** — org level (billing, credentials, hubs). Access is per-level, not inherited (an org admin isn't automatically a hub admin)
- **Contact access control** — with `non_app_permission: require_permission`, unknown channel contacts are held `pending` (localized auto-reply, overridable via `access_request_message`) until approved/blocked by the role in `access_approval_role`
- **MCP access** — whether external MCP clients reach the hub's tools: `mcp_access` (see Hub Settings; UI-only)

## Hub Environments

| Environment | Description |
|-------------|-------------|
| `preview` | Default. Editable workspace for configuring and testing |
| `production` | Read-only. Serves live traffic. Changes flow from preview via publish/sync |

**Lifecycle:**
1. New hubs start as `preview` — edit freely. `wayai create --label <l>` (or `wayai push --label <l>` on auto-create) names the first preview at creation
2. **Publish** (CLI `wayai publish`, or UI) — first promotion creates a `production` hub cloned from preview
3. **Sync** (CLI `wayai publish` / alias `wayai sync`, or UI) — pushes subsequent preview changes to the linked production. The one command auto-detects first-publish vs sync; it confirms by default (shows the preview→production diff) and `-y` skips the prompt. Promotes the pushed **preview** state, so `wayai push` first
4. **Replicate Preview** (CLI `wayai replicate [hub] --label <l>` or UI) — creates a new sibling preview (from a preview or production) for experimentation
5. **Relabel** (CLI `wayai relabel <label>` / `--clear`, or UI) — set/clear a preview's `preview_label` (the sibling disambiguator). NOT editable via `hub.yaml` + push — it's server-owned

Production is read-only — all config mutations flow through preview. Multiple previews can link to the same production (many-to-1). Channel uniqueness is enforced on production only — previews can share phone/email/SID with their production.

WhatsApp/Instagram/Telegram channels can be exercised on a preview before publishing — register a tester via a `#test CODE` claim code (see `references/connections.md` → Channel → "Testing a channel on a preview before publishing").

Only preview hubs are editable. `wayai pull` also writes the linked production hub as a **read-only mirror** folder (bare slug — no `--<label>` suffix — with a marker comment in `hub.yaml`) so the live production config is browsable alongside the preview; `wayai push` refuses it and it's excluded from auto-select. Use `wayai diff --production` for a clean preview-vs-production diff.

## Hub Settings

| Setting | Values | Default | Description |
|---------|--------|---------|-------------|
| `non_app_permission` | `everyone`, `require_permission`, `not_allowed` | `everyone` | Who can reach hub via external channels |
| `timezone` | IANA timezone | — | For scheduling and display |
| `language` | `en`, `pt`, `es` | `en` | Language for hub-sent text (e.g. the pending-access notice on `require_permission` channels) |
| `access_approval_role` | `admin`, `team` | `admin` | Who may approve/block a pending contact: hub admins only, or also support team members |
| `access_request_message` | string | — | Optional override for the "your access is pending approval" auto-reply (else a localized default by `language`) |
| `auto_close_inactive_days` | `1`–`180` | `7` | Days of inactivity (no user/team message) before a conversation is force-closed. Every hub has one |
| `conversation_retention_days` | `1`–`30` | `7` | Days an ended conversation's DO stays alive for post-hoc `annotate` before cleanup (archival still happens at close) |
| `mcp_access` | `disabled`, `read_only`, `read_write` | `read_only` | Whether external MCP clients can reach this hub's tools. **UI only** (Hub → Users tab) — not settable via `hub.yaml`; `read_write` downgrades to `read_only` when published to production |

## First-time setup (cold start)

The user's entry point is `wayai.pro/docs/get-started`, which routes the agent to the install command for its harness. Once the skill loads (here), this section drives everything from "skill loaded" to "hub responding to test messages." The agent must self-bootstrap from this section alone — returning users (second hub, new project) skip the page.

**Always start by running `wayai status --json`.** It returns a single state snapshot. Branch from the result; re-run between steps to confirm progress before moving on (idempotent).

### Phrasing templates (use verbatim)

- **Agent action** — narrate one line then act, then one-line receipt:
  - "Installing the CLI…" → run command → "Done — `wayai 2.4.1` installed."
- **User handoff** — exactly one URL, one action, one return signal:
  - "Open `<URL>`. Do `<one action>`. Tell me when done."

### State machine

| # | Detection (from `status --json` or env) | Action |
|---|---|---|
| 1 | CLI missing (`wayai --version` not found) | Agent runs `npm i -g @wayai/cli@latest`. |
| 1b | No harness skill install present at project root (none of `<root>/.claude/skills/wayai/SKILL.md`, `<root>/.opencode/skills/wayai/SKILL.md`, `<root>/.agents/skills/wayai/SKILL.md` exists, where `<root>` is `git rev-parse --show-toplevel` or cwd if not in a git repo) | **If you are Claude Code, first `mkdir -p <root>/.claude`** — the `skills` installer links Claude Code's `.claude/skills/wayai` only when `.claude/` already exists, else it silently skips it while still printing "symlinked: Claude Code". Then run `npx skills add wayai-pro/wayai-skill -y` from `<root>`. After it completes, verify **your own harness's** path resolves (Claude Code → `<root>/.claude/skills/wayai/SKILL.md`, following the symlink; other harnesses → `<root>/.agents/skills/wayai/SKILL.md`). If it resolves, exit (the harness loads the skill next turn). If it doesn't but `<root>/.agents/skills/wayai/SKILL.md` exists, self-heal: `mkdir -p <root>/.claude/skills && ln -sfn ../../.agents/skills/wayai <root>/.claude/skills/wayai`, then re-verify. If still nothing, surface the install error and halt — do not silently exit (would loop on re-entry). |
| 1c | `skill.installed: true`, `skill.latest` is set, and `skill.latest` is newer than `skill.version` (CLI nightly check populates `skill.latest`) | Agent runs `npx skills add wayai-pro/wayai-skill -y` from `<root>` to refresh the install in place, then exits (the updated skill loads on the next turn). On install failure, surface the error to the user and continue with the existing skill. |
| 2 | `auth.logged_in: false` | Agent runs `wayai login` (opens browser). User handoff: "Open the page that just opened. Sign in or sign up. Tell me when done." |
| 3 | `auth.logged_in: true`, `orgs: []` | Ask once: "What should we name your organization? (usually your company name)". Then agent runs `wayai org create "<name>"` and re-runs `status --json` to pick up the new org. (Manual fallback only if the CLI create errors: open `https://app.wayai.pro/settings/organizations/new`.) |
| 3b | `git rev-parse --show-toplevel` fails (cwd is not in a git repo) | The CLI requires git for workspace detection. **Before acting, surface the resolved cwd and confirm with the user** — handoff: "I'll initialize a git repo at `<cwd>`. Confirm or pick a different folder." This prevents accidentally initializing a repo in `~` or another unintended directory. After confirmation, agent runs `git init` in cwd. Then checks `git config --global user.name` and `git config --global user.email`; if either is empty, asks the user once for their name and email and runs `git config --global user.name "<name>"` / `git config --global user.email "<email>"`. A GitHub remote is not required for the onboarding flow — only set one up later if the user wants the GitOps/CI loop. |
| 4 | `workspace.scoped: false` | Agent runs `wayai init --org <active_org.id>`. |
| 5 | Workspace scoped, hub goal not yet known | User handoff: "What should this hub do? Describe the goal, who talks to it, and the main use case." |
| 6 | LLM credential missing for chosen provider | User handoff: "Paste your OpenAI/Anthropic/Google API key here." Then agent runs `wayai create-credential --name <name> --type "Bearer Token" --stdin`. |
| 7 | Hub needs an OAuth connection (WhatsApp / Instagram / Google Calendar / MCP OAuth) | Apply the **OAuth connection handoff** (Connections & Credentials → OAuth connection handoff): send the full-path connections deeplink for the connector, wait for completion, then `wayai pull -y`. The same handoff applies any time an OAuth connection is needed later, not only here. |
| 8 | Prerequisites met | Read [`references/canonical-example/README.md`](references/canonical-example/README.md) once for end-to-end wiring, then generate `wayai-ws/hubs/<hub>/hub.yaml` + `agents/*.yaml` + `agents/*.md` from the user's description (per-domain refs below for individual shapes), then `wayai create -y` to create + push the new hub (`wayai push -y` also auto-creates when the workspace has just this one new folder). |
| 9 | Push succeeded | Agent runs `wayai send-message "Hi"` and shows the response. User handoff: "Refine, add tools, or publish?" |
| 10 | User confirms publish | Agent runs `wayai publish` — shows the preview→production diff, then confirms (or `wayai publish -y` to skip the prompt). First publish clones preview → a new production hub; later runs sync. **Paid plans only** — if the CLI reports publishing requires a paid plan, manual fallback: open the publish deeplink below to upgrade + Publish in the UI. |

### Deeplinks (canonical URLs — never breadcrumbs)

| State | URL |
|-------|-----|
| 3 (org create — manual fallback) | `https://app.wayai.pro/settings/organizations/new` |
| 6 (credential pre-fill) | `https://app.wayai.pro/settings/organizations/<org_id>/credentials?type=bearer&name=<key-name>&prefill=true` |
| 7 (OAuth connection) | `https://app.wayai.pro/settings/organizations/<org_id>/hubs/<hub_id>/connections?connector=<whatsapp\|instagram\|google-calendar\|mcp-server>` |
| 10 (publish — manual fallback) | `https://app.wayai.pro/settings/organizations/<org_id>/hubs/<hub_id>/overview?action=publish` |

### Rules

- Detect, don't assume — every step starts with a fresh `wayai status --json`.
- Agent actions get a one-line receipt; user handoffs get one URL, one action, one return signal.
- Never invent URLs outside the table above. Never instruct "go to Settings → …" — always a deeplink. For non-onboarding URLs see [`references/navigation.md`](references/navigation.md).
- One question at a time during hub scoping (state 5): goal, then channel, then LLM provider.
- Never auto-commit anything created during onboarding — show `git diff` and wait for user approval before any commit.

## Workflow

### Existing hub
1. **Update CLI** — `wayai update` (always run before any operation; if the CLI isn't installed yet, bootstrap with `npm i -g @wayai/cli@latest`)
2. **Update skill if stale** — run `wayai status --json`; if `skill.latest` is set and newer than `skill.version`, run `npx skills add wayai-pro/wayai-skill -y` and exit (the refreshed skill loads on the next turn). Otherwise continue. (Cold-start onboarding runs the same check as state-machine row 1c.)
3. **Pull** — `wayai pull -y` (sync local files from platform; catches out-of-band changes)
4. **Read context** — first ensure the repo-root `AGENTS.md` matches [`references/agents-md-template.md`](references/agents-md-template.md) (write it if missing or drifted — it's the session bootstrap the CLI seeds), then read `wayai-ws/hubs/<hub>/AGENTS.md` for this hub's notes (purpose, decisions, ongoing work). AGENTS.md-aware harnesses (Codex, Cursor, OpenCode, Aider) auto-load `AGENTS.md` natively
5. **Edit** — modify `hub.yaml`, `agents/*.yaml`, `agents/*.md`
6. **Push** — `wayai push -y` (apply to preview hub; auto-pulls server-assigned IDs back)
7. **Test** — `wayai send-message "Hello"`
8. **Review** — run `git diff`, ask user to confirm. **Never auto-commit.** User commits and pushes to `main`
9. **Go live** — `wayai publish` (or the platform UI) when ready: shows the preview→production diff, confirms, then first-publishes (clones preview → new production hub) or syncs subsequent changes. `push` first — it promotes the pushed preview state, not unpushed local edits

If the hub folder has a `progress.html`, refresh it after the task's last push or publish — see [`references/progress-report.md`](references/progress-report.md) for when and how.

### New hub (from scratch)
1. **Credentials** — `wayai create-credential --name "openai-key" --type "Bearer Token"` (one-time per org per credential)
2. **Init** — `wayai init` (interactive) or `wayai init --org <uuid>`
3. **Create files** — `wayai-ws/hubs/<hub>/hub.yaml` + `agents/*.yaml` + `agents/*.md`
4. **Create** — `wayai create -y` creates the hub, non-OAuth connections, and applies all config (auto-binds the worktree). `create <folder>` when the workspace has more than one hub folder. (`wayai push -y` also auto-creates when it resolves to a single new folder; in a multi-hub workspace it requires `--hub`, so `create` is the explicit, unambiguous verb.)
5. **Test** — `wayai send-message "Hello"`

After the hub exists, follow the existing-hub workflow.

### Hub-Folder Memory

`wayai-ws/hubs/<hub>/AGENTS.md` is the **hub-specific memory** for this hub — the contextual information the config files (`hub.yaml`, `agents/`) can't capture: purpose, key decisions and *why*, ongoing work, business rules that only apply here, terminology, integration quirks. Read it at the start of every hub-related task. `wayai pull`/`push` seed a placeholder `AGENTS.md` (+ a `CLAUDE.md` shim) if the folder has none, absence-guarded. AGENTS.md-aware harnesses (Codex, Cursor, OpenCode, Aider) auto-load it natively when the agent's cwd is inside the hub folder.

**Maintain it actively:**
- After significant changes (new agent, new tool, business rule update), update `AGENTS.md` so future sessions inherit the context
- If a hub folder has no `AGENTS.md` (or only the seeded placeholder), fill it with what you know — purpose, current agents, recent decisions — and ask the user to confirm or enrich
- Keep it focused on *why* decisions were made and *what* makes this hub different. Don't restate platform mechanics — those live in this skill
- Record agreed scope in an optional `## Build plan` checkbox section and tick items as they land — the progress report renders it as plan-vs-actual ([`references/progress-report.md`](references/progress-report.md#the-build-plan-convention))
- **Rekor bases get the same treatment.** If you're also working with a Rekor base, record the context its settings can't capture in the base folder's `AGENTS.md` (create it + a `CLAUDE.md` shim if missing) — the Rekor skill owns the details

**Overflow content goes into `wayai-ws/hubs/<hub>/references/`:**
- When `AGENTS.md` grows past ~200 lines or starts mixing topics, extract the deeper material into focused files under `wayai-ws/hubs/<hub>/references/`
- Examples: `references/business-rules.md`, `references/integrations.md`, `references/glossary.md`, `references/<api-name>-spec.md`, `references/<persona>-tone.md`
- Keep `AGENTS.md` as the always-on entry point with a short pointer to each reference file: e.g., "Detailed pricing rules: `references/pricing-rules.md`"
- Hub-folder `references/` are **not synced** to the platform — they're for agent context only, just like `AGENTS.md`

## Common CLI Commands

```bash
wayai update            # Update CLI (run before any operation)
wayai login             # OAuth — or `wayai login --token` for headless/CI
wayai logout            # Sign out and clear stored credentials
wayai whoami            # Show the authenticated identity
wayai org create        # Create a new organization (you become its owner): `wayai org create "<name>"` [--region <r>] [--json]
wayai org pull          # Org-as-code: fetch org-level shared resources → wayai-ws/org/
wayai org push          # Org-as-code: apply local org resources to the platform (fans out to linked hubs); `wayai org diff` previews
wayai create-credential # Create org credential (--name, --type "API Key"|"Bearer Token"|"Basic Auth", --org, --stdin)
wayai update-credential # Rotate/edit an org credential (--name <cred>; --stdin/--secret rotates the secret, --rename, --description, --tag, --environment)
wayai set-connection-credential  # Set a connection's credential directly — --connection <name> + either --org-credential <name> (link) or --field <f> --stdin (raw secret). Works on preview + production (the sanctioned production-credential write)
wayai init              # Set up .wayai.yaml (interactive — creates an org inline if you have none); --org <uuid> to skip prompt
wayai migrate           # Move a legacy workspace/ + root org/ layout to wayai-ws/
wayai pull              # Pull hub config from platform (-y skips confirmation; auto-binds worktree on first pull). Also writes the linked production hub as a read-only mirror folder
wayai push              # Push local changes (-y skips confirmation; auto-pulls IDs back). Auto-creates a lone new folder; a multi-hub workspace needs --hub or `wayai create`
wayai create [folder]   # Explicitly create a new hub from an idless folder, then push (--label names the preview). The discoverable verb when a workspace has more than one hub folder
wayai diff              # Dry-run diff of local files vs preview (read-only); --production diffs vs the linked production hub
wayai replicate [hub]   # Clone a hub (preview or production) into a new sibling preview; --label <l> names it. Pulls the new preview into its own folder
wayai relabel <label>   # Set a preview hub's label (--clear removes it; --hub to target). Renames the local folder. The server-owned way to change preview_label
wayai publish           # Promote the preview to production (alias: wayai sync). Auto-detects first-publish (clones preview → new production) vs sync (pushes changes to the linked production); shows the preview→production diff and confirms (-y skips). Paid plans only. `wayai push` first — promotes the pushed preview state
wayai use <hub>         # Bind this worktree to a specific hub (UUID or folder name)
wayai unbind            # Clear the worktree hub binding
wayai template list     # List ready-made hub templates (gym, clinic, …) — browse what's available
wayai template pull <slug> [--lang <locale>] [--force] [--dry-run]  # Write a template's config into wayai-ws/hubs/<slug>/, then `wayai push`. --lang picks a localized variant (en|pt|es); omitted = the template's default, an untranslated language falls back with a note. Refuses to overwrite existing files in the target dir (lists them) — pass --force to overwrite; --dry-run previews the file map without writing
wayai send-message      # Test message to a hub (preview or production)
wayai conversations     # List or inspect conversations (default text view omits message_id — use --json or `observability` to discover ids)
                        # `wayai conversations <id> observability` — list LLM turns with message_id, latency, tool_calls (assistant turns only)
                        # `wayai conversations <id> observability --message-id <id>` — full record for one turn (prompt, completion, tool calls, tokens; --json for raw)
                        # `wayai conversations <id> annotate --set key=value [--type numeric|categorical|text]` — set a post-hoc business outcome (e.g. customer_purchased=true) on an ended conversation as an analytics dimension; repeat --set for multiple keys (needs the hub within its conversation_retention_days window)
wayai delete-history    # Clear conversation history (testing); --conversation-id <id> deletes just one
wayai sync-skills       # Sync skills to provider connections; --connection-id <uuid> to scope
wayai sync-mcp          # Re-discover an MCP connection's tools (refresh stale schemas); --connection <name|uuid>
wayai analytics         # Summary + per-variable aggregates; --metric, --filter, --period, --json
wayai analytics query   # Structured ClickHouse query (multi-variable, group_by, correlations)
wayai evals             # List eval scenarios for the hub (--enabled / --disabled)
wayai evals sql         # Raw single-SELECT SQL over the hub's eval result rows ("SELECT …"; --schema prints the column + eval-score-path catalog; --limit, --json)
wayai run-eval          # Run a scenario set's enabled evals (sole set by default; --set/--eval to pick on multi-set hubs; --pacing conservative|balanced|fast|<ms> to throttle run dispatch)
wayai eval-results      # Inspect eval results (--session <id> or --eval <name>; --runs for per-run detail, --json for raw)
wayai eval capture      # Capture production conversation as eval YAML (<conversation_id> [--set <name>])
wayai eval journey capture  # Capture a conversation's FULL transcript as a journey (<conversation_id> [--name <n>]); then `wayai pull` to sync it to journeys/<slug>.yaml
wayai eval session delete   # Delete an eval session + its run history (<session_id>, or --all for every session on the hub; -y to skip confirm)
# Journeys are hub-as-code: edit journeys/<slug>.yaml and `wayai push` (pull after first create to sync step ids)
wayai list              # List organizations and hubs
wayai status            # Show workspace status
wayai report create     # Create platform bug report (--title, --description, --hub, --conversation, --error)
wayai report edit       # Amend your own pending report (<id> --title/--description/--error/--steps/--context)
wayai report list       # List your reports (newest first; --status <s>, --json)
wayai report get        # Show a report's status + message thread (<id>, --json)
wayai report accept     # Accept a shipped fix (<id>) → addressed
wayai report contest    # Contest a shipped fix or a dismissal (<id> --reason "...") → back to triage
```

**Closing the loop on a report you filed.** After triage escalates and the fix ships, your report
moves to `shipped` — you'll get an email, and `wayai login`/`wayai status` remind you once (or find it
with `wayai report list --status shipped`). Read `wayai report get <id>` (status + the fixer's note in
the thread), then `accept` if it works or `contest --reason "..."` if it doesn't. You can also `contest` a `dismissed` report you believe is real — it routes back to triage.
Contests are bounded (a cap, and triage may mark a dismissal final); past those, contact support.

Most commands accept `--hub <uuid|folder>` to disambiguate when multiple hubs live in `wayai-ws/hubs/`.

### Worktree hub binding

Each git checkout (main or linked worktree) can be bound to a single hub. `wayai push` and `wayai pull` refuse to run against a different hub once bound — this catches the common mistake of a prompt being routed to the wrong terminal/worktree. The binding is auto-set on the first successful pull (or new-hub creation) into an unbound checkout, and lives at `<git-dir>/wayai-binding` (per-checkout, never tracked). It is a routing tripwire, not a concurrency lock — it does not coordinate concurrent edits to the same hub.

If `push`/`pull` errors with a binding mismatch, **stop and ask the user before doing anything else**. It usually means a prompt was meant for a different worktree. Do **not** run `wayai unbind`, `wayai use`, or modify `.git/wayai-binding` without explicit user instruction in the current session — these are session-routing actions, equivalent to changing which hub the user thinks you're working on.

## Repository Structure

```
.wayai.yaml                              # Repo config — organization scope (init-only)
AGENTS.md                                # Session bootstrap — seeded by the CLI (init/pull/push), reconciled against references/agents-md-template.md; yours to edit
CLAUDE.md                                # Root Claude Code shim — `@AGENTS.md` (seeded if absent, NOT overwritten)
.claude/skills/wayai/                    # Claude Code skill install (provisioned by `npx skills add wayai-pro/wayai-skill -y`)
├── SKILL.md
└── references/                          # On-demand deep-dive references (see index below)
.opencode/skills/wayai/                  # OpenCode skill install (same provisioner; same SKILL.md + references/ layout)
.agents/skills/wayai/                    # Neutral skill install (same provisioner; same SKILL.md + references/ layout)
wayai-ws/                                # All WayAI hub-as-code (init creates wayai-ws/hubs/)
├── org/                                 # Org-as-code — shared resources (wayai org pull/push)
│   ├── resources.yaml
│   └── resources/<slug>/
└── hubs/
    └── <hub-slug>--<label>/             # One folder per preview hub (disambiguated)
        ├── hub.yaml                     # Hub config + states + connections + outbound + resources
        ├── agents/
        │   ├── <slug>.yaml              # Agent config (one per agent, slugified name)
        │   └── <slug>.md                # Agent instructions (one per agent)
        ├── evals/                       # Eval scenarios (synced)
        │   ├── <name>.yaml
        │   └── <set>/<name>.yaml
        ├── journeys/                    # Eval journeys (synced; flat folder, one file per journey)
        │   └── <slug>.yaml
        ├── resources/                   # Knowledge & skill resource files (synced)
        ├── AGENTS.md                    # Hub-specific memory — scaffold seeded on pull/push (NOT synced; fill it in)
        ├── CLAUDE.md                    # Per-hub Claude Code shim — `@AGENTS.md` (seeded if absent, NOT synced)
        └── references/                  # Hub-specific supporting files (NOT synced)
    └── <hub-slug>/                      # Linked production hub: READ-ONLY mirror (bare slug, no --label). Refreshed each pull; never pushed
```

Preview hub folders use `hub-slug--<preview_label>` or `hub-slug--<hub_id_prefix>` for disambiguation; the linked production hub mirrors to the bare `hub-slug` (no suffix). **Existing repos** that still use the legacy `workspace/` + root `org/` layout keep working (the CLI reads either) — run `wayai migrate` to move to `wayai-ws/`.

## `hub.yaml` Shape

```yaml
version: 1
hub_id: "abc-123-def"           # set by `wayai pull` — do not edit
hub_environment: preview         # set by `wayai pull` — do not edit
preview_label: experiment-a      # server-owned, set by `wayai pull` — do not edit (only on previews; sets the hub folder's `--<label>` suffix). Editing here is IGNORED on push (push warns); change it with `wayai relabel <label>` / `--clear`, or set it at creation with `wayai create --label` / `wayai replicate --label` / `wayai push --label`

hub:
  name: Customer Support
  # description: Handles refunds, order status, and billing questions   # optional, human-facing
  hub_type: chat                 # chat | task
  ai_mode: pilot+copilot         # pilot | copilot | pilot+copilot | turned_off
  timezone: America/New_York
  non_app_permission: everyone
  # tags: [retail, vip]          # org tag slug names (create in UI first); gate which org credentials this hub can resolve. Omit to leave unchanged; [] clears. See references/connections.md#organization-tags
  # auto_close_inactive_days: 7  # force-close a conversation after N days of inactivity (see Hub Settings)
  # conversation_retention_days: 7  # keep an ended conversation's DO alive N days for post-hoc `annotate` (see Hub Settings)
  kanban_statuses:               # full field specs + constraints: references/kanban.md
    - slug: new
      name: New
      order: 0
      color: "#22c55e"
      isInitialStatus: true
      allowed_next_statuses: [in_progress, waiting_for_customer, resolved]
    - slug: in_progress
      name: In Progress
      order: 1
      color: "#3b82f6"
      triggersAgentResponse: true   # transition fires an agent turn; can carry additional_context_schema + additional_instructions (see references/kanban.md)
      allowed_next_statuses: [waiting_for_customer, resolved]
    - slug: waiting_for_customer
      name: Waiting for Customer
      order: 2
      color: "#f59e0b"
      allowed_next_statuses: [resolved]
      followups:
        - order: 0
          type: inactivity
          threshold: 30
          timeUnit: minutes
          instructions: "Hi! Just checking in — do you still need help?"
    - slug: resolved
      name: Resolved
      order: 3
      color: "#ef4444"
      isTerminalStatus: true

states:
  - id: "state-uuid-789"          # set by pull
    slug: order_tracking           # immutable identifier; auto-derived from name if omitted
    name: order_tracking
    scope: conversation            # conversation | user
    description: Tracks current order
    json_schema:
      type: object
      properties:
        order_id: { type: string }
        status: { type: string, enum: [pending, shipped, delivered] }
    initial_value: { order_id: null, status: null }

connections:
  - name: anthropic
    type: Agent
    service: Anthropic
    # sync_credentials_to_production: true   # default; false keeps production's credential separate (set it via `wayai set-connection-credential`). See references/connections.md#credential-propagation-to-production-sync_credentials_to_production
  - name: my-api-connection
    type: Tool
    service: REST API
```

`hub.yaml` also holds `resources:`, `outbound_contacts:`, `outbound_lists:`, `outbound_schedules:` blocks. See the per-domain references for full schemas.

## `agents/<slug>.yaml` Shape

```yaml
id: "agent-uuid-123"               # set by pull
name: Pilot Agent
role: pilot
connection: anthropic              # connection display name
# instructions resolved by convention from agents/<slug>.md
# additional_context_template:     # per-turn context, emitted as an <additional_context> tag on the last user message (same {{...}} syntax as instructions). Put high-churn placeholders ({{now()}}, {{state()}}) HERE, not in instructions, to keep the system prompt cache-stable (see references/agents/instructions.md#additional-context-cache-friendly)
# response_format:                 # {schema_name, schema_json} — force structured JSON output instead of free text (see references/agents/roles-and-settings.md#response-format-structured-output)
# enabled: true                    # default; omitted
# include_message_timestamps: false  # default; when true, appends [timestamp, weekday, daypart] to user messages
settings:
  model: claude-sonnet-5
  temperature: 0.7
  max_tokens: 4096
  # reasoning per provider: Anthropic thinking_enabled + effort · OpenAI/OpenRouter reasoning_effort · Gemini reasoning_level (see roles-and-settings.md)
  # file_handling_mode: always_attach  # or metadata_only (historical files sent as metadata; agent fetches via read_file). max_attachment_size_mb caps always_attach size (see roles-and-settings.md#file-handling-all-llm-connectors)
tools:
  native:
    - update_kanban_status
    - get_state
    - update_state
  delegation:
    - type: agent
      tool: transfer_to_agent
      target: Specialist - Billing
    - type: team
      tool: transfer_to_team
      target: Tier 2 Support
  custom:
    - name: check_order_status
      description: Check order status by email
      method: post
      path: /api/orders/status
      body_format: json
      connection: my-api-connection
      config:
        name: check_order_status
        description: Check order status by email
        parameters:
          type: object
          properties:
            email: { type: string, description: Customer email }
          required: [email]
resources:
  - name: Company FAQ              # links a hub resource by name (see Resources)
    resource_id: "resource-uuid"   # set by pull
    priority: 0
```

For full agent options (settings per connector, `additional_context_template`, `response_format`, file handling, native tool params, custom tool fields, `composed_tools`, placeholders), see [`references/agents/`](references/agents/).

**Evaluation variables** — `conversation_evaluator` / `message_evaluator` agents carry an `evaluation_variables` list (the structured fields they emit per conversation/message, which become `data.variables.*` in Analytics), round-tripped via pull/push. See [`references/agents/roles-and-settings.md`](references/agents/roles-and-settings.md#evaluation-variables).

## Key Rules

- **Read-only fields:** `hub_id`, `hub_environment`, `id` — set by `wayai pull`, never edit
- **Connection auto-creation:** non-OAuth connections in `hub.yaml` resolve to org credentials by matching `service` + `authentication_type`. Use `credential` field to disambiguate when multiple org credentials share the same auth type. OAuth connections (WhatsApp, Instagram, Google Calendar, MCP OAuth) must already exist (UI setup — see OAuth connection handoff) — referenced by name only
- **Production credentials:** a connection copies its credential into production on publish/sync by default. Set `sync_credentials_to_production: false` to keep production's credential separate, then set it directly with `wayai set-connection-credential` (production is otherwise read-only). See [`references/connections.md`](references/connections.md#credential-propagation-to-production-sync_credentials_to_production)
- **Org tags:** `hub.tags` (slug names, created in the UI first) gate which org credentials the hub can resolve (matching rule in Connections & Credentials above). See [`references/connections.md`](references/connections.md#organization-tags)
- **Tool groups:** `native` (platform built-ins by name), `delegation` (agent-to-agent/team handoff), `custom` (HTTP endpoints with connection), `mcp` (tools from an MCP Server connection, by `name` + `connection` — push discovers + assigns; see references/agents/native-tools.md). Designing *which* params/tools to expose: [`references/agents/tool-principles.md`](references/agents/tool-principles.md)
- **Names are foreign keys:** cross-entity references resolve by display name at push/runtime — agent `connection:`, delegation `target:` (agent name or UI-managed team name), agent `resources[].name`, eval `agent:`, custom tool `connection:`. A dangling name fails the push or the runtime call — when renaming anything, update its referrers in the same edit
- **Renaming:** change the `name` field — the stable `id` ensures it's detected as a rename, not delete + create. For agents, `wayai push` auto-renames the `.yaml` and `.md` files
- **Default omission:** fields matching defaults are omitted (e.g., `enabled: true`, kanban flags default `false`, `excludeHolidays` defaults `true`)
- **Entity matching (sync/diff):** `id` first (stable UUID), then fallback. Exceptions: states match by `name` (unique per hub regardless of scope); evals match by `name + path`; native tools by `tool_name` per agent

## Slugification & Entity Matching

Names → URL-safe slugs for filenames:
1. Lowercase
2. Normalize accents (NFD + strip diacritics)
3. Replace non-alphanumeric with `-`
4. Collapse consecutive `-`
5. Trim leading/trailing `-`
6. Limit to 50 chars

Examples: `Mario's Pizza` → `marios-pizza`; `Suporte Nível 2` → `suporte-nivel-2`; `Specialist - Billing` → `specialist-billing`.

## Editing Agent Instructions

- File: `agents/<slugified-agent-name>.md` alongside the corresponding `.yaml`
- Always save under `wayai-ws/hubs/<hub>/agents/`, never `/tmp` or other paths
- Always `wayai pull` before editing to avoid clobbering out-of-band changes
- Instructions support dynamic placeholders: `{{now()}}`, `{{user_name()}}`, `{{state()}}`, etc. — see [`references/agents/instructions.md`](references/agents/instructions.md)
- **Structure for reliability:** order each `.md` **procedure → guardrails → voice**; say what to do (not what to avoid); make every action a named tool call. Structure the prompt so the right action is the first and easiest thing the model can do. Full principles: [`references/agents/prompt-principles.md`](references/agents/prompt-principles.md)

## Reference Documentation

One reference per domain, following the hub navigation order. Concepts live in this file; open the domain's reference when you're about to author its YAML, need field-level schemas or per-provider specifics, or are debugging behavior in that domain.

| Domain | Reference | When to read |
|--------|-----------|--------------|
| **Connections** | [`references/connections.md`](references/connections.md) | Wiring up a channel, agent provider, tool API, or speech connector; credential binding, org tags, production credentials |
| **Agents** | [`references/agents/roles-and-settings.md`](references/agents/roles-and-settings.md) | Choosing an agent role, delegation flow, connector-specific settings, evaluation variables, response format |
| **Agents** | [`references/agents/native-tools.md`](references/agents/native-tools.md) | Native tool catalog + parameters, meta tools (`get_tool_schema`, `execute_tool`), MCP tool assignment |
| **Agents** | [`references/agents/custom-tools.md`](references/agents/custom-tools.md) | Custom HTTP tool format, OpenAI function schema, `composed_tools` side effects |
| **Agents** | [`references/agents/tool-principles.md`](references/agents/tool-principles.md) | Designing a tool surface a fallible agent calls reliably — surface curation > validation > prompt, one-tool-one-intent, fail-loud, guarded atomic ops |
| **Agents** | [`references/agents/instructions.md`](references/agents/instructions.md) | Placeholder syntax (`{{now()}}`, `{{state()}}`, etc.) for `agents/<slug>.md` |
| **Agents** | [`references/agents/prompt-principles.md`](references/agents/prompt-principles.md) | Placing & structuring context for reliable execution — context-to-slot-lifetime, cache hygiene, state vs tool-history, flow-before-style, positive framing, voice as its own section |
| **Kanban** | [`references/kanban.md`](references/kanban.md) | Kanban field specs: flags, transitions, followups, additional-context schema/instructions, lanes, constraint matrix, warnings |
| **States** | [`references/states.md`](references/states.md) | State JSON Schemas, scope, agent read/write, initial values |
| **Resources** | [`references/resources.md`](references/resources.md) | Knowledge bases, skill resources, agent linkage, provider sync (`wayai sync-skills`) |
| **Evals** | [`references/evals.md`](references/evals.md) | Eval scenario YAML, scenario sets, journeys-as-code, seed fixtures + variables, `wayai eval capture` / `wayai eval journey capture`, run pacing, authoring & interpreting principles |
| **Outbound** | [`references/outbound.md`](references/outbound.md) | Outbound contacts, lists, schedules, channel rules, execution modes |
| **Analytics** | [`references/analytics.md`](references/analytics.md) | Variable categories/types, filter operators, time analysis, query workflows |
| **Canonical example** | [`references/canonical-example/README.md`](references/canonical-example/README.md) | End-to-end hub showing how `hub.yaml` + `agents/*` + `resources/` + `evals/` + `journeys/` cross-reference. Read once before generating a new hub from scratch |
| **Navigation** | [`references/navigation.md`](references/navigation.md) | App URL surface (`/chat`, `/task`, `/support`, `/settings/...`), hub-detail tabs, query-string deep links — any time you hand the user a URL |
| **Progress report** | [`references/progress-report.md`](references/progress-report.md) | Creating or refreshing `wayai-ws/hubs/<hub>/progress.html` — the shareable build-progress & readiness snapshot — and the AGENTS.md `## Build plan` convention. Code-harness agents only |
| **AGENTS.md files** | [`references/agents-md-template.md`](references/agents-md-template.md) | Canonical repo-root `AGENTS.md` bootstrap (reconcile a stale copy against it at session start) + the per-hub / Rekor-base memory pattern |

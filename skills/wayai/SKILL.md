---
name: wayai
version: 6.22.2
description: |
  Configure WayAI hubs, agents, tools, resources, states, evals, outbound, and analytics.
  Use when: creating or editing a hub or hub config; adding/configuring agents, tools, channels,
  connections, kanban, states, resources, eval scenarios, outbound campaigns; running analytics
  or evals; reviewing or editing workspace YAML (hub.yaml, agents/*.yaml) or agent instruction
  Markdown; installing a ready-made hub template (`wayai template list`/`pull`); using the wayai CLI
  (push, pull, send-message, conversations, sync-skills, create-credential, analytics, run-eval,
  eval capture, template, init); or interpreting WayAI platform
  terminology (pilot/copilot, preview/production, kanban statuses, AI modes, agent roles).
---

# WayAI Skill

WayAI is a SaaS platform for AI-powered communication hubs. Each hub combines AI agents and a human team across channels (WhatsApp, Email, Instagram, native App). This workspace stores **one hub** as code: `hub.yaml` + `agents/*.{yaml,md}` synced bidirectionally to the platform via the `wayai` CLI.

**Platform is the source of truth.** Workspace files are the edit surface — changes flow through files → `wayai push` → platform. Always `wayai pull` before editing to catch out-of-band changes.

## Agent Guidelines

- **Interface:** if you have filesystem/shell access (code-harness agents — Claude Code, Codex, Cursor, OpenCode), drive WayAI through the **`wayai` CLI and workspace files**, and do not call any `mcp__wayai__*` tools that may also be in your toolset. If you do **not** have filesystem/shell access (app-harness agents — Claude Desktop, etc.), use the `mcp__wayai__*` tools — they are your interaction surface for everything below
- Only provide information from this skill, tool descriptions, or reference documentation
- Do not invent URLs, paths, or steps
- Hub config flows through files + the `wayai` CLI; one-time setup (orgs, OAuth, publish) goes through the platform UI
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
| Skills sync to providers | CLI (`wayai sync-skills`) |
| Conversation testing | CLI (`wayai send-message`, `wayai conversations`, `wayai delete-history`) |
| Inspect what an agent actually received (resolved prompt, rendered context, injected timestamps, tool calls) | CLI (`wayai conversations <id> observability [--message-id <id>]`) |
| Record a post-hoc business outcome on an ended conversation (e.g. customer purchased) as an analytics dimension | CLI (`wayai conversations <id> annotate --set key=value [--type ...]`) |
| Analytics | CLI (`wayai analytics`, `wayai analytics query`) |
| Eval runs and results | CLI (`wayai run-eval`, `wayai eval-results`) |
| Capture production conversation as eval | CLI (`wayai eval capture <conversation_id>`) |
| Capture production conversation as a journey (full multi-turn transcript) | CLI (`wayai eval journey capture <conversation_id>`) |
| Delete eval session(s) / run history | CLI (`wayai eval session delete <session_id>`, or `--all` for every session on the hub) |
| Org credentials | CLI (`wayai create-credential`) or UI |
| Bug reporting | CLI (`wayai report create`) |
| Workspace discovery | CLI (`wayai list`) |
| Organization — create | CLI (`wayai org create`) or UI |
| Organization — update, delete | UI |
| Publish/sync to production, delete hubs, replicate previews | UI |
| User management | UI |

## Entity Hierarchy

```
Organization              ← CLI (`wayai org create`) or UI
├── Org Credentials       ← CLI or UI (store API keys once, reuse across hubs)
└── Project               ← CLI or UI
    └── Hub               ← CLI (auto-creates on push) or UI; publish/sync via UI
        ├── Connections   ← Auto-created from org credentials (non-OAuth); OAuth via UI
        └── Agents        ← CLI
            ├── Tools     ← CLI
            └── Resources ← CLI (linked from hub.yaml)
```

Setup order: Organization (CLI `wayai org create` or UI) → Org Credentials (CLI or UI) → Project (CLI or UI) → Hub (CLI push or UI) → configure agents, tools, connections via CLI.

The `wayai` connection (native tools) is auto-created when a hub is created — no setup needed.

## Hub Types

| Type | Conversations | Channels | Use Case |
|------|--------------|----------|----------|
| `chat` | ONE per end user | WhatsApp, Instagram, Email, App | Person-centered: support, sales, helpdesk |
| `task` | MULTIPLE per user | App only | Task-centered: invoices, inventory, approvals |

Decision: WhatsApp/Instagram/Email needed → `chat`. Object/task processing → `task`.

## AI Modes

| Mode | Behavior |
|------|----------|
| `pilot` | AI handles end users autonomously |
| `copilot` | AI suggests responses to the support team (no channel delivery) |
| `pilot+copilot` | Switches dynamically based on `current_responder_type` |
| `turned_off` | AI disabled; humans only |

## Agent Roles

| Role | Track | Per Hub | Description |
|------|-------|---------|-------------|
| `pilot` | Pilot | 1 | Responds to end users autonomously |
| `copilot` | Copilot | 1 | Suggests responses to the support team |
| `pilot_specialist` / `copilot_specialist` | Both | Multiple | Delegation target — full transfer via `transfer_to_agent` |
| `pilot_advisor` / `copilot_advisor` | Both | 1 each | Advisory input via `consult_agent`; returns control |
| `monitor` | Background | 1 | Observes silently |
| `conversation_evaluator` / `message_evaluator` | Background | 1 each | Async quality assessment; excluded from normal routing |
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

## Connection Types

| Category | Examples |
|----------|----------|
| **Agent** | OpenAI, Anthropic, Google AI Studio, OpenRouter (required for AI) |
| **Channel** | WhatsApp, Instagram (OAuth — UI only); Resend, Telegram (API Key — auto-created) |
| **Tool — Native** | Wayai (auto-created), Google Calendar (OAuth), External Resources (API Key) |
| **Tool — Custom** | User-defined HTTP endpoints (API Key, Bearer Token, Basic Auth) |
| **Tool — MCP** | External MCP servers (Streamable HTTP) — Bearer Token via CLI; OAuth via UI |
| **Speech** | Groq STT, OpenAI TTS, ElevenLabs |

**Auto-creation rule:** Non-OAuth connections (Agent, STT, TTS, Tool — Custom, Tool — MCP via Bearer Token) are auto-created from matching organization credentials when `hub.yaml` is pushed. OAuth connections must be set up in the UI first.

**OAuth connection handoff (any time — not just onboarding):** OAuth connections (WhatsApp, Instagram, Google Calendar, **MCP OAuth**) can't be created from the CLI — they need a one-time UI flow. **Whenever** one is needed — first-time setup *or* later (a new channel, an OAuth MCP server) — hand the user the full-path connections-tab deeplink `https://app.wayai.pro/settings/organizations/<orgId>/hubs/<hubId>/connections?connector=<slug>` (`<orgId>`/`<hubId>` from `wayai status --json`; `<slug>` ∈ `whatsapp`, `instagram`, `google-calendar`, `mcp-server`), then `wayai pull -y` once they're done. The deeplink opens the **Connections** tab (and highlights the connector if a connection already exists — e.g. re-auth); to create one the user clicks **Add Connection**, picks the **\<Connector\>** card, chooses **OAuth**, and finishes the provider flow. Use this tab form — **not** `/connections/new?connector=…`, which takes a `connector_id` UUID and defaults to the first auth type (MCP → Bearer Token), so it can't reach MCP OAuth (see [navigation.md](references/navigation.md)).

For per-provider setup, see [`references/connections.md`](references/connections.md).

## Tool Types

| Type | Source | How |
|------|--------|-----|
| Native | Platform built-ins (e.g., `send_text_message`, `update_kanban_status`, `transfer_to_human`) | Listed by name in `agents/<slug>.yaml` |
| Custom | HTTP endpoints you define | Defined in `agents/<slug>.yaml` with `connection`, `method`, `path`, `config` |
| MCP | Tools from connected MCP servers | Dual-origin — declared per-agent under `tools.mcp` in `agents/<slug>.yaml` **and/or** assigned in the Platform UI. `wayai push` discovers + assigns in one run; a present `mcp` key (even `[]`) is authoritative, an omitted one preserves UI-assigned tools. See [native-tools.md](references/agents/native-tools.md#mcp-tools) |
| Delegation | Agent-to-agent (`transfer_to_agent`, `consult_agent`) or agent-to-team (`transfer_to_team`) | Declared with `target` in `agents/<slug>.yaml` |

Meta tools (`get_tool_schema`, `execute_tool`) let agents call tools whose schemas are excluded from the inline list. See [`references/agents/native-tools.md`](references/agents/native-tools.md).

## Hub Environments

| Environment | Description |
|-------------|-------------|
| `preview` | Default. Editable workspace for configuring and testing |
| `production` | Read-only. Serves live traffic. Changes flow from preview via UI sync |

**Lifecycle (UI-managed):**
1. New hubs start as `preview` — edit freely
2. **Publish** (UI) — first promotion creates a `production` hub cloned from preview
3. **Sync** (UI) — pushes subsequent preview changes to the linked production
4. **Replicate Preview** (UI) — creates a new preview from production for experimentation

Production is read-only — all config mutations flow through preview. Multiple previews can link to the same production (many-to-1). Channel uniqueness is enforced on production only — previews can share phone/email/SID with their production.

WhatsApp/Instagram/Telegram channels can be exercised on a preview before publishing — register a tester via a `#test CODE` claim code (see `references/connections.md` → Channel → "Testing a channel on a preview before publishing").

Only preview hubs are tracked in this repository.

## Kanban & States

**Kanban statuses** are workflow stages for conversations (visible in support/task views), defined per hub in `hub.yaml`. Each status has a stable `slug` (immutable identifier), a `name` (display label), behavioral flags, and optional time-based followups.

**Identity model:**
- `slug` — immutable lowercase identifier (`^[a-z][a-z0-9_]{0,49}$`). Stored in `conversation.kanban_status`, ClickHouse `data.meta.kanban_status`, native tool params, and transition references. **Cannot be renamed** after the status is saved.
- `name` — display label shown in the UI and tool descriptions. Freely editable.

Behavioral flags: `isInitialStatus`, `triggersAgentResponse`, `allowsAgentUpdate`, `isTerminalStatus`, `isSchedulingStatus` (requires `eventName`).

**Allowed transitions** (`allowed_next_statuses`) — optional list of slugs this status can transition to. Omit (= `undefined`) for unrestricted (any next status). Empty array `[]` is rejected; use `isTerminalStatus: true` for "no outbound."

Followup types:
- `inactivity` — sent after a period of silence
- `before_event` — sent before a scheduled event (requires parent status `isSchedulingStatus: true`)

Both support `threshold` + `timeUnit` (seconds/minutes/hours/days), optional quiet hours (`excludedTimeStart`/`excludedTimeEnd`), and `excludeHolidays` (default `true`).

**Additional context on transition** (only meaningful when `triggersAgentResponse: true`):
- `additional_context_schema` — JSON Schema (Draft 2020-12) defining the form rendered when a team member transitions a conversation into this status. Submitted values are validated against the schema server-side; any violation fails the request with a path-anchored error. Capped at 16 KB JSON-serialized. Top-level property name `additional_data` is reserved (collides with the dump placeholder below).
- `additional_instructions` — prose template injected into the agent turn on transition. Supports two placeholder forms:
  - `{{path.to.field}}` — dotted access, whitespace-tolerant. Strings/numbers/booleans render verbatim; arrays of primitives render comma-joined; nested objects/arrays of objects render as 2-space-indented JSON.
  - `{{additional_data}}` — full payload dumped as 2-space-indented JSON. Useful when the schema has lots of fields and you want them all in one block without per-field placeholders.
- The agent receives a single `<system_additional_instructions>` block containing the interpolated prose. The tag is emitted only when there's content to send (no empty tag noise); the agent turn fires regardless via `triggersAgentResponse`.

**Constraints** (enforced server-side on every write — REST, CLI `wayai push`, MCP):
- Exactly one status must have `isInitialStatus: true`
- Slugs must be unique within a hub and match `^[a-z][a-z0-9_]{0,49}$`
- Every entry in `allowed_next_statuses` must reference a sibling slug; `[]` is rejected
- Mutually exclusive flags (cannot both be `true` on the same status):
  - `isInitialStatus` ↔ `triggersAgentResponse` / `allowsAgentUpdate` / `isTerminalStatus` / `isSchedulingStatus`
  - `triggersAgentResponse` ↔ `allowsAgentUpdate` / `isTerminalStatus`
- `isSchedulingStatus: true` requires non-empty `eventName`
- `before_event` followups require the parent status to have `isSchedulingStatus: true`
- Followups with `delivery_mode: direct` require non-empty `direct_text`
- `additional_context_schema` must be a well-formed JSON Schema; the schema's `properties` must not declare a top-level `additional_data` key (reserved placeholder name); JSON-serialized size ≤ 16 KB

**Non-blocking warnings** (returned alongside successful saves):
- `unreachable` — a non-initial status that no other status lists in its `allowed_next_statuses`.
- `placeholder_unresolved` — a `{{path.to.field}}` placeholder in `additional_instructions` does not resolve against `additional_context_schema`. At runtime that placeholder renders as empty string.
- `unused_lane` — a declared lane no status is assigned to.

**Lanes** (optional, presentational grouping for the kanban board filter). A hub may declare a `lanes` array (`{ slug, name, color?, order? }`); each non-initial status may set `lane_slug` to one of them. Lanes are organizational only — they do **not** change transitions (those stay in `allowed_next_statuses`) and are not visible to agents. Constraints (server-side, all write paths): lane slugs unique + `^[a-z][a-z0-9_]{0,49}$`; a status's `lane_slug` must reference a declared lane; the initial status must **not** have a `lane_slug` (it's the shared entry point). The board filters by lane; the initial status shows in every lane.

**Runtime transition gate:** When a conversation transitions kanban status (drag-drop, native tool, REST, MCP), the configured `allowed_next_statuses` is enforced. Disallowed transitions return `invalid_kanban_transition` with the allowed targets. `undefined` `allowed_next_statuses` keeps the legacy "any → any" behavior.

**States** are JSON-schema data the agent reads/writes during conversations. Each state has either `conversation` or `user` scope, a `json_schema` (shape), and an optional `initial_value` (opt-in pre-populated virtual record rendered in both the `{{state()}}` placeholder and the `<state>` tag until the first real write). Omit `initial_value` to keep state silent until something is actually written.

**Kanban vs State:** kanban tracks workflow progression; state tracks structured data. Both coexist.

For deeper schemas, see [`references/states.md`](references/states.md).

## Hub Settings

| Setting | Values | Default | Description |
|---------|--------|---------|-------------|
| `non_app_permission` | `everyone`, `require_permission`, `not_allowed` | `everyone` | Who can reach hub via external channels |
| `timezone` | IANA timezone | — | For scheduling and display |
| `language` | `en`, `pt`, `es` | `en` | Language for hub-sent text (e.g. the pending-access notice on `require_permission` channels) |
| `access_approval_role` | `admin`, `team` | `admin` | Who may approve/block a pending contact: hub admins only, or also support team members |
| `access_request_message` | string | — | Optional override for the "your access is pending approval" auto-reply (else a localized default by `language`) |

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
| 1b | No harness skill install present at project root (none of `<root>/.claude/skills/wayai/SKILL.md`, `<root>/.opencode/skills/wayai/SKILL.md`, `<root>/.agents/skills/wayai/SKILL.md` exists, where `<root>` is `git rev-parse --show-toplevel` or cwd if not in a git repo) | Agent runs `npx skills add wayai-pro/wayai-skill -y` from `<root>`. After it completes, agent re-checks the three paths: if at least one exists, exit (the harness will load the installed skill on the next turn). If none exist, surface the install error to the user and halt — do not silently exit (would loop on re-entry). |
| 1c | `skill.installed: true`, `skill.latest` is set, and `skill.latest` is newer than `skill.version` (CLI nightly check populates `skill.latest`) | Agent runs `npx skills add wayai-pro/wayai-skill -y` from `<root>` to refresh the install in place, then exits (the updated skill loads on the next turn). On install failure, surface the error to the user and continue with the existing skill. |
| 2 | `auth.logged_in: false` | Agent runs `wayai login` (opens browser). User handoff: "Open the page that just opened. Sign in or sign up. Tell me when done." |
| 3 | `auth.logged_in: true`, `orgs: []` | Ask once: "What should we name your organization? (usually your company name)". Then agent runs `wayai org create "<name>"` and re-runs `status --json` to pick up the new org. (Manual fallback only if the CLI create errors: open `https://app.wayai.pro/settings/organizations/new`.) |
| 3b | `git rev-parse --show-toplevel` fails (cwd is not in a git repo) | The CLI requires git for workspace detection. **Before acting, surface the resolved cwd and confirm with the user** — handoff: "I'll initialize a git repo at `<cwd>`. Confirm or pick a different folder." This prevents accidentally initializing a repo in `~` or another unintended directory. After confirmation, agent runs `git init` in cwd. Then checks `git config --global user.name` and `git config --global user.email`; if either is empty, asks the user once for their name and email and runs `git config --global user.name "<name>"` / `git config --global user.email "<email>"`. A GitHub remote is not required for the onboarding flow — only set one up later if the user wants the GitOps/CI loop. |
| 4 | `workspace.scoped: false` | Agent runs `wayai init --org <active_org.id>`. |
| 5 | Workspace scoped, hub goal not yet known | User handoff: "What should this hub do? Describe the goal, who talks to it, and the main use case." |
| 6 | LLM credential missing for chosen provider | User handoff: "Paste your OpenAI/Anthropic/Google API key here." Then agent runs `wayai create-credential --name <name> --type "Bearer Token" --stdin`. |
| 7 | Hub needs an OAuth connection (WhatsApp / Instagram / Google Calendar / MCP OAuth) | Apply the **OAuth connection handoff** (Connection Types → OAuth connection handoff): send the full-path connections deeplink for the connector, wait for completion, then `wayai pull -y`. The same handoff applies any time an OAuth connection is needed later, not only here. |
| 8 | Prerequisites met | Read [`references/canonical-example/README.md`](references/canonical-example/README.md) once for end-to-end wiring, then generate `wayai-ws/hubs/<hub>/hub.yaml` + `agents/*.yaml` + `agents/*.md` from the user's description (per-domain refs below for individual shapes), then `wayai push -y`. |
| 9 | Push succeeded | Agent runs `wayai send-message "Hi"` and shows the response. User handoff: "Refine, add tools, or publish?" |
| 10 | User confirms publish | User handoff: "Open `https://app.wayai.pro/settings/organizations/<org_id>/hubs/<hub_id>/overview?action=publish` and click Publish." |

### Deeplinks (canonical URLs — never breadcrumbs)

| State | URL |
|-------|-----|
| 3 (org create — manual fallback) | `https://app.wayai.pro/settings/organizations/new` |
| 6 (credential pre-fill) | `https://app.wayai.pro/settings/organizations/<org_id>/credentials?type=bearer&name=<key-name>&prefill=true` |
| 7 (OAuth connection) | `https://app.wayai.pro/settings/organizations/<org_id>/hubs/<hub_id>/connections?connector=<whatsapp\|instagram\|google-calendar\|mcp-server>` |
| 10 (publish) | `https://app.wayai.pro/settings/organizations/<org_id>/hubs/<hub_id>/overview?action=publish` |

### Rules

- Detect, don't assume — every step starts with a fresh `wayai status --json`.
- Agent actions get a one-line receipt; user handoffs get one URL, one action, one return signal.
- Never invent URLs outside the table above. Never instruct "go to Settings → …" — always a deeplink. For non-onboarding URLs see [`references/navigation.md`](references/navigation.md).
- One question at a time during hub scoping (state 5): goal, then channel, then LLM provider.
- Never auto-commit anything created during onboarding — show `git diff` and wait for user approval before any commit.

## Workflow

### Existing hub
1. **Update CLI** — `wayai update` (always run before any operation)
2. **Update skill if stale** — `wayai status --json` and apply state machine row 1c: if `skill.latest` is set and newer than `skill.version`, run `npx skills add wayai-pro/wayai-skill -y` and exit (the refreshed skill loads on the next turn). Otherwise continue.
3. **Pull** — `wayai pull -y` (sync local files from platform; catches out-of-band changes)
4. **Read context** — `wayai-ws/hubs/<hub>/AGENTS.md` for hub-specific notes (purpose, decisions, ongoing work). AGENTS.md-aware harnesses (Codex, Cursor, OpenCode, Aider) auto-load it natively
5. **Edit** — modify `hub.yaml`, `agents/*.yaml`, `agents/*.md`
6. **Push** — `wayai push -y` (apply to preview hub; auto-pulls server-assigned IDs back)
7. **Test** — `wayai send-message "Hello"`
8. **Review** — run `git diff`, ask user to confirm. **Never auto-commit.** User commits and pushes to `main`
9. **Go live** — sync preview → production via the platform UI when ready

### New hub (from scratch)
1. **Credentials** — `wayai create-credential --name "openai-key" --type "Bearer Token"` (one-time per org per credential)
2. **Init** — `wayai init` (interactive) or `wayai init --org <uuid>`
3. **Create files** — `wayai-ws/hubs/<hub>/hub.yaml` + `agents/*.yaml` + `agents/*.md`
4. **Push** — `wayai push -y` auto-creates the hub, non-OAuth connections, and applies all config
5. **Test** — `wayai send-message "Hello"`

After the hub exists, follow the existing-hub workflow.

### Hub-Folder Memory

`wayai-ws/hubs/<hub>/AGENTS.md` is the **hub-specific memory** for this hub — purpose, key decisions, ongoing work, business rules that only apply here, terminology, integration quirks. Read it at the start of every hub-related task. AGENTS.md-aware harnesses (Codex, Cursor, OpenCode, Aider) auto-load it natively when the agent's cwd is inside the hub folder.

**Maintain it actively:**
- After significant changes (new agent, new tool, business rule update), update `AGENTS.md` so future sessions inherit the context
- If `AGENTS.md` is missing for a hub, create it with what you know — purpose, current agents, recent decisions — and ask the user to confirm or enrich
- Keep it focused on *why* decisions were made and *what* makes this hub different. Don't restate platform mechanics — those live in this skill

**Overflow content goes into `wayai-ws/hubs/<hub>/references/`:**
- When `AGENTS.md` grows past ~200 lines or starts mixing topics, extract the deeper material into focused files under `wayai-ws/hubs/<hub>/references/`
- Examples: `references/business-rules.md`, `references/integrations.md`, `references/glossary.md`, `references/<api-name>-spec.md`, `references/<persona>-tone.md`
- Keep `AGENTS.md` as the always-on entry point with a short pointer to each reference file: e.g., "Detailed pricing rules: `references/pricing-rules.md`"
- Hub-folder `references/` are **not synced** to the platform — they're for agent context only, just like `AGENTS.md`

## Common CLI Commands

```bash
wayai update            # Update CLI (run before any operation)
wayai login             # OAuth — or `wayai login --token` for headless/CI
wayai org create        # Create a new organization (you become its owner): `wayai org create "<name>"` [--region <r>] [--json]
wayai create-credential # Create org credential (--name, --type "API Key"|"Bearer Token"|"Basic Auth", --org, --stdin)
wayai init              # Set up .wayai.yaml (interactive — creates an org inline if you have none); --org <uuid> to skip prompt
wayai pull              # Pull hub config from platform (-y skips confirmation; auto-binds worktree on first pull)
wayai push              # Push local changes (-y skips confirmation; auto-pulls IDs back)
wayai use <hub>         # Bind this worktree to a specific hub (UUID or folder name)
wayai unbind            # Clear the worktree hub binding
wayai template list     # List ready-made hub templates (gym, clinic, …) — browse what's available
wayai template pull <slug> [--lang <locale>] [--force] [--dry-run]  # Write a template's config into wayai-ws/hubs/<slug>/, then `wayai push`. --lang picks a localized variant (en|pt|es); omitted = the template's default, an untranslated language falls back with a note. Refuses to overwrite existing files in the target dir (lists them) — pass --force to overwrite; --dry-run previews the file map without writing
wayai send-message      # Test message to a preview hub
wayai conversations     # List or inspect conversations (default text view omits message_id — use --json or `observability` to discover ids)
                        # `wayai conversations <id> observability` — list LLM turns with message_id, latency, tool_calls (assistant turns only)
                        # `wayai conversations <id> observability --message-id <id>` — full record for one turn (prompt, completion, tool calls, tokens; --json for raw)
                        # `wayai conversations <id> annotate --set key=value [--type numeric|categorical|text]` — set a post-hoc business outcome (e.g. customer_purchased=true) on an ended conversation as an analytics dimension; repeat --set for multiple keys (needs the hub within its conversation_retention_days window)
wayai delete-history    # Clear conversation history (testing); --conversation-id <id> deletes just one
wayai sync-skills       # Sync skills to provider connections; --connection-id <uuid> to scope
wayai sync-mcp          # Re-discover an MCP connection's tools (refresh stale schemas); --connection <name|uuid>
wayai analytics         # Summary + per-variable aggregates; --metric, --filter, --period, --json
wayai analytics query   # Structured ClickHouse query (multi-variable, group_by, correlations)
wayai run-eval          # Run a scenario set's enabled evals (sole set by default; --set/--eval to pick on multi-set hubs)
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
AGENTS.md                                # Pointer to this skill (init-only — yours to edit)
AGENTS.local.md                          # Project-specific overrides (init-only — yours to edit)
CLAUDE.md                                # Claude Code shim — `@AGENTS.md @AGENTS.local.md` (init-only)
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
        ├── AGENTS.md                    # Hub-level agent context (NOT synced; yours to edit)
        ├── CLAUDE.md                    # Per-hub Claude Code shim — `@AGENTS.md` (init-only, NOT synced)
        └── references/                  # Hub-specific supporting files (NOT synced)
```

Preview hub folders use `hub-slug--<preview_label>` or `hub-slug--<hub_id_prefix>` for disambiguation. **Existing repos** that still use the legacy `workspace/` + root `org/` layout keep working (the CLI reads either) — run `wayai migrate` to move to `wayai-ws/`.

## `hub.yaml` Shape

```yaml
version: 1
hub_id: "abc-123-def"           # set by `wayai pull` — do not edit
hub_environment: preview         # set by `wayai pull` — do not edit

hub:
  name: Customer Support
  hub_type: chat                 # chat | task
  ai_mode: pilot+copilot         # pilot | copilot | pilot+copilot | turned_off
  timezone: America/New_York
  non_app_permission: everyone
  kanban_statuses:
    - slug: new
      name: New
      order: 0
      color: "#22c55e"
      isInitialStatus: true
      allowed_next_statuses: [qualified, waiting_for_customer, resolved]
    - slug: qualified
      name: Qualified
      order: 1
      color: "#3b82f6"
      triggersAgentResponse: true
      allowed_next_statuses: [waiting_for_customer, resolved]
      additional_context_schema:           # JSON Schema (Draft 2020-12)
        type: object
        properties:
          contact:
            type: object
            properties:
              display_name: { type: string }
              relationship: { type: string }
            required: [display_name]
          students:
            type: array
            items:
              type: object
              properties:
                student_name: { type: string }
                modality: { type: string }
              required: [student_name]
        required: [contact, students]
      additional_instructions: |
        A qualified contact just transitioned in. Greet {{contact.display_name}}
        ({{contact.relationship}} of the student) and confirm interest.
        Students:
        {{additional_data}}
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
  - name: my-api-connection
    type: Tool - Custom
    service: User Tool
```

`hub.yaml` also holds `resources:`, `outbound_contacts:`, `outbound_lists:`, `outbound_schedules:` blocks. See the per-domain references for full schemas.

## `agents/<slug>.yaml` Shape

```yaml
id: "agent-uuid-123"               # set by pull
name: Pilot Agent
role: pilot
connection: anthropic              # connection display name
# instructions resolved by convention from agents/<slug>.md
# enabled: true                    # default; omitted
# include_message_timestamps: false  # default; when true, appends [timestamp, weekday, daypart] to user messages
settings:
  model: claude-sonnet-4-6
  temperature: 0.7
  max_tokens: 4096
  # reasoning per provider: Anthropic thinking_enabled · OpenAI/OpenRouter reasoning_effort · Gemini reasoning_level (see roles-and-settings.md)
tools:
  native:
    - send_text_message
    - update_kanban_status
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
```

For full agent options (settings per connector, native tool params, custom tool fields, `composed_tools`, placeholders), see [`references/agents/`](references/agents/).

**Evaluation variables** — `conversation_evaluator` / `message_evaluator` agents carry an `evaluation_variables` list (the structured fields they emit per conversation/message), round-tripped via pull/push. See [`references/agents/roles-and-settings.md`](references/agents/roles-and-settings.md#evaluation-variables).

## Key Rules

- **Read-only fields:** `hub_id`, `hub_environment`, `id` — set by `wayai pull`, never edit
- **Connection auto-creation:** non-OAuth connections in `hub.yaml` resolve to org credentials by matching `service` + `authentication_type`. Use `credential` field to disambiguate when multiple org credentials share the same auth type. OAuth connections (WhatsApp, Instagram, Google Calendar, MCP OAuth) must already exist (UI setup — see OAuth connection handoff) — referenced by name only
- **Tool groups:** `native` (platform built-ins by name), `delegation` (agent-to-agent/team handoff), `custom` (HTTP endpoints with connection), `mcp` (tools from a `Tool - MCP` connection, by `name` + `connection` — push discovers + assigns; see references/agents/native-tools.md). Designing *which* params/tools to expose: [`references/agents/tool-principles.md`](references/agents/tool-principles.md)
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

References mirror the hub navigation. Open the relevant file when working on that domain.

| Domain | Reference | When to read |
|--------|-----------|--------------|
| **Connections** | [`references/connections.md`](references/connections.md) | Wiring up a channel, agent provider, tool API, or speech connector |
| **Agents** | [`references/agents/roles-and-settings.md`](references/agents/roles-and-settings.md) | Choosing an agent role, delegation flow, connector-specific settings |
| **Agents** | [`references/agents/native-tools.md`](references/agents/native-tools.md) | Native tool parameters per connector, meta tools (`get_tool_schema`, `execute_tool`) |
| **Agents** | [`references/agents/custom-tools.md`](references/agents/custom-tools.md) | Custom HTTP tool format, OpenAI function schema, `composed_tools` side effects |
| **Agents** | [`references/agents/tool-principles.md`](references/agents/tool-principles.md) | Designing a tool surface a fallible agent calls reliably — surface curation > validation > prompt, one-tool-one-intent, fail-loud, guarded atomic ops |
| **Agents** | [`references/agents/instructions.md`](references/agents/instructions.md) | Placeholder syntax (`{{now()}}`, `{{state()}}`, etc.) for `agents/<slug>.md` |
| **Agents** | [`references/agents/prompt-principles.md`](references/agents/prompt-principles.md) | Placing & structuring context for reliable execution — context-to-slot-lifetime, cache hygiene, state vs tool-history, flow-before-style, positive framing, voice as its own section |
| **Resources** | [`references/resources.md`](references/resources.md) | Knowledge bases, skill resources, agent linkage, provider sync (`wayai sync-skills`) |
| **States** | [`references/states.md`](references/states.md) | State JSON Schemas, scope, agent read/write, initial values |
| **Outbound** | [`references/outbound.md`](references/outbound.md) | Outbound contacts, lists, schedules, channel rules, execution modes |
| **Evals** | [`references/evals.md`](references/evals.md) | Eval scenario YAML, scenario sets, journeys-as-code (`journeys/<slug>.yaml`, `wayai pull`/`push`), `wayai eval capture` / `wayai eval journey capture` from production |
| **Analytics** | [`references/analytics.md`](references/analytics.md) | `wayai analytics` and `wayai analytics query` flags, metric paths, filters |
| **Canonical example** | [`references/canonical-example/README.md`](references/canonical-example/README.md) | End-to-end hub showing how `hub.yaml` + `agents/*` + `resources/` + `evals/` + `journeys/` cross-reference. Read once before generating a new hub from scratch |
| **Navigation** | [`references/navigation.md`](references/navigation.md) | App URL surface (`/chat`, `/task`, `/support`, `/settings/...`, `/user/...`), hub-detail tabs, query-string deep links |

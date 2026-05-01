---
name: wayai
version: 6.0.0
description: |
  Configure WayAI hubs, agents, tools, resources, states, evals, outbound, and analytics.
  Use when: creating or editing a hub or hub config; adding/configuring agents, tools, channels,
  connections, kanban, states, resources, eval scenarios, outbound campaigns; running analytics
  or evals; reviewing or editing workspace YAML (hub.yaml, agents/*.yaml) or agent instruction
  Markdown; using the wayai CLI (push, pull, send-message, conversations, sync-skills,
  create-credential, analytics, run-eval, eval capture, init); or interpreting WayAI platform
  terminology (pilot/copilot, preview/production, kanban statuses, AI modes, agent roles).
---

# WayAI Skill

WayAI is a SaaS platform for AI-powered communication hubs. Each hub combines AI agents and a human team across channels (WhatsApp, Email, Instagram, native App). This workspace stores **one hub** as code: `hub.yaml` + `agents/*.{yaml,md}` synced bidirectionally to the platform via the `wayai` CLI.

**Platform is the source of truth.** Workspace files are the edit surface — changes flow through files → `wayai push` → platform. Always `wayai pull` before editing to catch out-of-band changes.

## Agent Guidelines

- Only provide information from this skill, tool descriptions, or reference documentation
- Do not invent URLs, paths, or steps
- Hub config flows through files + the `wayai` CLI; one-time setup (orgs, OAuth, publish) goes through the platform UI
- Always `wayai pull -y` before editing — catches out-of-band changes
- Always `wayai push -y` immediately after editing — editing and pushing are a single action
- Never auto-commit — show `git diff`, wait for user approval

## Quick Decision: What Can I Do?

| Entity | How |
|--------|-----|
| Hub settings, agents, agent instructions, tools, kanban, states, resources, evals, outbound, custom tools | CLI (`wayai push`) |
| Connections — non-OAuth (Agent providers, STT/TTS, Tool API key, MCP API key) | CLI (auto-created from org credentials) |
| Connections — OAuth (WhatsApp, Instagram, Google Calendar, MCP OAuth) | Platform UI |
| Skills sync to providers | CLI (`wayai sync-skills`) |
| Conversation testing | CLI (`wayai send-message`, `wayai conversations`, `wayai delete-history`) |
| Analytics | CLI (`wayai analytics`, `wayai analytics query`) |
| Eval runs and results | CLI (`wayai run-eval`, `wayai eval-results`) |
| Capture production conversation as eval | CLI (`wayai eval capture <conversation_id>`) |
| Org credentials | CLI (`wayai create-credential`) or UI |
| Bug reporting | CLI (`wayai report-bug`) |
| Workspace discovery | CLI (`wayai list`) |
| Organization (signup, update, delete) | UI |
| Publish/sync to production, delete hubs, replicate previews | UI |
| User management | UI |

## Entity Hierarchy

```
Organization              ← UI only (signup)
├── Org Credentials       ← CLI or UI (store API keys once, reuse across hubs)
└── Project               ← CLI or UI
    └── Hub               ← CLI (auto-creates on push) or UI; publish/sync via UI
        ├── Connections   ← Auto-created from org credentials (non-OAuth); OAuth via UI
        └── Agents        ← CLI
            ├── Tools     ← CLI
            └── Resources ← CLI (linked from hub.yaml)
```

Setup order: Organization (signup) → Org Credentials (CLI or UI) → Project (CLI or UI) → Hub (CLI push or UI) → configure agents, tools, connections via CLI.

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

For role flow, delegation, and settings depth, see [`references/agents/roles-and-settings.md`](references/agents/roles-and-settings.md).

## Connection Types

| Category | Examples |
|----------|----------|
| **Agent** | OpenAI, Anthropic, Google AI Studio, OpenRouter (required for AI) |
| **Channel** | WhatsApp, Instagram (OAuth — UI only); Resend, Telegram (API Key — auto-created) |
| **Tool — Native** | Wayai (auto-created), Google Calendar (OAuth), External Resources (API Key) |
| **Tool — Custom** | User-defined HTTP endpoints (API Key, Bearer Token, Basic Auth) |
| **Tool — MCP** | External MCP servers (Streamable HTTP) — API Key via CLI; OAuth via UI |
| **Speech** | Groq STT, OpenAI TTS, ElevenLabs |

**Auto-creation rule:** Non-OAuth connections (Agent, STT, TTS, Tool — Custom, Tool — MCP via API Key) are auto-created from matching organization credentials when `hub.yaml` is pushed. OAuth connections must be set up in the UI first.

For per-provider setup, see [`references/connections.md`](references/connections.md).

## Tool Types

| Type | Source | How |
|------|--------|-----|
| Native | Platform built-ins (e.g., `send_text_message`, `update_kanban_status`, `transfer_to_human`) | Listed by name in `agents/<slug>.yaml` |
| Custom | HTTP endpoints you define | Defined in `agents/<slug>.yaml` with `connection`, `method`, `path`, `config` |
| MCP | Tools from connected MCP servers | Listed by name once the MCP connection is set up |
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

Only preview hubs are tracked in this repository.

## Kanban & States

**Kanban statuses** are workflow stages for conversations (visible in support/task views), defined per hub in `hub.yaml`. Each status supports behavioral flags and optional time-based followups.

Behavioral flags: `isInitialStatus`, `triggersAgentResponse`, `allowsAgentUpdate`, `isTerminalStatus`, `isSchedulingStatus` (requires `eventName`).

Followup types:
- `inactivity` — sent after a period of silence
- `before_event` — sent before a scheduled event

Both support `threshold` + `timeUnit` (seconds/minutes/hours/days), optional quiet hours (`excludedTimeStart`/`excludedTimeEnd`), and `excludeHolidays` (default `true`).

**States** are JSON-schema data the agent reads/writes during conversations. Each state has either `conversation` or `user` scope, a `json_schema` (shape), and `initial_value` (defaults).

**Kanban vs State:** kanban tracks workflow progression; state tracks structured data. Both coexist.

For deeper schemas, see [`references/states.md`](references/states.md).

## Hub Settings

| Setting | Values | Default | Description |
|---------|--------|---------|-------------|
| `app_permission` | `everyone`, `require_permission` | `require_permission` | Who can start conversations via the native app |
| `non_app_permission` | `everyone`, `require_permission`, `not_allowed` | `everyone` | Who can reach hub via external channels |
| `file_handling_mode` | `metadata_only`, `always_attach` | `metadata_only` | How files are sent to AI |
| `hub_sla` | JSON `time_threshold1/2/3` (seconds) | `60`/`300`/`600` | SLA escalation thresholds |
| `max_file_size_for_attachment` | integer (bytes) | — | Optional cap |
| `timezone` | IANA timezone | — | For scheduling and display |

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
| 3 | `auth.logged_in: true`, `orgs: []` | User handoff: "Open `https://app.wayai.pro/login` (or sign up there if you don't have an account yet), then `https://app.wayai.pro/settings/organizations` to create your organization. Tell me when done." Then re-run `status --json`. |
| 4 | `workspace.scoped: false` | Agent runs `wayai init --org <active_org.id>`. |
| 5 | Workspace scoped, hub goal not yet known | User handoff: "What kind of hub? Pick one: pizzeria, dental, swimming-academy, sdr, or describe a custom one." See [`references/templates.md`](references/templates.md). |
| 6 | LLM credential missing for chosen template | User handoff: "Paste your OpenAI/Anthropic/Google API key here." Then agent runs `wayai create-credential --name <name> --type "Bearer Token" --stdin`. |
| 7 | Template requires an OAuth channel (WhatsApp / Instagram / Google Calendar) | User handoff: "Open `https://app.wayai.pro/settings/connections?connector=<connector>`, finish the provider's flow, tell me when done." |
| 8 | Prerequisites met | Agent copies the template (`assets/templates/{en\|pt}/.../hub.md` + `<role>-instructions.md`) into `workspace/<hub>/`, replaces placeholders, then `wayai push -y`. |
| 9 | Push succeeded | Agent runs `wayai send-message "Hi"` and shows the response. User handoff: "Refine, add tools, or publish?" |
| 10 | User confirms publish | User handoff: "Open `https://app.wayai.pro/settings/organizations/<org_id>/hubs/<hub_id>/overview?action=publish` and click Publish." |

### Deeplinks (canonical URLs — never breadcrumbs)

| State | URL |
|-------|-----|
| 3 (signup / org create) | `https://app.wayai.pro/login`, `https://app.wayai.pro/settings/organizations/new` |
| 6 (credential pre-fill) | `https://app.wayai.pro/settings/credentials?type=bearer&name=<key-name>&prefill=true` |
| 7 (OAuth channel) | `https://app.wayai.pro/settings/connections?connector=<whatsapp\|instagram\|google-calendar>` |
| 10 (publish) | `https://app.wayai.pro/settings/organizations/<org_id>/hubs/<hub_id>/overview?action=publish` |

### Rules

- Detect, don't assume — every step starts with a fresh `wayai status --json`.
- Agent actions get a one-line receipt; user handoffs get one URL, one action, one return signal.
- Never invent URLs outside the table above. Never instruct "go to Settings → …" — always a deeplink. For non-onboarding URLs see [`references/navigation.md`](references/navigation.md).
- One question at a time during template selection (state 5): hub kind, then channel, then LLM provider.
- Never auto-commit anything created during onboarding — show `git diff` and wait for user approval before any commit.

## Workflow

### Existing hub
1. **Update CLI** — `wayai update` (always run before any operation)
2. **Update skill if stale** — `wayai status --json` and apply state machine row 1c: if `skill.latest` is set and newer than `skill.version`, run `npx skills add wayai-pro/wayai-skill -y` and exit (the refreshed skill loads on the next turn). Otherwise continue.
3. **Pull** — `wayai pull -y` (sync local files from platform; catches out-of-band changes)
4. **Read context** — `workspace/<hub>/AGENTS.md` for hub-specific notes (purpose, decisions, ongoing work). AGENTS.md-aware harnesses (Codex, Cursor, OpenCode, Aider) auto-load it natively
5. **Edit** — modify `hub.yaml`, `agents/*.yaml`, `agents/*.md`
6. **Push** — `wayai push -y` (apply to preview hub; auto-pulls server-assigned IDs back)
7. **Test** — `wayai send-message "Hello"`
8. **Review** — run `git diff`, ask user to confirm. **Never auto-commit.** User commits and pushes to `main`
9. **Go live** — sync preview → production via the platform UI when ready

### New hub (from scratch)
1. **Credentials** — `wayai create-credential --name "openai-key" --type "Bearer Token"` (one-time per org per credential)
2. **Init** — `wayai init` (interactive) or `wayai init --org <uuid>`
3. **Create files** — `workspace/<hub>/hub.yaml` + `agents/*.yaml` + `agents/*.md`
4. **Push** — `wayai push -y` auto-creates the hub, non-OAuth connections, and applies all config
5. **Test** — `wayai send-message "Hello"`

After the hub exists, follow the existing-hub workflow.

### Hub-Folder Memory

`workspace/<hub>/AGENTS.md` is the **hub-specific memory** for this hub — purpose, key decisions, ongoing work, business rules that only apply here, terminology, integration quirks. Read it at the start of every hub-related task. AGENTS.md-aware harnesses (Codex, Cursor, OpenCode, Aider) auto-load it natively when the agent's cwd is inside the hub folder.

**Maintain it actively:**
- After significant changes (new agent, new tool, business rule update), update `AGENTS.md` so future sessions inherit the context
- If `AGENTS.md` is missing for a hub, create it with what you know — purpose, current agents, recent decisions — and ask the user to confirm or enrich
- Keep it focused on *why* decisions were made and *what* makes this hub different. Don't restate platform mechanics — those live in this skill

**Overflow content goes into `workspace/<hub>/references/`:**
- When `AGENTS.md` grows past ~200 lines or starts mixing topics, extract the deeper material into focused files under `workspace/<hub>/references/`
- Examples: `references/business-rules.md`, `references/integrations.md`, `references/glossary.md`, `references/<api-name>-spec.md`, `references/<persona>-tone.md`
- Keep `AGENTS.md` as the always-on entry point with a short pointer to each reference file: e.g., "Detailed pricing rules: `references/pricing-rules.md`"
- Hub-folder `references/` are **not synced** to the platform — they're for agent context only, just like `AGENTS.md`

## Common CLI Commands

```bash
wayai update            # Update CLI (run before any operation)
wayai login             # OAuth — or `wayai login --token` for headless/CI
wayai create-credential # Create org credential (--name, --type "API Key"|"Bearer Token"|"Basic Auth", --org, --stdin)
wayai init              # Set up .wayai.yaml (interactive); --org <uuid> to skip prompt
wayai pull              # Pull hub config from platform (-y skips confirmation)
wayai push              # Push local changes (-y skips confirmation; auto-pulls IDs back)
wayai send-message      # Test message to a preview hub
wayai conversations     # List or inspect conversations
wayai delete-history    # Clear conversation history (testing)
wayai sync-skills       # Sync skills to provider connections; --connection-id <uuid> to scope
wayai analytics         # Summary + per-variable aggregates; --metric, --filter, --period, --json
wayai analytics query   # Structured ClickHouse query (multi-variable, group_by, correlations)
wayai run-eval          # Run eval scenarios
wayai eval-results      # Inspect eval results
wayai eval capture      # Capture production conversation as eval YAML (<conversation_id> [--set <name>])
wayai list              # List organizations and hubs
wayai status            # Show workspace status
wayai report-bug        # Create platform bug report (--title, --description, --hub, --conversation, --error)
```

Most commands accept `--hub <uuid|folder>` to disambiguate when multiple hubs live in `workspace/`.

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
workspace/                               # Local copy of hub configuration
└── <hub-slug>-<label>/                  # One folder per preview hub (disambiguated)
    ├── hub.yaml                         # Hub config + states + connections + outbound + resources
    ├── agents/
    │   ├── <slug>.yaml                  # Agent config (one per agent, slugified name)
    │   └── <slug>.md                    # Agent instructions (one per agent)
    ├── evals/                           # Eval scenarios (synced)
    │   ├── <name>.yaml
    │   └── <set>/<name>.yaml
    ├── resources/                       # Knowledge & skill resource files (synced)
    ├── AGENTS.md                        # Hub-level agent context (NOT synced; yours to edit)
    ├── CLAUDE.md                        # Per-hub Claude Code shim — `@AGENTS.md` (init-only, NOT synced)
    └── references/                      # Hub-specific supporting files (NOT synced)
```

Preview hub folders use `hub-slug-<preview_label>` or `hub-slug-<hub_id_prefix>` for disambiguation.

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
  app_permission: require_permission
  non_app_permission: everyone
  file_handling_mode: metadata_only
  hub_sla:
    time_threshold1: 60
    time_threshold2: 300
    time_threshold3: 600
  kanban_statuses:
    - name: New
      order: 0
      color: "#22c55e"
      isInitialStatus: true
      triggersAgentResponse: true
    - name: Waiting for Customer
      order: 1
      color: "#f59e0b"
      followups:
        - order: 0
          type: inactivity
          threshold: 30
          timeUnit: minutes
          instructions: "Hi! Just checking in — do you still need help?"
    - name: Resolved
      order: 2
      color: "#ef4444"
      isTerminalStatus: true

states:
  - id: "state-uuid-789"          # set by pull
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
    service: User Tool - API Key
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
# include_message_timestamps: false  # default; when true, appends [timestamp, day] to user messages
settings:
  model: claude-sonnet-4-6
  temperature: 0.7
  max_tokens: 4096
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

## Key Rules

- **Read-only fields:** `hub_id`, `hub_environment`, `id` — set by `wayai pull`, never edit
- **Connection auto-creation:** non-OAuth connections in `hub.yaml` resolve to org credentials by matching `service` + `authentication_type`. Use `credential` field to disambiguate when multiple org credentials share the same auth type. OAuth connections (WhatsApp, Instagram, Google Calendar) must already exist (UI setup) — referenced by name only
- **Tool groups:** `native` (platform built-ins by name), `delegation` (agent-to-agent/team handoff), `custom` (HTTP endpoints with connection)
- **Renaming:** change the `name` field — the stable `id` ensures it's detected as a rename, not delete + create. For agents, `wayai push` auto-renames the `.yaml` and `.md` files
- **Default omission:** fields matching defaults are omitted (e.g., `enabled: true`, kanban flags default `false`, `excludeHolidays` defaults `true`)
- **Entity matching (sync/diff):** `id` first (stable UUID), then fallback. Exceptions: states match by `name + scope`; evals match by `name + path`; native tools by `tool_name` per agent

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
- Always save under `workspace/<hub>/agents/`, never `/tmp` or other paths
- Always `wayai pull` before editing to avoid clobbering out-of-band changes
- Instructions support dynamic placeholders: `{{now()}}`, `{{user_name()}}`, `{{state()}}`, etc. — see [`references/agents/instructions.md`](references/agents/instructions.md)

## Reference Documentation

References mirror the hub navigation. Open the relevant file when working on that domain.

| Domain | Reference | When to read |
|--------|-----------|--------------|
| **Connections** | [`references/connections.md`](references/connections.md) | Wiring up a channel, agent provider, tool API, or speech connector |
| **Agents** | [`references/agents/roles-and-settings.md`](references/agents/roles-and-settings.md) | Choosing an agent role, delegation flow, connector-specific settings |
| **Agents** | [`references/agents/native-tools.md`](references/agents/native-tools.md) | Native tool parameters per connector, meta tools (`get_tool_schema`, `execute_tool`) |
| **Agents** | [`references/agents/custom-tools.md`](references/agents/custom-tools.md) | Custom HTTP tool format, OpenAI function schema, `composed_tools` side effects |
| **Agents** | [`references/agents/instructions.md`](references/agents/instructions.md) | Placeholder syntax (`{{now()}}`, `{{state()}}`, etc.) for `agents/<slug>.md` |
| **Resources** | [`references/resources.md`](references/resources.md) | Knowledge bases, skill resources, agent linkage, provider sync (`wayai sync-skills`) |
| **States** | [`references/states.md`](references/states.md) | State JSON Schemas, scope, agent read/write, initial values |
| **Outbound** | [`references/outbound.md`](references/outbound.md) | Outbound contacts, lists, schedules, channel rules, execution modes |
| **Evals** | [`references/evals.md`](references/evals.md) | Eval scenario YAML, scenario sets, `wayai eval capture` from production |
| **Analytics** | [`references/analytics.md`](references/analytics.md) | `wayai analytics` and `wayai analytics query` flags, metric paths, filters |
| **Templates** | [`references/templates.md`](references/templates.md) | Hub templates catalog, file format, placeholders |
| **Navigation** | [`references/navigation.md`](references/navigation.md) | App URL surface (`/chat`, `/task`, `/support`, `/settings/...`, `/user/...`), hub-detail tabs, query-string deep links |

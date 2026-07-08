# Progress Report (progress.html)

An agent-generated, self-contained HTML snapshot of a hub's **build progress and go-live readiness**, written to `wayai-ws/hubs/<hub>/progress.html`. It opens straight from disk (`file://`), makes zero network requests, and is safe to hand to stakeholders who have no platform access. It reports what only the agent side of the fence knows: local ↔ preview ↔ production drift, plan-vs-actual from the hub's `AGENTS.md`, readiness gaps, decisions, and next steps. It is **not** a config viewer — the settings UI already shows live config; config appears here as counts plus deeplinks only.

**Scope: code-harness agents only** (Claude Code, Codex, Cursor, OpenCode — anything with filesystem + shell access). App-harness agents (MCP tools, e.g. Claude Desktop): this feature does not apply to you — if the user asks for it, explain that the report is generated from a code-harness session in the hub workspace.

The file is never synced: `wayai push` doesn't read it and `wayai pull` never touches it (same non-synced status as the hub folder's `AGENTS.md` and `references/`).

## Table of Contents

- [Scope and tripwires](#scope-and-tripwires)
- [When to generate and refresh](#when-to-generate-and-refresh)
- [Data collection recipe](#data-collection-recipe)
- [The Build plan convention](#the-build-plan-convention)
- [JSON status model](#json-status-model)
- [Pipeline states](#pipeline-states)
- [Scope coverage](#scope-coverage)
- [Readiness rules](#readiness-rules)
- [Generating the file](#generating-the-file)
- [Security and sharing](#security-and-sharing)
- [Edge cases](#edge-cases)
- [The template](#the-template)

---

## Scope and tripwires

Three hard rules, in priority order:

1. **No config mirror.** Never reproduce a settings tab inside the report — no agent lists with settings, no connection tables, no kanban boards, no state schemas. Config appears as counts plus deeplinks into the settings UI, nothing more. If you find yourself rendering per-entity config rows, stop and link instead.
2. **Honesty boundary.** Anything not verifiable from local files or a documented CLI output — credential validity, team existence, OAuth completion, channel provisioning — is reported at `info` level with a deeplink, never as a pass/fail claim. And **no aggregate readiness score, ever**: a single percentage would launder info-level uncertainty into false precision. The renderer shows per-status counts only.
3. **Public-safe.** Assume the file will be emailed or PDF'd outside the org. Rules in [Security and sharing](#security-and-sharing).

---

## When to generate and refresh

Trigger on task boundaries, not on every command (the workflow pushes after every edit — nobody reads a report mid-iteration):

| # | Trigger | Action |
|---|---------|--------|
| T1 | User asks for status/progress ("where are we?", "how close to launch?", "can I share progress?", "update the report") | Generate — create or refresh. If no `## Build plan` exists in the hub's `AGENTS.md`, offer once to write one from session context, then generate either way |
| T2 | End of an initial hub build (hub created this session and responding to a test message) | Create the report; tell the user in one line that it exists and is shareable |
| T3 | `progress.html` already exists AND the current task ran `wayai push` or `wayai publish` | Refresh **once**, after the task's last state-changing command. If that lands after the git-diff review step (publish is workflow step 9), show the report file's incremental diff before any commit |
| T4 | Anything else | Never create the file unprompted — a generated HTML appearing in a repo whose owner never asked is a surprise, not a feature |

Placement rules:

- Always `wayai-ws/hubs/<hub>/progress.html` — hub-folder root, one report per hub
- **Preview folders only.** Never generate inside the production mirror folder (bare slug, `hub_environment: production`, read-only marker in `hub.yaml`) — the preview's report already covers production via the pipeline strip, and `wayai pull` rewrites mirror contents
- Readers see a staleness banner once `generated_at` is more than 14 days old; when a T1–T3 trigger is ambiguous, prefer to refresh (regeneration is cheap)

---

## Data collection recipe

Ordered steps. Later steps never block on earlier failures — **generation never aborts**; missing signals degrade to `unknown` per the table:

| Step | Source | Yields | On failure |
|------|--------|--------|------------|
| 1 | `wayai status --json` | `auth.logged_in`, `active_org.id`/`name` (for deeplinks), `cli.version`, `skill.version`, `worktree_binding.hub_id` | CLI missing/errored → take the org id from `.wayai.yaml`; if still unknown, set `hub.org_id: null` (drops every deeplink) and pipeline edges `unknown` |
| 2 | Read `hub.yaml`, `agents/*.yaml`, `agents/*.md`, `evals/`, `journeys/`, `resources/` | Hub identity fields, all local readiness rules, config counts, and the **per-agent scope facts** — tool count (`tools.*` entries) and eval count (scenarios/journeys whose `agent:` names each agent) | Malformed YAML → mark the affected checks `unknown`, mention it in `narrative.summary` |
| 3 | Read the hub's `AGENTS.md` | `## Build plan` section (goal, items), `## Agent scope` per-agent responsibilities, decisions, next steps | Missing or still the seed placeholder → `plan.source: "none"`, `scope.*` responsibilities `null`; narrative from session knowledge or minimal |
| 4 | `wayai diff` | `pipeline.local_to_preview` | Command fails / not logged in → `unknown`; no `hub_id` in hub.yaml → `not_created` |
| 5 | `wayai diff --production` | `pipeline.preview_to_production` (corroborate `not_published` with the absence of the bare-slug production mirror folder) | Command fails → `unknown` |
| 6 | Session memory | `testing.send_message` check, from a `wayai send-message` run earlier in this session | Not run → status `unknown` |
| 7 | Assemble the JSON island, then write the file per [Generating the file](#generating-the-file) | — | — |

**Never run `wayai send-message` solely to populate the report** — it creates a real test conversation. Record its outcome only when it already ran in this session, or the user asks for a live test.

Read `wayai diff` output **coarsely**: `No differences` (or `(no changes)` under every changed section) means in-sync; any `+`/`~`/`-` line means drift. Name the changed sections in the edge `summary`. Some delete lines print raw UUIDs — summarize ("2 tools removed"), never reproduce them.

---

## The Build plan convention

An **optional** section in `wayai-ws/hubs/<hub>/AGENTS.md` that makes progress trackable across sessions. The report renders it as plan-vs-actual; it is a *view* of this section, never a separate copy.

- Exact heading `## Build plan` (write it exactly; match case-insensitively when reading)
- Optional first line `Goal: <one sentence>` directly under the heading → becomes `plan.goal`
- Then GitHub task-list items: `- [ ] text` / `- [x] text` (accept `X`). One nesting level allowed via 2-space-indented child checkboxes. Plain text only
- Keep it under ~30 items — past that, split into phases and archive completed phases into prose above the list
- **Ownership:** write the plan when scope is agreed with the user; tick items as work lands (a normal `AGENTS.md` edit — not synced to the platform); keep it honest — an item is done when it's pushed and working, not when the YAML exists
- Absent plan → the report renders "No build plan recorded" and, when generating, you offer once to create one (prefilled from session context)
- Per-agent responsibilities (what each agent does / doesn't do) live in a sibling `## Agent scope` section — see [Scope coverage](#scope-coverage)

Example:

```markdown
## Build plan
Goal: Atender alunos por WhatsApp e agendar aulas experimentais

- [x] Agente pilot com instruções
- [ ] Conexão WhatsApp (OAuth)
  - [ ] Registrar tester com #test
- [ ] Evals do fluxo de agendamento
```

---

## JSON status model

`schema_version: 1`. This model is the stable machine layer — check `id`s and enums are frozen so future tooling (e.g. a `wayai hub-status --json`) can supply the mechanical fields without touching the renderer.

Authoring rules:

- **All free text** (`title`, `detail`, `action`, `summary`, plan items, narrative) is authored by you **in the hub's `language`** (`en`/`pt`/`es`, from `hub.yaml`; omitted → `en`). Section headings and badge words come from the template's built-in dictionaries — never author those
- `generated_at` — current UTC instant, ISO-8601 (`2026-07-08T14:30:00Z`)
- `hub.org_id: null` ⇒ **omit every `link`/`links[].href`** — never guess a URL
- Deeplinks: only `https://app.wayai.pro/settings/organizations/<org_id>/hubs/<hub_id>/<tab>` where `<tab>` is one of the hub-detail tabs in [navigation.md](navigation.md#hub-level). Kanban has no tab — the kanban count gets no link. The renderer refuses any href not starting with `https://app.wayai.pro/`
- Serialize with every `<` escaped as `\u003c` (valid JSON — `JSON.parse` decodes it back to `<`). Escaping the whole `<`, not just `</`, is deliberate: it neutralizes every HTML sequence that can break the JSON island out of its `<script>` element — `</script>` (and its case/whitespace variants), plus `<!--`/`<script`, which would otherwise flip the parser into script-data-double-escaped state and swallow the island's real closing tag

Annotated example (hub language `pt`, mid-build):

```json
{
  "schema_version": 1,
  "language": "pt",
  "generated_at": "2026-07-08T14:30:00Z",
  "generated_by": { "harness": "Claude Code", "cli_version": "2.4.1", "skill_version": "6.27.0" },
  "hub": {
    "name": "Suporte Acqua",
    "folder": "suporte-acqua--main",
    "hub_id": "9f8e7d6c-…",
    "preview_label": "main",
    "hub_type": "chat",
    "ai_mode": "pilot+copilot",
    "timezone": "America/Sao_Paulo",
    "org_id": "1a2b3c4d-…",
    "org_name": "Acqua Training"
  },
  "pipeline": {
    "local_to_preview":      { "status": "drift", "summary": "Edições em Agents e Tools ainda não enviadas", "counts": null },
    "preview_to_production": { "status": "not_published", "summary": "Hub ainda não publicado", "counts": null }
  },
  "scope": {
    "hub": "Atender alunos por WhatsApp e agendar aulas experimentais",
    "agents": [
      { "name": "Pilot de Suporte", "role": "pilot",
        "responsibility": "Responde dúvidas e encaminha agendamentos ao especialista",
        "instructions": true, "tools": 6, "evals": 3 },
      { "name": "Especialista - Agendamento", "role": "pilot_specialist",
        "responsibility": "Agenda aulas experimentais; não responde cobrança",
        "instructions": true, "tools": 4, "evals": 0 }
    ]
  },
  "checks": [
    { "id": "platform.hub_created", "status": "pass", "title": "Hub criado na plataforma",
      "detail": "hub_id presente em hub.yaml", "action": null, "link": null },
    { "id": "connections.credentials", "status": "info", "title": "Credenciais das conexões",
      "detail": "2 de 3 conexões referenciam credenciais da organização; validade não é verificável localmente",
      "action": "Confirme na aba Connections",
      "link": "https://app.wayai.pro/settings/organizations/1a2b3c4d-…/hubs/9f8e7d6c-…/connections" }
  ],
  "plan": {
    "source": "agents_md",
    "goal": "Atender alunos por WhatsApp e agendar aulas experimentais",
    "items": [
      { "text": "Agente pilot com instruções", "done": true },
      { "text": "Conexão WhatsApp (OAuth)", "done": false,
        "children": [ { "text": "Registrar tester com #test", "done": false } ] }
    ],
    "note": null
  },
  "narrative": {
    "summary": "Configuração principal concluída; falta o canal WhatsApp e cobertura de evals.",
    "decisions": [ "Kanban com 4 estágios em vez de 6 — decidido em 2026-07-01" ],
    "next_steps": [ "Concluir OAuth do WhatsApp", "Capturar uma journey do fluxo feliz" ]
  },
  "config": {
    "counts": { "agents": 3, "connections": 2, "kanban_statuses": 4, "states": 1,
                "resources": 2, "evals": 0, "journeys": 0, "outbound_schedules": 0 },
    "links": [
      { "label": "Overview",    "href": "https://app.wayai.pro/settings/organizations/1a2b3c4d-…/hubs/9f8e7d6c-…/overview" },
      { "label": "Agents",      "href": "https://app.wayai.pro/settings/organizations/1a2b3c4d-…/hubs/9f8e7d6c-…/agents" },
      { "label": "Connections", "href": "https://app.wayai.pro/settings/organizations/1a2b3c4d-…/hubs/9f8e7d6c-…/connections" }
    ]
  }
}
```

Field rules:

| Field | Rules |
|-------|-------|
| `schema_version` | Integer, `1`. The renderer tolerates newer values (banner + best-effort render) |
| `language` | `en` \| `pt` \| `es`; renderer falls back to `en` |
| `generated_at` | ISO-8601 UTC; the renderer localizes it and computes relative age offline |
| `generated_by` | `{harness, cli_version, skill_version}` — all optional; rendered in the footer |
| `hub.*` | Every field nullable except `name` and `folder`. `hub_id: null` = never created |
| `pipeline.<edge>` | `{status, summary, counts}` — enums in [Pipeline states](#pipeline-states); `summary` is your one-liner. `counts` is a reserved machine-layer slot (not rendered) — set it `null` |
| `scope` | `{hub, agents[]}` — see [Scope coverage](#scope-coverage). `hub` is the authored one-line purpose (nullable). Each `agents[]` entry is `{name, role, responsibility?, instructions, tools, evals}`; the renderer derives the pass/warn status itself. Omit the whole block only when no conversational agent exists yet |
| `checks[]` | `{id, status, title, detail, action?, link?}` in the catalog's canonical order; `status` ∈ `pass\|warn\|fail\|info\|unknown`; omit conditional rules that don't apply. Only ids from the [catalog](#readiness-rules) — no ad-hoc checks |
| `plan` | `source` ∈ `agents_md`\|`none`; `items[].children` one level max; the renderer computes done/total itself |
| `narrative` | Plain text only (no markdown); `summary` may contain `\n` for paragraphs; `decisions[]`/`next_steps[]` optional |
| `config.counts` | Fixed key set, all eight always present (`0` allowed): `agents, connections, kanban_statuses, states, resources, evals, journeys, outbound_schedules` |
| `config.links` | Only documented tabs (see authoring rules above); typical set: Overview, Agents, Connections, Users |

---

## Pipeline states

| Edge | Status | Meaning / source |
|------|--------|------------------|
| local → preview | `in_sync` | `wayai diff` prints `No differences — local files match preview` (a fully-synced diff returns early; a partial diff shows `(no changes)` under each unchanged section) |
| | `drift` | Any `+`/`~`/`-` lines; `summary` names the changed sections |
| | `not_created` | No `hub_id` in `hub.yaml` |
| | `unknown` | Diff failed / not logged in / CLI missing |
| preview → production | `in_sync` / `drift` | `wayai diff --production` rendered diff |
| | `not_published` | The command's distinct never-published message; corroborated by the absence of the bare-slug production mirror folder |
| | `unknown` | Command failure |

Sync state lives **only** here — it is deliberately not duplicated as readiness checks.

---

## Scope coverage

The most important view, and the one the settings UI cannot give: **is the hub's job assigned to agents, and is each assignment defined and tested?** The UI shows agents, tools, and evals as three separate tabs with no notion of what any of them is *for*. This section synthesizes them into a hub-scope → agent-scope tree. It renders right after the pipeline strip, above the structural readiness checks.

**Structure** (the `scope` block in the island):

- `scope.hub` — one line: what the whole hub is for. Authored — take it from the AGENTS.md `## Build plan` `Goal:` line, or the hub's purpose in AGENTS.md. `null` → "No hub scope recorded".
- `scope.agents[]` — one entry **per conversational agent** (pilot / copilot / `*_specialist` / `*_advisor`). Do **not** list background agents (summarizer, monitor, evaluators) — they have no user-facing responsibility. Each entry:
  - `name`, `role` — from `agents/<slug>.yaml` (computed)
  - `responsibility` — what this agent does / explicitly does not do. **Authored** (see the convention below); `null` → "No scope stated"
  - `instructions` — boolean: is `agents/<slug>.md` non-empty (computed)
  - `tools` — integer: count of entries across the agent's `tools.{native,custom,delegation,mcp}` (computed)
  - `evals` — integer: number of eval scenarios/journeys whose `agent:` names this agent (computed)

**Honesty rules** (same boundary as everywhere else):

- **Computed = pass/warn; intent-match = narrative.** The renderer derives each agent's badge from the computable coverage only: **pass** when it has instructions *and* ≥1 eval; **warn** otherwise (undefined behavior, or defined-but-untested). Whether the instructions *actually fulfill* the stated responsibility is a judgment call — put that in `narrative`, never a green check.
- **Tool count is context, not a score** — a count can't tell you the *right* tools are present, and a pure-reply agent legitimately has none. It's displayed, never gated on.
- **Not a config mirror.** Responsibility line + the three coverage chips, nothing more. Never render the tool list, model, temperature, or schemas — that's the Agents tab (the tripwire from [Scope and tripwires](#scope-and-tripwires) is most tempting here).

**Authoring per-agent responsibilities (AGENTS.md).** Intended scope must be stated somewhere *other than the agent's own `.md`* — otherwise "do the instructions cover the scope?" is circular. Capture it in the hub's `AGENTS.md`, one line per agent under an optional `## Agent scope` heading:

```markdown
## Agent scope
- Pilot de Suporte: responde dúvidas e encaminha agendamentos ao especialista
- Especialista - Agendamento: agenda aulas experimentais; não responde cobrança
```

Match agents by display name. A missing entry → that agent's `responsibility` is `null` ("No scope stated"); when generating, offer once to capture it from what you know (same pattern as the Build plan). This section pairs with [The Build plan convention](#the-build-plan-convention): scope answers "is the job assigned and covered", the plan answers "what's done vs. todo".

---

## Readiness rules

Fixed catalog — emit in this order, only these ids, respecting each rule's "emit when" condition. `action` is one imperative sentence in the hub's language; attach `link` only where the table says so.

| id | Source | Emit when | Semantics | Action on non-pass |
|----|--------|-----------|-----------|--------------------|
| `platform.hub_created` | `hub.yaml` `hub_id` | always | present → **pass**; absent → **fail** | "Run `wayai create` (or `wayai push`) to create the hub" |
| `agents.mode_coverage` | `ai_mode` × agent roles | always | roles required by `ai_mode` present (pilot mode → one `pilot`; copilot → one `copilot`; pilot+copilot → both) → **pass**; missing or duplicated required role → **fail**; `turned_off` → **info** ("AI disabled by design") | "Add an agent with role `<role>`" |
| `agents.connections_resolve` | local | always | every agent `connection:` matches a `connections[].name` → **pass**; any dangling → **fail**, naming offenders | "Fix `connection:` on `<agent>` or add the connection to hub.yaml" |
| `agents.instructions` | `agents/<slug>.md` | always | every enabled **pilot/copilot/`*_specialist`/`*_advisor`** has a non-empty `.md` → **pass**; any missing → **fail**. Background/auto-provisioned roles (summarizer, monitor, evaluators) without `.md` → **info**, never fail | "Write `agents/<slug>.md`" |
| `agents.delegation_agents` | local | agent-type delegations exist | every `type: agent` `target` matches an agent display name → **pass**; dangling → **fail** | "Fix `target:` on `<agent>` — names are foreign keys" |
| `agents.delegation_teams` | local (unverifiable) | team-type delegations exist | always **info** — team names are UI-managed, not locally verifiable | "Verify team `<name>` exists in the Users tab" + `/users` link |
| `kanban.initial_status` | local | kanban configured | exactly one `isInitialStatus` → **pass**; zero or more than one → **fail** | "Mark exactly one status `isInitialStatus: true`" |
| `kanban.terminal_status` | local | kanban configured | ≥1 `isTerminalStatus` → **pass**; none → **warn** | "Add a terminal status so the team can close conversations from the board" |
| `kanban.configured` | local | **no** kanban | single **info** row — an empty `kanban_statuses` is legal and must not read as a failure | — |
| `connections.llm` | local | always | ≥1 `type: Agent` connection → **pass**; none → **fail**; `turned_off` mode → **info** | "Add an Agent connection (an org credential auto-creates it on push)" |
| `connections.credentials` | local (weak signal) | always | always **info**: report how many connections carry a `credential:` org link; absence is ambiguous (direct secret / OAuth / missing) — validity is not locally verifiable | "Check credential status on the Connections tab" + `/connections` link |
| `channels.external` | local (inferred) | `hub_type: chat` | always **info**: list channels inferred from Channel-type connections (WhatsApp / Instagram / email / Telegram); none → "reachable in-app only". Provisioning/OAuth state not locally verifiable | "Verify channel setup on the Connections tab" + `/connections` link |
| `connections.production_credentials` | local + mirror | any `sync_credentials_to_production: false` AND hub published | always **info** — production credential is managed separately | "Set it with `wayai set-connection-credential` or in the UI" |
| `resources.content` | local | `resources:` declared | every declared resource has `resources/<slug>/` with ≥1 file → **pass**; missing/empty folder → **warn** | "Add content under `resources/<slug>/` and push" |
| `evals.coverage` | local | **no evals exist** | **warn** — zero evals hub-wide. Per-agent eval coverage lives in [Scope coverage](#scope-coverage), so this fires only as the all-zero fallback | "Capture a journey after a good test run (`wayai eval journey capture`)" |
| `memory.agents_md` | local | always | hub `AGENTS.md` exists and differs from the CLI seed → **pass**; missing or still placeholder → **warn**. Detect the seed via the contiguous substring `placeholder once you do` (the seed line-wraps mid-sentence — do not match across the line break). Fail-open: if the CLI seed wording ever changes, this check passes | "Fill in the hub's AGENTS.md — it is the hub's memory" |
| `testing.send_message` | session | always | ran this session and replied → **pass**; ran and errored → **fail** with a short error summary (this is the strongest end-to-end signal: pilot + connection + credential); not run → **unknown** | unknown: "Ask me to run `wayai send-message` for an end-to-end check" |

---

## Generating the file

The template is copied **mechanically**, not retyped — a shell extraction is byte-perfect where manual transcription is not.

1. **Assemble the JSON island** per [JSON status model](#json-status-model). Then escape it **mechanically** — replace every `<` with `\u003c` across the whole island (one blind string replacement; `\u003c` is valid JSON that `JSON.parse` decodes back to `<`). Don't hand-escape — a single raw `</script>`, `<!--`, or `<script` inside any authored string (a hub name, a `detail`, the narrative) breaks the island out of its `<script>` element and turns the shared file into an XSS/exfiltration vector on the recipient's machine. Escaping every `<` closes all of those at once.
2. **Extract the template** from this reference file — use the path of the copy you are reading (the skill install dir, e.g. `.claude/skills/wayai/references/progress-report.md` or `.agents/skills/wayai/references/progress-report.md`):

   ```bash
   sed -n '/^<!-- BEGIN progress-template/,/^<!-- END progress-template/p' <path-to-this-file> \
     | sed '1d;$d' > wayai-ws/hubs/<hub>/progress.html
   ```

   (The outer `sed` grabs marker line through marker line — the `^<!--` anchor is what keeps this very paragraph from matching; the inner strips the two marker comments, leaving pure HTML.)
3. **Inject the island**: in the written file, replace the sentinel — the exact string `id="report-data">{}</script>` — so `{}` becomes your JSON. One exact-string edit; touch nothing else.
4. **Self-check** the result:

   ```bash
   grep -c 'id="report-data"' progress.html        # must print 1
   grep -c 'TEMPLATE_VERSION=1' progress.html       # must print 1
   grep -oiE '<!--|</?script' progress.html | wc -l # must print 4 — more means an authored string carried a raw HTML tag/comment (step-1 escaping was skipped)
   tail -1 progress.html                            # must print </html>
   ```

   Also confirm the island is no longer `{}`. The tag count is the security backstop: the template has exactly two `<script` opens + two `</script>` closes and zero `<!--`, so a count above 4 means an authored string carried a raw `<!--`, `<script`, or `</script>` (in any case/whitespace form) — i.e. step-1 escaping was skipped. Fix the escaping and regenerate. (Use `grep -oiE`, not `grep -c` — a breakout can share the island's own line, and the variants are case-insensitive.)

**Fallback** (no shell, or the reference path can't be resolved): copy everything between the BEGIN/END markers verbatim by hand — do not reformat, "fix", or reorder anything — then apply steps 3–4.

**Full regeneration, always.** Never patch, merge, or preserve any part of an existing `progress.html` — not even when only the data changed. An existing file whose `TEMPLATE_VERSION` differs from this reference is stale markup; regenerating wholesale is what keeps template updates and data consistent.

---

## Security and sharing

- **Never embed**: secrets, API keys, tokens (they never appear in workspace YAML — but also never paste CLI output containing them), `user_email` values, conversation transcripts, end-user names or identifiers. `hub_id`/`org_id` are fine — they already appear in deeplink URLs and grant nothing without an authenticated session
- **Audience filter**: narrative and plan text flow from `AGENTS.md` — internal memory — into a client-shareable file. Author `narrative.*` and check texts **for the sharing audience**; never paste `AGENTS.md` internals verbatim. An internal note like "chose the cheaper model to fit the client's budget" must not reach the client's copy
- The renderer builds all DOM via text nodes (no HTML injection) and refuses non-`app.wayai.pro` link targets; the CSP meta blocks every network request. Don't weaken either
- **Git**: the file is a normal committable artifact; full-file refreshes produce large diffs — expected. It rides the standard review flow (never auto-commit); don't add it to `.gitignore` unilaterally

---

## Edge cases

| Case | Detection | Behavior |
|------|-----------|----------|
| Never-pushed hub | no `hub_id` in `hub.yaml` | Generate anyway — a valid pre-creation snapshot: pipeline `not_created`/`not_published`, `platform.hub_created` fail, all deeplinks omitted |
| Production mirror folder | bare-slug folder, `hub_environment: production`, read-only marker | **Refuse** to generate there; point at the preview folder |
| Multi-hub workspace | binding / `--hub` target | One report per hub folder; act on the bound/targeted hub; ambiguous → ask, never fan out unprompted |
| Not logged in / CLI missing | `wayai status --json` fails | Generate a partial report (edges `unknown`); tell the user why it's partial |
| No `AGENTS.md` / no `## Build plan` | file read | `plan.source: "none"` → report shows "No build plan recorded"; offer once to create the plan |
| Island corruption | a raw `<`-initiated HTML sequence in authored text | Prevented by escaping every `<` as `\u003c` (step 1) and caught by the self-check tag count. A raw `<!--`+`<script` pair would blank the page (parser swallows the renderer) — which is exactly what the self-check exists to stop before you ship |

---

## The template

Current `TEMPLATE_VERSION`: **1**. Copy everything between the BEGIN/END markers exactly — the **only** region you may change is the content of the `<script type="application/json" id="report-data">` island (the `{}` sentinel). Extraction procedure: [Generating the file](#generating-the-file).

```html
<!-- BEGIN progress-template v1 -->
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<meta http-equiv="Content-Security-Policy" content="default-src 'none'; script-src 'unsafe-inline'; style-src 'unsafe-inline'; img-src data:">
<title>WayAI Progress Report</title>
<style>
:root{
  --bg:#f5f6f8;--card:#ffffff;--text:#1a2233;--muted:#66708a;--border:#dde1ea;
  --accent:#2563eb;--pass:#16a34a;--warn:#d97706;--fail:#dc2626;--info:#2563eb;--unk:#6b7280;
  --pass-bg:#f0faf3;--warn-bg:#fdf6ec;--fail-bg:#fdf0f0;--info-bg:#eff4fe;--unk-bg:#f2f3f5;
}
@media (prefers-color-scheme:dark){:root{
  --bg:#12151c;--card:#1a1f2a;--text:#e6e9f0;--muted:#9aa3b8;--border:#2c3342;
  --pass-bg:#15251b;--warn-bg:#2a2113;--fail-bg:#2a1616;--info-bg:#162031;--unk-bg:#20242e;
}}
*{box-sizing:border-box}
body{margin:0;padding:24px 16px 40px;background:var(--bg);color:var(--text);
  font:15px/1.55 system-ui,-apple-system,"Segoe UI",Roboto,sans-serif}
main{max-width:860px;margin:0 auto}
.card{background:var(--card);border:1px solid var(--border);border-radius:10px;padding:20px 24px;margin:0 0 16px}
h1{font-size:.8rem;font-weight:600;letter-spacing:.08em;text-transform:uppercase;color:var(--muted);margin:0 0 6px}
h2{font-size:1.05rem;margin:0 0 12px}
.hub-name{font-size:1.5rem;font-weight:700;margin:0 0 10px}
.chips{display:flex;flex-wrap:wrap;gap:6px;margin:0 0 10px}
.chip{font-size:.75rem;padding:2px 10px;border:1px solid var(--border);border-radius:999px;color:var(--muted)}
.meta{font-size:.8rem;color:var(--muted);margin:2px 0}
.mono{font-family:ui-monospace,Menlo,Consolas,monospace;font-size:.75rem}
.banner{border-radius:8px;padding:10px 14px;margin:12px 0 0;font-size:.85rem}
.banner-warn{background:var(--warn-bg);color:var(--warn);border:1px solid var(--warn)}
.banner-err{background:var(--fail-bg);border:1px solid var(--fail)}
.banner-err p{margin:4px 0;color:var(--text)}
.pipe{display:flex;align-items:stretch;gap:10px;flex-wrap:wrap}
.pipe-node{border:1px solid var(--border);border-radius:8px;padding:10px 16px;font-weight:600;
  font-size:.85rem;display:flex;align-items:center;white-space:nowrap}
.pipe-edge{flex:1;min-width:130px;text-align:center;align-self:center}
.pipe-arrow{color:var(--muted);font-size:.8rem;letter-spacing:.2em}
.pipe-status{display:block;font-size:.8rem;font-weight:600}
.pipe-note{display:block;font-size:.72rem;color:var(--muted)}
.s-in_sync{color:var(--pass)}.s-drift{color:var(--warn)}
.s-not_created,.s-not_published,.s-unknown{color:var(--unk)}
@media (max-width:640px){.pipe{flex-direction:column;align-items:stretch}
 .pipe-node{justify-content:center}.pipe-edge{min-width:0}}
.badge{display:inline-block;min-width:86px;text-align:center;font-size:.72rem;font-weight:700;
  padding:2px 8px;border-radius:999px;white-space:nowrap}
.b-pass{color:var(--pass);background:var(--pass-bg)}
.b-warn{color:var(--warn);background:var(--warn-bg)}
.b-fail{color:var(--fail);background:var(--fail-bg)}
.b-info{color:var(--info);background:var(--info-bg)}
.b-unknown{color:var(--unk);background:var(--unk-bg)}
.check{display:flex;gap:12px;padding:10px 0;border-top:1px solid var(--border);align-items:baseline}
.chips + .check{border-top:none}
.check-title{font-weight:600}
.check-detail{font-size:.85rem;color:var(--muted)}
.check-action{font-size:.85rem;color:var(--accent)}
.scope-role{font-weight:400;color:var(--muted);font-size:.8rem;margin-left:6px}
.scope-chips{display:flex;flex-wrap:wrap;gap:6px;margin:6px 0 0}
.scope-chip{font-size:.72rem;padding:1px 9px;border:1px solid var(--border);border-radius:999px;color:var(--muted)}
.chip-warn{color:var(--warn);border-color:var(--warn)}
.pill{display:inline-block;font-size:.78rem;padding:2px 12px;border:1px solid var(--border);
  border-radius:999px;color:var(--accent);text-decoration:none;margin:0 6px 6px 0}
.goal{font-size:.9rem;margin:0 0 10px}
.bar{height:6px;border-radius:3px;background:var(--unk-bg);margin:8px 0 14px;overflow:hidden}
.bar-fill{height:100%;background:var(--pass)}
ul.plain{list-style:none;padding:0;margin:0}
.plan-item{padding:3px 0}
.plan-child{margin-left:24px}
.plan-done{color:var(--muted);text-decoration:line-through}
.tick{display:inline-block;width:1.3em;font-weight:700}
.tick-done{color:var(--pass)}.tick-open{color:var(--muted)}
h3{font-size:.8rem;font-weight:600;letter-spacing:.05em;text-transform:uppercase;color:var(--muted);margin:16px 0 6px}
.narr p{margin:0 0 8px}
.narr ul{margin:0;padding-left:20px}
.counts{display:grid;grid-template-columns:repeat(auto-fill,minmax(108px,1fr));gap:10px;margin:0 0 14px}
.count-tile{border:1px solid var(--border);border-radius:8px;padding:10px 6px;text-align:center}
.count-num{font-size:1.35rem;font-weight:700}
.count-label{font-size:.7rem;color:var(--muted)}
footer{font-size:.72rem;color:var(--muted);text-align:center;margin-top:20px}
a{color:var(--accent)}
noscript{display:block;max-width:860px;margin:0 auto;padding:20px}
@media print{
  :root{--bg:#fff;--card:#fff;--text:#111;--muted:#555;--border:#ccc;
    --pass-bg:#fff;--warn-bg:#fff;--fail-bg:#fff;--info-bg:#fff;--unk-bg:#fff}
  body{padding:0}.card{break-inside:avoid;border-color:#ccc}
  a{color:inherit;text-decoration:underline}
}
</style>
</head>
<body>
<noscript>Enable JavaScript to view this report. / Ative o JavaScript para ver este relatório. / Active JavaScript para ver este informe.</noscript>
<main id="report-root"></main>
<script type="application/json" id="report-data">{}</script>
<script>
(function(){
"use strict";
var TEMPLATE_VERSION=1, SUPPORTED_SCHEMA=1, STALE_DAYS=14;
var GLYPH={pass:"✓",warn:"!",fail:"✕",info:"i",unknown:"?"};
var STATUSES=Object.keys(GLYPH);
var L={
en:{title:"WayAI Progress Report",generated:"Generated",refresh:"Ask the hub's coding agent to refresh this report.",
 stale:"This report may be out of date — ask the hub's coding agent to refresh it.",
 newer:"This report was generated with a newer template — ask the hub's coding agent to refresh it.",
 pipeline:"Sync pipeline",local:"Local files",preview:"Preview",production:"Production",
 in_sync:"In sync",drift:"Pending changes",not_created:"Not created yet",not_published:"Not published yet",unknown:"Unknown",
 checks:"Readiness",pass:"Pass",warn:"Attention",fail:"Missing",info:"Info",
 plan:"Build plan",noplan:"No build plan recorded.",goal:"Goal",done:"done",
 narrative:"Status and next steps",decisions:"Key decisions",next:"Next steps",
 config:"Configuration at a glance",open:"Open in WayAI",
 scope:"Scope coverage",no_hub_scope:"No hub scope recorded.",no_scope:"No scope stated",has_instructions:"instructions",no_instructions:"no instructions",tools_label:"tools",evals_label:"evals",
 agents:"Agents",connections:"Connections",kanban_statuses:"Kanban statuses",states:"States",
 resources:"Resources",evals:"Evals",journeys:"Journeys",outbound_schedules:"Outbound schedules"},
pt:{title:"Relatório de Progresso WayAI",generated:"Gerado",refresh:"Peça ao agente de código do hub para atualizar este relatório.",
 stale:"Este relatório pode estar desatualizado — peça ao agente de código do hub para atualizá-lo.",
 newer:"Este relatório foi gerado com um template mais novo — peça ao agente de código do hub para atualizá-lo.",
 pipeline:"Sincronização",local:"Arquivos locais",preview:"Preview",production:"Produção",
 in_sync:"Sincronizado",drift:"Alterações pendentes",not_created:"Ainda não criado",not_published:"Ainda não publicado",unknown:"Desconhecido",
 checks:"Prontidão",pass:"OK",warn:"Atenção",fail:"Faltando",info:"Info",
 plan:"Plano de construção",noplan:"Nenhum plano de construção registrado.",goal:"Objetivo",done:"concluídos",
 narrative:"Status e próximos passos",decisions:"Decisões principais",next:"Próximos passos",
 config:"Configuração em resumo",open:"Abrir no WayAI",
 scope:"Cobertura de escopo",no_hub_scope:"Nenhum escopo do hub registrado.",no_scope:"Escopo não definido",has_instructions:"instruções",no_instructions:"sem instruções",tools_label:"ferramentas",evals_label:"evals",
 agents:"Agentes",connections:"Conexões",kanban_statuses:"Status do kanban",states:"Estados",
 resources:"Recursos",evals:"Evals",journeys:"Journeys",outbound_schedules:"Agendamentos outbound"},
es:{title:"Informe de Progreso WayAI",generated:"Generado",refresh:"Pídele al agente de código del hub que actualice este informe.",
 stale:"Este informe puede estar desactualizado — pídele al agente de código del hub que lo actualice.",
 newer:"Este informe se generó con una plantilla más nueva — pídele al agente de código del hub que lo actualice.",
 pipeline:"Sincronización",local:"Archivos locales",preview:"Preview",production:"Producción",
 in_sync:"Sincronizado",drift:"Cambios pendientes",not_created:"Aún no creado",not_published:"Aún no publicado",unknown:"Desconocido",
 checks:"Preparación",pass:"OK",warn:"Atención",fail:"Falta",info:"Info",
 plan:"Plan de construcción",noplan:"Ningún plan de construcción registrado.",goal:"Objetivo",done:"completados",
 narrative:"Estado y próximos pasos",decisions:"Decisiones clave",next:"Próximos pasos",
 config:"Configuración de un vistazo",open:"Abrir en WayAI",
 scope:"Cobertura de alcance",no_hub_scope:"Ningún alcance del hub registrado.",no_scope:"Alcance no definido",has_instructions:"instrucciones",no_instructions:"sin instrucciones",tools_label:"herramientas",evals_label:"evals",
 agents:"Agentes",connections:"Conexiones",kanban_statuses:"Estados del kanban",states:"Estados",
 resources:"Recursos",evals:"Evals",journeys:"Journeys",outbound_schedules:"Programaciones outbound"}
};
var lang="en";
function t(k){return (L[lang]&&L[lang][k])||L.en[k]||k;}
function h(tag,cls){var e=document.createElement(tag);if(cls)e.className=cls;
 for(var i=2;i<arguments.length;i++){var k=arguments[i];if(k==null)continue;
  e.appendChild(typeof k==="string"?document.createTextNode(k):k);}return e;}
function safeHref(u){return (typeof u==="string"&&u.indexOf("https://app.wayai.pro/")===0)?u:null;}
function pill(label,url){var u=safeHref(url);if(!u)return h("span","chip",String(label));
 var a=h("a","pill",String(label));a.href=u;a.target="_blank";a.rel="noopener";return a;}
function badge(st){var s=STATUSES.indexOf(st)>=0?st:"unknown";
 return h("span","badge b-"+s,GLYPH[s]+" "+t(s));}
function card(title){var c=h("section","card");if(title)c.appendChild(h("h2",null,title));
 for(var i=1;i<arguments.length;i++)if(arguments[i]!=null)c.appendChild(arguments[i]);return c;}
function locale(){return lang==="pt"?"pt-BR":lang==="es"?"es":"en";}
function fmtWhen(iso){var d=new Date(iso);if(isNaN(d.getTime()))return String(iso);
 var s;try{s=d.toLocaleString(locale(),{dateStyle:"medium",timeStyle:"short"});}catch(e){s=d.toISOString();}
 try{var days=Math.round((Date.now()-d.getTime())/864e5);
  s+=" ("+new Intl.RelativeTimeFormat(locale(),{numeric:"auto"}).format(-days,"day")+")";}catch(e){}
 return s;}
function ageDays(iso){var d=new Date(iso);return isNaN(d.getTime())?0:(Date.now()-d.getTime())/864e5;}
function errorCard(){var c=h("section","card banner-err");
 c.appendChild(h("p",null,"This report's data is unreadable — ask the hub's coding agent to regenerate it."));
 c.appendChild(h("p",null,"Os dados deste relatório estão ilegíveis — peça ao agente de código do hub para regenerá-lo."));
 c.appendChild(h("p",null,"Los datos de este informe son ilegibles — pídele al agente de código del hub que lo regenere."));
 return c;}
function renderHeader(d){var hub=d.hub||{};
 var c=card(null,h("h1",null,t("title")));
 c.appendChild(h("div","hub-name",String(hub.name||t("unknown"))));
 var chips=h("div","chips");
 [hub.hub_type,hub.ai_mode,hub.preview_label,hub.timezone].forEach(function(v){
  if(v)chips.appendChild(h("span","chip",String(v)));});
 if(chips.childNodes.length)c.appendChild(chips);
 if(hub.folder)c.appendChild(h("div","meta mono",String(hub.folder)));
 if(d.generated_at)c.appendChild(h("div","meta",t("generated")+": "+fmtWhen(d.generated_at)));
 c.appendChild(h("div","meta",t("refresh")));
 if(d.generated_at&&ageDays(d.generated_at)>STALE_DAYS)c.appendChild(h("div","banner banner-warn",t("stale")));
 if(typeof d.schema_version==="number"&&d.schema_version>SUPPORTED_SCHEMA)c.appendChild(h("div","banner banner-warn",t("newer")));
 return c;}
var PIPE_STATUSES=["in_sync","drift","not_created","not_published","unknown"];
function pipeEdge(edge){var e=edge||{};var st=e.status;
 if(PIPE_STATUSES.indexOf(st)<0)st="unknown";
 var w=h("div","pipe-edge");w.appendChild(h("span","pipe-arrow","▸▸▸"));
 w.appendChild(h("span","pipe-status s-"+st,t(st)));
 if(e.summary)w.appendChild(h("span","pipe-note",String(e.summary)));
 return w;}
function renderPipeline(d){var p=d.pipeline||{};var row=h("div","pipe",
  h("div","pipe-node",t("local")),pipeEdge(p.local_to_preview),
  h("div","pipe-node",t("preview")),pipeEdge(p.preview_to_production),
  h("div","pipe-node",t("production")));
 return card(t("pipeline"),row);}
function scopeRow(a){
 var name=h("div","check-title",String(a.name||""));
 if(a.role)name.appendChild(h("span","scope-role",String(a.role)));
 var body=h("div",null,name,
  h("div","check-detail",a.responsibility?String(a.responsibility):t("no_scope")));
 body.appendChild(h("div","scope-chips",
  h("span","scope-chip"+(a.instructions?"":" chip-warn"),t(a.instructions?"has_instructions":"no_instructions")),
  h("span","scope-chip",(a.tools||0)+" "+t("tools_label")),
  h("span","scope-chip"+((a.evals||0)>0?"":" chip-warn"),(a.evals||0)+" "+t("evals_label"))));
 var st=(a.instructions&&(a.evals||0)>0)?"pass":"warn";
 return h("div","check",badge(st),body);}
function renderScope(d){var sc=d.scope;if(!sc||typeof sc!=="object")return null;
 var c=card(t("scope"),h("p","goal",sc.hub?String(sc.hub):t("no_hub_scope")));
 if(Array.isArray(sc.agents))sc.agents.forEach(function(a){if(a)c.appendChild(scopeRow(a));});
 return c;}
function renderChecks(d){if(!Array.isArray(d.checks)||!d.checks.length)return null;
 var chips=h("div","chips");
 STATUSES.forEach(function(s){var n=d.checks.filter(function(c){return c&&(STATUSES.indexOf(c.status)>=0?c.status:"unknown")===s;}).length;
  if(n)chips.appendChild(h("span","badge b-"+s,n+" "+GLYPH[s]));});
 var c=card(t("checks"),chips);
 d.checks.forEach(function(ch){if(!ch)return;
  var body=h("div",null,h("div","check-title",String(ch.title||ch.id||"")));
  if(ch.detail)body.appendChild(h("div","check-detail",String(ch.detail)));
  if(ch.action){var act=h("div","check-action",String(ch.action)+" ");
   if(safeHref(ch.link))act.appendChild(pill("→",ch.link));body.appendChild(act);}
  c.appendChild(h("div","check",badge(ch.status),body));});
 return c;}
function planItems(items,depth,acc){var ul=h("ul","plain");
 items.forEach(function(it){if(!it)return;acc.total++;if(it.done)acc.done++;
  var li=h("li","plan-item"+(depth?" plan-child":""),
   h("span","tick "+(it.done?"tick-done":"tick-open"),it.done?"✓":"○"),
   h("span",it.done?"plan-done":null,String(it.text||"")));
  ul.appendChild(li);
  if(Array.isArray(it.children)&&it.children.length&&depth<1)li.appendChild(planItems(it.children,depth+1,acc));});
 return ul;}
function renderPlan(d){var p=d.plan;
 if(!p||p.source==="none"||!Array.isArray(p.items)||!p.items.length)
  return card(t("plan"),h("p","meta",t("noplan")));
 var acc={done:0,total:0};var list=planItems(p.items,0,acc);
 var c=card(t("plan"));
 if(p.goal)c.appendChild(h("p","goal",t("goal")+": "+String(p.goal)));
 c.appendChild(h("div","meta",acc.done+"/"+acc.total+" "+t("done")));
 var bar=h("div","bar",h("div","bar-fill"));
 bar.firstChild.style.width=(acc.total?Math.round(100*acc.done/acc.total):0)+"%";
 c.appendChild(bar);c.appendChild(list);
 if(p.note)c.appendChild(h("p","meta",String(p.note)));
 return c;}
function textList(arr){var ul=h("ul");arr.forEach(function(x){if(x!=null)ul.appendChild(h("li",null,String(x)));});return ul;}
function renderNarrative(d){var n=d.narrative;if(!n)return null;
 var c=card(t("narrative"));c.className+=" narr";var any=false;
 if(n.summary){String(n.summary).split(/\n+/).forEach(function(pg){
  if(pg.trim()){c.appendChild(h("p",null,pg.trim()));any=true;}});}
 if(Array.isArray(n.decisions)&&n.decisions.length){c.appendChild(h("h3",null,t("decisions")));c.appendChild(textList(n.decisions));any=true;}
 if(Array.isArray(n.next_steps)&&n.next_steps.length){c.appendChild(h("h3",null,t("next")));c.appendChild(textList(n.next_steps));any=true;}
 return any?c:null;}
function renderConfig(d){var cfg=d.config;if(!cfg)return null;
 var c=card(t("config"));
 if(cfg.counts&&typeof cfg.counts==="object"){var grid=h("div","counts");
  Object.keys(cfg.counts).forEach(function(k){var v=cfg.counts[k];
   if(typeof v!=="number")return;
   grid.appendChild(h("div","count-tile",h("div","count-num",String(v)),h("div","count-label",t(k))));});
  if(grid.childNodes.length)c.appendChild(grid);}
 if(Array.isArray(cfg.links)&&cfg.links.length){var row=h("div",null,h("h3",null,t("open")));
  var pills=h("div");cfg.links.forEach(function(l){
   if(l&&l.label&&safeHref(l.href))pills.appendChild(pill(l.label,l.href));});
  if(pills.childNodes.length){row.appendChild(pills);c.appendChild(row);}}
 return c.childNodes.length>1?c:null;}
function renderFooter(d){var g=d.generated_by||{};var parts=[];
 if(g.harness)parts.push(String(g.harness));
 if(g.cli_version)parts.push("CLI "+g.cli_version);
 if(g.skill_version)parts.push("Skill "+g.skill_version);
 parts.push("Template v"+TEMPLATE_VERSION);
 return h("footer",null,parts.join(" · "));}
function render(d,root){
 lang=(d.language==="pt"||d.language==="es")?d.language:"en";
 document.documentElement.lang=lang;
 if(d.hub&&d.hub.name)document.title=t("title")+" — "+d.hub.name;else document.title=t("title");
 [renderHeader(d),renderPipeline(d),renderScope(d),renderChecks(d),renderPlan(d),renderNarrative(d),renderConfig(d),renderFooter(d)]
  .forEach(function(node){if(node)root.appendChild(node);});}
var root=document.getElementById("report-root");
try{var data=JSON.parse(document.getElementById("report-data").textContent);
 if(!data||typeof data!=="object"||Array.isArray(data))throw new Error("bad data");
 render(data,root);}
catch(e){root.innerHTML="";root.appendChild(errorCard());}
})();
</script>
</body>
</html>
<!-- END progress-template v1 -->
```

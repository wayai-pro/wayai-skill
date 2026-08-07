# Native Tools

Tools provided by native connectors. Added to agents by listing them under `tools.native` in `agents/<slug>.yaml` and running `wayai push`. This file mirrors the platform catalog (`packages/core/src/catalog/native-tools.ts` + `native-tool-schemas.ts`) — every tool below exists with exactly this name; a name not listed here does not exist.

All tools belong to the **Wayai** connector (auto-created and enabled with every hub — no setup).

## YAML form

Two forms are accepted under `tools.native`:

```yaml
tools:
  native:
    - update_kanban_status                                     # short form: catalog defaults
    - name: get_tool_schema                                    # object form: per-instance overrides
      keep_in_history: true
    - name: generate_monthly_report
      expose_schema_to_llm: false
```

| Field | Default | Notes |
|-------|---------|-------|
| `name` | required | Native tool name (must match catalog) |
| `id` | set by `wayai pull` | Stable tool_id for rename safety; do not author manually |
| `keep_in_history` | catalog default | When set, controls whether this tool's call/result remain in agent history. Omit to leave the existing value untouched (no diff). |
| `expose_schema_to_llm` | `true` | When `false`, the tool's schema is excluded from the agent's inline tool list (token optimization). Reachable via `execute_tool`. Omit to leave untouched. |

Other native fields (schema, instructions, operation) come from the platform catalog and are not round-trippable via YAML.

## Table of Contents
- [Conversation Tools](#conversation-tools)
- [State Tools](#state-tools)
- [Resource & File Tools](#resource--file-tools)
- [Skill Tools](#skill-tools)
- [History & Summary Tools](#history--summary-tools)
- [Meta Tools](#meta-tools)
- [MCP Tools](#mcp-tools)
- [Quick Reference](#quick-reference)

---

## Conversation Tools

Routing, workflow, and delivery. (Replying with plain text needs no tool — delivery is automatic.)

### transfer_to_agent

Hand off the conversation to another AI agent on the same track (specialist, or back to the entry pilot/copilot as router). The harness reinvokes the target immediately — call it silently (see [roles-and-settings.md](roles-and-settings.md#delegation-flow)).

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `agent_name` | string | Yes | Name of the AI agent to transfer to |

### transfer_to_team

Hand off the conversation to a human support team. The result returns to the caller — confirm to the user *after* the call.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `team_name` | string | Yes | Name of the team to transfer to (a team defined in Hub → Users → Teams; unknown names fail at runtime) |

### consult_agent

Consult another agent for advice without transferring; the advice returns to the caller.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `agent_name` | string | Yes | Name of the AI agent to consult |
| `consult_question` | string | Yes | The specific question to ask |

### delegate_to_hub

Delegate a self-contained request to **another hub**, which handles it as a task conversation. **Asynchronous:** the tool returns as soon as the request is handed over, and the target hub's answer is posted into the consult thread later — the calling agent must end its turn, never wait for the result.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `request` | string | Yes | The task to delegate, phrased so the target hub can act on it without seeing this conversation |

The target hub and the context boundary are **admin configuration on the tool**, not model choices. In YAML:

```yaml
- type: hub
  tool: delegate_to_hub
  target: Billing Hub        # a PRODUCTION (published) hub in the same organization
  context_boundary: summary  # instruction | summary | transcript
```

- **`target`** must be a published (**production**) **`hub_type: task`** hub in the **same organization**. Production matches branching semantics; *task* is a data-isolation requirement — a task hub gives each delegated request its own conversation, whereas a `chat` hub keeps one active conversation per user, so every delegation would land in a single thread and mix customers. A cross-org, deleted, unpublished, or chat-type target fails the push and refuses at runtime rather than spawning.
- **The target hub must be able to CLOSE the spawned conversation** — its close is the completion signal that delivers the answer back. The request text asks it to; a target whose agents cannot close (no `close_conversation`, no terminal kanban status) leaves the asker waiting, with no timeout until H1.7.
- Assign it to a **`consultant`**-role agent: it runs only on a consult turn.
- **`context_boundary`** is the data-crossing control — the only thing that decides what leaves this hub:

| Value | Crosses into the target hub |
|-------|-----------------------------|
| `instruction` | The `request` argument alone — nothing from the conversation |
| `summary` (default when omitted) | The request + the rolling conversation summary |
| `transcript` | The request + the full customer transcript |

  Internal consult traffic is excluded at every setting, and an unrecognized value falls back to `summary` rather than widening.

- Usable **only inside a consult thread** — the thread is where the answer is delivered. Delivery is idempotent (a redelivered result is posted once), and a result arriving after the conversation closed is dropped, with the thread recorded as `cancelled_at_close`.
- **v1 is config-as-code**: the Platform UI's tool "Add" grid omits this tool (it cannot yet collect a target hub or a `context_boundary`); assign it with `wayai push`. An already-assigned instance still shows and toggles in the UI.

### start_consult_thread

Ask the **consultant configured on the tool** a question, in a consult thread the support team can see. The target is admin configuration — the model supplies only the question.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `question` | string | Yes | What you need from the consultant, phrased so it can be answered without further back-and-forth |
| `title` | string | Yes | Short label for the thread, so the team can tell parallel consults apart |

```yaml
# same-hub consultant
- type: agent
  tool: start_consult_thread
  target: Billing Expert          # must be a `consultant`-role agent on this hub

# partner hub as the consultant
- type: hub
  tool: start_consult_thread
  target: Billing Hub             # a PRODUCTION `hub_type: task` hub, same org
  context_boundary: summary       # same meaning and defaults as delegate_to_hub
```

- **Two modes, and the target decides which.** A same-hub, non-harness consultant answers **inside the tool call**. A partner hub — or a **harness-backed** consultant, whose sandbox turn can run minutes — answers **asynchronously**: the calling agent must end its turn (tell the customer you are checking) and is **brought back automatically** when the answer arrives. A sync consult that overruns its turn-time budget converts to async on its own, so an agent must handle "the answer will follow" on any target.
- **Assignable to any non-background agent** (pilot, copilot, specialists). `monitor`, the evaluators, and `summarizer` can never start a consult.
- **Consultant→consultant chains are OFF by default, and SYNC-ONLY when enabled.** A `consultant`-role agent may start a consult only with the explicit opt-in below, and only against a same-hub, non-harness consultant — a chained consult must answer inside the call, because a consultant has no responder track to be brought back on:

```yaml
- type: agent
  tool: start_consult_thread
  target: Tax Specialist
  allow_consultant_chain: true
```

- **Budgets are enforced, not advisory** — every consult turn bills a foreground operation, and an agent can multiply them without a human asking. A consult chain is capped in **depth**, and a **cycle is refused outright** in either direction: agent A → agent B → agent A inside one hub, and hub A → hub B → hub A across hubs. There is also a cap per LLM call and a (durable) cap per conversation. A refusal comes back as a tool error the agent can act on ("answer with what you have").
- **An unanswered consult always resolves.** A pending thread carries an expiry; when it fires the asker is brought back with a timeout notice rather than waiting forever. If the conversation closed in the meantime, nothing is delivered and the thread is recorded as `cancelled_at_close`.
- **The support team can take a thread over at any time.** The moment a team member posts in an agent-started thread, the thread becomes theirs: the AI is no longer brought back for it, and a follow-up consult into that thread returns a fixed "taken over by the team" notice instead of an answer. This is one-way and permanent for that thread.
- **Configured via CI/YAML only in v1** (like `delegate_to_hub`) — the UI "Add" grid omits it until the target picker ships.

### close_conversation

Close the current conversation and mark it resolved. The conversation is archived and leaves the active list.

In a hub that declares a terminal kanban status — **with or without `outcomes`** — this close is executed **as** the terminal kanban transition and is gated exactly like `update_kanban_status`: `allowed_next_statuses` applies always, `from_statuses` when outcomes are configured. An agent that cannot reach the terminal status cannot close either. It takes one optional parameter, `outcome`, required when that status declares `outcomes`; omit it when retrying a close that already recorded one. Only a hub with no terminal status at all leaves the tool unchanged. See [kanban.md](../kanban.md).

### update_kanban_status

Update conversation's kanban board status. The enum is constrained to the hub's agent-updateable status **slugs**; the tool description renders `slug (Display Name)` pairs so the LLM can reason about the choice. Transitions are rejected when the target slug is not configured, the source status is `isTerminalStatus: true` (any outbound transition fails), or the source defines an `allowed_next_statuses` list that excludes the target. These rejections return an error message beginning with `invalid_kanban_transition:`. Unknown-target and `allowed_next_statuses` rejections list the allowed target slugs; a terminal-source rejection names the source and target.

When the target is terminal and declares `outcomes`, `outcome` is required. An outcome with `from_statuses` is eligible only when the conversation's **stored current status** appears in that list; omission means any source. A missing or unknown stored source allows all configured outcomes for compatibility. A same-as-target source (the re-close shape below) does too **for a non-agent REST caller, programmatic clients included**; an agent gets it only when reusing a stored outcome, when the status declares no outcomes, or when the outcome it selected is unrestricted — a *restricted* outcome is never reachable from a source that never entered its `from_statuses`. A missing or ineligible selection returns `invalid_kanban_outcome:` with only the eligible slugs (possibly none), without changing or closing the conversation. **Exception — a re-close:** when the conversation is still open and already sits in the terminal status with an outcome recorded (a close whose auto-close step failed), re-selecting that status with **no** `outcome` reuses the stored one and completes the close; submitting a *different* one returns `outcome_already_recorded` rather than overwriting it. If the hub workflow configuration cannot be loaded, the operation fails with `kanban_config_unavailable` and should be retried later. Transitioning **into** a terminal status with a valid outcome closes the conversation. These gates apply identically to `close_conversation` and the harness `end_conversation` in any hub declaring a terminal status — both are executed AS this transition.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `new_kanban_status` | string | Yes | Target status **slug** (immutable identifier from `hub.kanban_statuses[].slug`; display names appear only as labels in the description) |
| `outcome` | string | When required | Outcome **slug**. Required when the target terminal status declares outcomes; must be eligible from the conversation's stored current status |
| `scheduled_event_date` | string | No | For `isSchedulingStatus` statuses: event datetime, RFC3339 with timezone |
| `event_description` | string | No | Description of the scheduled event |
| `event_sid` | string | No | External event ID for integration purposes |

### schedule_followup

Schedule a custom manual followup at an exact future time. The receiver is determined automatically from conversation status + hub AI mode. Separate from automatic kanban-based followups.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `scheduled_time` | string | Yes | ISO 8601 datetime with timezone; must be in the future |
| `message` | string | Yes | The followup message. May embed AI instructions in brackets: `"Text [AI Context: instructions]"` |

---

## State Tools

Read/write the hub's configured [states](../states.md). All four are keyed by **`state_slug`** — the state's immutable slug, not its display name (the schema enumerates the hub's slugs with display names as labels). Scope (`conversation` vs `user`) is inferred from the state's definition — the agent never passes it.

### get_state

Retrieve a state's current value.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `state_slug` | string | Yes | Slug of the state to read |

### update_state

**Merge** key-value updates into the state's current value (not a full replace).

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `state_slug` | string | Yes | Slug of the state to update |
| `updates` | object | Yes | Key-value pairs merged into the current value |

### set_state_path

Set one path inside the state object without touching the rest.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `state_slug` | string | Yes | Slug of the state to update |
| `path` | array of strings | Yes | Path as an **ordered list of keys** (e.g. `["items", "0", "quantity"]`) |
| `value` | any JSON | Yes | Value written at that path |

### reset_state

Reset a state to its initial value (deletes the persisted row; reads fall back to `initial_value` if defined, else `{}`).

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `state_slug` | string | Yes | Slug of the state to reset |

---

## Resource & File Tools

Discover and read knowledge-resource content, exchange files with the user, and (optionally) edit resource files through the provider's code-execution sandbox. The knowledge resources linked to the agent are injected — as `id (name)` — into the `list_resource_folders` / `list_resource_files` `resource_id` parameter descriptions at turn time, so the agent discovers ids in-band — the hub author no longer needs to hand-write resource UUIDs into the instructions.

### list_resource_folders

List folders in a resource (with `parent_folder_id` for hierarchy).

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `resource_id` | string | Yes | Resource to query |
| `search_query` | string | No | Filter folders by name |
| `metadata_filter` | object | No | MongoDB-style operators (`$eq`, `$ne`, `$gt/$gte/$lt/$lte`, `$in/$nin`, `$contains`, `$and/$or/$not`; nested paths like `"location.city"`) against `folder_metadata_schema` |

### list_resource_files

List files in a resource; returns metadata including the `file_id` used by `read_file`.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `resource_id` | string | Yes | Resource to query |
| `folder_id` | string | No | Filter by folder. Omit = all files; `00000000-0000-0000-0000-000000000000` = root-level only |
| `search_query` | string | No | Search files by title |
| `tags` | array | No | Match files with any of the tags |
| `metadata_filter` | object | No | Same operator syntax as `list_resource_folders`, against `file_metadata_schema` |
| `limit` | integer | No | Max results (default 50, max 100) |
| `offset` | integer | No | Pagination offset |

### read_file

Retrieve a file by ID and inject it into the conversation context for analysis.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `file_id` | string | Yes | From conversation messages or `list_resource_files` |

### send_files

Send files to the end user through their channel (App, WhatsApp, email, …). WhatsApp sends files individually (caption on the last one); email sends all as attachments of one message.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `file_ids` | array | No | File IDs to send (conversation or resource files) |
| `folder_ids` | array | No | Resource folder IDs — sends all files in them (alternative to `file_ids`) |
| `message_text` | string | No | Accompanying text (WhatsApp caption / email body) |

### download_file

Mount a **text** resource file into the provider's code-execution sandbox for editing (Anthropic: `/tmp/<filename>`; OpenAI: `/mnt/data/<filename>` — use the returned `sandbox_path`). Returns `{ sandbox_path, filename, sandbox_file_id }`. Bytes never enter conversation context.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `file_id` | string | Yes | Resource file to mount (binary files rejected) |

### upload_file

Persist a sandbox file back to the resource library. Two modes: **UPDATE** (pass `file_id` of the existing resource file) or **CREATE** (pass `resource_id` + `filename`, optional `folder_id`). Requires `write_enabled` on the agent↔resource binding.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `sandbox_file_id` | string | Yes | Provider-specific ID of the file inside the sandbox |
| `file_id` | string | UPDATE mode | Existing resource file to overwrite |
| `resource_id` | string | CREATE mode | Resource the new file belongs to |
| `filename` | string | CREATE mode | Filename (MIME inferred from extension; text only) |
| `folder_id` | string | No | CREATE mode: target folder (omit for root) |

---

## Skill Tools

Read [skill-resource](../resources.md#skill-resources) content on demand (tool-based execution mode). `skill_id` = the skill's `resource_id`; the ids of the skills linked to the agent are injected into the `skill_id` parameter description at turn time, so the agent can discover them without any hand-written ids. (The handler also resolves a skill by name.)

### read_skill

Read a skill's `SKILL.md`.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `skill_id` | string | Yes | The skill's resource_id |

### read_skill_file

Read a file inside a skill by relative path (no need to call `read_skill` first if the path is known).

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `skill_id` | string | Yes | The skill's resource_id |
| `file_path` | string | Yes | Relative path from skill root (e.g. `references/menu.md`) |

---

## History & Summary Tools

### get_conversations_summary

Timeline of the current user's past conversations (summaries + conversation IDs).

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `max_conversations` | integer | No | Max past conversations (default 10) |
| `max_characters` | integer | No | Output cap (default 20000, max 40000; truncates oldest first) |

### get_conversation

Full transcript of one past conversation (find IDs via `get_conversations_summary`).

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `conversation_id` | string | Yes | Conversation to retrieve |
| `max_characters` | integer | No | Output cap (default 30000, max 60000) |

Also returns `previous_conversation_id` / `next_conversation_id` when they exist — the user's conversations closed nearest in time either side of this one, so the agent can keep walking history without another `get_conversations_summary` call. Neighbours by closing time, not a fixed chain; either may be absent.

### expand_summary

Retrieve the original messages behind a section of the rolling `<conversation_summary>` block (see the summarizer role in SKILL.md → Agent Roles). An unknown `section_id` returns `available_section_ids` for retry.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `section_id` | string | Yes | Section `id` from the `<conversation_summary>` block |

---

## Meta Tools

Tools for meta-level tool management and execution (part of the Wayai connector).

### get_tool_schema

Retrieve schema for a specific tool.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `tool_name` | string | Yes | Name of tool to get schema for |

### execute_tool

Execute a tool with parameters.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `tool_name` | string | Yes | Name of tool to execute |
| `parameters` | object | Yes | Tool parameters |

> Note: Cannot execute orchestration tools recursively.

### Usage requirements

- **Hide the target tool from the inline list:** set `expose_schema_to_llm: false` on the deferred tool (custom or native — see the YAML form section above for native tools). This is a prompt-token optimization — it keeps the schema out of the inline tool list. It is **not** an access control: an agent with `execute_tool` assigned can still invoke the tool by name.
- **Assign both meta tools to the agent** — `get_tool_schema` to fetch the schema, `execute_tool` to run it. Co-assignment is the intended pattern but is not enforced. The lookup is agent-scoped: meta tools only resolve tools assigned to the calling agent.
- **`keep_in_history` AND-combines across both meta tools.** For both `get_tool_schema` and `execute_tool`, the target tool's `keep_in_history` is AND-combined with the meta tool's own flag — **both must be `true`** for the call/result to remain in history. Both meta tools default to `keep_in_history: false`. Override per-instance via the YAML object form (see [YAML form](#yaml-form)).

---

## MCP Tools

**Connector:** MCP Server (`type: Tool`, `service: MCP Server`; supports Bearer Token and OAuth authentication)

An agent uses an external MCP server's tools by assigning them individually — there is **no dynamic "discover and call anything at runtime" path**. Each assigned tool is a reviewed, per-agent allowlist entry (the static allowlist keeps third-party tools least-privilege; the server's tool descriptions are wrapped as untrusted input).

### Assigning MCP tools via GitOps (`tools.mcp`)

Declare MCP tools under an agent's `tools:` map. The reference is `(name, connection)` — an MCP tool name is unique per server:

```yaml
# agents/<agent>.yaml
tools:
  mcp:
    - name: search_documents       # the discovered MCP tool's name
      connection: My MCP Server    # the MCP Server connection's display name
```

- **`wayai push` discovers + assigns in one run.** If the named tool isn't in the hub's discovered catalog yet, push runs discovery against the server (equivalent to the UI's "Sync MCP tools" action), then assigns it — so a single push can create the connection (declared in `hub.yaml`) and assign its tools. **The MCP server must be reachable**; if it isn't, push fails with a clear, retryable error.
- **`wayai sync-mcp --connection <name>` re-syncs an existing connection's catalog.** `push` only discovers tools *new* to the catalog — it does **not** refresh the cached schema of tools already catalogued. When the server changes a tool's input schema (adds/renames params), run `wayai sync-mcp --connection "My MCP Server"` to re-discover and refresh the stored catalog (the `--connection` value is the display name or the connection UUID). Preview-only. This keeps the GitOps catalog (what `pull` exports and the diff compares) accurate — and the catalog doubles as the runtime fallback: at turn time the schema is resolved live from the server on a best-effort basis, but when the server is slower than the turn-time budget, unreachable, or the connection's credential fails, the turn falls back to the last synced snapshot. A fresh catalog is what makes that fallback accurate.
- **Drift is now surfaced, not silent (you no longer have to already know to run `sync-mcp`).** `wayai push` re-verifies every referenced MCP connection whose tools were all already catalogued and **warns** when the cached schema is behind the server (naming the connection and the changed/removed tools) — best-effort, so a slow/unreachable server never blocks the push. And **`wayai sync-mcp --connection <name> --check`** (alias `--dry-run`) reports that drift **without writing anything**, exiting non-zero when the catalog is behind — run it in CI to gate on a stale catalog. `--check` is read-only and works against production hubs too.
- **Omitted vs present:** when the `mcp` key is **omitted**, existing MCP tool rows are left untouched (tools assigned in the Platform UI survive). When **present** (even `[]`), the list is **authoritative** — unlisted MCP tools are removed. (This differs from `native`/`custom`, whose absence deletes them, because MCP tools are dual-origin: UI + GitOps. Same rule as `evaluation_variables`.)
- **`wayai pull`** emits a `tools.mcp` block for the agent's MCP tools (`id` / `connection_id` included for stable round-trips). The tool schema and description stay derived from the connection's discovered catalog — they are not round-tripped.

### Assigning MCP tools via the Platform UI

hub → **Connections** → set up the MCP Server connection → **Sync MCP tools** → hub → **Agents** → the agent → **Add tool** → pick the MCP tool. Equivalent to `tools.mcp`; the next `wayai pull` materializes these as YAML.

**Refreshing a changed catalog (UI):** click **Sync MCP tools** on the connection again to re-discover and refresh the stored tool schemas after the server changes them — the UI equivalent of `wayai sync-mcp`. Use this (or the CLI command) whenever an MCP server adds/renames a tool's params; `wayai push` alone won't refresh an already-catalogued tool's schema.

> Connection-level sync/discovery (the server's live tool list) is handled by the connection, not by agent-assignable tools. There are no `mcp_discover_tools` / `mcp_refresh` agent tools.

---

## Quick Reference

| Module | Tools |
|--------|-------|
| Conversation | `transfer_to_agent`, `transfer_to_team`, `consult_agent`, `delegate_to_hub`, `start_consult_thread`, `close_conversation`, `update_kanban_status`, `schedule_followup` |
| State | `get_state`, `update_state`, `set_state_path`, `reset_state` |
| Resource & File | `list_resource_folders`, `list_resource_files`, `read_file`, `send_files`, `download_file`, `upload_file` |
| Skill | `read_skill`, `read_skill_file` |
| History & Summary | `get_conversations_summary`, `get_conversation`, `expand_summary` |
| Meta | `get_tool_schema`, `execute_tool` |
| MCP | per-agent tools assigned via `tools.mcp` (YAML) or the Platform UI |

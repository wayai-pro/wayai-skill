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
- [Skill & File Tools](#skill--file-tools)
- [Code Sandbox Tools](#code-sandbox-tools)
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

Consult another agent for advice without transferring; the advice returns to the caller. An advisor on a hosted agent connection is refused before it is asked: advisors run only on model connections ([roles-and-settings.md → Agent Settings](roles-and-settings.md#agent-settings)).

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

- **Two modes, and the target decides which.** A same-hub consultant answers **inside the tool call**. A partner hub answers **asynchronously**: the calling agent must end its turn (tell the customer you are checking) and is **brought back automatically** when the answer arrives. A sync consult that overruns its turn-time budget converts to async on its own, so an agent must handle "the answer will follow" on any target. A consultant on a hosted agent connection, or with no connection, is refused before anything is sent — consultants run only on model connections ([roles-and-settings.md → Agent Settings](roles-and-settings.md#agent-settings)).
- **Assignable to any non-background agent** (pilot, copilot, specialists). `monitor`, the evaluators, and `summarizer` can never start a consult.
- **Consultant→consultant chains are OFF by default, and SYNC-ONLY when enabled.** A `consultant`-role agent may start a consult only with the explicit opt-in below, and only against a same-hub consultant — a chained consult must answer inside the call, because a consultant has no responder track to be brought back on:

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
| `scheduled_event_date` | string | Conditional | Event datetime, RFC3339 with timezone. **Required** when the target is an `isSchedulingStatus` status configuring any event-relative followup (`before_event`, `inactivity_after_event`, `inactivity_after_before_event`) — the transition is rejected without it, since those followups have nothing to count from. Optional on any other status |
| `event_description` | string | No | Description of the scheduled event |
| `event_sid` | string | No | External event ID for integration purposes |

### schedule_followup

Schedule a custom manual followup at an exact future time. The receiver is determined automatically from conversation status + hub AI mode. Separate from automatic kanban-based followups.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `scheduled_time` | string | Yes | ISO 8601 datetime with timezone; must be in the future |
| `message` | string | Yes | The followup message. May embed AI instructions in brackets: `"Text [AI Context: instructions]"` |

### run_monitor

Run another monitor on this hub, and let it act on its own rules. Selected by a monitor's rule — see [rules and fallback](roles-and-settings.md#rules-and-fallback).

**Only a `monitor` agent can hold this tool**, and **no model is ever offered it**. The monitor it names must be one whose trigger is `manual`, so that nothing else ever fires it.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `monitor_name` | string | Yes | The name of the monitor to run. Must be a `const` you write — it chooses which monitor runs, so it cannot come from a variable. The named monitor must be on this hub, enabled, and have `trigger: manual`. |

### insert_note

Brief the agent that is about to reply, with a note **you** write. Selected by a monitor's rule — see [rules and fallback](roles-and-settings.md#rules-and-fallback).

A `manual` monitor can insert one too, when the cascade that reached it started on a customer's message. If it started anywhere else there is no reply left to brief, so the note is skipped and the reason recorded.

**Only a `monitor` agent can hold this tool**, and **no model is ever offered it**: a rule selects it, and the note is inserted into the answering agent's context for that one reply. It is the only tool in this file that works that way, so the usual "assign it and the model may call it" reading does not apply.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `template` | string | Yes | The note text. Must be a `const` you write — it cannot come from a variable. Write `{{variable_name}}` inside it to substitute the monitor's own variables. |
| `keep_in_history` | boolean | No | Default `false` — the note briefs this reply and is stored nowhere. `true` also records it as an internal message the support team can read and the customer never receives. |

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

## Skill & File Tools

Discover and read skill content and the documents of the agent's toolset, exchange files with the user, and write those documents under `data/` (`write_file`, `edit_file`). Skills are read-only to agents. `list_files` is the one discovery tool: the mounts an agent may browse are injected into its `path` parameter description at turn time, so it discovers what exists in-band — the hub author no longer needs to hand-write skill UUIDs into the instructions.

**Files are addressed by path.** Every skill file has a canonical path, `skills/<skill-slug>/<folders…>/<file-name>`, where the slug is the skill's `skill_name`. `read_file` takes that path, `list_files` returns it on every row, and `{{files('tree')}}` lists it under each skill. Knowledge resources, and the `resources/` mount they were read at, are retired: an organization's documents are read under `data/` (below). Prefer paths in instructions: publishing a hub regenerates every UUID, so an id written into a prompt breaks at preview → production, while the names a path is built from clone verbatim. `file_id` is still accepted everywhere.

**Documents of the agent's toolset are addressed under `data/`**, as `data/<file-type>/<path>`, where `<path>` is the file's full path in its [file type](../bases/files.md): `data/policies/eu/refunds.md`. An agent gets this mount from a [Base Toolset](../connections.md#base-toolset) connection when it holds both `list_files` and `read_file` as its own tools: each file type its assigned toolset tools can `list` or `get` is then a `data/<file-type>` mount, and `list_files` and `read_file` call those tools for it. What the toolset doesn't expose isn't in the mount: a tool reaches only its own file type and folder, and a toolset that lists only `public/` shows nothing outside it. Such an agent isn't also offered the toolset's `list` and `get` tools the mount covers, so the model reads documents one way. An agent that also holds both `write_file` and `edit_file` writes its documents through the mount too: a file type it holds a toolset `put` tool for is then mounted like any other, through its `list` or `get` tools, the `put` tools on that file type's base are left out of the agent's tools with them, and the agent writes by `data/` path (see [`write_file`](#write_file) and [`edit_file`](#edit_file)). Without both, such a file type isn't mounted, and its toolset tools stay as they are, because they take the toolset's own paths. A file type the agent holds a toolset `delete` tool for is never mounted, since the mount doesn't delete: its `list`, `get`, `put` and `delete` stay as they are. A `put` outside every mount stays offered as itself too: one for a file type the agent can't `list` or `get`, and one on a connection to another base than the mount's. A file type the agent reaches on several bases is mounted from one: the first connection's, by connection id, among those it can `list` or `get` the file type on. A toolset's own description of a mounted file type's tools (`description_override`) is announced beside its mount in `list_files`' mount list, and so is whether the agent can write it, whole or in some of its folders. Instructions that name the toolset's `list` or `get` tools for a mounted file type should name its `data/` paths instead: the agent reads them with `list_files` and `read_file`. An agent holding only one of `list_files` and `read_file` has no `data/` mount and keeps the toolset's tools exactly as they were, because the mount's listings name paths only `read_file` opens. The `read_file` that `metadata_only` file handling adds doesn't count: assign `read_file` to the agent. A `data/` path is always written in full and matched exactly, case included: file types and paths in a base are case-sensitive. A `data/<name>` that names no file type of the mount is resolved as before, so a shortened skill path such as `data/menu.md` (a skill folder named `data`) still works. A binary or large document (a Word or Excel file, a PDF, an image) is edited in the [code sandbox](#code-sandbox-tools) instead: `copy_to_sandbox` copies it in by its `data/` path, and `copy_from_sandbox` copies it back, with the same version check.

**Files attached to the conversation are addressed the same way**, under `conversation/<file-name>` — the address the transcript itself announces. Every message a USER or a TEAM member sent with attachments is annotated with `[Attached files: conversation/report.pdf (Type: application/pdf, Size: 2048 bytes)]`, so the agent learns what arrived without a listing call, and passes the path straight to `read_file` or `send_files`. Earlier files are announced, never re-sent as content (see *File handling* in `roles-and-settings.md`); `read_file` returns the content, and for an image or a PDF, the file itself. Duplicate names take the same `-2`/`-3` ordinals as skills — but unlike a skill ordinal they are never promoted, because a hidden file keeps its own reserved (see the consult paragraph below).

A file the AGENT sent with `send_files` is addressable by the same path but is **not** announced in history, so an agent cannot tell from the transcript alone whether it already sent a document — instruct it from its own tool results instead.

**Internal consult attachments are not addressable, on any turn.** A file attached inside a consult thread — a team member's private question to a consultant, an advisor's answer — is hidden from every agent-facing read, in this conversation and in an earlier one: it is never announced, never resolves by path, bare name or file ID, and never appears in a candidate list. The team still sees the file in the chat thread and the archive keeps every row.

That holds for the consultant's own turn too, which is deliberate: a consultant's answer is returned verbatim to whoever opened the consult, and an *agent* can open one, so a file read on a consult turn could come back to the customer as text. **This does cost one flow.** Attaching a file to an `@consultant` question and naming it in the text used to let the consultant open it by name — a consultant's context is text-only and never carries a file's path (see `roles-and-settings.md`), so the name a human typed was the only route, and it is now closed: the consultant gets "not found" and should be instructed to ask the team to paste the relevant content. Announcing consult-thread attachments in the consultant's own context, so that flow works properly, is a pending follow-up.

A hidden file still keeps its alias **reserved**: a conversation holding a consult `receipt.pdf` and a visible one announces the visible one as `conversation/receipt-2.pdf`, and `conversation/receipt.pdf` resolves for nobody. That is what keeps an address the transcript already announced pointing at the same file forever — so on the `conversation/` and `conversations/` mounts an ordinal is *not* the promotable rank it is for skills.

**Files from an earlier conversation with the same user** are addressed as `conversations/<conversation_id>/<file-name>` — plural mount, and the id is the one `get_conversations_summary` lists. The archived transcript announces them itself: each message that carried attachments is followed by an indented `[files: conversations/<id>/receipt.pdf]` line, and anything attached to a message the transcript does not show (a tool step, or one older than the 50-message cap) is listed under a trailing `Other files in this conversation:` section. Both lists are capped, with the remainder counted (`… and 12 more files`) rather than dropped. `read_file` and `send_files` take those paths directly, in full — this mount accepts no shortened form.

Only the current end user's own conversations resolve. Any other id — another user's, or one that never existed — reads as "not in this user's history"; the two are deliberately indistinguishable, so the namespace cannot be used to test whether a conversation id is real. A conversation of the user's own whose files have aged out says so separately.

`get_conversations_summary` marks a conversation holding files as `[3 files]` — counted on the same basis, so the number matches what the transcript announces. Transcripts are written once, at close, so conversations archived before this shipped announce nothing — their files remain reachable by path if the agent knows the name.

**Keep skill names — and file names within one — unique if you hard-wire paths.** Duplicates are allowed, and they get `-2`/`-3` suffixes (`support`, `support-2`), but a suffix is a *rank within the duplicate set*, not an identity: renaming or deleting the first "Support" promotes the second one to `support`, and publishing re-derives the ranks from scratch. There is nothing stable to key on — two skills with the same name are indistinguishable after a clone — so a hard-wired `…-2` path can silently come to mean a different file. Unique names have no suffix and no such exposure.

**How a path resolves.** With the `skills/` mount, the next segment is the *skill selector*: `skills/order-processing/references/menu.md` is looked up inside the skill `order-processing` and nowhere else, so another skill that happens to contain an `order-processing/` folder can never answer for it, and a skill is never found under `resources/`. A `skills/<name>/…` whose `<name>` is no skill the agent can read is resolved as a mount-less path, as before, so a `skills/` folder inside a skill keeps its short address (`skills/matrix.md`). A mounted path must be complete — you named the skill *and* the path, so it matches exactly (case-insensitively), and a shortened one reports not-found rather than guessing at a nested file you did not name.

Shortening is what the mount-less form is for. There the reference is matched across everything the agent can read — every linked skill **and** this conversation's attachments — in three tiers: exact path, then case-insensitively, then as a path *suffix*. The strictest tier that matches any file wins, so a file whose path **is** `notes.md` beats one at `archive/notes.md`. You only get the candidate list when several files match within the *same* tier. Dropping a *middle* segment still resolves nothing in either form: `order-processing/menu.md` will not find `skills/order-processing/references/menu.md`.

No namespace outranks another: a bare `report.pdf` that names both a skill file and something the user attached is reported as ambiguous, listing both full paths, rather than one quietly answering for the other. Name the mount (`conversation/report.pdf`) when both exist.

### list_files

Browse what the agent can reach, by path. Discovery is a drill-down: no `path` lists the mounts, a mount lists its top level, a folder path lists that folder.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `path` | string | No | What to list. Omit (or `""`) = the mounts. `conversation` = files attached to this conversation. `skills` = every skill mount. `skills/<slug>` = that skill's top level; `skills/<slug>/<folder>/<subfolder>` = that folder. `data` = the document mounts; `data/<file-type>` or `data/<file-type>/<folder>` = every document beneath it (see below) |
| `query` | string | No | Search files by title or file name. Under a skill or `data/` path this searches the **whole subtree**, not just the level named |
| `tags` | array | No | Match files with any of the tags (skill and `data/` files). Searches the subtree, like `query` |
| `metadata_filter` | object | No | MongoDB-style operators (`$eq`, `$ne`, `$gt/$gte/$lt/$lte`, `$in/$nin`, `$contains`, `$and/$or/$not`; nested paths like `"location.city"`; a bare value means equality) against a file's metadata; under `data/`, the file type's metadata schema refuses a field it does not declare. Skill and `data/` files; searches the subtree |
| `limit` | integer | No | Max rows (default 50, max 100) |
| `offset` | integer | No | Pagination offset |

**Filters need a skill or a file type.** A filter passed with no path — or with the bare `skills` or `data` path — is refused rather than ignored: mounts are not files, so answering with the unfiltered mount list would read as "these are your matches". Search one skill or file type at a time (`skills/<slug>` or `data/<file-type>` plus the filter). On the `conversation` mount `query` works and `tags` / `metadata_filter` are refused, because an attachment carries neither. A `metadata_filter` the tool cannot evaluate is **reported, not applied**: an unknown `$operator`, a `$and`/`$or` that is not a non-empty array of objects, and a key naming a prototype member (`constructor`, `__proto__`, `prototype`) each come back as an error naming the offending token. That is deliberate — an empty page would be indistinguishable from a genuine no-match, so the agent would tell the user a file is absent from a skill that holds it. (Underneath, the shared predicate still matches nothing rather than everything for such a filter; the error is what makes the refusal legible.)

**Path grammar.** The mount is the first segment and it decides the store: `conversation` is this conversation's attachments, `skills/<slug>` is one skill, `data/<file-type>` is one file type of the toolset's documents. Everything after `skills/<slug>` is the folder chain, matched case-insensitively. A `data/` path is matched exactly, case included, as the base stores it. Leading `./` and `/` are accepted.

**Document mounts (`data/`).** A `data/<file-type>` listing goes through the toolset's own `list` tool, which searches a whole folder tree: a folder lists every file beneath it, a page at a time. Each row is the toolset's own file row, whole — its `version`, `content_type`, `size`, `metadata`, and who changed it and when — marked `type: "file"`, with `path` as its full `data/` path. `query`, `tags` and `metadata_filter` search the same tree, as the toolset's `list` does: `query` matches the file name or the metadata `title`, and a `metadata_filter` naming a field the file type doesn't declare is refused. The response is `{ path, entries, limit, offset, has_more }`: there is no `total`, and `has_more` is true when the page is full, so page on with `offset`. A file type the toolset lists through one tool is always listed through it, and the toolset decides which folders that tool reaches. A file type it lists through several tools, one per folder, routes each folder to the tool whose folder is the deepest holding it, and a folder above all of theirs — `data/policies` of a toolset that lists `public/` and `faq/` — lists the folders leading down to them instead. A folder the agent can read but not list says so. `data/` rows are for `read_file`; `send_files` doesn't take them.

**Mount semantics.** The mounts are always `conversation` plus one `skills/<slug>` per **enabled** skill linked to the agent, and one `data/<file-type>` per file type of its toolset documents. Disabling a skill withdraws its mount even while the agent link stands, matching what `{{files()}}` and the injected mount list announce. `conversation` is listed even when it holds nothing, because it is a real mount; its row carries `file_count` (absent, not `0`, when the count could not be read). Consult-thread attachments are excluded here exactly as they are from `read_file` (see above). There is deliberately **no `conversations/` (plural) mount**: a past conversation's files are announced by its own archived transcript and counted by `get_conversations_summary`, so they are addressable without being browsable.

**Response.** `{ path, entries, total, limit, offset }`, where `path` is the resolved location and `total` is the count the page was drawn from. Each entry is a `mount`, `folder`, or `file` row carrying its own `path`; `folder` rows carry the folder's `description`, and `file` rows carry `file_id` alongside the path, because `read_file` and `send_files` also take an id. A `data/` listing differs as described above: `has_more` in place of `total`, and each row the toolset's own.

**Errors are directional, with one deliberate exception.** A path naming a file rather than a folder points at `read_file`. A `skills/<slug>` that is unlinked, disabled, or simply nonexistent all report the SAME "no such path" and point back at the mounts — indistinguishable on purpose, because a slug is a guessable name, and telling the two apart would let anyone who can steer an agent's tool calls enumerate which skills a hub holds but withheld from that agent. The "not linked" diagnostic survives only where a skill is named in a parameter (`read_file`'s `skill_id`, which takes a linked skill's id or slug), and there it reads the same for an unlinked one and one that does not exist. **Admin note:** if an agent reports "no such path" for a skill you can see in the UI, check that the skill is linked to that agent and that the skill is enabled — the tool cannot distinguish those for you.

### read_file

Retrieve a file and inject it into the conversation context for analysis. Text files come back inline; images and PDFs are attached to the same turn.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `path` | string | No\* | Canonical path — `skills/order-processing/references/menu.md` for a skill file, `conversation/report.pdf` for something attached to this conversation, or any unambiguous shortening of these (`references/menu.md`, `menu.md`). `conversations/<conversation_id>/receipt.pdf` addresses an earlier conversation's file and takes **no** shortening — mount, id and name in full. `data/<file-type>/<path>` addresses a toolset document, in full |
| `file_id` | string | No\* | From conversation messages or `list_files`. Prefer `path`: ids are regenerated at publish |
| `skill_id` | string | No | Restricts a path to one skill, named by its id or its slug (`order-processing`, or `skills/order-processing`) — use when the same file name exists in several skills and the reference comes back ambiguous. Refused beside a `data/` path |
| `view_range` | array | No | `[start_line, end_line]` of a text file: only those lines come back, counted from 1 with both included; `end_line` -1 reads to the end. A range starting past the last line is refused, naming how many lines there are; one ending past it stops at the last line. Ignored for a file that is not text |

Inline text from `skills/` and the conversations is cut at 60,000 characters, with a notice; read on with `view_range`. The text comes framed as `--- File: <name> ---`, and the frame says which lines a `view_range` returned (`lines 10-40 of 120`).

**A `data/` document** is read through the toolset's own `get` tool, and `read_file` answers that tool's own answer, whole and uncut, with `path` as its `data/` path: `{ path, version, content_type, size, metadata, …, encoding, content }`. The **version** is the one a version-checked write of the file names. Text comes back as UTF-8 and `view_range` selects its lines, adding `lines` (`lines 10-40 of 120`). A document that is not text (a PDF, an image) comes back as base64 (`encoding: "base64"`); it is not shown to the model as an image or a document, and `view_range` doesn't apply to it. The toolset's `get` carries at most 1 MiB, and refuses a larger file. A file type the toolset reads through one tool is always read through it, and the toolset decides that tool's reach; with several tools, one per folder, a file outside all of their folders reports "File not found". A file the agent can list but not read says so.

\* At least one of `path` / `file_id` is required; `path` wins when both are given. `skill_id` narrows only a path, and it is checked after a skill file's `file_id` but before a conversation attachment's: a skill file the agent can read, addressed by `file_id`, is never refused because of `skill_id`, while an attachment addressed by `file_id` is refused when `skill_id` names no skill the agent is linked to — omit `skill_id` for an attachment. An unknown reference reports "File not found" with the discovery hint. "Access denied" means one of two things: a `file_id` whose file sits in a skill this agent is not linked to, or a `skill_id` naming no skill the agent is linked to — the latter reads identically whether or not that skill exists, by id or by slug, so it never confirms one does.

### write_file

Create a document of the agent's toolset under `data/`, or replace one whole, through the toolset's own `put` tool. Only `data/<file-type>/<path>` files the agent writes through the mount are accepted (see the `data/` paragraph above); everything else — skills under `skills/`, the conversation's files and earlier conversations' — is read-only, and refused before anything is called.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `path` | string | Yes | The document's full `data/` path, as `list_files` gives it: `data/policies/eu/refunds.md` |
| `content` | string | Yes | The whole file, as text: at most 60,000 characters |
| `version` | integer | No | The version the agent read (from `read_file` or a `list_files` row), to replace that file. Omit it to create a new file |

**Without `version` it only creates:** a path that is already taken is refused, and the file there stays as it was. **With `version` it replaces that version only:** it reads the file first through the toolset's `get` (so the agent must be able to read it), and refuses, changing nothing, a file that isn't text or that has moved past that version since it was read (`changed since you read it: it is at version 4, not 3`); a write that lands between that read and this one is refused by the toolset's own version check. Either way the agent reads the file again and redoes its change on what it holds now. A version never repeats at a path, so a version read from a deleted file never matches the file created there later. A version counts content: a write of the content the file already holds keeps its version. The answer is the toolset's own row for the file as the write left it, with `path` as its `data/` path: `{ path, version, content_type, size, metadata, … }`, never its content. Its `version` is the one the next write of the file names. A new file's content type comes from its extension (`text/markdown` for `.md`), and a replaced file keeps its own, and its metadata. Text only: a file that isn't text is never replaced, and text over 60,000 characters isn't written. The toolset's own limits apply as to any call of its tools: which file types and folders its `put` reaches, and whom it serves.

### edit_file

Replace text in a document of the agent's toolset under `data/`: it reads the file through the toolset's `get` tool, replaces the text, and writes the result whole through its `put` tool, naming the version the agent read. The same files as `write_file` are accepted, and the agent must also be able to read the file.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `path` | string | Yes | The document's full `data/` path |
| `old_str` | string | Yes | The exact text to replace, whitespace and line breaks included |
| `new_str` | string | Yes | Its replacement; `""` deletes `old_str` |
| `version` | integer | Yes | The version `read_file` gave for the file: the edit is made on it |
| `replace_all` | boolean | No | Replace every occurrence of `old_str` rather than exactly one. Default `false` |

`old_str` must occur exactly once unless `replace_all` is set: zero matches, or several, are refused without a write, naming how many. A file read at another version than `version` is refused before any write, and a write that lands between the read and the write is refused by the toolset's version check, so an edit is never made on text the agent didn't read. `new_str` is inserted as written: `$&` and `$1` are text. The answer is `write_file`'s, plus `replacements`, how many were made. Text files only, of at most 60,000 characters before and after the edit.

### send_files

Send files to the end user through their channel (App, WhatsApp, email, …). WhatsApp sends files individually (caption on the last one); email sends all as attachments of one message.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `file_ids` | array | No | Files to send, each named by path (`conversation/report.pdf`, `conversations/<conversation_id>/receipt.pdf`, `skills/order-processing/price-list.pdf`) or by file ID |
| `folder_ids` | array | No | Skill folder IDs — sends all files in them (alternative to `file_ids`) |
| `message_text` | string | No | Accompanying text (WhatsApp caption / email body) |

---

## Code Sandbox Tools

Run code in a sandbox: a disposable Linux container (Debian, with Python 3, curl, jq, zip and other common tools), working directory `/workspace`. By default it is the conversation's, and its files are visible to every agent of the conversation that shares it; an agent whose `sandbox_scope` is `agent` has its own, which no other agent sees ([Roles & Settings › Code sandbox](roles-and-settings.md#code-sandbox-all-llm-connectors)). It is the only place a native agent runs code or edits a binary or large file, on every LLM connector: copy a file in, change it with a command, and copy the result out — into the conversation, or back to the document it came from.

Python comes with libraries for the files customers send, already installed, so they work with no network:
- **Spreadsheets:** `openpyxl` reads and edits `.xlsx` workbooks.
- **Word and PowerPoint:** `python-docx` and `python-pptx` read and edit `.docx` and `.pptx` files.
- **PDFs:** `pypdf` reads, splits, merges, rotates, fills and extracts text from them.
- **Images:** `Pillow` opens, converts, resizes and edits PNG, JPEG, WebP, GIF, TIFF and BMP.
- **Tables:** `pandas` reads and writes CSV and `.xlsx`, and filters, joins and aggregates them.

A sandbox whose network setting reaches a package index can `pip install` more: `pip install --user`, or a virtual environment under `/workspace`, is kept across the sandbox's stops (below), while a plain `pip install` is not. (A skill run with native integration on Anthropic or OpenAI runs in that provider's own skill container instead; see [Skill Execution Modes](../skills.md#skill-execution-modes).) These tools run only for an agent whose `code_execution_enabled` setting is on, on any LLM connector, held by a pilot- or copilot-track responder (`pilot`, `pilot_specialist`, `copilot`, `copilot_specialist`), and the sandbox reaches the network only as the agent's `sandbox_network` setting allows (see [Agent Settings › Code sandbox](roles-and-settings.md#code-sandbox-all-llm-connectors)). Consult turns cannot use them. A sandbox keeps one network setting for its life: a command or a copy in by an agent with a different one starts a new sandbox, with the `/workspace` files kept but nothing that was running, while `copy_from_sandbox` reads the running one whatever the setting.

### bash

Run a Bash command in the sandbox. The sandbox starts on the first command and keeps its files between commands in the conversation; it ends after 10 minutes without use, 60 minutes in all, or when the conversation closes, and the next command then starts a new one (its result says so).

**What a new sandbox keeps.** The files under `/workspace` are saved at the end of each turn that ran a command or copied a file in (the reply never waits for it), and again when the sandbox ends after 10 minutes without use, at 60 minutes, or for a new network setting, if anything changed since; the next sandbox of the same scope starts with them: restoring them, like starting the sandbox, takes time from the first command's own limit, and a command left with none does not run (its tool error says so, whether the new sandbox has the saved files or is empty, and to run it again). Only those files are kept: nothing that was running, and nothing outside `/workspace`. So a long job writes its progress to files under `/workspace`, and the agent picks the work up from them in a new sandbox. Commands run with `HOME` set to `/workspace`, and with pip's user base and npm's global prefix at `/workspace/.local` (its `bin` on the `PATH`, after the system's own programs), so dotfiles and those installs are kept with the other files; download caches stay outside `/workspace` and are not. A workspace of more than 512 MiB of files, or more than 100,000 files and folders, is saved without `/workspace/.local` (the installs) when that brings it within those bounds; otherwise it is not kept, and the next sandbox starts empty. A sandbox stopped without saving (a command outlived its limit, or the sandbox stopped on its own) keeps the last save, usually made at the end of the last turn that ran a command: the next sandbox starts with those files, and what changed since is lost. An agent's own sandbox (`sandbox_scope: agent`) gets back only its own files. Saved files are deleted when the conversation closes or is deleted, whenever the conversation's own files are deleted, and at the latest 30 days after their last save. A command, with everything it starts, can run at most 512 processes and threads at once: past that, starting another fails, so a runaway command (a fork bomb, say) stops there and the next command runs normally. Nothing a command starts outlives it: when it returns, everything it left running ends with it, a background process, a `setsid` one or one moved elsewhere alike. So a server and the requests that use it go in one command, and a file is the way to hand data to the next.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `command` | string | Yes | The script to run, as with `bash -c`, in `/workspace` |
| `timeout_seconds` | integer | No | Time limit: 120 by default, at most 540; never past the turn's own time limit |

**Response.** `{ exit_code, timed_out, timeout_seconds, stdout, stderr, duration_ms }`. Each stream keeps its first and last parts; `stdout_omitted_bytes` / `stderr_omitted_bytes` say how much was cut. `new_sandbox` appears when the command started a new sandbox, and says whether it started empty or with the saved `/workspace` files, and then what came back: how many files and folders, their size, the workspace's first 20 top-level names (and how many more), what the restore left out, and that nothing earlier calls started is running; `stopped` (with `exit_code: null`) appears when a command outlived its limit and the sandbox was stopped (the next command's `new_sandbox` says whether its new sandbox has the saved files). A sandbox that cannot run the command (the organization runs as many sandboxes as it may, the sandbox could not start, the network setting is invalid, or it could not limit the command's processes) returns a tool error saying so, and the command does not run.

### copy_to_sandbox

Copy one of the conversation's files, or one of the agent's documents, into the sandbox, so commands can read or edit it there. WayAI copies the file; the sandbox needs no network for it, and never reaches where documents are stored. The files are those `read_file` opens from the conversation's own files: this conversation's attachments and an earlier conversation's of the same user (never a consult attachment); and the documents of the agent's `data/` mount (see the `data/` paragraph under [Skill & File Tools](#skill--file-tools)), read through the toolset's own `get` tool, so the toolset's limits apply. Skill files cannot be copied. A file larger than 25 MiB is refused, and a document larger than 1 MiB.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `path` | string | Yes | The file, as `read_file` names it: `conversation/<name>` (a shorter unambiguous form or the file's id also works), `conversations/<conversation_id>/<name>`, or a document's full path `data/<file-type>/<path>` |
| `sandbox_path` | string | No | Where the copy lands, under `/workspace` (a relative path is taken from `/workspace`); defaults to the file's name in `/workspace`. Replaces any file there. A path through a symbolic link is refused, whatever the link points at |

**Response.** `{ sandbox_path, bytes }`, with `new_sandbox` when the copy started a new sandbox (as for `bash`). A document's copy adds `path`, its `data/` path, and `version`, the version copied: `copy_from_sandbox` names both to copy the edited document back.

### copy_from_sandbox

Copy a file out of the sandbox into the conversation, so what commands produced (a chart, a converted document) is kept after the sandbox ends. The file is stored for the support team on an internal note, which the customer never sees; it reaches the customer only when the agent sends it with `send_files`, by the path the result gives. A file larger than 25 MiB is refused, and when no sandbox is running nothing is copied and none is started: the next command starts one again with the saved `/workspace` files.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `sandbox_path` | string | Yes | The file to copy, under `/workspace` (a relative path is taken from `/workspace`). A path through a symbolic link is refused, whatever the link points at |
| `name` | string | No | The stored file's name, with no folders; defaults to the file's name in the sandbox. Not with `path` |
| `path` | string | No | A document to write the file to instead, by its full `data/` path: `data/contracts/acme/msa.docx`. Leave it out to keep the file with the conversation |
| `version` | integer | No | With `path`: the version `copy_to_sandbox` (or `read_file`) gave, to replace that document. Leave it out to create one |

**Response.** `{ path, file_id, bytes, note }`: `path` is the file's address under `conversation/`, which `read_file` and `send_files` accept. As with every file stored in a conversation, the stored name is made safe for storage and timestamped, so use the `path` returned.

**Copying a document back (`path`).** This is how an agent edits a binary or large document of its toolset (a Word or Excel file, a PDF, an image, a text file past `write_file`'s 60,000 characters): copy it in with `copy_to_sandbox`, change it with `bash`, and copy it back with `path` and the `version` it was copied in at. The file is written through the toolset's own `put` tool, as `write_file` writes, and only where the agent writes documents through its `data/` mount: a toolset that only reads (a customer-facing agent's always does) can copy a document in, never back. **With `version` it replaces that version only:** a document changed since it was copied in is refused, changing nothing (`expected version 3, but the current version is 4`), so copy it in again and redo the change. **Without `version` it only creates:** a path already taken is refused. The toolset's own limits apply as to any call of its tools: which file types and folders its `put` reaches, and whom it serves. A document holds at most 1 MiB, refused before anything is written. Nothing is stored with the conversation and no note is left: the document's own history records the write. The answer is `write_file`'s: the toolset's row for the document as the write left it, `{ path, version, content_type, size, metadata, … }`, its `version` the one the next write names; a file that is text is written as `write_file` writes it, so a new one is typed as `write_file`'s are; any other is written as is, and a new one takes its content type from its extension (`.pdf`, `.docx`, `.xlsx`, `.png`, `.svg`, …). A replaced document keeps its own type, and its metadata. A resumed turn never runs a cut copy back again: the agent is told its outcome is unknown, and reads the document to see whether it landed.

---

## Skill Tools

Read a [skill's](../skills.md#skill-files) content on demand (tool-based execution mode). `skill_id` = the skill's id (an agent link's `skill_id`); the ids of the skills linked to the agent are injected into the `skill_id` parameter description at turn time, so the agent can discover them without any hand-written ids. (The handler also resolves a skill by name.)

### read_skill

Read a skill's `SKILL.md`.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `skill_id` | string | Yes | The skill's id |

### read_skill_file

Read a file inside a skill by relative path (no need to call `read_skill` first if the path is known).

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `skill_id` | string | Yes | The skill's id |
| `file_path` | string | Yes | Relative path from skill root (e.g. `references/menu.md`) |

---

## History & Summary Tools

### get_conversations_summary

Timeline of the current user's past conversations (summaries + conversation IDs).

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `max_conversations` | integer | No | Max past conversations (default 10) |
| `max_characters` | integer | No | Output cap (default 20000, max 40000; truncates oldest first) |

A conversation that holds files is marked `[3 files]` beside its dates. Absent means unknown, not zero — entries written before the count existed carry no marker.

### get_conversation

Full transcript of one past conversation (find IDs via `get_conversations_summary`).

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `conversation_id` | string | Yes | Conversation to retrieve |
| `max_characters` | integer | No | Output cap (default 30000, max 60000) |

Also returns `previous_conversation_id` / `next_conversation_id` when they exist — the user's conversations closed nearest in time either side of this one, so the agent can keep walking history without another `get_conversations_summary` call. Neighbours by closing time, not a fixed chain; either may be absent.

The transcript announces that conversation's files by path (`  [files: conversations/<id>/receipt.pdf]` under the message that carried them, plus a trailing `Other files in this conversation:` section for the rest), so `read_file` / `send_files` can open one without any listing call. Internal consult traffic is hidden from the transcript, and so are its attachments.

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

**Connector:** MCP Server (`type: Tool`, `service: MCP Server`; supports Bearer Token and OAuth authentication). A [Base Toolset](../connections.md#base-toolset) connection's tools are assigned the same way: `connection:` names the Base Toolset connection, and its catalog comes from the toolset's own tool definitions.

An agent uses an external MCP server's tools by assigning them individually — there is **no dynamic "discover and call anything at runtime" path**. Each assigned tool is a reviewed, per-agent allowlist entry (the static allowlist keeps third-party tools least-privilege; the server's tool descriptions are wrapped as untrusted input).

### Assigning MCP tools via GitOps (`tools.mcp`)

Declare MCP tools under an agent's `tools:` map. The reference is `(name, connection)` — an MCP tool name is unique per server:

```yaml
# agents/<agent>.yaml
tools:
  mcp:
    - name: search_documents       # the discovered MCP tool's name
      connection: My MCP Server    # the MCP Server (or Base Toolset) connection's display name
```

- **`wayai push` discovers + assigns in one run.** If the named tool isn't in the hub's discovered catalog yet, push runs discovery against the server (equivalent to the UI's "Sync MCP tools" action), then assigns it — so a single push can create the connection (declared in `hub.yaml`) and assign its tools. **The MCP server must be reachable**; if it isn't, push fails with a clear, retryable error.
- **`wayai sync-mcp --connection <name>` re-syncs an existing connection's catalog.** `push` only discovers tools *new* to the catalog — it does **not** refresh the cached schema of tools already catalogued. When the server changes a tool's input schema (adds/renames params), run `wayai sync-mcp --connection "My MCP Server"` to re-discover and refresh the stored catalog (the `--connection` value is the display name or the connection UUID). Preview-only. This keeps the GitOps catalog (what `pull` exports and the diff compares) accurate — and the catalog doubles as the runtime fallback: at turn time the schema is resolved live from the server on a best-effort basis, but when the server is slower than the turn-time budget, unreachable, or the connection's credential fails, the turn falls back to the last synced snapshot. A fresh catalog is what makes that fallback accurate. A Base Toolset's tools are never resolved live: agents always get the synced catalog, so re-sync after the toolset changes.
- **Drift is now surfaced, not silent (you no longer have to already know to run `sync-mcp`).** `wayai push` re-verifies every referenced MCP connection whose tools were all already catalogued and **warns** when the cached schema is behind the server (naming the connection and the changed/removed tools) — best-effort, so a slow/unreachable server never blocks the push. And **`wayai sync-mcp --connection <name> --check`** (alias `--dry-run`) reports that drift **without writing anything**, exiting non-zero when the catalog is behind — run it in CI to gate on a stale catalog. `--check` is read-only and works against production hubs too.
- **Omitted vs present:** when the `mcp` key is **omitted**, existing MCP tool rows are left untouched (tools assigned in the Platform UI survive). When **present** (even `[]`), the list is **authoritative** — unlisted MCP tools are removed. (This differs from `native`/`custom`, whose absence deletes them, because MCP tools are dual-origin: UI + GitOps. Same rule as `evaluation_variables`.)
- **`wayai pull`** emits a `tools.mcp` block for the agent's MCP tools (`id` / `connection_id` included for stable round-trips). The tool schema and description stay derived from the connection's discovered catalog — they are not round-tripped.

### Assigning MCP tools via the Platform UI

hub → **Connections** → set up the MCP Server connection → **Sync MCP tools** → hub → **Agents** → the agent → **Add tool** → pick the MCP tool. Equivalent to `tools.mcp`; the next `wayai pull` materializes these as YAML. A Base Toolset connection catalogs its tools when its token is saved, and its button reads **Sync toolset tools**.

**Refreshing a changed catalog (UI):** click **Sync MCP tools** on the connection again to re-discover and refresh the stored tool schemas after the server changes them — the UI equivalent of `wayai sync-mcp`. Use this (or the CLI command) whenever an MCP server adds/renames a tool's params; `wayai push` alone won't refresh an already-catalogued tool's schema.

> Connection-level sync/discovery (the server's live tool list) is handled by the connection, not by agent-assignable tools. There are no `mcp_discover_tools` / `mcp_refresh` agent tools.

---

## Quick Reference

| Module | Tools |
|--------|-------|
| Conversation | `transfer_to_agent`, `transfer_to_team`, `consult_agent`, `delegate_to_hub`, `start_consult_thread`, `close_conversation`, `update_kanban_status`, `schedule_followup`, `insert_note` (monitor only), `run_monitor` (monitor only) |
| State | `get_state`, `update_state`, `set_state_path`, `reset_state` |
| Skill & File | `list_files`, `read_file`, `write_file`, `edit_file`, `send_files` |
| Code Sandbox | `bash`, `copy_to_sandbox`, `copy_from_sandbox` (need `code_execution_enabled`) |
| Skill | `read_skill`, `read_skill_file` |
| History & Summary | `get_conversations_summary`, `get_conversation`, `expand_summary` |
| Meta | `get_tool_schema`, `execute_tool` |
| MCP | per-agent tools assigned via `tools.mcp` (YAML) or the Platform UI |

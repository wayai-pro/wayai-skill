# Agent Roles & Settings

Deep reference for choosing an agent role, structuring delegation, and configuring connector-specific settings. The shapes of `agents/<slug>.yaml` and the basics of agent fields are covered in `SKILL.md` — this file goes beneath that surface.

## Table of Contents
- [Role Tracks](#role-tracks)
- [Role Reference](#role-reference)
- [Choosing a Role](#choosing-a-role)
- [Delegation Flow](#delegation-flow)
- [Agent Settings](#agent-settings)
- [Per-Connector `settings`](#per-connector-settings)
- [Response Format (Structured Output)](#response-format-structured-output)
- [Evaluation Variables](#evaluation-variables)
- [`enabled` Behavior](#enabled-behavior)
- [`include_message_timestamps` Behavior](#include_message_timestamps-behavior)

---

## Role Tracks

WayAI agents operate on three tracks. The conversation's `current_responder_type` (`agent` | `team` | `system`) plus the hub's `ai_mode` (`pilot` | `copilot` | `pilot+copilot` | `turned_off`) determine which track is active for a given message.

| Track | Active When | Roles |
|-------|-------------|-------|
| **Pilot** | AI talks to the end user | `pilot`, `pilot_specialist`, `pilot_advisor` |
| **Copilot** | AI suggests responses to the team | `copilot`, `copilot_specialist`, `copilot_advisor` |
| **Background** | Always (no message routing) | `monitor`, `conversation_evaluator`, `message_evaluator`, `summarizer` |
| **On-demand** | Only when explicitly consulted | `consultant` |

The Pilot agent's response is delivered through the channel; the Copilot agent's response surfaces in the team UI as a suggestion (no channel delivery). The `consultant` role is track-independent but foreground: it runs only when people (or agents) consult it in visible threads, and its turns bill as normal operations.

---

## Role Reference

| Role | Per Hub | Receives Conversation | Returns Control | Notes |
|------|---------|-----------------------|-----------------|-------|
| `pilot` | 1 | Yes (primary on Pilot track) | n/a | Main AI agent for end users; also a valid `transfer_to_agent` target — a specialist can route back to it (hub-and-spoke router) |
| `copilot` | 1 | Yes (primary on Copilot track) | n/a | Main AI assistant for the support team; also a valid `transfer_to_agent` target |
| `pilot_specialist` | Multiple | Yes (full transfer) | No | Domain expert; receives `transfer_to_agent` |
| `copilot_specialist` | Multiple | Yes (full transfer) | No | Copilot-track specialist |
| `pilot_advisor` | 1 | No (advisory only) | Yes (back to caller) | Receives `consult_agent`; runs once and returns |
| `copilot_advisor` | 1 | No (advisory only) | Yes (back to caller) | Copilot-track advisor |
| `monitor` | 1 per firing trigger | No (silent observer) | n/a | Excluded from message routing. At most one ENABLED monitor on each of `idle`, `user_message` and `assistant_reply`; any number on `manual` — see [One enabled monitor per trigger](#one-enabled-monitor-per-trigger) |
| `conversation_evaluator` | 1 | No (async) | n/a | Scores entire conversation after close |
| `message_evaluator` | 1 | No (async) | n/a | Scores each message |
| `summarizer` | 1 | No (async post-turn) | n/a | Auto-provisioned with the first pilot/copilot; rolling `conversation_summary` state (see SKILL.md) |
| `consultant` | Multiple | No (consulted on demand) | n/a | Track-independent; consulted by people (and agents) in visible consult threads. **An advisor advises an AI mid-turn and is invisible; a consultant is consulted by people (and agents) in visible threads.** Never a track responder and never a `transfer_to_agent`/`consult_agent` target. Configurable today; consult dispatch ships in a follow-up |

Background roles (`monitor`, evaluators, `summarizer`) and `consultant` are excluded from delegation flows — they cannot be the target of `transfer_to_agent` or `consult_agent`.

**`transfer_to_agent` targets any agent on the same track** — the entry `pilot`/`copilot` *or* a `*_specialist` (status `agent` → pilot track; status `team` → copilot track). Targeting the entry pilot enables the **hub-and-spoke router** pattern: the pilot dispatches to specialists, and a specialist can transfer back to the pilot to re-dispatch a request that belongs to a different domain. Cross-track agents and advisor/background roles are never transfer targets. (Within one turn an agent can't be delegated back to an agent already in the chain — the reinvoke cycle guard bounds ping-pong; across turns, re-routing is unrestricted.)

---

## Choosing a Role

| You want… | Use |
|-----------|-----|
| Single AI handling all end users autonomously | `pilot` only |
| AI helping a human team write replies (no channel delivery) | `copilot` only |
| AI handles end users; team can take over and AI shifts to suggesting | `pilot+copilot` mode + `pilot` + `copilot` |
| A specialized handler for billing / refunds / specific intents | Add a `pilot_specialist` (or `copilot_specialist`); transfer to it |
| A domain expert that answers a question and hands back | Add a `pilot_advisor` (or `copilot_advisor`); consult it |
| A dispatcher that routes to domain specialists and re-dispatches | `pilot` as the entry router + `pilot_specialist`s; a specialist transfers back to the `pilot` to re-route a cross-domain request |
| Conversation quality scoring | `conversation_evaluator` and/or `message_evaluator` |
| Silent monitoring/logging | `monitor` |
| A domain expert your support team (or agents) consult in visible threads | Add a `consultant` (configurable now; consult dispatch ships in a follow-up) |

---

## Delegation Flow

Three delegation patterns, all expressed as native tools assigned to the calling agent. The routing effect is only half the story — the harness handles each tool's **return** differently, and that dictates **who writes the next user-facing message**. Your agent instructions must match, or the user sees a redundant message or none at all.

| Tool | Tool result | Who writes the next user-facing message | Prompt rule |
|------|-------------|-----------------------------------------|-------------|
| `transfer_to_agent` | **Skipped** — the harness does not give the caller a reply turn; it reinvokes the target in the same turn | The **target specialist** (immediately) | Call it **silently** — no message at all |
| `consult_agent` | **Returns to the caller** | The **caller** (advisor output is internal) | Use the advice, then reply normally |
| `transfer_to_team` | **Returns to the caller** | The **caller** (one final confirmation) | Call it, **then confirm** — never pre-announce |

**`transfer_to_agent`** — full transfer to a specialist. The harness **skips the tool return to the caller** and **reinvokes the target agent immediately in the same turn**; the specialist produces the next user-facing message. The conversation then stays with the specialist until it transfers back or ends — the user does **not** need to send another message for the specialist to engage.

- **Prompt rule:** call it **silently** — no user-facing text. Any text the caller emits alongside the call **is delivered** to the user, and the specialist then replies in the same turn, so a pre-announcement ("I'll forward you to…") lands as a redundant message in front of the specialist's response. (The harness only skips the caller's *post-result* reply turn — it never reinvokes the caller after the transfer — but that turn is not where a pre-announcement lives.)
- **In evals:** a `transfer_to_agent` scenario is scored as the **complete handoff** — the specialist's continuation, not the intermediate transfer turn (the transfer call + the specialist's final reply). See [`../evals.md`](../evals.md) → "Delegation evals".

```yaml
tools:
  delegation:
    - type: agent
      tool: transfer_to_agent
      target: Specialist - Billing      # display name of the target agent
```

**`consult_agent`** — one-shot advisory. The advisor runs once with the question, **its result returns to the caller**, and control stays with the caller agent, which writes the next user-facing message. The user does not see the advisor's output directly.

```yaml
tools:
  delegation:
    - type: agent
      tool: consult_agent
      target: Compliance Advisor
```

**`transfer_to_team`** — agent-to-team handoff. Switches the conversation's `current_responder_type` to `team` (with `ai_mode: pilot+copilot`, the active track shifts Pilot → Copilot). Unlike `transfer_to_agent`, **the tool result returns to the calling agent** — so the caller writes the next user-facing message itself.

- **Prompt rule:** call the tool, **then confirm** to the user ("You're now with our team — someone will help you shortly."). Do **not** pre-announce before the call.

```yaml
tools:
  delegation:
    - type: team
      tool: transfer_to_team
      target: Tier 2 Support
```

> **Limbo footgun.** The two transfer tools have *opposite* announcement rules. Pre-announcing a `transfer_to_agent` adds a redundant "I'll forward you" before the specialist's immediate reply. Pre-announcing a `transfer_to_team` — then staying silent because the agent "already said it" — leaves the user in limbo: the team has been notified, but the user never gets a closing confirmation. Rule of thumb: **`transfer_to_agent` → say nothing; `transfer_to_team` → say it *after*.**

See [native-tools.md](native-tools.md) for the full parameter list.

---

## Agent Settings

Top-level `agents/<slug>.yaml` fields that affect agent behavior (independent of the LLM `settings` block):

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `enabled` | boolean | `true` | Whether the agent is active for new conversations, transfers, and consultations. Disabled agents are skipped during routing |
| `include_message_timestamps` | boolean | `false` | When `true`, appends `[2026-04-30 14:30:00 (America/New_York), Thursday, afternoon]` to user messages (same line, after the content) in the LLM history. Useful when temporal context affects responses |
| `response_format` | object | omitted (text) | Structured output. When set: `{ schema_name: "...", schema_json: {...} }` (both required). Omit for plain text |
| `connection` | string | required | Display name of the agent's `Agent` connection (LLM provider) |
| `settings` | object | varies | Connector-specific settings (model, temperature, max_tokens, etc.) — see [Per-Connector `settings`](#per-connector-settings) |

Default omission: `enabled: true` is omitted in pulled YAML. `include_message_timestamps: false` is omitted. `response_format` is omitted when the agent is text-only.

---

## Per-Connector `settings`

The `settings` block under each agent is connector-specific — it mirrors the `agent_settings_schema` declared by the connector. Common shapes:

**Anthropic** (`Agent` connection — service: Anthropic):
```yaml
settings:
  model: claude-sonnet-5
  max_tokens: 4096
  # temperature only on Opus 4.6 / Sonnet 4.6 & older — Sonnet 5 / Opus 4.7+ / Fable strip it (see below)
  thinking_enabled: true        # extended thinking on/off (Claude 4+) — WayAI toggle mapping to the Anthropic `thinking` param; default false (off)
  effort: high                  # reasoning effort: low|medium|high|xhigh|max (Opus 4.5+/Sonnet 4.6+/Sonnet 5/Fable)
```

**OpenAI** (service: OpenAI):
```yaml
settings:
  model: gpt-6-sol
  max_tokens: 4096
  # temperature omitted — GPT-6 Sol is a reasoning model and strips it (set temperature only on non-reasoning OpenAI models)
  reasoning_effort: medium      # reasoning models: low|medium|high|xhigh|max|none (xhigh needs gpt-5.2+, max needs gpt-5.6+; GPT-6 Astra has no none; minimal is legacy gpt-5/5.1 only)
```

**Google AI Studio** (service: Google AI Studio):
```yaml
settings:
  model: gemini-3.8-flash
  temperature: 0.7
  max_tokens: 4096
  reasoning_level: high         # dynamic|low|medium|high
```

**OpenRouter** (service: OpenRouter):
```yaml
settings:
  model: openai/gpt-6-sol
  temperature: 0.7
  max_tokens: 4096
  reasoning_effort: medium      # minimal|low|medium|high|xhigh|max|none (OpenRouter maps to the nearest level each model supports)
```

**xAI** (service: Xai):
```yaml
settings:
  model: grok-4.7
  temperature: 0.7
  max_tokens: 4096
  reasoning_effort: none        # low|medium|high|none (none omits the param; direct Grok, not via OpenRouter)
```

### Reasoning / thinking by provider

Each provider exposes one or more reasoning controls, named to match its schema. The key name must be exact — a mismatched key (e.g. a nested `thinking:` object on Anthropic, or `max_output_tokens` on Gemini) is **silently stripped on `wayai push`**.

| Connector | Setting | Values | Maps to |
|---|---|---|---|
| Anthropic | `thinking_enabled` (boolean, default `false`) | `true` / `false` | WayAI's on/off toggle for the Anthropic `thinking` param. `true` → adaptive thinking (Claude 4.6+) or a fixed budget (Sonnet 4.5 / Haiku 4.5 / Opus 4.5). `false` → thinking off — sent as an explicit disable on models that reason by default (Sonnet 5), so "off" always means off. Temperature/top_p are ignored when on. **Fable and Opus 5.5 always think** — the toggle is hidden there and can't turn thinking off (depth is still shaped by `effort`). |
| Anthropic | `effort` | `low` / `medium` / `high` / `xhigh` / `max` | `output_config.effort` (Opus 4.5+, Sonnet 4.6+, Fable). `xhigh` = Opus 4.7+/Sonnet 5/Fable; `max` excludes Opus 4.5; Sonnet 4.5/Haiku 4.5 reject it. Levels the model can't use clamp down. |
| OpenAI | `reasoning_effort` | `low` / `medium` / `high` / `xhigh` / `max` / `none` | `reasoning.effort` (pairs with `verbosity`). `none` = no reasoning (GPT-6 Astra has no `none` level — it runs at `low`). `xhigh` needs gpt-5.2+ (clamped to `high` below); `max` needs gpt-5.6+ (clamped to `xhigh` below). `minimal` is a legacy level only gpt-5 / gpt-5.1 accept — kept as a deprecated value but clamped to `low` on gpt-5.2 and newer. |
| Google Gemini | `reasoning_level` | `dynamic` / `low` / `medium` / `high` | `thinkingLevel` (Gemini 3.x) or `thinkingBudget` (Gemini 2.5). `dynamic` = model default. |
| OpenRouter | `reasoning_effort` | `minimal` / `low` / `medium` / `high` / `xhigh` / `max` / `none` | `reasoning.effort`. `none` disables the override. OpenRouter maps the level to the nearest one each model supports. |
| xAI | `reasoning_effort` | `low` / `medium` / `high` / `none` | Top-level `reasoning_effort` (OpenAI-standard chat-completions form). `none` omits the param. Models without reasoning control ignore/reject a non-none value. |

**Temperature on newer Anthropic models.** `temperature` is the only sampling knob the Anthropic connector exposes (no `top_p`/`top_k`). Sonnet 5, Opus 4.7+, and Fable **strip a non-default `temperature`** before the call (silently ignored, not an error) — set it only on Opus 4.6 / Sonnet 4.6 / Sonnet 4.5 / Opus 4.5 / Haiku 4.5. It's also ignored on any model whenever `thinking_enabled` is on.

**Choosing thinking & effort (Anthropic).** Thinking is **off by default** (`thinking_enabled: false`) on every controllable model — including Sonnet 5, which won't reason unless you set `thinking_enabled: true` (despite reasoning by default at the raw API). **Fable and Opus 5.5 always think** and have no toggle: thinking tokens bill as output, so for simple agents on them lower `effort` rather than looking for an off switch — or pick Sonnet 5 / Haiku 4.5. When setting up an agent:
- **Simple agents** (FAQ, routing, intent classification, short replies) — leave thinking off; optionally set `effort: low`/`medium` to trim tokens.
- **Reasoning-heavy agents** (multi-step problem solving, complex tool orchestration, evaluators) — set `thinking_enabled: true` and/or raise `effort` to `high`/`xhigh`.
- `effort` shapes total token spend (text + tool calls) with or without thinking, so it's the lever even when thinking is off (and the only one on Fable and Opus 5.5).

### File handling (all LLM connectors)

A conversation's **earlier** files are announced to every agent as a text annotation naming each file by path — `[Attached files: conversation/report.pdf (Type: application/pdf, Size: 2048 bytes)]` — and never re-sent as content, in any mode. Re-attaching them cost a download and a block of context per file on every turn, for content the agent can ask for by path in one call. The current message's own attachments are unaffected: they are always sent in full.

| Setting | What it still does |
|---|---|
| `file_handling_mode` | No longer changes what the model receives. `metadata_only` still auto-enables `read_file` for the agent — the one way, besides assigning the tool, to let it open a path it was told about. Not on a `user_message` monitor, which is offered no tools at all (see `trigger`). |
| `max_attachment_size_mb` | Nothing. There is no attachment to cap. |

Which agents see the annotations: the pilot/copilot track, `monitor`, and `conversation_evaluator` — the roles whose context is assembled from turn history. Three roles are not, so instructions that key on `[Attached files: …]` never match for them: `message_evaluator` (dispatched with a single synthesized instruction message), `summarizer` (reads its own synthesized message list), and `consultant` (its customer transcript is rendered by hand, text-only, into a `<customer_conversation>` frame). Any of them can still hold `read_file`: the auto-append gate is the per-agent `metadata_only` setting and carries no role check, so an evaluator configured that way gets the tool even though nothing in its context names a path. One exception to holding it and being able to USE it: a monitor on the `user_message` trigger is offered no tools, so the tool is on the agent and never reaches the model.

### Pre-tool preamble delivery (all LLM connectors)

Models often write a short user-facing message *before* calling tools ("Let me check that, one moment…"). `deliver_preamble` controls what happens to it:

| Setting | Values | Default | Effect |
|---|---|---|---|
| `deliver_preamble` | boolean | `true` | Deliver the pre-tool text immediately as its own message — the user sees/hears it during the tool-execution wait, then the final reply arrives as a separate message. `false` = only the final reply is delivered; the pre-tool text is discarded (from the user AND from the model's context, so the final answer is always self-contained — except on Claude Fable / Opus 5.5, whose progress update stays in the model's context; see below). |

Notes:
- Applies to **every** tool-loop round — a multi-step turn can deliver several progress messages before the final answer.
- **Claude Fable and Opus 5.5** write their pre-tool narration as a short progress update. The Anthropic connector delivers it as the preamble; other connectors serving these models (OpenRouter, the Claude harness) don't yet. The update is part of the model's own reasoning, so it stays in the model's context even when not delivered.
- **Email is exempt**: each preamble would be a separate email, so the setting is ignored on email channels (pre-tool text is discarded there).
- **Voice conversations**: each delivered preamble is synthesized as its own TTS audio clip — the "preamble technique" that avoids dead air while tools run — **when spoken replies are enabled** (the TTS connector's `voice_reply_enabled`, on by default; see Connections › TTS). With it off, preambles deliver as text only.
- Billing: each delivered preamble is a normal delivered message (+1 operation; +1 TTS operation on voice turns when spoken replies are enabled).
- Set `deliver_preamble: false` for agents that shouldn't narrate progress or whose channel UX wants exactly one message per turn (only pilot-track agents deliver to users; copilot/background roles never deliver preambles regardless).

### Copilot suggestion trigger (all LLM + harness connectors)

Controls *when* a Copilot-track agent drafts its suggestion. Consulted **only** for an agent running on the Copilot track (status `team` under `copilot` / `pilot+copilot`); ignored for Pilot and background agents.

| Setting | Values | Default | Effect |
|---|---|---|---|
| `copilot_trigger` | `every_message` / `on_demand` | `every_message` | `every_message` drafts a suggestion automatically on every inbound message (today's behavior). `on_demand` suppresses the automatic suggestion — the support team pulls one explicitly with the "Suggest reply" button in `/support`. |

Notes:
- A **cost + noise lever**: a Copilot bills a foreground operation per drafted suggestion, so `on_demand` avoids paying for suggestions on messages the team can already handle.
- `on_demand` changes only *when* the suggestion fires — the suggestion itself is the same internal, team-facing draft (never delivered to the customer).
- No effect on Pilot agents; a Pilot always responds to the end user regardless of this setting.

When the agent's connection changes, existing `settings` are sanitized against the new connector's schema — unknown keys are dropped. To see the exact schema for a specific connector at any time, run `wayai pull` and inspect a freshly pulled agent's `settings` block.

---

## Response Format (Structured Output)

Force the agent to return structured JSON instead of free text:

```yaml
response_format:
  schema_name: order_extraction
  schema_json:
    type: object
    properties:
      order_id: { type: string }
      items:
        type: array
        items:
          type: object
          properties:
            sku: { type: string }
            quantity: { type: integer }
    required: [order_id]
```

JSON Schema property order is also the model's structured-output emission order. For evaluator or rubric schemas, declare rationale fields before conclusion fields so the model reasons before committing to its verdict:

```yaml
response_format:
  schema_name: response_evaluation
  schema_json:
    type: object
    properties:
      notes: { type: string }
      response_match: { type: boolean }
    required: [notes, response_match]
    additionalProperties: false
```

Both `schema_name` and `schema_json` are required when `response_format` is set. The `wayai pull` includes `type: json_schema` for informational consistency — the platform sets it automatically.

Agents with `response_format` set behave like API endpoints — the response is the JSON object, not a chat reply. Useful for evaluators, structured intent classification, and data-extraction agents called via `consult_agent`.

---

## Evaluation Variables

`conversation_evaluator`, `message_evaluator` and `monitor` agents declare the structured fields they emit — their **evaluation variables** (stored as agent parameters). An evaluator's are surfaced as Analytics columns; a monitor's feed only its own `flag_conditions` and `rules` (see [Monitor Configuration](#monitor-configuration-monitor-only)), so `label`, `category` and `is_summary` do nothing on a monitor. They round-trip via `wayai pull` / `wayai push` under an `evaluation_variables` list on the agent:

```yaml
# agents/conversation-evaluator.yaml
name: Conversation Evaluator
role: conversation_evaluator
connection: anthropic
evaluation_variables:
  - name: conversation_summary
    type: string
    label: Conversation Summary
    description: A concise summary of the conversation.
    category: quality
    is_summary: true            # rolling-summary field (type: string only)
  - name: user_sentiment
    type: enum
    label: User Sentiment
    enum: [positive, negative, neutral]
    category: satisfaction
  - name: goal_achieved
    type: enum
    enum: [yes, no]
    required: false
```

| Field | Type | Default | Notes |
|-------|------|---------|-------|
| `name` | string | required | Variable name the agent writes — unique per agent |
| `type` | enum | `string` | `string` \| `number` \| `boolean` \| `integer` \| `enum` |
| `description` | string | `""` | What the variable measures (guides the evaluator) |
| `label` | string | omitted | Display label (Analytics columns, UI) — evaluators only |
| `enum` | string[] | omitted | Allowed values — required when `type: enum`, ignored otherwise |
| `required` | boolean | `true` | Whether the evaluator must populate it |
| `category` | string | omitted | Analytics grouping: `quality` \| `satisfaction` \| `compliance` \| `categorization` — evaluators only |
| `is_summary` | boolean | `false` | Marks the rolling conversation-summary field; honored only for `type: string` |

**Replace semantics:** the list is authoritative. Omit the `evaluation_variables` key to leave the agent's current variables untouched; include it (even as `[]`) and `push` makes the platform match it exactly — added entries are created, missing ones deleted. Default-valued fields are omitted on pull, so a pulled-then-pushed file is a no-op.

---

## Flag Conditions (`conversation_evaluator` only)

`flag_conditions` is a setting on the **`conversation_evaluator` agent** — conditions evaluated after a conversation closes (against its system metrics + evaluation variables). Any match (OR semantics) flags the conversation for review in the support sidebar. Round-trips via `wayai pull` / `wayai push` as a top-level key on the evaluator agent:

```yaml
# agents/conversation-evaluator.yaml
name: Conversation Evaluator
role: conversation_evaluator
connection: anthropic
flag_conditions:
  - variable: goal_achieved      # system metric or evaluation-variable name
    operator: "="                # "=" | "!=" | ">=" | "<=" | ">" | "<"
    value: no
  - variable: user_sentiment
    operator: "="
    value: negative
```

New hubs seed the two defaults above onto the evaluator agent automatically. Omit the key to leave the current value untouched; set `flag_conditions: []` to clear it. Configurable from the UI in the evaluator agent's detail view (Agents tab) and through `wayai pull` / `wayai push`. (Previously a hub-level Overview setting — relocated to the evaluator agent.)

**Operators.** `=` and `!=` compare the value as written. `>=`, `<=`, `>` and `<` are how
you threshold a numeric evaluation variable:

```yaml
  - variable: satisfaction_score
    operator: "<"
    value: 3
```

They compare numerically when both the variable's value and the condition's `value` are
numbers (a number written as a string counts); when neither side is a number they compare
as text, and when only one side is a number the condition does not match. Any operator
outside this list is refused when you save, on every path.

---

## Summarization Threshold (`summarizer` only)

`summarization_threshold_tokens` is a setting on the **`summarizer` agent** — the rolling summarizer fires async post-turn once a turn's effective input tokens (input + cache_read + cache_creation) reach this value. Default 120000; min 1000, max 1000000. Round-trips via `wayai pull` / `wayai push` as a top-level key on the summarizer agent:

```yaml
# agents/summarizer.yaml
name: Summarizer
role: summarizer
connection: anthropic
summarization_threshold_tokens: 120000   # lower for testing, raise for very long conversations
```

New hubs seed the default onto the summarizer agent automatically. Omit the key to leave the current value untouched; an empty value (`summarization_threshold_tokens:`) clears it back to the 120000 default. Configurable from the UI in the summarizer agent's detail view (Agents tab) and through `wayai pull` / `wayai push`. (Previously a hub-level Overview setting — relocated to the summarizer agent.)

---

## Previous Conversations Context

`previous_conversations_count` gives an agent the user's most recent **ended** conversations as context, without any prompt engineering. `0` disables it; the maximum is 20.

Push semantics match every other optional agent key: **absent on create** means off, **absent on update leaves the stored value alone**, and `0` is what clears one. So removing the key from an agent's YAML does not turn the feature off — set it to `0`.

```yaml
# agents/support-pilot.yaml
name: Support Pilot
role: pilot
connection: anthropic
previous_conversations_count: 3   # this user's 3 most recent ended conversations
```

The block is prepended to the conversation's **first user message** as `<previous_conversations>`, carrying `conversation_id`, `ended_at`, and `summary` for each. That placement is deliberate: putting the same content in `instructions` would make the cached system prefix — which covers the instructions AND every tool schema, shared across the whole hub — different for every user, so each conversation pays to write that prefix to cache instead of reading a shared one. On the first user message it sits inside the reusable history prefix instead.

**It is resolved once and frozen.** The set is read at the first agent turn that actually needs it — the first turn whose agent has the setting on — and replayed unchanged afterwards. For a conversation that starts with the feature already enabled, that is its first turn, so the context is "as of when this conversation started", and a conversation ending later does not appear until the *next* one begins. Enabling the setting on a hub whose conversations are already open is the exception: those conversations have no snapshot yet, so each freezes on its next turn and can include a conversation that ended after it started. Once frozen, it stays frozen either way. This is what keeps the injected bytes stable; re-resolving mid-conversation would rewrite the first message and invalidate the whole cached history. Raising the setting takes effect on conversations already open; lowering it applies immediately.

**Summaries come from the Conversation Evaluator.** `summary` is that agent's post-close output. A hub without an enabled `conversation_evaluator` — or a conversation that ended before one existed — yields an entry with an ID and `ended_at` but no summary. The IDs remain useful: an agent with the `get_conversation` tool assigned can read the full transcript of any of them.

Nothing fails when that chain is broken — the block simply arrives without summaries — so the agent's settings page checks the hub and says which link is missing. The conditions, in the order it reports them:

| Reported | Meaning |
|---|---|
| `no_conversation_evaluator` | The hub has none. Auto-provisioning fires on **pilot** creation, so a copilot-only hub never gets one. |
| `evaluator_disabled` | It exists but is switched off. |
| `evaluator_no_connection` | It has no connection, so it throws at every conversation close — provisioned, but no more useful than absent. |
| `evaluator_summary_not_emitted` | The evaluator no longer emits the summary field — free-text output, a cleared schema, or a schema whose properties dropped that name. The close path reads that key out of the evaluator's structured output, so it finds nothing. The settings UI shows an agent's response format read-only, so this one is fixed through `wayai push`. |
| `no_system_channel` | The evaluator turn is dispatched on the hub's system channel; without one it never runs. |

The setting is **not** blocked on any of these — a hub that wants ids plus `get_conversation` is a legitimate configuration.

Ignored on background roles (`monitor`, `conversation_evaluator`, `message_evaluator`, `summarizer`), which never address the end user.

Rows age out with the hub's retention settings — a conversation past the hub's retention cutoff is dropped from the frozen block and erased from the stored snapshot, so the context does not outlive the window the operator configured. Pruning is best-effort per turn: if the platform retention config can't be read on a given turn it is skipped rather than guessed at (guessing short would erase rows that are still live, which nothing can undo) and retried on the next turn.

Configurable from the UI in the agent's detail view (Agents tab) and through `wayai pull` / `wayai push`.

**Not the same as `{{previous_conversations(N)}}`.** That placeholder resolves the same rows but renders wherever you put it, re-resolving every turn. Prefer this setting; see [instructions.md](instructions.md) for the placeholder.

---

## Monitor Configuration (`monitor` only)

`monitor_config` is a setting on the **`monitor` agent** — it controls **when** the monitor evaluates a conversation and which conditions flag it. `flag_conditions` use OR semantics (any match flags the conversation). Unlike the evaluator/summarizer, the monitor is **not** auto-provisioned — create it explicitly. Round-trips via `wayai pull` / `wayai push` as a top-level key on the monitor agent:

```yaml
# agents/monitor.yaml
name: Monitor
role: monitor
connection: anthropic
monitor_config:
  trigger: idle                 # when this monitor runs; defaults to idle
  delay_seconds: 300            # idle only: inactivity wait before it runs; >= 10
  flag_conditions:
    - variable: user_sentiment  # a field this monitor declares below
      operator: "="             # same operators as the evaluator's flag_conditions
      value: negative
evaluation_variables:           # the fields of this monitor's own answer its conditions read
  - name: user_sentiment
    type: enum
    enum: [positive, neutral, negative]
```

**What a condition can read.** A monitor's `flag_conditions` and `rules` read the monitor's OWN answer — never another agent's — through the fields it declares in its `evaluation_variables`: the same list, with the same types, an evaluator carries (see [Evaluation Variables](#evaluation-variables)). Declare every field a condition names, including a `{field}_confidence` one (`type: number`) on a decisions model. A monitor's declarations are read up to about two million characters of names and enum values in all; any past that are left out. A `boolean` field compares as `true` or `false`. A monitor's variables feed its own conditions only; they are not Analytics columns.

### `trigger`

| Value | When the monitor runs |
|---|---|
| `idle` (default) | After the conversation has been quiet for `delay_seconds`. This is what a monitor does when `trigger` is omitted. |
| `user_message` | Right after the customer's message, before the answering agent replies. The customer waits for it, so it is meant for a fast, closed-set monitor. |
| `assistant_reply` | After the answering agent has drafted its reply, before any of it is sent — the **reply gate**. See [The reply gate](#the-reply-gate--assistant_reply). |
| `manual` | Never on its own — it runs only when another monitor calls it with `run_monitor`. A hub may have any number. |

**All four run.**

A `user_message` monitor runs **inside** the customer's turn rather than as a turn of its own. Its `flag_conditions` are evaluated and the conversation flagged exactly as an idle monitor's are, and it writes no message of its own.

**The model is offered no tools.** A monitor judges the window it is given: whatever tools it holds — assigned, or `read_file` auto-enabled by `metadata_only` — are not offered to the model on this trigger, so it cannot look anything up mid-judgement. The same agent on `idle` does run its tools, because that is a turn of its own. A monitor that needs a tool to reach its verdict belongs on `idle`. Its `rules` can still CALL a tool once the verdict is in — see [`rules` and `fallback`](#rules-and-fallback) — and that is a different thing: the model chooses values, and your configuration chooses what they mean.

**It adds its own latency to every reply.** The customer waits for it, so keep it on a fast model and a small window. It cannot change the reply or stop it being sent. If its model is misconfigured, fails, or returns nothing usable, the monitor is skipped and the turn proceeds without it; if the model is merely slow, its call is abandoned after a few seconds and the turn proceeds without waiting for it.

**Only the model call is abandoned.** Reading the conversation and resolving the monitor's own instructions happen first and are not. Placeholders in a monitor's instructions are resolved for a `user_message` monitor exactly as they are for an `idle` one — so a monitor whose instructions pull in file or resource content (`{{file_content(…)}}`, `{{files()}}`, `{{previous_conversations(N)}}`) fetches that content on every customer message, before its model is called. Prefer a monitor that judges the conversation it is handed; move one that needs to look things up to `idle`.

### One enabled monitor per trigger

**A hub runs at most one enabled monitor on each of `idle`, `user_message` and `assistant_reply`**, and saving a second one is refused. `manual` is not limited — nothing fires it on its own, so any number of monitors can wait there to be called.

What each surface does with the rule:

| Surface | Behaviour |
|---|---|
| The agent editor, and `POST`/`PATCH` on the agents API | Refuses a save that would put a second enabled monitor on a trigger another enabled monitor holds, naming the agent in the way. Only a save that changes the monitor's trigger, its `enabled` flag or its role is judged — renaming one, editing its conditions or uploading instructions is not. |
| `wayai push` and `wayai diff` | Judged on the configuration the push would produce, not one agent at a time — so two monitors can SWAP triggers in a single push, and a push that replaces a monitor with another on the same trigger lands. A push that would leave two enabled monitors sharing a trigger is refused, with no agent change applied; `wayai diff` reports the same refusal before you push. |
| Publishing a preview to production, and syncing preview → production | Not re-checked. These copy a configuration that was already accepted where it was authored. |

**A monitor with no `monitor_config` holds `idle`.** An absent `trigger` *is* `idle`, and that is true of a monitor carrying no `monitor_config` at all — so such a monitor takes the hub's idle slot, and having no `delay_seconds` to run on, it schedules nothing there. It counts for this rule: a second `idle` monitor beside it is refused naming it. Give it a `delay_seconds` and it becomes the hub's working idle monitor; otherwise move it to another trigger, disable it, or delete it.

**A hub that already has two.** Nothing limited monitors before this rule, so a hub can still hold a pair — most often a monitor that predates `monitor_config` sitting beside a working one. The rule never blocks a save or a push that does not make it worse, including the one that repairs it by disabling or moving one of them. Until then only the first of the pair is reachable, and if that one is the monitor with no `monitor_config`, nothing runs on that trigger at all. `wayai pull` lists them.

### `history_messages` and `include_tool_results`

These shape what a `user_message`, `assistant_reply` or `manual` monitor READS, and are refused on an `idle` monitor, which runs as a full turn and takes the ordinary history window.

| Key | Meaning |
|---|---|
| `history_messages` | How many of the newest messages the monitor sees. Default 10, maximum 100. Keep it small — a closed-set monitor's accuracy falls as unrelated content grows. |
| `include_tool_results` | `false` by default: the monitor sees that a tool ran, but not what it returned. Set `true` only when the monitor's judgement depends on the tool's output. |

**`delay_seconds` belongs to `idle` and is required for it.** An `idle` monitor — one that says so, or one that omits `trigger` entirely — must declare a delay of at least 10 seconds, and a push without one is refused. The other three triggers do not use a delay and may omit it.

### `rules` and `fallback`

A `user_message` monitor can act on what it found, and so can an `assistant_reply` one (the reply gate) and a `manual` one when another monitor runs it. `rules` is an ordered list: the first rule whose conditions ALL hold selects an action, and `fallback` supplies one when no rule matched. Both are refused on `idle`, which has nothing to interpret them.

```yaml
monitor_config:
  trigger: user_message
  history_messages: 10
  rules:
    - when:                                   # every condition must hold (AND)
        - variable: urgency
          operator: "="
          value: high
        - variable: urgency_confidence        # only on a decisions model
          operator: ">="
          value: 0.8
      action:
        kind: call_tool
        tool_name: notify_ops                 # an external tool assigned to THIS monitor
        args:
          level: { from_variable: urgency }   # a value the monitor produced
          note:  { const: escalate }          # a value you wrote
  fallback:
    kind: none                                # the default: do nothing
```

**Conditions are the same conditions.** `when` uses the operators `flag_conditions` uses, over the same variables. The difference is quantification: every condition in a `when` must hold, while `flag_conditions` flags on any single match. A threshold is just a condition on a `{field}_confidence` variable — there is no separate threshold setting, and those variables exist only on a model that returns calibrated confidence.

**The model never names a tool or an argument.** It produces values. Your rule decides which tool runs and which argument each value fills — either `from_variable` (something the monitor decided) or `const` (something you wrote). A value coming back as text is passed to the tool as a value and nothing else; it is validated by that tool's own schema exactly as the answering agent's tool calls are.

**One call per message.** First match wins; later rules are not tried, and a monitor makes at most one tool call per customer message.

**The tool must be assigned to the monitor itself.** A rule naming a tool the monitor does not carry is refused when you save it, and refused again at run time if the tool is unassigned later. Assigning it to another agent is not enough — a monitor reaches only its own tools. The order is: create the monitor, assign its tools, then add the rule.

**Available actions.** `call_tool`, and `none` (evaluate and flag, but do nothing) — which is also the default `fallback`. Holding a reply (`hold`) and asking for one revision of it (`rewrite`) belong to the reply gate (`assistant_reply`) — see [The reply gate](#the-reply-gate--assistant_reply) — and are refused on every other trigger.

**What a rule can call.** `update_state`, `schedule_followup`, `insert_note`, `run_monitor`, `transfer_to_agent`, `transfer_to_team`, and this hub's own external HTTP and MCP tools. Nothing else — the list is what rules may call, not what they may not, so a tool is unavailable to a rule unless it is named here.

Why the others are not on it. `close_conversation` and `update_kanban_status` (a move to a status you marked terminal ends the conversation) decide whether the conversation is still open, which this turn settled before the monitor ran — so the call would change the conversation record without changing the reply the customer is about to get. `consult_agent` would run a second model call inside the customer's wait. Assign any of them to the answering agent instead, which can act on them mid-reply.

#### Briefing the answering agent — `insert_note`

A monitor judges the customer's message before the answering agent replies. `insert_note` is how it passes what it found forward: **you** write the sentence, the monitor's variables are filled into it, and the answering agent reads it as context for that one reply.

```yaml
# on the monitor agent
tools:
  native:
    - insert_note

# in the monitor's rules
      action:
        kind: call_tool
        tool_name: insert_note
        args:
          template:
            const: "This customer is {{urgency}} urgency and sounds {{sentiment}}. Lead with an apology and skip the upsell."
```

**You write the note, not the model.** `template` must be a constant you wrote — it cannot come from a variable. The answering agent reads the note as guidance, so its sentences have to be yours; a note whose whole text came from a model would be instructions written by whoever sent the message being judged. Variables go *inside* your sentence as `{{variable_name}}`, where they arrive as data. A rule whose template comes from a variable is refused when you save it, and refused again at run time.

**It reaches this reply only.** The note is gone once the reply is written — it is not part of the conversation and the agent never sees it again. That is what makes it safe to say something true only of this message. To leave something durable instead, use `update_state`, which the answering agent reads through `{{state(slug)}}` from the customer's *next* message onward.

**Keep a copy for your team.** Add `keep_in_history: { const: true }` and the note is also recorded in the conversation for the support team to read. **The customer never receives it or sees it**, on any channel or in any app view. It still does not come back to the agent on later replies.

**Only a monitor can hold this tool.** Assigning `insert_note` to a pilot, a specialist or any other agent is refused — in the editor, in `wayai push`, and through the API. Nothing else has a reply to brief.

**One note per message**, like every rule action: the first matching rule wins. A variable the monitor did not produce this run comes out as nothing rather than cancelling the note, and the names it could not fill are recorded with the run. A very long note is shortened.

**Unlike every other tool a rule can call, this one costs nothing to run** — no waiting, no external call, and no operation on your bill. It does make the answering agent's prompt slightly longer.

#### Chaining monitors — `run_monitor`

One monitor can run another. The caller judges every message cheaply; on the rare path its rule calls `run_monitor`, and a second monitor — one you keep on `trigger: manual` — takes a closer look and acts on **its own** rules.

```yaml
# the caller, on every customer message
monitor_config:
  trigger: user_message
  rules:
    - when: [{ variable: urgency, operator: "=", value: high }]
      action:
        kind: call_tool
        tool_name: run_monitor
        args:
          monitor_name: { const: Refund Judge }   # ← a manual monitor on this hub

# the callee
monitor_config:
  trigger: manual
  rules:
    - when: [{ variable: refund_risk, operator: ">=", value: 4 }]
      action:
        kind: call_tool
        tool_name: transfer_to_team
        args: { team_name: { const: Tier 2 Support } }
```

**The callee acts on its own rules, and tells the caller nothing.** What it found is recorded against the callee, not returned to the caller — so a cascade cannot be used to let a caller do something its own rules may not. The callee's rules go through the same list of what a rule can call.

**The callee must be `manual`.** That is what keeps it from also running on its own, and it is checked when you save the rule *and* again each time the rule fires — because a monitor can be renamed, disabled, deleted, re-roled or moved to another trigger long after the rule was written. When any of that has happened the run is skipped and the reason is recorded; the customer is answered exactly as if no rule had matched.

**`monitor_name` names a monitor, and must be a `const` you write.** It chooses which monitor runs, so it cannot come from a variable. It is a *name*, not an id, which is why a cascade keeps working after `wayai push`, `wayai pull`, publishing a preview, syncing, or replicating a hub — the name means the same thing on the far side. Rename the callee and you must update the rule, exactly as you would after renaming a tool a rule calls.

**Limits, so one message cannot spend an unbounded amount of work.** A chain may be **three** callees deep, and a monitor already in the current chain cannot be called again — so a loop is refused rather than run. Those two bound it completely: a monitor takes at most one action per run, so a cascade is a single chain and never branches. Each limit records which one stopped it.

**Each run is recorded separately.** Every monitor in the chain writes its own entry, carrying its depth and which run invoked it, so you can read the whole cascade back afterwards.

**The callee is told what the caller found.** The caller's variables are passed to it as context, so the callee can look closer at a judgement already made rather than re-deriving it. It still produces its **own** variables, and its rules read those — not the caller's.

**What it costs.** Each callee is another model call inside the same customer's wait — a cascade of two monitors is two calls before the reply begins. No callee starts a turn of its own, so none of them adds a *turn* to your bill — but the time they add is part of the same turn, and a turn is billed for how long it runs as well as for starting. A cascade that makes a reply take noticeably longer costs noticeably more. Prefer a cheap closed-set caller that only cascades on the rare path.

**Handing the conversation over is the one thing a rule can change about who writes this reply.** The two transfer tools are on the list precisely because changing who responds is what they are for, and a rule's transfer takes effect on the SAME message rather than the next one:

- `transfer_to_team` — the conversation becomes a person's, and the AI says nothing at all on this message. This is the escalation judge: your monitor reads the customer's message, decides it needs a human, and no AI reply is sent.
- `transfer_to_agent` — the AI you named answers instead of the one that would have. It answers from its own instructions, its own context and its own tools, not the original agent's.

Both are declared under `tools.delegation` and assigned to the monitor — that is the only place these two tools can be declared, for a monitor exactly as for a pilot. **Where each one gets its destination differs, and it is the same difference a pilot sees:**

- `transfer_to_team` goes to the team you pinned as the delegation's `target`. The rule must still pass `team_name` — the tool requires it — but that value does not choose the team; it is what the handoff note records. **Write the same name you pinned**, or the note will say one team and the conversation will go to the queue of another. Declare one delegation per destination team and let the rule pick which one to call.
- `transfer_to_agent` goes to the agent the CALL names, not to the delegation's `target`. So `agent_name` is what decides, and the rule must supply it.

```yaml
# on the monitor agent
tools:
  delegation:
    - type: team
      tool: transfer_to_team
      target: Tier 2 Support          # ← this is where a transfer_to_team rule goes
    - type: agent
      tool: transfer_to_agent
      target: Refunds Specialist

# in the monitor's rules
      action:
        kind: call_tool
        tool_name: transfer_to_team
        args:
          team_name: { const: Tier 2 Support }        # ← matches the target above
      action:
        kind: call_tool
        tool_name: transfer_to_agent
        args:
          agent_name: { const: Refunds Specialist }   # ← this is what routes it
```

A team handoff puts the conversation in the team's queue, unclaimed — a member picks it up in the support UI, exactly as for a handoff from a pilot.

> **On an account-mode hub, neither the pinned target nor `team_name` chooses the team.** When the hub's `support_model` is `account`, every `transfer_to_team` — from a rule or from a pilot — goes to the customer's own assigned team, falling back to the hub's default team, and fails if the hub has neither. Set the customer's team (or a hub default) rather than expecting the delegation's `target` to route the escalation.

A transfer that cannot be carried out — an agent that is disabled, on the other track, or named by something that resolves to nothing — is recorded as a failed call and changes nothing: the original agent answers as it would have. The usual transfer rules still apply; a rule is simply another caller.

**What a transfer costs.** The customer waits for two replies' worth of work rather than one: the monitor's own call, then the incoming agent's. A team handoff has no second model call — nothing answers. Neither adds an operation of its own to your bill; what they add is the time the conversation spends being handled, which is metered as one continuous stretch rather than twice over. And the monitor judges once per customer message — it does not run again on the message it handed over.

**An argument that is not a plain value.** A rule's arguments are text, numbers and true/false. A tool that expects an object or a list still works — write the value as JSON in a constant and it is read back as the shape the tool declares:

```yaml
      action:
        kind: call_tool
        tool_name: update_state
        args:
          state_slug: { const: triage }
          updates:    { const: '{"urgency":"high"}' }
```

**A tool's own follow-on actions do not run from a rule.** If you have configured a tool with composed actions — a chain that fires after it succeeds — that chain is skipped when a rule calls it. It still runs normally when the answering agent calls the same tool. Without that, a chain ending in "close the conversation" would reach from a rule the very thing the list above excludes.

**Telling the answering agent what the monitor found.** Not on this turn. The agent's instructions and its state block are both assembled before the monitor runs, so a state a rule writes is read on the customer's NEXT message. A rule is for acting — writing state, notifying a system, scheduling a follow-up, handing the conversation over — not for briefing the reply the customer is waiting for. (A transfer is the exception that proves it: the incoming agent assembles its own instructions and state from scratch, so it sees what the rule wrote.)

**One reply per message, from whoever owns the conversation.** Every rule but a transfer leaves the reply exactly as it was: the tool succeeds, fails or is refused, and the customer is answered once, by the agent this turn had already chosen. A transfer changes WHO answers, never HOW MANY times — after a handoff to an agent the customer gets one reply, from the new agent; after a handoff to a team, none. What every rule costs is time: its tool call adds its own wait to the reply, bounded at a few seconds, after which the call is abandoned and the turn goes on without it — and a transfer abandoned that way is treated as not having happened, so the original agent still answers.

**Where to see what happened.** A run records which rule matched, which tool it named, and whether the call was refused or failed, beside the monitor's own output.

Omit the key to leave the current value untouched; set `monitor_config: null` to clear it. Configurable from the UI in the monitor agent's detail view (Agents tab) and through `wayai pull` / `wayai push`. (Previously a hub-level Overview setting — relocated to the monitor agent.)

---

## `enabled` Behavior

When `enabled: false`:
- The agent is skipped during conversation routing
- Existing conversations already attached to this agent stay attached (the agent is not re-routed automatically)
- Delegation tools targeting this agent will fail at runtime
- Pull continues to round-trip the agent (it's still part of the hub config)

Use `enabled: false` to take an agent offline without deleting it — its config, instructions, and history stay intact.

---

## `include_message_timestamps` Behavior

When `true`, every user message in the LLM history is augmented — the stamp is **appended to the same line** after the message content, space-separated:

```
The user's actual message text [2026-04-30 14:30:00 (America/New_York), Thursday, afternoon]
```

The hub's `timezone` setting determines the timezone shown; the weekday and the `daypart` (`morning` before 12:00, `afternoon` until 18:00, `night` after) are computed from the timestamp. The `daypart` is a coarse English label the model maps to the locale-appropriate greeting (e.g. pt `bom dia` / `boa tarde` / `boa noite`) instead of re-deriving the bucket from the clock.

Use this when:
- The agent reasons about time deltas ("how long since the last message?")
- The agent needs to know business hours / day-of-week
- Followup logic depends on absolute time

Skip this for agents where temporal context is irrelevant — the extra tokens add up over long histories.

**`include_message_timestamps` vs `{{now()}}` — pick by granularity.** This setting is the **per-message** way to give temporal context (a `[timestamp, weekday, daypart]` on *every* user message). The other way is `{{now()}}` in `additional_context_template` — a **single per-turn "now"** (see [`instructions.md`](instructions.md#additional-context-cache-friendly)). Both are cache-safe (neither touches the cached system-prompt prefix) and both resolve into eval replay, so the choice is purely granularity: reach for `{{now()}}` when the agent only needs the current time, and this setting when it must reason about *when each* message arrived. They compose — you can set both — but don't enable per-message timestamps just to answer "what time is it now?"; that's `{{now()}}`'s job and it's leaner.

### The reply gate — `assistant_reply`

A monitor on `trigger: assistant_reply` reads the reply the answering agent has just drafted — **before any of it is sent** — and decides what happens to it.

```yaml
monitor_config:
  trigger: assistant_reply
  rules:
    - when: [{ variable: tone, operator: "=", value: inappropriate }]
      action: { kind: hold }
    - when: [{ variable: promises_refund, operator: "=", value: true }]
      action:
        kind: call_tool
        tool_name: notify_ops
        args: { note: { const: "AI promised a refund" } }
```

**A gate with rules needs structured output.** Its rules read variables, and a variable comes only from a structured answer — so a gate with `rules` must set a [`response_format`](#response-format-structured-output) whose schema carries the variables they read (`tone` and `promises_refund` above), each declared in its `evaluation_variables`. On text output no rule could ever match, and every reply would be sent unjudged. The settings view shows the response format read-only, so set it with `wayai push`. A `fallback` alone needs none.

- **Creating such a gate, or turning a gate into one, is refused** — in the agent editor, on the agents API, and by `wayai push`.
- **A gate saved before this rule still saves** in the editor and on the agents API — renamed, disabled or otherwise edited — and publishing or replicating its hub copies it as it is. The editor flags it beside its rules, and each reply it lets through is logged as sent unjudged.
- **`wayai push` refuses its declaration** until the YAML sets a `response_format` or drops the rules, and nothing else in that push is applied until it does. `wayai pull` of such a hub gives you exactly that YAML, so fix it before your next push.
- **A rule on a variable nothing declares** (a typo, or one deactivated since) can never match, and is not reported as unjudged — declare it.

**What it judges.** The same window a `user_message` monitor reads, plus the draft, labelled as the reply not yet sent. A callee it runs with `run_monitor` is shown the draft too.

**Actions.**

| Action | What happens |
|---|---|
| `none` | The reply is sent as drafted. |
| `call_tool` | The tool runs, then the reply is sent as drafted. |
| `hold` | The reply is **not sent**. It is kept in the conversation for your support team to read — the customer never receives it or sees it, on any channel or in any app view. The conversation is flagged and moved to the team's queue, unclaimed, where a person decides what to send. If moving it to the queue fails, the reply is still held and the conversation still flagged. |
| `rewrite` | The answering agent is asked **once** to revise its reply, following a `note` you write. It revises the text only: it sees what its tools returned while it answered, and none of its tools runs again. The gate then judges the revision: if it passes, the **revision** is sent and the original never is. If the gate objects again — a second `rewrite` or a `hold` — or anything on the way fails, the reply is **held** as above. |

```yaml
      action:
        kind: rewrite
        note: "Remove the discount — only a manager can offer one."   # your words; required
```

**A rewrite never sends text the gate has not passed.** There is one revision per reply, never a loop. If the revision cannot be made (the agent's model fails, is too slow, returns nothing, or tries to call a tool instead of replying — the call is not run), or the gate cannot judge the revision, the reply is held rather than sent. The note is yours: it is shown to the answering agent as guidance for that one revision, and not kept in the conversation.

**What a gate's rule can call.** Everything a `user_message` rule can, except two, because the reply already exists when the gate runs:

- `transfer_to_agent` — it would have another agent answer the same message a second time. Refused when you save it and again when it fires.
- `insert_note` — there is no agent left to brief. Refused when you save it; a `manual` monitor the gate runs has its note refused when it fires, and the reason is recorded.

`transfer_to_team` is allowed: the reply is sent, and the conversation then belongs to the team. Nothing a gate's rule calls makes the AI answer again on this message.

**It judges every reply that would reach the customer** — including the reply of an agent a transfer just handed the conversation to, a retried reply, a scheduled follow-up, and replies in an evaluation run. It never judges a Copilot suggestion or any other reply written for your team, which the customer never sees anyway.

**What changes for a hub with a gate.**

- **No token streaming, no mid-reply updates.** On a streaming app, the reply appears whole once the gate has passed it, instead of word by word. Text the agent would otherwise send while it works ("Let me check that for you…") is not sent. Neither can be taken back once shown, and the gate has not judged them yet.
- **It adds its own latency to every reply**, as a `user_message` monitor does — keep it on a fast model. If its model fails, is misconfigured, returns nothing usable, answers without a variable its rules read, or is too slow, the gate is skipped and **the reply is sent**: a gate that could not judge never leaves the customer unanswered. (A revision the gate cannot judge is the exception: the gate already objected to the original, so it is held.)
- **A rewrite adds a second answering-agent call and a second gate judgement** to that reply, all inside the customer's wait.
- **Cost.** The gate starts no turn of its own; the time it adds is part of the same turn, and a turn is billed for how long it runs.

**What a gate does not cover.** Anything the answering agent's own **tools** send while it works — `send_files` and the message it carries, or an external tool that messages the customer — goes out before any reply exists, so the gate never sees it. Assign such tools with that in mind. Replies from a harness-backed agent are not gated yet.

**One enabled gate per hub**, like the other firing triggers.


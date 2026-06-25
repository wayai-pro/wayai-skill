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
| **Background** | Always (no message routing) | `monitor`, `conversation_evaluator`, `message_evaluator` |

The Pilot agent's response is delivered through the channel; the Copilot agent's response surfaces in the team UI as a suggestion (no channel delivery).

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
| `monitor` | 1 | No (silent observer) | n/a | Excluded from message routing |
| `conversation_evaluator` | 1 | No (async) | n/a | Scores entire conversation after close |
| `message_evaluator` | 1 | No (async) | n/a | Scores each message |

Background roles (`monitor`, evaluators) are excluded from delegation flows — they cannot be the target of `transfer_to_agent` or `consult_agent`.

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
| `include_message_timestamps` | boolean | `false` | When `true`, appends `[2026-04-30 14:30:00 (America/New_York), Wednesday]` to user messages in the LLM history. Useful when temporal context affects responses |
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
  model: claude-sonnet-4-6
  temperature: 0.7
  max_tokens: 4096
  thinking_enabled: true        # optional extended thinking (Claude 4+)
```

**OpenAI** (service: OpenAI):
```yaml
settings:
  model: gpt-5.5
  temperature: 0.7
  max_tokens: 4096
  reasoning_effort: medium      # reasoning models: minimal|low|medium|high|none
```

**Google AI Studio** (service: Google AI Studio):
```yaml
settings:
  model: gemini-3.1-pro
  temperature: 0.7
  max_tokens: 4096
  reasoning_level: high         # dynamic|low|medium|high
```

**OpenRouter** (service: OpenRouter):
```yaml
settings:
  model: openai/gpt-5.5
  temperature: 0.7
  max_tokens: 4096
  reasoning_effort: medium      # reasoning-capable models: low|medium|high|none
```

### Reasoning / thinking by provider

Each provider exposes one reasoning control, named to match its schema. The key name must be exact — a mismatched key (e.g. a nested `thinking:` object on Anthropic, or `max_output_tokens` on Gemini) is **silently stripped on `wayai push`**.

| Connector | Setting | Values | Maps to |
|---|---|---|---|
| Anthropic | `thinking_enabled` (boolean) | `true` / `false` | Adaptive thinking on Claude 4.6+; fixed budget on older 4.x. Temperature/top_p ignored when on. |
| OpenAI | `reasoning_effort` | `minimal` / `low` / `medium` / `high` / `none` | `reasoning.effort` (pairs with `verbosity`). `none` = no reasoning. |
| Google Gemini | `reasoning_level` | `dynamic` / `low` / `medium` / `high` | `thinkingLevel` (Gemini 3.x) or `thinkingBudget` (Gemini 2.5). `dynamic` = model default. |
| OpenRouter | `reasoning_effort` | `low` / `medium` / `high` / `none` | `reasoning.effort`. `none` disables the override. |

### File handling (all LLM connectors)

Two optional `settings` keys control how a conversation's **historical** files are sent to this agent's LLM (the current message's own attachments are always sent in full):

| Setting | Values | Default | Effect |
|---|---|---|---|
| `file_handling_mode` | `always_attach` / `metadata_only` | `always_attach` | `always_attach` includes file content; `metadata_only` sends only an `[Attached files: …]` annotation and the agent fetches content on demand via the `read_file` tool (auto-enabled for the agent in this mode). |
| `max_attachment_size_mb` | integer (MB) | — (no cap) | In `always_attach`, any historical file larger than this is downgraded to metadata-only. |

Background observer roles (`conversation_evaluator`, `message_evaluator`, `monitor`) ignore these settings and always receive full file content.

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

Both `schema_name` and `schema_json` are required when `response_format` is set. The `wayai pull` includes `type: json_schema` for informational consistency — the platform sets it automatically.

Agents with `response_format` set behave like API endpoints — the response is the JSON object, not a chat reply. Useful for evaluators, structured intent classification, and data-extraction agents called via `consult_agent`.

---

## Evaluation Variables

`conversation_evaluator` and `message_evaluator` agents emit structured fields per conversation/message — their **evaluation variables** (stored as agent parameters, surfaced as Analytics columns). They round-trip via `wayai pull` / `wayai push` under an `evaluation_variables` list on the agent:

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
| `label` | string | omitted | Display label (Analytics columns, UI) |
| `enum` | string[] | omitted | Allowed values — required when `type: enum`, ignored otherwise |
| `required` | boolean | `true` | Whether the evaluator must populate it |
| `category` | string | omitted | Analytics grouping: `quality` \| `satisfaction` \| `compliance` \| `categorization` |
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
    operator: "="                # "=" | "!="
    value: no
  - variable: user_sentiment
    operator: "="
    value: negative
```

New hubs seed the two defaults above onto the evaluator agent automatically. Omit the key to leave the current value untouched; set `flag_conditions: []` to clear it. Configurable from the UI in the evaluator agent's detail view (Agents tab) and via the MCP `update_agent` tool. (Previously a hub-level Overview setting — relocated to the evaluator agent.)

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

New hubs seed the default onto the summarizer agent automatically. Omit the key to leave the current value untouched; an empty value (`summarization_threshold_tokens:`) clears it back to the 120000 default. Configurable from the UI in the summarizer agent's detail view (Agents tab) and via the MCP `update_agent` tool. (Previously a hub-level Overview setting — relocated to the summarizer agent.)

---

## Monitor Configuration (`monitor` only)

`monitor_config` is a setting on the **`monitor` agent** — it controls when the background monitor re-evaluates an active conversation and which conditions flag it. `delay_seconds` is the user-inactivity wait (in seconds, minimum 10) before the monitor runs; `flag_conditions` use OR semantics (any match flags the conversation). Unlike the evaluator/summarizer, the monitor is **not** auto-provisioned — create it explicitly. Round-trips via `wayai pull` / `wayai push` as a top-level key on the monitor agent:

```yaml
# agents/monitor.yaml
name: Monitor
role: monitor
connection: anthropic
monitor_config:
  delay_seconds: 300            # inactivity wait before re-evaluation; >= 10
  flag_conditions:
    - variable: user_sentiment  # system metric or evaluation-variable name
      operator: "="             # "=" | "!="
      value: negative
```

Omit the key to leave the current value untouched; set `monitor_config: null` to clear it. Configurable from the UI in the monitor agent's detail view (Agents tab) and via the MCP `update_agent` tool. (Previously a hub-level Overview setting — relocated to the monitor agent.)

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

When `true`, every user message in the LLM history is augmented:

```
[2026-04-30 14:30:00 (America/New_York), Wednesday]
The user's actual message text
```

The hub's `timezone` setting determines the timezone shown; the day of the week is computed from the timestamp.

Use this when:
- The agent reasons about time deltas ("how long since the last message?")
- The agent needs to know business hours / day-of-week
- Followup logic depends on absolute time

Skip this for agents where temporal context is irrelevant — the extra tokens add up over long histories.

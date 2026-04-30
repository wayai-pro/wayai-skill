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
| `pilot` | 1 | Yes (primary on Pilot track) | n/a | Main AI agent for end users |
| `copilot` | 1 | Yes (primary on Copilot track) | n/a | Main AI assistant for the support team |
| `pilot_specialist` | Multiple | Yes (full transfer) | No | Domain expert; receives `transfer_to_agent` |
| `copilot_specialist` | Multiple | Yes (full transfer) | No | Copilot-track specialist |
| `pilot_advisor` | 1 | No (advisory only) | Yes (back to caller) | Receives `consult_agent`; runs once and returns |
| `copilot_advisor` | 1 | No (advisory only) | Yes (back to caller) | Copilot-track advisor |
| `monitor` | 1 | No (silent observer) | n/a | Excluded from message routing |
| `conversation_evaluator` | 1 | No (async) | n/a | Scores entire conversation after close |
| `message_evaluator` | 1 | No (async) | n/a | Scores each message |

Background roles (`monitor`, evaluators) are excluded from delegation flows — they cannot be the target of `transfer_to_agent` or `consult_agent`.

---

## Choosing a Role

| You want… | Use |
|-----------|-----|
| Single AI handling all end users autonomously | `pilot` only |
| AI helping a human team write replies (no channel delivery) | `copilot` only |
| AI handles end users; team can take over and AI shifts to suggesting | `pilot+copilot` mode + `pilot` + `copilot` |
| A specialized handler for billing / refunds / specific intents | Add a `pilot_specialist` (or `copilot_specialist`); transfer to it |
| A domain expert that answers a question and hands back | Add a `pilot_advisor` (or `copilot_advisor`); consult it |
| Conversation quality scoring | `conversation_evaluator` and/or `message_evaluator` |
| Silent monitoring/logging | `monitor` |

---

## Delegation Flow

Two delegation patterns, both expressed as native tools assigned to the calling agent:

**`transfer_to_agent`** — full transfer. The conversation moves to the target agent (specialist). Subsequent user messages go to the specialist until it transfers back or the conversation ends.

```yaml
tools:
  delegation:
    - type: agent
      tool: transfer_to_agent
      target: Specialist - Billing      # display name of the target agent
```

**`consult_agent`** — one-shot advisory. The advisor runs once with the question, returns a response to the caller, and control returns to the caller agent. The user does not see the advisor's output directly.

```yaml
tools:
  delegation:
    - type: agent
      tool: consult_agent
      target: Compliance Advisor
```

**`transfer_to_team`** — agent-to-team handoff. Switches the conversation's `current_responder_type` to `team`. With `ai_mode: pilot+copilot`, the active track shifts from Pilot to Copilot.

```yaml
tools:
  delegation:
    - type: team
      tool: transfer_to_team
      target: Tier 2 Support
```

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
  thinking:                   # optional extended thinking
    type: enabled
    budget_tokens: 8000
```

**OpenAI** (service: OpenAI):
```yaml
settings:
  model: gpt-5
  temperature: 0.7
  max_tokens: 4096
  reasoning_effort: medium    # for reasoning models
```

**Google AI Studio** (service: Google AI Studio):
```yaml
settings:
  model: gemini-2.5-pro
  temperature: 0.7
  max_output_tokens: 4096
```

**OpenRouter** (service: OpenRouter):
```yaml
settings:
  model: anthropic/claude-sonnet-4.6
  temperature: 0.7
  max_tokens: 4096
```

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

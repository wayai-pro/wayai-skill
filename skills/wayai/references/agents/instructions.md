# Agent Instructions (Placeholders)

Agent instructions live in `agents/<slug>.md` files. They support dynamic `{{placeholder()}}` syntax — placeholders are resolved at runtime against live conversation data before the instructions reach the LLM.

## Table of Contents
- [Overview](#overview)
- [Syntax](#syntax)
- [Available Placeholders](#available-placeholders)
- [Examples](#examples)
- [Additional Context (cache-friendly)](#additional-context-cache-friendly)

---

## Overview

Agent instructions support `{{placeholder()}}` syntax. Before the instructions reach the LLM, all placeholders are replaced with live data (current time, user info, conversation state, etc.). This lets you write instructions that adapt to each conversation without hardcoding values.

Placeholders are processed in:
- Agent instructions (uploaded via `upload_agent_instructions`)
- Prepend instructions (injected at conversation start)
- Kanban followup instructions
- **Additional Context** — see below

---

## Syntax

**Basic call** — returns the full result as text:
```
{{now()}}
```

**With arguments** — passes parameters to the function:
```
{{now(America/New_York)}}
{{previous_conversations(3)}}
{{state(conversation, cart)}}
```

**Path access** — extracts a specific property from the JSON result:
```
{{now().weekday}}
{{user_info().user_email}}
{{state(conversation, cart).total}}
```

**Array access** — navigates into arrays:
```
{{previous_conversations(1).conversations[0].message_text}}
```

**Rules:**
- Context parameters (`hub_id`, `user_id`, `conversation_id`, `agent_id`) are auto-injected — never pass them manually
- If a placeholder function fails, it's silently removed from the text. If a property path doesn't exist, it's replaced with an empty string
- A placeholder can appear multiple times in the same text — all occurrences are replaced

---

## Available Placeholders

### `{{now()}}`

Current date and time information.

| Argument | Default | Description |
|----------|---------|-------------|
| `timezone` | `America/Sao_Paulo` | IANA timezone |

| Property | Example |
|----------|---------|
| `.timestamp` | `2024-01-15T14:30:00Z` |
| `.iso_string` | `2024-01-15T14:30:00Z` |
| `.date` | `2024-01-15` |
| `.time` | `14:30:00` |
| `.weekday` | `Monday` |
| `.time_of_day` | `morning` / `afternoon` / `evening` / `night` |
| `.timezone` | `America/Sao_Paulo` |

```
Today is {{now().weekday}}, {{now().date}}.
Good {{now().time_of_day}}!
Current time in NYC: {{now(America/New_York).time}}
```

---

### `{{user_name()}}`

User's display name.

| Property | Description |
|----------|-------------|
| `.user_name` | Profile name |

```
The user's name is {{user_name().user_name}}.
```

---

### `{{user_info()}}`

Extended user profile data.

| Property | Example |
|----------|---------|
| `.user_name` | `João Silva` |
| `.user_phone` | `+5511999999999` |
| `.user_email` | `joao@example.com` |
| `.user_timezone` | `America/Sao_Paulo` |
| `.user_language` | `pt` |

```
Respond in the user's language: {{user_info().user_language}}.
User timezone: {{user_info().user_timezone}}.
```

---

### `{{state()}}`

Conversation-level or user-level state values (defined in Hub > State).

| Argument | Required | Description |
|----------|----------|-------------|
| `scope` | Yes | `conversation` or `user` |
| `state_name` | Yes | Name as defined in the state table |

Returns the full state JSON. Use path access for specific fields.

```
{{state(conversation, cart)}}
Cart total: {{state(conversation, cart).total}}
User preference: {{state(user, preferences).language}}
```

---

### `{{resources()}}`

Markdown-formatted metadata for resources (knowledge bases and skills) linked to the agent.

| Argument | Default | Description |
|----------|---------|-------------|
| `resource_type` | `all` | `all`, `kb` (knowledge only), or `skill` (skills only) |

Returns a Markdown block with resource names, IDs, descriptions, and file counts. If `include_structure_in_prompt` is enabled on the resource, also includes folder/file listings.

```
{{resources()}}
{{resources(kb)}}
{{resources(skill)}}
```

---

### `{{agent_skills()}}`

XML-formatted skill metadata for skills linked to the agent. Used for progressive disclosure — the agent sees available skills and can load them on demand.

Returns:
```xml
<available_skills>
<skill>
  <skill_id>abc-123</skill_id>
  <name>order-processing</name>
  <description>Handles pizza order workflow</description>
</skill>
</available_skills>
```

```
{{agent_skills()}}
```

---

### `{{previous_conversations()}}`

Previous ended conversations for the same user in this hub.

| Argument | Default | Description |
|----------|---------|-------------|
| `conversation_limit` | `5` | Max conversations to return |

Returns JSON with conversations array, each containing messages (sender_type, message_text, created_at).

```
{{previous_conversations(3)}}
Last message from user: {{previous_conversations(1).conversations[0].message_text}}
```

---

### `{{event_info()}}`

Event information from conversation events (scheduling, appointments).

| Argument | Default | Description |
|----------|---------|-------------|
| `kanban_status` | *(none)* | Filter by status (e.g., `scheduled`, `confirmed`) |

| Property | Example |
|----------|---------|
| `.event_name` | `Consultation with Dr. Silva` |
| `.event_description` | `Initial consultation` |
| `.event_sid` | External calendar event ID |
| `.datetime_formatted` | `January 15, 2024 at 02:30 PM` |
| `.weekday` | `Monday` |
| `.is_today` | `true` / `false` |
| `.is_tomorrow` | `true` / `false` |
| `.days_until` | `3` |

```
{{event_info(scheduled).event_name}} is on {{event_info(scheduled).weekday}}.
{{event_info(confirmed).datetime_formatted}}
```

---

### `{{agent_settings()}}`

Another agent's instructions and tools configuration (by agent name).

| Argument | Required | Description |
|----------|----------|-------------|
| `agent_name` | Yes | Name of the agent in the same hub |

| Property | Description |
|----------|-------------|
| `.agent_id` | Agent UUID |
| `.agent_name` | Agent name |
| `.instructions` | Processed instructions (with placeholders resolved) |
| `.tools_config` | JSON array of enabled tool configs |
| `.reasoning_effort` | Reasoning effort setting |
| `.verbosity` | Verbosity setting |

```
{{agent_settings(Pilot).instructions}}
```

---

### `{{eval_info()}}`

Evaluation test case data for evaluator agents.

| Argument | Required | Description |
|----------|----------|-------------|
| `eval_name` | Yes | Eval name or UUID |

| Property | Description |
|----------|-------------|
| `.eval_id` | Eval UUID |
| `.message_text` | Test message |
| `.actual_response` | Agent's actual response |
| `.expected_response` | Expected response |
| `.message_history` | Conversation history for context |
| `.agent_settings` | Responder agent's settings |

```
{{eval_info(greeting-test)}}
```

---

### `{{current_conversation_id()}}`

The current conversation UUID.

| Property | Description |
|----------|-------------|
| `.conversation_id` | Conversation UUID |

```
Conversation ID: {{current_conversation_id().conversation_id}}
```

---

## Examples

### Personalized greeting with time awareness

```markdown
You are a friendly assistant for Acme Corp.

Current time: {{now()}}
User: {{user_name().user_name}}

Greet the user by name and adjust your tone to the time of day ({{now().time_of_day}}).
```

### Stateful order assistant

```markdown
You help customers manage their orders.

## Current Cart
{{state(conversation, cart)}}

## User Preferences
{{state(user, preferences)}}

Use the cart state to answer questions about the current order.
If the cart is empty, suggest popular items.
```

### Context-aware support with history

```markdown
You are a support agent.

## User Info
- Name: {{user_name().user_name}}
- Language: {{user_info().user_language}}
- Timezone: {{user_info().user_timezone}}

## Previous Conversations
{{previous_conversations(3)}}

Always respond in the user's language. Use previous conversations for context — don't ask questions already answered before.
```

### Appointment reminder agent

```markdown
You are a scheduling assistant.

## Upcoming Event
- Event: {{event_info(scheduled).event_name}}
- Date: {{event_info(scheduled).datetime_formatted}} ({{event_info(scheduled).weekday}})
- Days until: {{event_info(scheduled).days_until}}

Confirm the appointment details with the user and offer to reschedule if needed.
```

---

## Additional Context (cache-friendly)

Agents have a separate `additional_context_template` field — same `{{...}}` syntax as instructions, but the resolved output is emitted as a `<additional_context>` XML tag prepended to the **last user message** on every turn instead of being substituted inline into the system prompt.

**Why this exists.** Anthropic prompt caching matches the cached prefix byte-for-byte. Any placeholder that changes per turn (`{{now()}}`, `{{state(...)}}`) inside `instructions` invalidates the cached system prompt every turn, so cache reads never land. Moving those high-churn values into `additional_context_template` keeps the system prompt stable and lets the cache absorb the agent's identity, tools, and skills — typical cache-read savings are large because tools + agent definitions are most of the prefix.

**When to use which:**
- `instructions` — stable agent identity, tone, rules. Static placeholders are fine here (`{{user_info()}}`, `{{previous_conversations(N)}}`).
- `additional_context_template` — anything that changes per turn or per minute. `{{now()}}` always belongs here; `{{state(...)}}` belongs here unless the agent's reasoning hard-depends on reading state from inline prose.

**YAML / agent file:**

```yaml
agents:
  - name: Support Pilot
    role: pilot
    connection: anthropic-prod
    instructions: agents/support-pilot.md
    additional_context_template: |
      Current time: {{now()}}
      Open tickets: {{state(user, open_tickets)}}
```

**Behavior:**
- Empty / whitespace-only → no tag emitted (fully backward-compatible)
- The tag sits next to existing state tags (`<conversation_state>`, `<user_state>`) on the last user turn
- Evaluator agents (`message_evaluator`, `conversation_evaluator`) do NOT receive the tag — they score raw history
- All `{{...}}` placeholders documented above work identically in this field


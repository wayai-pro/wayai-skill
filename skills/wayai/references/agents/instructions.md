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

**Array access** — navigates into arrays. The path always starts with a dot, so
indexing a function that itself returns an array reads `.[0]`:
```
{{previous_conversations(1).[0].summary}}
```

**Rules:**
- Context parameters (`hub_id`, `user_id`, `conversation_id`, `agent_id`) are auto-injected — never pass them manually
- If a placeholder can't resolve — unknown function, wrong args, or a missing property path — it is **left literal**: the raw `{{...}}` text stays in the prompt (deliberate, so a bad path is visible to the author). The one exception is `{{var(...)}}`: a missing subfield of a *present* variable renders as empty string
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

### `{{week_horizon()}}`

Upcoming days as a `date → weekday` lookup table, in the hub timezone. Use it so the agent **reads** the weekday of a date instead of computing it (which LLMs do unreliably) — e.g. when proposing or confirming scheduling dates.

| Argument | Default | Description |
|----------|---------|-------------|
| `days` | `8` | Number of days to list, including today (capped at 31) |

```
Available days:
{{week_horizon()}}

Next two weeks: {{week_horizon(14)}}
```

Output — one line per day, with `today:` / `tomorrow:` prefixes on the first two:

```
today: 2026-06-11/Thursday
tomorrow: 2026-06-12/Friday
2026-06-13/Saturday
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
| `state_slug` | Yes | The state's immutable slug (`hub.yaml` `states[].slug`) |

Returns the full state JSON. Use path access for specific fields. A wrong scope or slug — or a state that has no written value and no `initial_value` — leaves the placeholder **literal** in the prompt. (`{{state(scope)}}` returns that scope's full object; a bare `{{state()}}` returns the conversation scope.)

```
{{state(conversation, cart)}}
Cart total: {{state(conversation, cart).total}}
User preference: {{state(user, preferences).language}}
```

---

### `{{resources()}}`

Markdown-formatted catalog of the resources (knowledge bases and skills) linked to the current agent.

| Argument | Default | Description |
|----------|---------|-------------|
| `resource_type` | `all` | `all`, `kb` (knowledge only), or `skill` (skills only) |

Returns a Markdown list — one bullet per resource with its name, type, id, description, and file count. When the agent's link to a resource has `include_structure_in_prompt` enabled, each file's canonical path (`resources/<resource-slug>/<folders…>/<file-name>`) is listed underneath. Those paths are addresses, not labels: the agent can pass one straight to `read_file` with no listing round-trip, and unlike an id a path survives publishing (which regenerates every UUID). An agent with no linked resources renders a short "(no resources are linked to this agent)" note. This is a prompt-side catalog; the `list_resource_folders` / `list_resource_files` native tools also expose these ids for on-demand browsing (see [native-tools.md](native-tools.md)).

```
{{resources()}}
{{resources(kb)}}
{{resources(skill)}}
```

---

### `{{agent_skills()}}`

XML-formatted skill metadata for the skills linked to the current agent. Used for progressive disclosure — the agent sees available skills and can load them on demand. Draws from the same linked-resource set as `{{resources(skill)}}`. The `read_skill` / `read_skill_file` native tools also list these skill ids directly in their `skill_id` parameter (see [native-tools.md](native-tools.md)).

Returns (empty `<available_skills></available_skills>` when no skills are linked):
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

### `{{resource_content(NAME)}}`

Inlines the full text content of a resource linked to the current agent, resolved by name.

| Argument | Required | Description |
|----------|----------|-------------|
| `resource_name` | Yes | The resource's `resource_name` (or `skill_name` for a skill) |

For a skill, returns its `SKILL.md`. For a knowledge base, returns its files concatenated, each headed by its canonical path — `## resources/company-faq/refunds.md`. The heading is an address, not a label: the agent can pass it straight to `read_file`, and it is the same path `{{resources()}}` and `list_resource_files` report for that file. Content is fetched per turn and capped (~100 KB per resource) to bound prompt size; an unknown or unlinked name leaves the placeholder literal. Reach for `read_skill` / `list_resource_files` when the agent should fetch on demand instead of always inlining the whole resource.

> **Note:** content is inlined **raw and unescaped** into the system prompt (like `{{state()}}`), so only reference resources you trust — a knowledge base populated from externally-authored documents is placed verbatim into the agent's instructions.

```
{{resource_content(refund-policy)}}
```

---

### `{{previous_conversations()}}`

> **Prefer the `previous_conversations_count` agent setting** for the common case of "give this agent the user's recent history". It resolves the same rows but injects them into the conversation's first user message and freezes them there, which keeps the hub-wide cached prefix intact and stops the block being re-paid every turn. See [roles-and-settings.md](roles-and-settings.md#previous-conversations-context). Reach for this placeholder only when you need the rows at a specific point in a template.
>
> Putting this placeholder in `instructions` is the costly case: the cached system prefix covers the instructions AND every tool schema and is shared across the whole hub, so per-user content there makes each conversation pay to WRITE that prefix instead of reading a shared one.

Previous ended conversations for the same user in this hub, newest first.

| Argument | Default | Description |
|----------|---------|-------------|
| `conversation_limit` | *(required)* | Max conversations to return (capped at 20) |

Renders a JSON array of `{ conversation_id, summary, ended_at }`. It is an index,
not a transcript: give the agent the `get_conversation` tool and it can pull the
full text of any `conversation_id` listed here.

`summary` is the **Conversation Evaluator**'s summary output, written after the
conversation ends. It is therefore absent when:

- the hub's evaluator agent is disabled or produced no summary;
- the conversation ended **before this feature shipped** — there is no backfill, so
  existing hubs see summaries only on conversations closed from that point on;
- the conversation was reaped past the hub's `ended_index_retention_days`, which
  bounds how far back this placeholder reaches.

Long summaries are truncated. `conversation_id` and `ended_at` are always present,
so an agent can still reach for `get_conversation` when the summary is missing.

```
{{previous_conversations(3)}}
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

> **Note:** this placeholder is not yet populated at runtime — an unresolved `{{agent_settings(NAME)}}` is left as the literal token (tracked separately). For the *current* agent's resources and skills, use `{{resources()}}` / `{{agent_skills()}}` instead.

| Argument | Required | Description |
|----------|----------|-------------|
| `agent_name` | Yes | Name of the agent in the same hub |

| Property | Description |
|----------|-------------|
| `.agent_id` | Agent UUID |
| `.agent_name` | Agent name |
| `.instructions` | Processed instructions (with placeholders resolved) |
| `.tools_config` | JSON array of enabled tool configs |
| `.reasoning_effort` | Reasoning effort setting (OpenAI / OpenRouter / xAI) |
| `.reasoning_level` | Reasoning level setting (Google Gemini) |
| `.verbosity` | Verbosity setting |

```
{{agent_settings(Pilot).instructions}}
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

> For the full **lifetime model** (timeless → `instructions`, cross-turn → `state`, this-turn → here) and why churning **tool schemas** also busts the cache, see "Context placement" in [`prompt-principles.md`](prompt-principles.md).

**When to use which:**
- `instructions` — stable agent identity, tone, rules. Static placeholders are fine here (`{{user_info()}}`). **Not `{{previous_conversations(N)}}`** — it is static per turn but PER USER, which forks the hub-wide cached prefix; see below.
- `additional_context_template` — anything that changes per turn or per minute. `{{now()}}` always belongs here; `{{state(...)}}` belongs here unless the agent's reasoning hard-depends on reading state from inline prose.

**Two ways to give temporal context — pick by granularity (both are cache-safe and both replay in evals):**
- `{{now()}}` in `additional_context_template` (here) — **one "now" per turn**, prepended to the last user message. Resolves at render time, so it's present in eval replay. Use when the agent only needs *the current time*.
- `include_message_timestamps: true` (see [`roles-and-settings.md`](roles-and-settings.md#include_message_timestamps-behavior)) — **per-message** `[timestamp, weekday, daypart]` appended to every user message. Use when the agent reasons about *when each* message arrived or deltas between messages.

Both sit outside the cached system-prompt prefix, so neither busts the cache. If you only need a single per-turn clock, prefer `{{now()}}` — it's the leaner one.

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


# Custom Tools

Create custom API integrations for agents using REST API connections (`type: Tool`, `service: REST API`).

## Table of Contents
- [Overview](#overview)
- [Creating a Custom Tool](#creating-a-custom-tool)
- [Tool Configuration](#tool-configuration)
- [Placeholder System](#placeholder-system)
- [composed_tools (side effects on success)](#composed_tools-side-effects-on-success)
- [Examples](#examples)

---

## Overview

Custom tools allow agents to call external APIs. They require:
1. A **REST API connection** (API Key or Basic Auth) — defined in `hub.yaml` connections section (auto-created from org credentials during push) or created via UI. Give it the API's **base URL**: a connection's credential is only sent to its base URL's origin, so a connection that holds a credential takes no tool until it has one, and moving the base URL to another origin clears the credential ([Moving a connection's base URL](../connections.md#moving-a-connections-base-url))
2. A **custom tool** attached to an agent — defined in `agents/<slug>.yaml` under the agent's `tools.custom` section

---

## Creating a Custom Tool

### Via Files + CLI (Recommended)

Add custom tools to the agent's YAML file (`agents/<slug>.yaml`) under `tools.custom`, and declare the connection in `hub.yaml`:

```yaml
# agents/order-agent.yaml
name: Order Agent
role: Pilot
connection: anthropic
tools:
  custom:
    - name: get_order_status
      description: Retrieve the current status of a customer order
      method: get
      path: /orders/{{order_id}}
      connection: my-api-connection    # Must match a connection name
    - name: create_ticket
      description: Create a new support ticket
      method: post
      path: /tickets
      body_format: json
      connection: my-api-connection
```

```yaml
# hub.yaml (connections section)
connections:
  - name: my-api-connection
    type: Tool
    service: REST API
    base_url: https://api.example.com   # Required for tools on a connection that holds a credential
```

Then run `wayai push` to create the tools on the hub.

### Via UI

1. Hub → Agents → Select agent
2. Tools → Add Tool → Custom
3. Fill in configuration fields

---

## Tool Configuration

| Field | Description |
|-------|-------------|
| `tool_name` | Display name for the tool |
| `tool_method` | HTTP method: GET, POST, PUT, DELETE, PATCH |
| `tool_path` | URL path — either relative to connection base_url (e.g., `/orders/{{order_id}}`) or a full URL (e.g., `https://api.example.com/orders`). Full URLs are used as-is, ignoring base_url. A connection's credential is only ever sent to its base URL's origin (scheme, host and port), never past a redirect off it: on a connection that holds a credential, creating a tool whose URL leaves that origin, or editing a tool's path or connection so that it does, is refused (UI or `wayai push`); an existing such tool can still be edited in every other field, and its calls go out without the credential. A connection with no base URL takes no such tool — nor does one whose base URL has no fixed host (a `{{placeholder}}` host, or no `https://` or `http://` scheme), which has no origin, so its credential is sent nowhere. Publishing, syncing or replicating a hub copies an existing such tool as it is, and its calls go out without the credential. Supports `{{placeholders}}` |
| `tool_config` | OpenAI function schema (auto-generated or custom) |
| `tool_headers` | Additional headers array: `[{key, value}]` |
| `tool_body_format` | Body encoding: `json` (default) or `form` (`application/x-www-form-urlencoded`) |
| `tool_instructions` | AI guidance for when/how to use the tool |
| `keep_in_history` | When `true`, this tool's call and result are kept in the agent's message history across turns. Default `false` — call/result are dropped after the turn to keep context lean. Enable for tools whose output the agent should reference later (e.g. fetched order details). When invoked via a meta tool (`get_tool_schema` or `execute_tool`), this flag AND-combines with the meta tool's own — both must be `true` to remain in history. See [native-tools.md](native-tools.md#meta-tools). |
| `expose_schema_to_llm` | When `false`, the tool's schema is excluded from the agent's inline tool list. Default `true`. This is a prompt-token optimization — the tool stays executable via `execute_tool` (no access-control gating). Set this on the deferred tool itself; the meta tools stay exposed inline. See [native-tools.md](native-tools.md#meta-tools). |
| `idempotency_header` | A header your API reads to recognize a repeated request (e.g. `Idempotency-Key`). A write's call (any method but GET, HEAD, OPTIONS) that the platform records sends a key in it, unique to that call, so your API can tell a repeat of the call from a new one. A header name: letters, digits and dashes, at most 64; not one the platform sets (`Authorization`, `Content-Type`, `Host`, `Cookie`, …). The key replaces a `tool_headers` entry of the same name. See [If a call is interrupted](#if-a-call-is-interrupted). |
| `replay_override` | `rerun` or `reconcile`: what may follow a call of this tool whose outcome was lost, when you know better than its method. See [If a call is interrupted](#if-a-call-is-interrupted). |
| `irreversible` | When `true`, a call whose outcome was lost is never repeated automatically: a person decides. Default `false`. |

### If a call is interrupted

A turn can be interrupted while a tool call is out (a restart, say), and then whether the call acted
is not known. The platform records, in an agent's turn, each call that may act before it runs, and
what came back after. An interrupted turn can resume from its last finished step, and the calls it
finished are not run again. The turn a customer's message starts resumes this way on every channel,
and so do an agent's turn after a transfer to it, a Copilot suggestion your team asked for, an agent's turn when a consult it started comes back, the turn
an automation, a scheduled trigger, a delegation from another hub or a system message starts (its
instructions, and the data they came with, still in front of the agent), and the turn a scheduled
follow-up or an eval run starts: the follow-up's instructions are still in front of
the agent, the follow-ups after it are still scheduled, and the schedule shows as fired once. When
that consult's turn would have to wait behind another turn, or another turn has answered since, the
consult's answer (or the notice that it timed out) goes to your team instead. A turn interrupted
after its agent wrote its reply goes on from there too: the reply is stored once and spoken once (a
reply interrupted while it was being spoken is spoken again, and charged again by your speech
provider), the work after it (the inactivity follow-ups it schedules, the agent it transfers to) is
done once, and the reply is sent once. When the interruption may have cut its send, it is sent again once: on
Telegram and Instagram marked as a possible duplicate; on WhatsApp only once Meta has not confirmed
the first within a minute, then marked the same way; by email under the same key, so it arrives
once. The notice that a conversation was handed to your team is sent the same way when an
interruption cuts it. A turn the platform cannot resume safely is handed to your team instead, and
nothing is run again: one interrupted while its `user_message` monitor was judging the message or
while a harness-backed agent's turn was starting, one interrupted while a reply check
(an `assistant_reply` monitor) was judging its reply or while an eval run's reply was being scored,
one interrupted after its agent wrote its reply and before the reply was stored that must wait
behind another turn (or that another turn has answered since),
a turn that continues the one before it with files that turn fetched, and one already resumed three
times. An automation's, a trigger's, a
delegation's or a system message's turn that cannot resume is handed to your team only in the first three
of those cases; otherwise (one already resumed three times, say) it is run again from the conversation's
history, or, for a system message the history does not keep, it is not run again and your team is told
it was interrupted. A customer's message's turn that cannot resume is handed to your team in those same
three cases too; otherwise its message is answered by a new turn, as when it first arrived (also when,
interrupted before its agent wrote its reply, the turn would have to wait behind another turn, or another
turn has answered since), unless its voice message or files were not stored before the interruption, when
it is handed to your team. An eval run interrupted while
it was being started fails, and is not run again. A resumed turn does not transcribe a voice message or run its `user_message`
monitor again: it keeps what the interrupted turn got from them. Neither does a new turn that answers the
interrupted turn's message in its place: one started when the message is delivered again (it transcribes
the voice message again only when the interrupted turn had not stored it yet), or one that answers it after
the platform dispatched it again (alone, or together with what arrived meanwhile). A
message delivered again while its interrupted turn's `user_message` monitor was judging it starts no new
turn: that turn is handed to your team. These settings classify the tool for
what a resumed turn does with a lost call:

| What the tool is | What a resumed turn does with a lost call |
|------------------|-------------------------------------------|
| A read (`GET`, `HEAD`, `OPTIONS`), or `replay_override: rerun` | Runs it again. |
| A write with an `idempotency_header` | Runs it again with the same key, which your API recognizes as the repeat. |
| Any other write, or `replay_override: reconcile` | Never runs it again. The agent is told the call's outcome is unknown, and goes on from there. |
| `irreversible: true` | Never runs it again. Unless the platform had recorded that the call took effect (the agent is then told so), the conversation goes to your team, whose notice names the call and its arguments, so a person checks whether it took effect. |

In the resumed turn, while a lost call's outcome is unknown, the agent's identical call (the same tool,
the same arguments) is not run either: it is answered that an identical call's outcome is unknown.

When the interrupted turn is not resumed but its message is answered by another turn instead (run again
from the conversation's history; a customer's message answered by a new turn, as above, alone or together
with what arrived meanwhile; or a turn on a message the customer sent afterwards, whose conversation
history holds the interrupted turn's message), that turn is shown each call the interrupted turn made of
a tool that is not a read or `replay_override: rerun`, where it was made, with what is known of it: that
it took effect (with what your API answered), that it did not, or that its outcome is unknown. None of
these calls is run again by the platform. While a call's outcome is unknown, that turn's identical call
is answered as in a resumed turn; an `irreversible: true` call whose outcome is unknown sends the
conversation to your team, as above, before the new turn's agent runs.

Any other later turn is never refused: there the identical call runs, so a write your API cannot
recognize as a repeat (no `idempotency_header`) can then act twice. A call that failed in a turn that was
not interrupted (your API answered 5xx, say) is not affected: retrying it runs it.

Set `replay_override: rerun` only on a tool whose second run leaves what the first left (a lookup sent
as POST, an update that sets a fixed value). A write with an `idempotency_header` keeps sending its key
under `replay_override: rerun`; `replay_override: reconcile` takes precedence over the header. All
three are a custom tool's fields, in its YAML and in its form in the console.

```yaml
tools:
  custom:
    - name: create_order
      method: post
      path: /orders
      connection: shop-api
      idempotency_header: Idempotency-Key   # each call sends its own key in this header
    - name: refund_payment
      method: post
      path: /refunds
      connection: shop-api
      irreversible: true                    # never run again on its own after a lost call
```

### tool_config Schema

OpenAI function calling format. Required fields: `name`, `description`, `parameters`.

```json
{
  "name": "get_order_status",
  "description": "Get the status of a customer order",
  "parameters": {
    "type": "object",
    "properties": {
      "order_id": {
        "type": "string",
        "description": "The unique order identifier"
      }
    },
    "required": ["order_id"]
  }
}
```

> **Note:** `tool_name` and `tool_description` are auto-populated from `tool_config.name` and `tool_config.description` by a database trigger.

---

## Placeholder System

Placeholders are replaced at runtime in URLs, headers, query params, and body.

| Placeholder | Source | Description |
|-------------|--------|-------------|
| `{{api_key}}` | Connection secrets | API key from REST API connection |
| `{{access_token}}` | Connection secrets | Access token (for dual-credential APIs) |
| `{{param_name}}` | AI parameters | Value provided by AI during tool call |

### Where Placeholders Work

- **URL/Path:** `/orders/{{order_id}}/status`
- **Headers:** `Authorization: Bearer {{api_key}}`
- **Query params:** `?token={{api_key}}&id={{order_id}}`
- **Body params:** Merge additional params with AI-provided body

---

## composed_tools (side effects on success)

Any custom tool can declare `composed_tools` — an array of native tools to execute as side effects after successful execution. Each entry maps parameters from the agent's call or the tool's response onto the native tool's schema.

**Whitelist of native tools usable as side effects:** `update_kanban_status`, `update_state`, `set_state_path`, `schedule_followup`, `close_conversation`.

`close_conversation` takes an `outcome` when the hub's terminal kanban status declares `outcomes` — map it (`outcome: params.<field>`, or a literal such as `outcome: resolved`). Without one the composed close is refused with `invalid_kanban_outcome`, because an agent close executes as the terminal kanban transition — unless the conversation is a re-close that already has an outcome recorded, in which case the stored value is reused and the omission is accepted. A mapped `outcome` is **omitted from the injected schema entirely** when the hub configures no outcomes — there is nothing for the model to choose, so the parameter would be noise. A mapped `new_kanban_status` is always injected; only its enum drops when no status is `allowsAgentUpdate`. In that configuration the parameter is still **required** but carries no allowed values, so use the object mapping form and give it a `description` naming the slugs — with the bare `params.X` string form the model has nothing to go on and the transition gate rejects whatever it guesses.

**Entry fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `tool_name` | string | Yes | Native tool to execute (whitelist above) |
| `mappings` | object | Yes | Parameter resolution map (key = native tool param, value = mapping string or object) |
| `appended_description` | string | No | Text appended to the primary tool's description for the AI |
| `success_path` | string | No | Dot-notation path checked in the tool result for business-logic success. Falsy → composed tool skipped |

**Mapping values:**

| Format | Behavior | Example |
|--------|----------|---------|
| `params.X` | Inject parameter `X` into the primary tool's schema; extract from agent's call | `params.target_status` |
| `result.X` | Extract value at path `X` from the tool's response | `result.id` |
| `result` | Use the entire tool response | `result` |
| (other string) | Literal/fixed value | `conversation` |
| Object `{source, type?, description?}` | Same as string source, with type/description overrides for the injected parameter | See below |

**Examples:**

Auto kanban update (simple):
```yaml
composed_tools:
  - tool_name: update_kanban_status
    mappings:
      new_kanban_status: params.target_status
```

With `success_path` (skip on business-logic failure):
```yaml
composed_tools:
  - tool_name: update_kanban_status
    success_path: success
    mappings:
      new_kanban_status: params.target_status
```

Mapping overrides (custom type/description for injected params):
```yaml
composed_tools:
  - tool_name: update_kanban_status
    success_path: data.success
    mappings:
      new_kanban_status: params.target_status
      scheduled_event_date:
        source: params.data_agendamento_iso
        type: string
        description: "Data e hora em formato ISO 8601 (ex: 2026-03-23T10:00:00-03:00)"
```

Kanban + state update (calendar tool):
```yaml
composed_tools:
  - tool_name: update_kanban_status
    appended_description: Select the workflow status based on the action taken
    mappings:
      new_kanban_status: params.target_status
      scheduled_event_date: params.event_date
      event_sid: result.id
      event_description: result.summary
  - tool_name: update_state
    mappings:
      state_slug: appointment
      updates: result
```

---

## Examples

### Example 1: Simple GET endpoint

**Use case:** Fetch order status from an e-commerce API.

```yaml
# agents/support-agent.yaml
name: Support Agent
tools:
  custom:
    - name: get_order_status
      description: Retrieve the current status of a customer order by order ID
      path: /orders/{{order_id}}
      method: get
      connection: my-api
```

```yaml
# hub.yaml (connections section)
connections:
  - name: my-api
    type: Tool
    service: REST API
    base_url: https://api.example.com/v1
```

**Connection setup (UI, or the org credential `wayai push` binds):**
- Auth type: API Key
- Base URL: `https://api.example.com/v1` (on the connection: `base_url:` in `hub.yaml`, or the connection form)
- API Key: `sk-xxx`

**Headers (connection level):**
```json
{"Authorization": "Bearer {{api_key}}"}
```

**Resulting request:**
```
GET https://api.example.com/v1/orders/ORD-12345
Authorization: Bearer sk-xxx
```

### Example 2: POST with body

**Use case:** Create a support ticket in an external system.

```yaml
# agents/support-agent.yaml
name: Support Agent
tools:
  custom:
    - name: create_ticket
      description: Create a new support ticket with customer issue details
      path: /tickets
      method: post
      body_format: json
      connection: my-api
```

**AI call generates:**
```
POST https://api.example.com/v1/tickets
Content-Type: application/json
Authorization: Bearer sk-xxx

{
  "subject": "Cannot access my account",
  "description": "Customer reports login issues since yesterday",
  "priority": "high"
}
```

### Example 3: Deferred schema (hidden from inline tool list)

**Use case:** A rarely-used reporting endpoint. Keep its schema out of the agent's prompt to save tokens, but still allow execution via meta tools.

```yaml
# agents/support-agent.yaml
name: Support Agent
tools:
  native:
    - name: get_tool_schema     # Required to discover hidden tools
      keep_in_history: true     # so the schema stays visible after the fetch turn
    - execute_tool              # Required to invoke hidden tools
  custom:
    - name: generate_monthly_report
      description: Generate a detailed monthly account report (rarely needed)
      path: /reports/monthly
      method: post
      body_format: json
      connection: my-api
      expose_schema_to_llm: false   # Schema excluded from inline tool list
```

The agent doesn't see `generate_monthly_report` in its tool list each turn. To use it, the agent calls `get_tool_schema(tool_name="generate_monthly_report")` to fetch the schema, then `execute_tool(tool_name="generate_monthly_report", …)` to run it. The schema lookup is agent-scoped — the meta tools only resolve tools assigned to the calling agent. `expose_schema_to_llm: false` is a token optimization, not access control: any agent with `execute_tool` assigned can still invoke the tool by name. See [native-tools.md](native-tools.md#meta-tools) for history-retention rules and the full setup.

### Example 4: Dual-credential API

**Use case:** API requiring both API key and access token.

**Connection setup (UI):**
- Connector: REST API (`type: Tool`, `service: REST API`) · Auth type: API Key
- Base URL: `https://api.service.com`
- API Key: `key-xxx`
- Access Token: `token-yyy`

**Headers (connection level):**
```json
{
  "X-API-Key": "{{api_key}}",
  "Authorization": "Bearer {{access_token}}"
}
```

---

## Quick Reference

| Operation | Method |
|-----------|--------|
| Create | Edit `agents/<slug>.yaml` → `wayai push` |
| Update | Edit `agents/<slug>.yaml` → `wayai push` |
| Delete | Remove from `agents/<slug>.yaml` → `wayai push` |
| View | `wayai pull` to refresh local files |

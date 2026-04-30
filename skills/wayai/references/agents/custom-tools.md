# Custom Tools

Create custom API integrations for agents using `Tool - Custom` connections.

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
1. A **Tool - Custom connection** (API Key or Basic Auth) — defined in `hub.yaml` connections section (auto-created from org credentials during push) or created via UI
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
    type: Tool - Custom
    service: User Tool - API Key
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
| `tool_path` | URL path — either relative to connection base_url (e.g., `/orders/{{order_id}}`) or a full URL (e.g., `https://api.example.com/orders`). Full URLs are used as-is, ignoring base_url. Supports `{{placeholders}}` |
| `tool_config` | OpenAI function schema (auto-generated or custom) |
| `tool_headers` | Additional headers array: `[{key, value}]` |
| `tool_body_format` | Body encoding: `json` (default) or `form` (`application/x-www-form-urlencoded`) |
| `tool_instructions` | AI guidance for when/how to use the tool |
| `keep_in_history` | When `true`, this tool's call and result are kept in the agent's message history across turns. Default `false` — call/result are dropped after the turn to keep context lean. Enable for tools whose output the agent should reference later (e.g. fetched order details). When invoked via a meta tool (`get_tool_schema` or `execute_tool`), this flag AND-combines with the meta tool's own — both must be `true` to remain in history. See [native-tools.md](native-tools.md#meta-tools). |
| `expose_schema_to_llm` | When `false`, the tool's schema is excluded from the agent's inline tool list. Default `true`. This is a prompt-token optimization — the tool stays executable via `execute_tool` (no access-control gating). Set this on the deferred tool itself; the meta tools stay exposed inline. See [native-tools.md](native-tools.md#meta-tools). |

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
| `{{api_key}}` | Connection secrets | API key from User Tool connection |
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
      state_scope: conversation
      state_name: appointment
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
    type: Tool - Custom
    service: User Tool - API Key
```

**Connection setup (UI — org credential):**
- Auth type: API Key
- Base URL: `https://api.example.com/v1`
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
- Type: User Tool - API Key
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

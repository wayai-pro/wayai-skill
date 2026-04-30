# Native Tools

Tools provided by native connectors. Added to agents by listing them under `tools.native` in `agents/<slug>.yaml` and running `wayai push`.

## YAML form

Two forms are accepted under `tools.native`:

```yaml
tools:
  native:
    - send_text_message                                        # short form: catalog defaults
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
- [Resource Tools](#resource-tools)
- [Google Calendar Tools](#google-calendar-tools)
- [Meta Tools](#meta-tools)
- [MCP Client Tools](#mcp-client-tools)

---

## Conversation Tools

**Connector:** Wayai
**connector_id:** `b17d9f3a-4e1b-46c9-b648-a2f0c3611aa4`

Tools for managing conversations, transfers, and agent consultations.

| Tool | tool_native_id |
|------|----------------|
| Close Conversation | `8db2461f-7b43-4e06-aca7-b110dbd4317d` |
| Consult Agent | `f3c7d8e9-1a2b-4e5f-8901-234567abcdef` |
| Get Internal Files | `e5e6f7a8-9abc-0def-1234-567890abcdef` |
| Schedule Followup | `9f8e7d6c-5b4a-3e2d-1c0b-9a8f7e6d5c4b` |
| Send Internal Files | `f6f7a8b9-0bcd-1def-2345-678901bcdef0` |
| Transfer to Agent | `e1f2a3b4-5c6d-7e8f-9012-345678abcdef` |
| Transfer to Team | `1fcac563-34d5-4546-80cd-9ac9c3f19ef7` |
| Update Kanban Status | `c8f7e2b1-9a4d-4e8c-b3f6-1d5a8e9c7b2f` |

### transfer_to_agent

Transfer to agent by name (dynamic lookup).

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `agent_name` | string | Yes | Name of agent to transfer to |

### transfer_to_team

Transfer to team by name (dynamic lookup).

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `team_name` | string | Yes | Name of team to transfer to |

### close_conversation

Close/end a conversation.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `new_status` | string | No | Status after closing (default: 'ended') |

### update_kanban_status

Update conversation's kanban board status.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `new_kanban_status` | string | Yes | New kanban status value |

### schedule_followup

Schedule a followup message for later.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `scheduled_time` | string | Yes | ISO 8601 datetime |
| `message` | string | Yes | Message to send |

### consult_agent

Consult with another agent without transferring.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `agent_name` | string | Yes | Name of agent to consult |
| `consult_question` | string | Yes | Question to ask |

---

## Resource Tools

**Connector:** Wayai Resource
**connector_id:** `d45e6f78-9abc-4def-8901-23456789abcd`

Tools for accessing and managing resource bases.

| Tool | tool_native_id |
|------|----------------|
| Get Resource Item | `d4d5e6f7-8a9b-0c1d-2e3f-4a5b6c7d8e9f` |
| List Resources | `c3c4d5e6-7f8a-9b0c-1d2e-3f4a5b6c7d8e` |
| Retrieve Resource Content | `a1a2b3c4-5d6e-7f8a-9b0c-1d2e3f4a5b6c` |
| Update Resource Content | `b2b3c4d5-6e7f-8a9b-0c1d-2e3f4a5b6c7d` |

### retrieve_resource_content

Search and retrieve resource items with filtering.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `entity_type` | string | No | 'organization' or 'hub' |
| `entity_id` | string | No | Entity ID to scope search |
| `resource_id` | string | No | Specific resource base ID |
| `search_query` | string | No | Search text |
| `tags` | array | No | Filter by tags |
| `content_format` | array | No | Filter by format |
| `limit` | number | No | Max results (default 10, max 100) |
| `offset` | number | No | Pagination offset |

### list_resources

List available resources.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `resource_name` | string | No | Filter by specific name |

### get_resource_item

Get a specific resource item by ID.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `resource_item_id` | string | Yes | Resource item ID |

### update_resource_content

Update an existing resource item.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `resource_item_id` | string | Yes | Item ID to update |
| `resource_item_content` | string | Yes | New content |
| `resource_item_title` | string | No | New title |

---

## Google Calendar Tools

**Connector:** Google Calendar
**connector_id:** `189c2e74-2275-43b6-8dac-0fb3b782e9de`

Tools for managing calendar events and availability.

| Tool | tool_native_id |
|------|----------------|
| Check Availability - Google Calendar | `a5e8c649-0f7d-4b3e-b9ac-96efb8e4c93b` |
| Create Event - Google Calendar | `2482de79-2f7d-444f-a6a1-e943faf59ec6` |
| Delete Event - Google Calendar | `763413b8-4464-44d2-989e-682d4c2e8385` |
| List Events - Google Calendar | `37f60e18-eb76-4efa-968d-1f961bd8325d` |
| Update Event - Google Calendar | `24f82d08-ee88-439e-851f-a33f48c8471e` |

### google_calendar_list_calendars

List all calendars available to user.

*No parameters required.*

### google_calendar_get_calendar

Get specific calendar info.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `calendarId` | string | No | Calendar ID (default: 'primary') |

### google_calendar_list_events

List events within a time range.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `calendarId` | string | No | Calendar ID |
| `timeMin` | string | No | Start time (ISO 8601) |
| `timeMax` | string | No | End time (ISO 8601) |
| `timeZone` | string | Yes | Timezone (e.g., 'America/Sao_Paulo') |
| `maxResults` | number | No | Max events to return |

### google_calendar_create_event

Create a new calendar event.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `calendarId` | string | No | Calendar ID |
| `event.summary` | string | Yes | Event title |
| `event.start` | string | Yes | Start datetime (ISO 8601) |
| `event.end` | string | Yes | End datetime (ISO 8601) |
| `event.timeZone` | string | Yes | Timezone |
| `event.description` | string | No | Event description |
| `event.attendees` | array | No | List of attendee emails |

### google_calendar_update_event

Update an existing event.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `calendarId` | string | No | Calendar ID |
| `eventId` | string | Yes | Event ID to update |
| `event` | object | Yes | Event fields to update |

### google_calendar_delete_event

Delete a calendar event.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `calendarId` | string | No | Calendar ID |
| `eventId` | string | Yes | Event ID to delete |

### google_calendar_check_availability

Check if a time slot is available.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `calendarId` | string | No | Calendar ID |
| `timeMin` | string | Yes | Start time (ISO 8601) |
| `timeMax` | string | Yes | End time (ISO 8601) |
| `timeZone` | string | Yes | Timezone |

---

## Meta Tools

**Connector:** Wayai
**connector_id:** `b17d9f3a-4e1b-46c9-b648-a2f0c3611aa4`

Tools for meta-level tool management and execution (part of the Wayai connector).

| Tool | tool_native_id |
|------|----------------|
| Execute Tool | `b4c997fe-91ab-4f4e-85f9-5bc66d2d9e7e` |
| Get Tool Schema | `a28d74c7-5b78-4847-9a5b-046e398fa6ae` |

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

## MCP Client Tools

**Connector:** MCP Server
**connector_id:** `f1a2b3c4-d5e6-7890-abcd-ef1234567890` (supports API Key and OAuth authentication)

Tools for interfacing with external MCP servers. Tool IDs are dynamically discovered from each connected MCP server.

### mcp_discover_tools

Discover available tools from MCP server.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `connection_id` | string | No | MCP connection (uses context if not provided) |

### mcp_discover_resources

Discover available resources from MCP server.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `connection_id` | string | No | MCP connection |

### mcp_read_resource

Read a resource from MCP server.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `uri` | string | Yes | Resource URI |
| `connection_id` | string | No | MCP connection |

### mcp_refresh

Refresh tools and resources from MCP server.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `connection_id` | string | No | MCP connection |

### Dynamic MCP Tools

Any tool discovered from connected MCP servers can be executed directly by name. Tool parameters depend on the specific MCP server implementation.

---

## Quick Reference

| Module | Key Tools |
|--------|-----------|
| Conversation | `transfer_to_agent`, `transfer_to_team`, `close_conversation`, `consult_agent` |
| Resource | `retrieve_resource_content`, `list_resources`, `get_resource_item` |
| Google Calendar | `google_calendar_list_events`, `google_calendar_create_event`, `google_calendar_check_availability` |
| Meta Tools | `get_tool_schema`, `execute_tool` |
| MCP Client | `mcp_discover_tools`, `mcp_read_resource`, dynamic tools |

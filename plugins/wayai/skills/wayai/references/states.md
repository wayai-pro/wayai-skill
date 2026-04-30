# States

States are JSON-schema-defined data slots that agents read and write during conversations. Use them to track structured information (orders, appointments, user preferences, multi-turn intake) that should persist across messages.

## Table of Contents
- [Scope](#scope)
- [State Definition (`hub.yaml`)](#state-definition-hubyaml)
- [Agent Read/Write](#agent-readwrite)
- [`initial_value`](#initial_value)
- [JSON Schema Patterns](#json-schema-patterns)
- [State vs Kanban](#state-vs-kanban)
- [Entity Matching](#entity-matching)

---

## Scope

| Scope | Lifetime | Use Case |
|-------|----------|----------|
| `conversation` | Tied to a single conversation; resets per conversation | Order in progress, appointment being booked, multi-turn intake |
| `user` | Tied to the end user across all their conversations on the hub | User preferences, profile data, lifetime customer info |

Each state has exactly one scope, fixed at creation.

---

## State Definition (`hub.yaml`)

States are declared at the top level of `hub.yaml` under `states:`:

```yaml
states:
  - id: "state-uuid-789"           # set by pull — do not edit
    name: order_tracking
    scope: conversation
    description: Tracks the order being placed
    json_schema:
      type: object
      properties:
        order_id: { type: string }
        status:
          type: string
          enum: [pending, confirmed, shipped, delivered]
        items:
          type: array
          items:
            type: object
            properties:
              sku: { type: string }
              quantity: { type: integer, minimum: 1 }
        notes: { type: string }
      required: []
    initial_value:
      order_id: null
      status: pending
      items: []
      notes: ""

  - id: "state-uuid-456"
    name: user_preferences
    scope: user
    description: User's communication preferences
    json_schema:
      type: object
      properties:
        language: { type: string, enum: [en, pt, es] }
        contact_method: { type: string, enum: [whatsapp, email] }
    initial_value:
      language: en
      contact_method: whatsapp
```

**Required fields:** `name`, `scope`, `json_schema`, `initial_value`.

---

## Agent Read/Write

Agents access states through native tools (provided by the Wayai connector):

| Tool | Purpose |
|------|---------|
| `get_state` | Read a state's current value (returns the full object) |
| `update_state` | Replace the entire state value |
| `set_state_path` | Update a single dot-notation path (e.g., `items.0.quantity`) |
| `reset_state` | Reset to `initial_value` |

To make these available, list them under `tools.native` for the agent:

```yaml
tools:
  native:
    - get_state
    - update_state
    - set_state_path
    - reset_state
```

Inside agent instructions, reference state values using the `{{state(name)}}` placeholder:

```markdown
Current order: {{state(order_tracking)}}
```

See [agents/instructions.md](agents/instructions.md) for full placeholder syntax.

---

## `initial_value`

`initial_value` is what the state holds when:
- A new conversation starts (for `conversation`-scoped states)
- A new user first interacts with the hub (for `user`-scoped states)
- An agent calls `reset_state`

The shape of `initial_value` MUST validate against `json_schema`. The platform rejects state definitions where `initial_value` doesn't conform.

Patterns:
- Use `null` for fields you want to be "unset" — combined with `nullable` in the schema (or omit `required` so the field can be absent)
- Use empty arrays `[]` and empty strings `""` over `null` when you want the agent to append rather than replace
- Default enum-typed fields to a sensible starting value (e.g., `status: pending`)

---

## JSON Schema Patterns

States use standard JSON Schema. Common shapes:

**Simple flat object:**
```yaml
json_schema:
  type: object
  properties:
    name: { type: string }
    age: { type: integer, minimum: 0 }
    is_active: { type: boolean }
```

**Nested object:**
```yaml
json_schema:
  type: object
  properties:
    address:
      type: object
      properties:
        street: { type: string }
        city: { type: string }
        zip: { type: string }
```

**Array of objects:**
```yaml
json_schema:
  type: object
  properties:
    items:
      type: array
      items:
        type: object
        properties:
          sku: { type: string }
          quantity: { type: integer, minimum: 1 }
        required: [sku, quantity]
```

**Enum field:**
```yaml
json_schema:
  type: object
  properties:
    status:
      type: string
      enum: [pending, confirmed, shipped, delivered, cancelled]
```

**Optional field with explicit `null`:**
```yaml
json_schema:
  type: object
  properties:
    delivery_date:
      oneOf:
        - { type: string, format: date }
        - { type: "null" }
```

The platform doesn't enforce JSON Schema's `additionalProperties: false` by default — agents can add fields the schema doesn't know about. To prevent that, set it explicitly.

---

## State vs Kanban

| Concern | State | Kanban |
|---------|-------|--------|
| What it tracks | Structured data | Workflow stage |
| Visibility | Read/write by agent; not displayed in views | Visible in support/task views |
| Triggers | None — passive data | `triggersAgentResponse`, `isSchedulingStatus`, etc. |
| Defined in | `hub.yaml` `states:` | `hub.yaml` `hub.kanban_statuses:` |
| Updated via | `update_state`, `set_state_path` | `update_kanban_status` (native tool) |

Both coexist: kanban tracks "where in the workflow am I?", state tracks "what data have I gathered?". A typical conversation moves through kanban stages while accumulating state.

---

## Entity Matching

States match on `name + scope` (composite key) instead of name alone — you can have an `order_tracking` state at both `conversation` and `user` scope without conflict. The `id` is the primary match; `name + scope` is the fallback for renames.

Renaming a state: change the `name` field — the `id` keeps continuity, so it's detected as a rename, not delete + create. Changing `scope` is treated as delete + create (scope is structural).

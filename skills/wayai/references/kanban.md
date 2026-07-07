# Kanban Statuses

Deep reference for kanban workflow configuration in `hub.yaml`. The conceptual model (what kanban is, slug identity, flag overview, kanban vs state) lives in `SKILL.md` → Kanban & States — this file holds the full field specs, constraint matrix, and mechanics.

## Table of Contents
- [Identity Model](#identity-model)
- [Behavioral Flags](#behavioral-flags)
- [Allowed Transitions](#allowed-transitions)
- [Followups](#followups)
- [Additional Context on Transition](#additional-context-on-transition)
- [Lanes](#lanes)
- [Constraints (server-enforced)](#constraints-server-enforced)
- [Non-blocking Warnings](#non-blocking-warnings)
- [Runtime Transition Gate](#runtime-transition-gate)
- [Full Example](#full-example)

---

## Identity Model

- `slug` — immutable lowercase identifier (`^[a-z][a-z0-9_]{0,49}$`). Stored in `conversation.kanban_status`, analytics (`data.meta.kanban_status`), native tool params (`update_kanban_status`), and transition references. **Cannot be renamed** after the status is saved.
- `name` — display label shown in the UI and tool descriptions. Freely editable.

Tool wire values are slug-only — the `update_kanban_status` enum accepts slugs, with display names appearing only as `slug (Display Name)` labels in the tool description. Instructions that reference statuses (e.g. "move to `in_progress`") must use the slug.

---

## Behavioral Flags

| Flag | Effect |
|------|--------|
| `isInitialStatus` | Conversations start here. Exactly one status must have it |
| `triggersAgentResponse` | Transitioning a conversation into this status fires an agent turn |
| `allowsAgentUpdate` | The agent may move conversations into this status via `update_kanban_status` |
| `isTerminalStatus` | No outbound transitions — and **entering this status closes the conversation** (ends + archives it, from any surface: agent tool, team drag-drop, REST, MCP). Moving a card to "Resolved" is a close action, not just a column change |
| `isSchedulingStatus` | Status represents a scheduled event; requires non-empty `eventName`. Enables `before_event` followups |

All flags default `false` (omitted in YAML when false).

---

## Allowed Transitions

`allowed_next_statuses` — optional list of sibling slugs this status can transition to.

- Omit (= `undefined`) for unrestricted (any next status — the legacy "any → any" behavior)
- Empty array `[]` is rejected; use `isTerminalStatus: true` for "no outbound"
- Every entry must reference a sibling slug in the same hub

---

## Followups

Per-status timed messages, listed under `followups:`.

| Type | Fires |
|------|-------|
| `inactivity` | After a period of silence in the conversation |
| `before_event` | Before a scheduled event — requires the parent status to have `isSchedulingStatus: true` |

Fields per followup:
- `order` — position within the status
- `threshold` + `timeUnit` — delay (`seconds`/`minutes`/`hours`/`days`)
- `instructions` — what to send (supports `{{...}}` placeholders — see [agents/instructions.md](agents/instructions.md))
- `delivery_mode: direct` + `direct_text` — send fixed text instead of triggering the agent (`direct_text` required and non-empty)
- Optional quiet hours: `excludedTimeStart` / `excludedTimeEnd`
- `excludeHolidays` — default `true`

---

## Additional Context on Transition

Only meaningful when the status has `triggersAgentResponse: true`. Lets a team member hand structured data to the agent turn fired by the transition.

- `additional_context_schema` — JSON Schema (Draft 2020-12) defining the form rendered when a team member transitions a conversation into this status. Submitted values are validated against the schema server-side; any violation fails the request with a path-anchored error. Capped at 16 KB JSON-serialized. Top-level property name `additional_data` is reserved (collides with the dump placeholder below).
- `additional_instructions` — prose template injected into the agent turn on transition. Two placeholder forms:
  - `{{path.to.field}}` — dotted access, whitespace-tolerant. Strings/numbers/booleans render verbatim; arrays of primitives render comma-joined; nested objects/arrays of objects render as 2-space-indented JSON.
  - `{{additional_data}}` — full payload dumped as 2-space-indented JSON. Useful when the schema has lots of fields and you want them all in one block without per-field placeholders.
- The agent receives a single `<system_additional_instructions>` block containing the interpolated prose. The wrapper tag is **always emitted** on a `triggersAgentResponse` transition — empty body when there's no template or submitted data — so the agent turn fires unconditionally.

---

## Lanes

Optional, presentational grouping for the kanban board filter. A hub may declare a `lanes` array (`{ slug, name, color?, order? }`); each non-initial status may set `lane_slug` to one of them.

Lanes are organizational only — they do **not** change transitions (those stay in `allowed_next_statuses`) and are not visible to agents. The board filters by lane; the initial status shows in every lane.

Constraints (server-side, all write paths):
- Lane slugs unique + `^[a-z][a-z0-9_]{0,49}$`
- A status's `lane_slug` must reference a declared lane
- The initial status must **not** have a `lane_slug` (it's the shared entry point)

---

## Constraints (server-enforced)

Enforced on every write — REST, CLI `wayai push`, MCP:

- Exactly one status must have `isInitialStatus: true`
- Slugs must be unique within a hub and match `^[a-z][a-z0-9_]{0,49}$`
- Every entry in `allowed_next_statuses` must reference a sibling slug; `[]` is rejected
- Mutually exclusive flags (cannot both be `true` on the same status):
  - `isInitialStatus` ↔ `triggersAgentResponse` / `allowsAgentUpdate` / `isTerminalStatus` / `isSchedulingStatus`
  - `triggersAgentResponse` ↔ `allowsAgentUpdate` / `isTerminalStatus`
- `isSchedulingStatus: true` requires non-empty `eventName`
- `before_event` followups require the parent status to have `isSchedulingStatus: true`
- Followups with `delivery_mode: direct` require non-empty `direct_text`
- `additional_context_schema` must be a well-formed JSON Schema; the schema's `properties` must not declare a top-level `additional_data` key (reserved placeholder name); JSON-serialized size ≤ 16 KB

---

## Non-blocking Warnings

Returned alongside successful saves:

- `unreachable` — a non-initial status that no other status lists in its `allowed_next_statuses`
- `placeholder_unresolved` — a `{{path.to.field}}` placeholder in `additional_instructions` does not resolve against `additional_context_schema`. At runtime that placeholder renders as empty string
- `unused_lane` — a declared lane no status is assigned to

---

## Runtime Transition Gate

When a conversation transitions kanban status (drag-drop, native tool, REST, MCP), the configured `allowed_next_statuses` is enforced. Disallowed transitions return `invalid_kanban_transition` with the allowed targets. `undefined` `allowed_next_statuses` keeps the legacy "any → any" behavior. Transitions out of an `isTerminalStatus: true` status are rejected.

---

## Full Example

```yaml
# hub.yaml
hub:
  kanban_statuses:
    - slug: new
      name: New
      order: 0
      color: "#22c55e"
      isInitialStatus: true
      allowed_next_statuses: [qualified, waiting_for_customer, resolved]
    - slug: qualified
      name: Qualified
      order: 1
      color: "#3b82f6"
      triggersAgentResponse: true
      allowed_next_statuses: [waiting_for_customer, resolved]
      additional_context_schema:           # JSON Schema (Draft 2020-12)
        type: object
        properties:
          contact:
            type: object
            properties:
              display_name: { type: string }
              relationship: { type: string }
            required: [display_name]
          students:
            type: array
            items:
              type: object
              properties:
                student_name: { type: string }
                modality: { type: string }
              required: [student_name]
        required: [contact, students]
      additional_instructions: |
        A qualified contact just transitioned in. Greet {{contact.display_name}}
        ({{contact.relationship}} of the student) and confirm interest.
        Students:
        {{additional_data}}
    - slug: waiting_for_customer
      name: Waiting for Customer
      order: 2
      color: "#f59e0b"
      allowed_next_statuses: [resolved]
      followups:
        - order: 0
          type: inactivity
          threshold: 30
          timeUnit: minutes
          instructions: "Hi! Just checking in — do you still need help?"
    - slug: resolved
      name: Resolved
      order: 3
      color: "#ef4444"
      isTerminalStatus: true
```

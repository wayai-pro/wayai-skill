# Evals

Evals are test scenarios that verify agent behavior. Each scenario is a YAML file under `workspace/<hub>/evals/`, synced bidirectionally to the platform via `wayai push` / `wayai pull`. Run them with `wayai run-eval` and inspect results with `wayai eval-results`.

## Table of Contents
- [Directory Structure](#directory-structure)
- [Scenario File Format](#scenario-file-format)
- [Filename ↔ Eval Name](#filename--eval-name)
- [Scenario Sets (Subfolders)](#scenario-sets-subfolders)
- [Capturing a Production Conversation](#capturing-a-production-conversation)
- [Running Evals](#running-evals)
- [Inspecting Results](#inspecting-results)
- [Entity Matching](#entity-matching)

---

## Directory Structure

```
workspace/<hub>/evals/
├── greeting.yaml                    # eval_name = "greeting", set = none
├── order-issues/                    # subfolder = scenario set name
│   ├── cancellation.yaml            # set = "order-issues"
│   └── refund.yaml
└── refund-flow/
    ├── happy-path.yaml
    └── edge-case.yaml
```

Subfolders are flat — only **one level deep** is supported. Each subfolder maps 1:1 to a scenario set on the platform; nesting (`evals/a/b/c.yaml`) is rejected by the parser.

---

## Scenario File Format

```yaml
# evals/order-issues/cancellation.yaml
id: "eval-uuid-123"                  # set by pull — do not edit
name: Order Cancellation             # original name (only when it differs from filename slug)
agent: Pilot                         # resolved to responder_agent_id
agent_id: "agent-uuid-456"           # stable ref (set by pull)
runs: 3                              # default 1 — omitted when 1
enabled: false                       # default true — omitted when true

history:                             # message_history (omitted when empty)
  - role: user
    content: "I placed order #12345"
  - role: assistant
    content: "I can see order #12345."

input:                               # message_text — required
  role: user
  content: "Cancel it please"

expected:                            # message_expected_response — required
  role: assistant
  tool_calls:
    - function:
        name: cancel_order
        arguments: '{"order_id": "12345"}'

evaluator_instructions: |
  The agent MUST call cancel_order with the correct order_id.
```

**Required fields:** `agent` (or `agent_id`), `input`, `expected`, `evaluator_instructions`.

`expected` can match on text content, tool calls, or both. The evaluator (a `message_evaluator` agent on the hub) compares the agent's actual response against `expected` using `evaluator_instructions` as its rubric.

---

## Filename ↔ Eval Name

The filename without extension is the eval's `name` (slugified). When the original name differs from its slug, an explicit `name` field is included in the YAML and takes precedence.

| Original name | Filename | YAML `name` field |
|--------------|----------|-------------------|
| `cancellation` | `cancellation.yaml` | omitted (matches slug) |
| `Order Cancellation` | `order-cancellation.yaml` | `name: Order Cancellation` (preserved) |

---

## Scenario Sets (Subfolders)

The first-level subfolder name is the scenario set:

| File path | Scenario set |
|-----------|--------------|
| `evals/greeting.yaml` | (none — root) |
| `evals/order-issues/cancellation.yaml` | `order-issues` |
| `evals/refund-flow/happy-path.yaml` | `refund-flow` |

**Auto-create on push:** If a folder references a scenario set that doesn't exist on the hub, `wayai push` creates it. Sets are not auto-deleted when their last scenario is removed — explicit deletion stays UI-only.

---

## Capturing a Production Conversation

Capture a real conversation as an eval scenario:

```bash
wayai eval capture <conversation_id> [--set <name>] [--name <eval_name>] [--instructions "..."]
```

Flow:
1. CLI calls `POST /api/evals/scenarios/from-conversation`
2. Platform reconstructs the conversation as a scenario (history + input + expected response)
3. CLI materializes it as `evals/<set>/<slug>.yaml` (or `evals/<slug>.yaml` if `--set` is omitted)
4. The scenario is **already on the platform** — review the local file, edit if needed, commit, then push

This is the fastest way to grow eval coverage from real production behavior. Capture once, then refine `evaluator_instructions` to match your acceptance criteria.

---

## Running Evals

```bash
wayai run-eval                           # all evals on the hub
wayai run-eval --set <name>              # scope to a scenario set
wayai run-eval --eval <name>             # run a single eval
```

Each run executes the scenario `runs` times (default 1, configurable per scenario), passes the result to the hub's `message_evaluator` agent, and records the score.

---

## Inspecting Results

```bash
wayai eval-results                       # latest results across the hub
wayai eval-results --set <name>          # scope to a set
wayai eval-results --json                # machine-readable
```

Results come from ClickHouse — eval rows are tagged `is_eval = true` and excluded from production analytics. See [analytics.md](analytics.md) for the eval-only analytics surface.

---

## Entity Matching

Evals match on `name + path` (composite key) — you can have a `cancellation` eval at both root and inside `order-issues/` without conflict. The `id` is the primary match; `name + path` is the fallback for renames.

Renaming an eval: change the filename — the `id` keeps continuity. Moving an eval between scenario sets is a rename of `path` (treated as update, not delete + create).

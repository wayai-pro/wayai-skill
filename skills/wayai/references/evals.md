# Evals

Evals are test scenarios that verify agent behavior. Each scenario is a YAML file under `wayai-ws/hubs/<hub>/evals/`, synced bidirectionally to the platform via `wayai push` / `wayai pull`. Run them with `wayai run-eval` and inspect results with `wayai eval-results`.

## Table of Contents
- [Directory Structure](#directory-structure)
- [Scenario File Format](#scenario-file-format)
- [Filename ↔ Eval Name](#filename--eval-name)
- [Scenario Sets (Subfolders)](#scenario-sets-subfolders)
- [Capturing a Production Conversation](#capturing-a-production-conversation)
- [Running Evals](#running-evals)
- [Inspecting Results](#inspecting-results)
- [Principles — authoring & interpreting](#principles--authoring--interpreting)
- [Entity Matching](#entity-matching)

---

## Directory Structure

```
wayai-ws/hubs/<hub>/evals/
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

**Required fields:** `agent` (or `agent_id`), `input`, `expected`. `evaluator_instructions` is optional.

`expected` can match on text content, tool calls, or both. The evaluator (a `message_evaluator` agent on the hub) is automatically given the `expected` response and the agent's actual response — both text **and** tool calls — and scores whether they match. A required tool call the agent skipped fails the eval even if it replied with plausible text. `evaluator_instructions` is optional: it layers extra, scenario-specific scoring criteria on top of that automatic comparison.

> **History replay.** Scenario `history` is replayed into the responder agent's context before `input`, so scenarios can be **multi-turn**. The `history` field accepts `user` / `assistant` turns, `assistant` turns carrying `tool_calls` (text optional), and `tool` result turns (each needs a `tool_call_id` pairing it to a preceding call). Unpaired tool calls/results are dropped. `role: system` items are accepted but **not** replayed (a mid-history system note isn't a conversational turn — this matches how live conversations reconstruct history). Capturing from a real conversation (`scenario-from-conversation`) preserves tool turns: prior `assistant` tool calls and their `tool` results are reconstructed into `history` with `tool_calls` / `tool_call_id` intact. The captured `expected` is the agent's final turn, so it carries `tool_calls` only when that final turn was itself a tool call — a turn that called a tool and *then* replied with text captures only the text (the `expected` is a single message). To assert that mid-turn tool call, edit the captured scenario.

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

### Capture as a journey (full multi-turn transcript)

`wayai eval capture` extracts only the **last** user→agent exchange. To capture the **entire** conversation as a happy-path journey (one eval step per agent turn):

```bash
wayai eval journey capture <conversation_id> [--name <journey_name>] [--instructions "..."]
```

This calls `POST /api/evals/journeys/from-conversation`. Unlike scenario capture it writes **no local file** — the journey is platform-managed: it owns its own scenario set and materializes one single-turn eval per agent turn (each graded against the ideal prefix). Review/edit it in the hub UI's **Journeys** tab.

---

## Running Evals

```bash
wayai run-eval                           # run the hub's sole enabled scenario set
wayai run-eval --set <name>              # run a specific scenario set
wayai run-eval --eval <name>             # run the set the named eval belongs to
```

A run session targets **exactly one scenario set**. With no flag the hub's sole enabled set runs; on a multi-set hub the run fails with `ambiguous_scenario_set` — pick one with `--set`/`--eval` (mutually exclusive). `--eval <name>` runs the *whole set* that eval belongs to, not just that one eval.

Each run executes the scenario `runs` times (default 1, configurable per scenario, capped at 100), passes the result to the hub's `message_evaluator` agent, and records the score.

A session is capped at **1000 total runs** across all enabled scenarios in the set (Σ `runs`). A session that would exceed it is rejected with `too_many_runs` — reduce the enabled scenarios or their `runs`.

---

## Inspecting Results

```bash
wayai eval-results                       # latest results across the hub
wayai eval-results --set <name>          # scope to a set
wayai eval-results --json                # machine-readable
```

Results come from ClickHouse — eval rows are tagged `is_eval = true` and excluded from production analytics. See [analytics.md](analytics.md) for the eval-only analytics surface.

---

## Principles — authoring & interpreting

The sections above are the *mechanics*; these are the *judgment calls*. Domain-neutral — each was forced by a real agent, not theorized.

### Authoring

1. **Test KNOWN failure modes, not happy paths.** An eval that only ever passes proves nothing. Target where the agent actually breaks.
2. **For ACTION evals, assert on the tool call** (`expected.tool_calls`), not the text. Plausible prose is not proof the agent *did* anything — the harness fails a skipped-but-required tool call even when the reply reads fine (see [Scenario File Format](#scenario-file-format)).
3. **Pin only what's deterministic.** Hard-code the stable parts in `expected`; push runtime ids/timestamps into `evaluator_instructions` ("the `order_id` must match the one in history") rather than a brittle literal.
4. **Unit-test a DECISION, not just the whole flow.** Isolating one mid-flow decision makes the failure point sharp. Stage the decision's precondition either as prior turns in `history` (replayed into the responder — see [Scenario File Format](#scenario-file-format)) or inline in the `input` message, then pin criteria in `expected` / `evaluator_instructions`. Keep the setup minimal: the fewer turns it takes to reach the decision, the sharper the signal.
5. **Mutating evals need a seed/reset.** WayAI runs the **real** agent with its **real** tools, so an eval that triggers a write hits the live backend — without a known starting state it passes on residue from the last run. Seed/reset, or assert against state you control.
6. **Repurpose or retire an eval when a structural fix makes its failure mode impossible.** A surface change (see [`tool-principles.md`](agents/tool-principles.md)) can make a bad call un-expressible — the eval guarding it now tests nothing. Retire it or re-point it at the next real risk.

### Interpreting

7. **A false PASS is the most dangerous result** — it hides a regression and makes every PASS suspect. Before trusting a PASS, confirm the evaluator actually *saw* the evidence (text **and** tool calls).
   - *WayAI specific:* the `message_evaluator` is given the agent's actual response **including its tool calls** for the scored turn, **plus an AGENT OBSERVABILITY block** — the agent's resolved system prompt and the exact messages it saw, including any replayed scenario `history`. So it sees the context that shaped the answer, not just the final reply. The remaining blind spot is narrow: *ephemeral* tool results (`keep_in_history: false`) are stripped from what the agent itself retains, so they're absent from the captured observability too. If a judgment depends on one, surface it (see [`prompt-principles.md`](agents/prompt-principles.md), "Context placement").
8. **Read the evaluator's REASONING + the actual tool_calls, not just the score.** A result must be self-contained — `wayai eval-results --json` carries the verdict, the reasoning, and the actual calls; a bare PASS/FAIL tells you nothing about *why*.
9. **Localize the failure: agent, tools, data, or the eval/evaluator?** A FAIL can be a bad agent, a broken tool, wrong fixture data, or a wrong `expected` / `evaluator_instructions`. Diagnose which before "fixing the agent."
10. **Reliability is a distribution — `runs: 1` is one sample.** A 1/1 PASS is not "reliable" (a booking eval read 1/1 while true reliability was ~50%). Raise `runs` for anything you're trusting, and read the pass *rate*, not the first result.
11. **Distinguish text-judged from tool-judged evals.** Tone/voice scenarios are judged on text; action scenarios on `tool_calls`. Don't grade an action eval on how nice the prose sounds.

## Entity Matching

Evals match on `name + path` (composite key) — you can have a `cancellation` eval at both root and inside `order-issues/` without conflict. The `id` is the primary match; `name + path` is the fallback for renames.

Renaming an eval: change the filename — the `id` keeps continuity. Moving an eval between scenario sets is a rename of `path` (treated as update, not delete + create).

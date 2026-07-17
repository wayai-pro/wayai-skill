# Evals

Evals are test scenarios that verify agent behavior. Each scenario is a YAML file under `wayai-ws/hubs/<hub>/evals/`, synced bidirectionally to the platform via `wayai push` / `wayai pull`. Run them with `wayai run-eval` and inspect results with `wayai eval-results`.

> **Good-practice defaults — reach for these first, not advanced extras.** A solid eval suite leans on three primitives; under-using them on tool/mutating evals is the common mistake:
> - **[Journeys](#journeys-hub-as-code)** — capture a real conversation (`wayai eval journey capture`) and let it materialize one eval per turn. The default for covering a whole multi-turn flow as regression protection; hand-write targeted scenarios for failure modes *on top* (see [Authoring](#authoring) principle 1).
> - **Per-run [`variables`](#per-run-variables-var--parallelize-mutating-evals) + higher `runs`** — reliability is a *distribution*, not a 1/1 sample. The default for anything you trust: raise `runs`, give each run a distinct input row, read the pass *rate*.
> - **Seed [`fixture:`](#seed-fixtures-fixture--repeatable-mutating-evals)** — the default for any eval that *writes*. `run-eval` resets the fixture before the session and clears it after, so every run starts from a known baseline instead of the last run's residue.
>
> **They compose.** When an eval's correctness depends on **tool values** (the agent reads or writes backend data), the good practice is all three *together* — **journey + `fixture:` + `variables`** — for **repeatable, parallel** session runs. A stateless text/tone eval that touches no tools needs none of them. See [the recommended pattern](#seed-fixtures-fixture--repeatable-mutating-evals) for the full shape.

## Table of Contents
- [Directory Structure](#directory-structure)
- [Scenario File Format](#scenario-file-format)
- [Filename ↔ Eval Name](#filename--eval-name)
- [Scenario Sets (Subfolders)](#scenario-sets-subfolders)
- [Capturing a Production Conversation](#capturing-a-production-conversation)
- [Running Evals](#running-evals)
- [Inspecting Results](#inspecting-results)
- [Debug: what did the agent actually see?](#debug-what-did-the-agent-actually-see)
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
  role: user                         # `user` or `system` only — declare it (see below)
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

**`input.role` is `user` or `system` only** — push rejects anything else. `input` is the turn *trigger*: it is what the responder answers. An `assistant` role would ask the agent to answer its own utterance, and a `tool` result arrives stripped of the `tool_call` it was answering — neither gives the agent anything to respond to. **Always declare `role`** — a role-less `input` is still accepted, but the turn kind it fires as is then the producer's default rather than something your file states. `history` is deliberately unconstrained — a `tool` result there answers a preceding assistant `tool_calls`, which is what history is for.

### Attachments (images/files on a user turn)

Give the responder a file — a photo of an insurance card, a receipt, a screenshot — so vision/document behaviour is testable, not just discoverable against real traffic:

```yaml
input:
  role: user
  content: "is this plan accepted?"      # optional — a photo with no caption is fine
  attachments:
    - attachments/insurance-card.jpg      # hub-relative path
```

- **`attachments:` is a list of hub-relative file paths.** Put the files under `attachments/` at the hub root (shared by evals and journeys); `wayai push` reads each, uploads it once (content-addressed by SHA-256 — the same file across many scenarios uploads once), and rewrites the turn to a reference. `wayai pull` writes the bytes back under `attachments/` and restores the paths, so the round-trip is stable.
- **User turns only.** Valid on `input` and on `user` turns in `history` (and on journey transcript `user` turns). Push rejects attachments on any other role.
- **Caps:** ≤20 files per turn, ≤10 MB each, ≤1,000 distinct attachment files per push, and a bounded total attachment payload per push. A push carries every referenced file's bytes, so it can't be split to get under the total — reduce the attachment set (fewer/smaller files, or drop attachments from some scenarios) if a push exceeds it.
- **Delivery:** the trigger turn's files reach the responder as a signed URL exactly as a live channel delivers them (images as a URL, never inline base64). Historical turns replay as a `[Attachment: <name>]` placeholder — the responder learns a file was present, not its contents. The **evaluator** sees presence markers, not the image, so assert on the agent's *response* (did it read the plan name correctly?), not on the evaluator inspecting the file.
- **Lifecycle:** bytes are stored per hub and deleted with the hub; deleting a scenario does not delete shared bytes. Templates omit attachments (a text-only slice can't carry the binaries).
- **No symlinks.** `attachments/` and every path component leading to it must be a real directory. `wayai pull` skips attachment sync (with a warning) if `attachments/` — or an ancestor — is a symlink, even one pointing back inside the hub, so a pre-planted link can't redirect writes.

### Per-run variables (`{{var()}}`) — parallelize mutating evals

**Good practice: declare `variables:` whenever you raise `runs` on a scenario that varies by input or writes data** — it buys reliability sampling and safe parallelism in one move. `variables:` is a map of author-provided arrays. Run *i* of `runs: N` resolves `{{var(NAME)}}` / `{{var(NAME).field}}` against `variables.NAME[i-1]` in `input`, `history`, `expected`, and `evaluator_instructions`. Resolution happens **once per run**, so the prompt and the assertion always see the same value — and each run leases a **disjoint** fixture, so a data-mutating scenario can run all its `runs` in parallel without colliding with itself. Values may be objects, so one index yields a coherent bundle (a patient's name + id + slot move together).

```yaml
runs: 3
variables:
  case:
    - { name: "Mariana Alves Pereira", cpf: "258.147.036-44", patient_id: pat-0001,
        starts_at: "2026-06-18T10:00:00-03:00", slot_id: "andre.fernandes::2026-06-18T10:00:00-03:00" }
    - { name: "João Ribeiro Souza",     cpf: "391.204.517-88", patient_id: pat-0002,
        starts_at: "2026-06-18T11:00:00-03:00", slot_id: "andre.fernandes::2026-06-18T11:00:00-03:00" }
    - { name: "Beatriz Nunes Carvalho", cpf: "702.845.193-05", patient_id: pat-0003,
        starts_at: "2026-06-18T12:00:00-03:00", slot_id: "andre.fernandes::2026-06-18T12:00:00-03:00" }
input:
  role: user
  content: "Sou {{var(case).name}}, CPF {{var(case).cpf}}, quero marcar para {{var(case).starts_at}}."
expected:
  role: assistant
  tool_calls:
    - function:
        name: book_appointment
        arguments: '{"data": {"patient_id": "{{var(case).patient_id}}", "starts_at": "{{var(case).starts_at}}", "slot_id": "{{var(case).slot_id}}"}}'
```

Rules: every declared array MUST have at least `runs` entries — a session whose arrays are too short is rejected with `insufficient_variables` before any run starts (short arrays are an authoring error, not silently wrapped). Unknown `{{var(NAME)}}` stays literal (typo signal); a missing subfield of a *present* variable resolves to empty string. Each run's resolved row is recorded in its run snapshot (`resolved_variables`) so a flaky run shows exactly which fixture it used. `variables` is the templating + per-run indexing primitive that makes the runs disjoint; the fixture reset/clear around them is the [seed hooks](#seed-fixtures-fixture--repeatable-mutating-evals) below.

### Seed fixtures (`fixture:`) — repeatable mutating evals

**Any eval that writes SHOULD start from a known baseline — declare a `fixture:`** (or assert against state you control; see [Authoring](#authoring) principle 5). It is the recommended default for mutating evals, not an optional add-on. A scenario **set** or a **journey** names the Rekor fixture its agent turn mutates, and `run-eval` owns the session lifecycle, so it **resets the fixture before the session and clears it after, in `try/finally`**: a data-mutating eval starts from a known baseline every run and a mid-run failure or abort can't leave the shared base dirty. Sets/journeys with no `fixture:` behave exactly as before — but an un-fixtured mutating eval silently passes on the *previous* run's residue.

```yaml
# evals/failure-modes/book-exact-slot.yaml — declare on the scenario
fixture: clinic-baseline    # the Rekor fixture this set depends on
target_base: clinic-preview # the Rekor base it is seeded against
runs: 3
variables: { case: [ ... ] }   # pairs with the fixture (see below)
input: { role: user, content: "..." }
expected: { role: assistant, tool_calls: [ ... ] }
```

```yaml
# journeys/create-agreement.yaml — a journey owns its set, so declare it once
name: Create Agreement
agent: Collections Pilot
fixture: debtor-baseline
target_base: collections-preview
transcript: [ ... ]
```

Lifecycle and rules:
- **Reset before** — at session start `run-eval` resets the fixture against the base (delete-owned + re-apply → the declared baseline). If the pre-session seed **fails** (no hub seed connection, no `target_base`, no credential, or Rekor rejects), the session is **aborted** with a clear `fixture_seed_failed` — no runs are scored, because an unseeded base is a setup failure, not an assertion failure.
- **Clear after** — on every terminal path (completed, stopped, deleted mid-run, or watchdog-reaped) the fixture is cleared (delete-owned). Teardown is guaranteed, not best-effort.
- **Set-level agreement** — the fixture is a property of the shared base, so **all enabled scenarios in a set must declare the same fixture** (or none); two different fixtures in one set is rejected with `fixture_mismatch`. A journey declares it once for its whole flow.
- **`target_base`** — the Rekor **base** the fixture is seeded against. Required alongside `fixture:` (the hub seed connection supplies the API origin + token; the base rides the eval config). Must agree across a set, same as the fixture.

**Setup (once per hub):** create a **seed connection** — a **REST API** connection (`type: Tool`, `service: REST API`) whose **Base URL** is the Rekor API origin (`https://api.rekor.pro`) and whose bearer/API-key credential is a Rekor token scoped **`write:seeds`** (the ops are preview-only) — then point the hub at it. **Config-as-code (recommended for CI):** declare that connection under `connections:` and set the hub's **`seed_connection: <connection name>`** field in `hub.yaml`, then `wayai push`. The binding resolves by name (the Base URL is validated on push, same guard as the setup PATCH) and round-trips on `wayai pull`. Omitting the field leaves the binding unchanged; `seed_connection: null` clears it. It is also settable imperatively via the `update_hub` MCP tool or `PATCH /api/setup/hubs/:id`. One connection per hub covers every fixture; authors then just declare `fixture:` + `target_base:`.

```yaml
# hub.yaml — bind the seed connection by name (config-as-code)
connections:
  - name: Rekor Seeds
    type: Tool
    service: REST API
    base_url: https://api.rekor.pro
    credential: Rekor Write Seeds   # org credential holding the write:seeds token
hub:
  seed_connection: Rekor Seeds
```

**The recommended pattern for tool-dependent evals** — anywhere the agent's correctness hinges on values its tools read or write — is **[journey](#journeys-hub-as-code) + `fixture:` + per-run [`variables`](#per-run-variables-var--parallelize-mutating-evals)** together, and its whole point is **repeatable and parallel session runs**. The **journey** captures the multi-turn tool flow (one graded eval per turn); the **`fixture:`** makes each session **repeatable** (reset-before / clear-after → a known baseline every run, not last run's residue); disjoint **`variables`** make the runs **parallel** (each run mutates its own leased row) — collision-free as long as each run's write footprint fits its row (parallel depth ≤ the number of disjoint fixture rows). Seed a pool, give each run one `variables.case` row, run at `runs: N`; reset-before/clear-after handles the baseline automatically. For a stateless text/tone eval that touches no tools, none of the three is needed — the pattern earns its place precisely when tool values are in play.

`expected` can match on text content, tool calls, or both. The evaluator (a `message_evaluator` agent on the hub) is automatically given the `expected` response and the agent's actual response — both text **and** tool calls — and scores whether they match. A required tool call the agent skipped fails the eval even if it replied with plausible text. `evaluator_instructions` is optional: it layers extra, scenario-specific scoring criteria on top of that automatic comparison.

> **History replay.** Scenario `history` is replayed into the responder agent's context before `input`, so scenarios can be **multi-turn**. The `history` field accepts `user` / `assistant` turns, `assistant` turns carrying `tool_calls` (text optional), and `tool` result turns (each needs a `tool_call_id` pairing it to a preceding call). Unpaired tool calls/results are dropped. `role: system` items are accepted but **not** replayed (a mid-history system note isn't a conversational turn — this matches how live conversations reconstruct history). Capturing from a real conversation (`scenario-from-conversation`) preserves tool turns: prior `assistant` tool calls and their `tool` results are reconstructed into `history` with `tool_calls` / `tool_call_id` intact. The captured `expected` is the agent's final exchange — its concluding text plus **every** tool call the agent made to produce that reply, frozen into `expected.tool_calls`. This holds whether the final turn was itself a tool call or a text reply that followed a tool loop (e.g. `book_makeup` → "Done, booked"), so action turns assert their required calls automatically — no need to hand-edit the captured scenario to re-add the tool call.

> **Delegation (`transfer_to_agent`) evals — scored as the *complete handoff*.** When the responder calls `transfer_to_agent`, the harness reinvokes the specialist in the same run (see [`agents/roles-and-settings.md`](agents/roles-and-settings.md) → Delegation Flow), and the eval scores the **delegated agent's continuation**, not the intermediate transfer turn. So the scored `actual` carries: `tool_calls` = the `transfer_to_agent` call **plus any calls the specialist made**, and `content` = the **specialist's final reply** (the entry agent's own pre-transfer text, if any, is *not* the scored reply — last text wins). Author `expected` for the whole handoff: assert `transfer_to_agent` in `expected.tool_calls`, and assert the specialist's outcome (its reply and/or its calls) — or push the non-deterministic specialist behavior into `evaluator_instructions`. A successful transfer alone is **not** treated as proof the specialist answered: the specialist actually runs and its reply is what's scored. (`transfer_to_team` is different — it returns to the caller with no agent continuation, so that turn is scored directly.)

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

This calls `POST /api/evals/journeys/from-conversation`. Like scenario capture, the journey is created **on the platform** — but it doesn't write the local file inline. Run `wayai pull` to bring it down to `journeys/<slug>.yaml` (this also syncs the server-minted step ids). Journeys are first-class **hub-as-code** (see next section): edit the YAML, `wayai push`, or edit in the hub UI's **Journeys** tab.

---

## Journeys (hub-as-code)

A **journey** is a stored happy-path transcript that materializes into single-turn evals — one per agent turn. Journeys are first-class hub-as-code: `wayai pull` writes them to `journeys/<slug>.yaml`, `wayai push` creates/updates/deletes them, `wayai diff` previews the changes.

**Prefer a journey to cover a whole multi-turn flow.** One `wayai eval journey capture` of a real conversation gives per-step regression coverage in a single command — a truer baseline than a hand-written multi-turn scenario, and every step is graded independently against the ideal prefix. This is the recommended way to grow broad coverage fast; reserve standalone `evals/*.yaml` scenarios for isolating a specific decision or failure mode.

**On-disk layout.** `journeys/` is a **flat folder** — one file per journey, `journeys/<slug>.yaml` (filename slug = journey name). There are NO scenario-set subfolders (that's evals): a journey owns its own **server-managed scenario set**, which is **not authored** and never appears in YAML. The derived single-turn evals are also server-managed (materialized from the transcript) and are **excluded from the eval diff** — you never see or edit them as `evals/*.yaml`.

**What a "step" is.** A step is one user→agent exchange in the transcript. The stable `step_id` lives on the **concluding assistant turn** of each step. The platform explodes the transcript into one derived eval per step, each graded against the ideal conversation prefix up to that point.

**YAML shape:**

```yaml
id: <journey-uuid>              # set by pull; primary match key (omit when authoring a new journey)
name: Cancel Order             # journey name (omit if it equals the filename slug)
agent: Pilot                   # responder agent, resolved by display name
agent_id: <agent-uuid>         # stable agent FK (set by pull; survives a rename)
enabled: false                 # omit when true
evaluator_instructions: "..."  # optional; default grading instructions for all steps
transcript:                    # ordered turns; the happy path
  - role: user
    content: "cancel my order #123"
  - role: assistant            # concluding turn of step 1 — carries the step_id
    content: null
    tool_calls:
      - id: call_1
        type: function
        function: { name: cancel_order, arguments: '{"order_id":"123"}' }
    step_id: <step-uuid>       # set by pull / minted server-side
  - role: tool
    tool_call_id: call_1
    content: "Order 123 cancelled"
  - role: assistant
    content: "Your order #123 is cancelled."
    step_id: <step-uuid-2>
step_config:                   # optional per-step overrides, keyed by step_id
  <step-uuid>:
    runs: 3                    # number of times to run this step's eval (default 1)
    evaluator_instructions: "Confirm the cancel_order tool was called"
variables:                     # optional per-run variables (gh #3007) — propagate to every step
  case:
    - { name: "Ana", order_id: "123" }
    - { name: "Bruno", order_id: "124" }
```

Transcript turn fields: `role` (`user`/`assistant`/`tool`), `content`, `tool_calls` (on assistant turns), `tool_call_id` (on tool-result turns), `attachments` (a list of hub-relative file paths, on `user` turns only — see [Attachments](#attachments-imagesfiles-on-a-user-turn)), and `step_id` (on concluding assistant turns). `step_config` is keyed by `step_id`; each override takes `runs` and/or `evaluator_instructions`.

**Per-run variables on a journey.** A journey-level `variables:` map (same shape and `{{var()}}` semantics as [scenario variables](#per-run-variables-var--parallelize-mutating-evals)) **propagates to every derived step eval**, so run *i* of the whole journey — every step — resolves against the same `variables.NAME[i-1]` row. That makes a multi-turn **mutating** journey runnable at `runs: N` in parallel with each run on its own disjoint fixture. Give each step the same `runs` (or a `step_config` that never exceeds the array length) so the per-step indices stay aligned; a step whose `runs` exceeds a variable array is rejected with `insufficient_variables` at run-launch, same as a scenario.

**Push / pull / diff / apply.** Journeys diff by `id` first, then by `name`. A push creates new journeys, updates changed ones (transcript, step_config, evaluator_instructions, variables, agent, enabled), and deletes journeys absent from the folder. Creating/deleting a journey on push also creates/deletes its managed scenario set and re-materializes the derived evals — all server-side.

> **Pull after creating a journey.** The backend mints stable `step_id`s when a journey is first created (whether via `journey capture` or a `push` of a hand-authored transcript with no `step_id`s). Those ids land in the DB but not on your disk. **Run `wayai pull` after creating or first-pushing a journey** to sync the ids (and the assigned `id`) back — same discipline as entity `id`s. Re-pushing a transcript that's missing the server's `step_id`s re-mints them, which severs the derived evals' history.

---

## Running Evals

```bash
wayai run-eval                           # run the hub's sole enabled scenario set
wayai run-eval --set <name>              # run a specific scenario set
wayai run-eval --eval <name>             # run the set the named eval belongs to
wayai run-eval --pacing conservative     # slower run pacing (see below)
wayai run-eval --pacing 1500             # custom interval: 1500ms between runs
```

A run session targets **exactly one scenario set**. With no flag the hub's sole enabled set runs; on a multi-set hub the run fails with `ambiguous_scenario_set` — pick one with `--set`/`--eval` (mutually exclusive). `--eval <name>` runs the *whole set* that eval belongs to, not just that one eval.

Each run executes the scenario `runs` times (default 1, configurable per scenario, capped at 100), passes the result to the hub's `message_evaluator` agent, and records the score.

**Run pacing (`--pacing`).** Controls how fast runs are dispatched, so a big eval set doesn't exhaust the LLM provider's rate-limit budget (which is org-scoped — the same key serving production shares it). Accepts a named preset or a millisecond interval; omit it to take the platform default (`balanced`).

| Value | Interval | When |
|---|---|---|
| `conservative` | ~3s/run (~20/min) | Key shared with heavy production traffic, tighter providers (Gemini / low-tier OpenAI), or Start-tier orgs |
| `balanced` *(default)* | ~1s/run (~60/min) | Anthropic Build+/Scale and most modern provider tiers |
| `fast` | ~0.3s/run (~200/min) | High-tier providers, off-peak |
| `<milliseconds>` | that exact gap (50–60000) | A custom pace |

Pacing is a per-run choice, not stored in the scenario YAML — set it each `run-eval` (the UI and MCP `create_eval_session` expose the same presets).

A session is capped at **1000 total runs** across all enabled scenarios in the set (Σ `runs`). A session that would exceed it is rejected with `too_many_runs` — reduce the enabled scenarios or their `runs`.

If the set/journey declares a [`fixture:`](#seed-fixtures-fixture--repeatable-mutating-evals), the session first resets it against Rekor; a failed pre-session seed aborts the run with `fixture_seed_failed` (nothing is scored), and enabled scenarios declaring different fixtures are rejected with `fixture_mismatch`.

---

## Inspecting Results

```bash
wayai eval-results --session <id>            # aggregate results for a session
wayai eval-results --session <id> --runs     # per-run detail: response, tool calls, and the conversation + message ids
wayai eval-results --eval "<name>"           # latest session results for one eval (by name)
wayai eval-results --session <id> --json     # machine-readable
```

Requires `--session <id>` or `--eval <name>` — there is no hub-wide default. The per-run `Observability:` line carries the `conversation` + `evaluated`/`evaluator` message ids, so a turn's full trace is one command away: `wayai conversations <conversation_id> observability --message-id <id>`.

Results come from ClickHouse — eval rows are tagged `is_eval = true` and excluded from production analytics. See [analytics.md](analytics.md) for the eval-only analytics surface.

### Listing scenarios & ad-hoc SQL

```bash
wayai evals                          # list the hub's eval scenarios (--enabled / --disabled to filter)
wayai evals sql "SELECT count() FROM conversation"                          # raw SQL over eval result rows
wayai evals sql "SELECT eval_id, count() FROM conversation GROUP BY 1" --json
wayai evals sql --schema             # column + eval-score-path catalog (incl. discovered data.eval_scores.* dimensions)
```

`wayai evals sql` is for aggregate questions `eval-results` can't answer (pass rates across sessions, score trends, per-dimension breakdowns). Server-enforced guardrails: scoped to the current hub + `is_eval = true`, single-SELECT only, 10k row cap (`--limit` to lower).

---

## Debug: what did the agent actually see?

When an eval (or a live turn) does something inexplicable — wrong date, ignored a rule, hallucinated a value — **stop guessing at the instructions and read the exact input the model received for that turn**: the resolved system prompt, the rendered `additional_context_template` (what `{{now()}}` actually expanded to), the replayed scenario `history`, any `[timestamp, weekday, daypart]` from `include_message_timestamps`, and the tool calls it made. That input is captured in conversation **observability**.

```bash
# 1. Run (or re-find) the eval and grab the ids from the per-run Observability: line.
wayai eval-results --session <session_id> --runs        # → conversation id + message id per run

# 2. Read the FULL record for the scored turn: resolved prompt + exact messages + tool calls.
wayai conversations <conversation_id> observability --message-id <message_id> --json
```

`wayai conversations <id> observability` (no `--message-id`) lists every LLM turn in the conversation with its `message_id`, latency, and tool-call count; adding `--message-id <id>` expands one turn into the complete record (prompt, completion, tool calls, tokens). This works for any conversation, not just evals — it's the answer to *"the agent did X — what did it ACTUALLY receive?"*.

> **Why this beats re-reading `agents/<slug>.md`:** the file is the *template*; observability is the *resolved* prompt for that specific turn. A relative-date bug, a stale `{{state(...)}}`, or a placeholder that silently rendered empty is invisible in the source file and obvious in the resolved record.

---

## Deleting Sessions

Each `run-eval` creates a **session** — its runs, results, and the eval conversations they spawned. Sessions accumulate; delete the noise once you've read what you need.

```bash
wayai eval session delete <session_id>   # delete one session
wayai eval session delete --all          # delete every session on the hub
wayai eval session delete --all -y       # skip the confirmation prompt
```

Destructive and irreversible. The cascade removes the session row, its runs + results, and the eval conversations everywhere they live (ConversationDO storage, ClickHouse rows, R2 archives). Scenarios, scenario sets, and journeys are **not** touched — only the run history. Both forms prompt for confirmation unless `-y`/`--yes` is passed.

---

## Principles — authoring & interpreting

The sections above are the *mechanics*; these are the *judgment calls*. Domain-neutral — each was forced by a real agent, not theorized.

### Authoring

1. **Test KNOWN failure modes, not happy paths.** An eval that only ever passes proves nothing. Target where the agent actually breaks. (A happy-path **journey** is the deliberate exception: it's regression coverage of a known-good flow — pair it with failure-mode scenarios, never rely on it alone.)
2. **For ACTION evals, assert on the tool call** (`expected.tool_calls`), not the text. Plausible prose is not proof the agent *did* anything — the harness fails a skipped-but-required tool call even when the reply reads fine (see [Scenario File Format](#scenario-file-format)).
3. **Pin only what's deterministic.** Hard-code the stable parts in `expected`; push runtime ids/timestamps into `evaluator_instructions` ("the `order_id` must match the one in history") rather than a brittle literal.
4. **Unit-test a DECISION, not just the whole flow.** Isolating one mid-flow decision makes the failure point sharp. Stage the decision's precondition either as prior turns in `history` (replayed into the responder — see [Scenario File Format](#scenario-file-format)) or inline in the `input` message, then pin criteria in `expected` / `evaluator_instructions`. Keep the setup minimal: the fewer turns it takes to reach the decision, the sharper the signal.
5. **Mutating evals need a seed/reset.** WayAI runs the **real** agent with its **real** tools, so an eval that triggers a write hits the live backend — without a known starting state it passes on residue from the last run. Declare a [`fixture:`](#seed-fixtures-fixture--repeatable-mutating-evals) so `run-eval` resets it before the session and clears it after (`try/finally`), or assert against state you control. To run a mutating scenario at `runs: N` **in parallel** without it colliding with itself, pair the fixture with per-run [`variables`](#per-run-variables-var--parallelize-mutating-evals): seed a pool, give each run one disjoint `variables.case` row within the reset baseline.
6. **Repurpose or retire an eval when a structural fix makes its failure mode impossible.** A surface change (see [`tool-principles.md`](agents/tool-principles.md)) can make a bad call un-expressible — the eval guarding it now tests nothing. Retire it or re-point it at the next real risk.

### Interpreting

7. **A false PASS is the most dangerous result** — it hides a regression and makes every PASS suspect. Before trusting a PASS, confirm the evaluator actually *saw* the evidence (text **and** tool calls).
   - *WayAI specific:* the `message_evaluator` is given the agent's actual response **including its tool calls** for the scored turn, **plus an AGENT OBSERVABILITY block** — the agent's resolved system prompt and the exact messages it saw, including any replayed scenario `history`. So it sees the context that shaped the answer, not just the final reply. The remaining blind spot is narrow: *ephemeral* tool results (`keep_in_history: false`) are stripped from what the agent itself retains, so they're absent from the captured observability too. If a judgment depends on one, surface it (see [`prompt-principles.md`](agents/prompt-principles.md), "Context placement").
8. **Read the evaluator's REASONING + the actual tool_calls, not just the score.** A result must be self-contained — `wayai eval-results --session <id> --runs --json` carries the verdict, the reasoning, and the actual calls per run; a bare PASS/FAIL tells you nothing about *why*.
9. **Localize the failure: agent, tools, data, or the eval/evaluator?** A FAIL can be a bad agent, a broken tool, wrong fixture data, or a wrong `expected` / `evaluator_instructions`. Diagnose which before "fixing the agent."
10. **Reliability is a distribution — `runs: 1` is one sample.** A 1/1 PASS is not "reliable" (a booking eval read 1/1 while true reliability was ~50%). Raise `runs` for anything you're trusting, and read the pass *rate*, not the first result.
11. **Distinguish text-judged from tool-judged evals.** Tone/voice scenarios are judged on text; action scenarios on `tool_calls`. Don't grade an action eval on how nice the prose sounds.

## Entity Matching

Evals match on `name + path` (composite key) — you can have a `cancellation` eval at both root and inside `order-issues/` without conflict. The `id` is the primary match; `name + path` is the fallback for renames.

Renaming an eval: change the filename — the `id` keeps continuity. Moving an eval between scenario sets is a rename of `path` (treated as update, not delete + create).

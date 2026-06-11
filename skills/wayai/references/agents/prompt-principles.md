# Context & Prompt Engineering Principles (Reliable Agent Instructions)

How to **place and structure** an agent's context so a cheap, fallible model executes reliably — *where* each piece of context belongs ("Context placement" below) and *how* to phrase the instructions (the rest). Distinct from [`instructions.md`](instructions.md), which covers placeholder *mechanics*, and [`tool-principles.md`](tool-principles.md), which covers the tool surface. Domain-neutral: applies to any hub and any vertical.

**Meta:** a prompt is **procedure → guardrails → voice**, in that priority order. Design for the *fallible* model you deploy, not the ideal one. Structure the prompt so the right action is the first and easiest thing the model can do, say what you want (not what you don't), and keep style out of the way of the flow.

> Field-tested: on a small/cheap model, an agent went from flaky (over-gathering, skipped writes, drifting tone) to reliable (correct action, 0 errors, consistent voice) **purely by restructuring the instructions** this way — no model upgrade.

## Context placement — match each piece to the lifetime of its slot

Before structuring instructions, decide *what even belongs in them*. Match each piece of context to the **lifetime of its slot** — a longer-lived slot than it needs makes it stale; a shorter-lived one makes it churn (and busts the cache, below).

| Lifetime | Slot | Examples |
|----------|------|----------|
| Timeless — every conversation | `instructions` (`agents/<slug>.md`) | persona, data model, flow, rules, tool conventions, voice |
| Per-conversation behavior | prepend / additional instructions | one-off overlays that shouldn't pollute the general prompt |
| Must survive across turns | conversation / user **`state`** | ids, cart, booking-in-progress — durable working memory |
| This turn only | `additional_context_template` + the message | `{{now()}}`, volatile `{{state()}}`, placeholders |

**Cache hygiene gives this teeth.** The provider caches the **stable prefix** (system prompt + tool schemas) byte-for-byte. Any per-turn value baked into `instructions` invalidates that cache every turn — a staleness bug *and* a per-turn cost/latency tax. So keep `instructions` and the tool surface byte-stable, and inject volatile content after the cache boundary via `additional_context_template` (mechanics + the why: [`instructions.md`](instructions.md#additional-context-cache-friendly)). Corollary: churning **tool schemas** busts the cache too — another reason to curate and stabilize the surface ([`tool-principles.md`](tool-principles.md)).

**State vs tool-history.** Tool call/result is **ephemeral by default** (`keep_in_history: false`) — good hygiene; raw tool I/O is verbose and noisy. Enabling `keep_in_history` is a **last resort** (it pollutes the window, grows tokens, pressures the cache). Prefer having the agent **annotate the distilled fact to `state`** — extract the id from the ephemeral result and write it compact (see "State discipline" below). *Caveat:* because tool I/O isn't in history by default, downstream consumers — including the `message_evaluator` — are blind to it unless given. That blindness is the root of the eval **false-PASS** ([`../evals.md`](../evals.md)). Know what each consumer actually sees.

## A. Structure & order

1. **Flow before style.** Put what the agent *does* (the procedure) first; how it *talks* (voice/tone) last. Front-loading style starves execution attention — small models over-gather and stall before reaching the action. Order each `agents/<slug>.md` as **Role → Procedure → Guardrails → Voice**.
2. **Keep it lean.** Every section competes for attention; a long or early section destabilizes the flow. Cut anything that doesn't change an action. Move per-turn / per-minute context (`{{now()}}`, volatile `{{state()}}`) out of `instructions` into `additional_context_template` — keeps the procedure short *and* the cache warm (see [`instructions.md`](instructions.md#additional-context-cache-friendly)).

## B. Framing

3. **Positive over negative.** Tell the model what **to do** ("confirm the date by reading `{{week_horizon()}}`"), not what **not** to do. Affirmative directives execute more reliably, and a prohibition can prime the very behavior you forbid (pink-elephant effect). Reserve negation for hard safety lines — and even then pair it with the positive ("use the `cart_id` returned by the tool" rather than only "never invent an id").
4. **Concrete beats abstract.** Replace tone adjectives and vague rules with **do-say / don't-say pairs** and exact call-shape recipes. "Friendly" is an opinion; a do-say example is a spec.

## C. Execution discipline

5. **One action = one explicit procedure.** Write multi-step actions as a **numbered, imperative** list. Every external effect is a **named tool call** (native or custom — e.g. `send_text_message`, `update_kanban_status`, a custom write tool). State the rule plainly: **never claim an action is done until its tool call returns success.** Models silently skip steps otherwise.
6. **State discipline.** Carry exact identifiers in conversation / user state and **copy them verbatim** from tool results — never let the model reconstruct, guess, or reformat an id. Read them back with `{{state(scope, name).field}}`; persist them with the agent's write tools.

## D. Voice

7. **Specify voice as its own section.** A dedicated `## Voice` block: **persona + a constant voice + context-aware tone + do-say / don't-say**. Left implicit, tone is an artifact of whichever model runs — specify it and the same cheap model matches a warmer, pricier one.

## Recommended instruction skeleton

```markdown
# Role
You are <persona>. Your goal is <the outcome you drive>.

## Procedure
1. <imperative step> — call `<tool_name>` with <inputs>.
2. <imperative step>. Do not state it's done until `<tool_name>` returns success.
3. ...

## Guardrails
- Always <positive rule>.
- When <condition>, <positive action>.

## Voice
- Persona: <who the agent is>.
- Constant voice: <traits that never change>.
- Tone by context: <e.g. brisk when confirming, warm when apologizing>.
- Do say: "<example>"   ·   Don't say: "<counter-example>"
```

Keep `{{now()}}` and volatile `{{state()}}` in `additional_context_template`, not in this file.

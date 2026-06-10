# Prompt Engineering Principles (Reliable Agent Instructions)

How to **structure** an agent's instructions so a cheap, fallible model executes them reliably — distinct from [`instructions.md`](instructions.md), which covers placeholder *mechanics*. Domain-neutral: applies to any hub and any vertical.

**Meta:** a prompt is **procedure → guardrails → voice**, in that priority order. Design for the *fallible* model you deploy, not the ideal one. Structure the prompt so the right action is the first and easiest thing the model can do, say what you want (not what you don't), and keep style out of the way of the flow.

> Field-tested: on a small/cheap model, an agent went from flaky (over-gathering, skipped writes, drifting tone) to reliable (correct action, 0 errors, consistent voice) **purely by restructuring the instructions** this way — no model upgrade.

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

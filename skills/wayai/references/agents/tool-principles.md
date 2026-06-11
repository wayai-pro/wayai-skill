# Agent Tooling Principles (Reliable Tool Surfaces)

How to **design a tool surface** a cheap, fallible model can call correctly — distinct from [`custom-tools.md`](custom-tools.md) and [`native-tools.md`](native-tools.md), which cover the *mechanics* (YAML shapes, config fields). The surface-side sibling of [`prompt-principles.md`](prompt-principles.md): that doc structures the *instructions*; this one structures the *tools the instructions call*. Domain-neutral: applies to any hub and any vertical.

**Meta — hierarchy of robustness: surface curation > value validation > prompt instruction.** To stop the agent doing the wrong thing, prefer making the bad call **un-expressible** (remove the param, close it to an `enum`, constrain it with a `pattern`) over validating the value, and validate over instructing against it. Each rung down is weaker and fires later.

> Field-tested: prompting the agent *not* to pass a blank id failed — it cycled `""` → `" "` → `"null"` → `"undefined"` → `"dummy"` → even turned the instruction word into a value (`"__omit__"`). Value validation alone wasn't enough either (a real-looking id still broke the query). Only **removing the param from the surface** fixed it.

## Where the lever lives in WayAI

Surface curation is author-owned only where you write the schema:

| Tool type | Schema owner | Surface-curation lever |
|-----------|--------------|------------------------|
| **Custom** | You — `tool_config` ([`custom-tools.md`](custom-tools.md)) | Full: add/remove params, `enum`, `pattern`, `required` |
| **MCP** | The external server | Curate on the server; or choose which tools you assign (each is a per-agent allowlist entry) |
| **Native** | The platform catalog — parameter shape not author-editable | Limited: you can't reshape params, but you *can* hide the whole tool's schema (`expose_schema_to_llm: false`), pick *whether* to assign it, and fall back to validation/instruction |

## A. Surface design

1. **One tool = one intent.** A tool serving two intents grows optional params that are wrong for one of them — and the agent fills them. Split by intent.
2. **Don't expose a param the agent shouldn't fill** in a given intent. An absent param can't be mis-filled.
3. **Minimize the surface — hide machinery.** Sorting, pagination (`limit`/`offset`), field selection rarely belong in the agent's hands. Fewer params = fewer ways to err + fewer prompt tokens. WayAI lever: `expose_schema_to_llm: false` keeps a tool executable via `execute_tool` while removing its schema from the inline list ([`native-tools.md`](native-tools.md#meta-tools)).
4. **Names self-explanatory and symmetric.** The split only pays off if the agent picks the right half — name the pair so the choice is obvious. Avoid ambiguous or foreign-language tool names.

## B. Validation

5. **Constrain structurally where the shape is known:** `enum` (closed set), `pattern` (structured id), non-blank / `required`. A schema the provider enforces beats a runtime error beats a prompt rule. *Gotcha:* an `enum` **invites** the agent to pick a value — use it where picking is the norm, not where *omitting* is (it'll pick rather than omit).
6. **Fail loud and actionable, never silent.** An invalid value returns a specific error (`X invalid — omit it to skip`), never a silent empty result. A silent empty reads to the agent as "no data," and it proceeds on the wrong premise.

## C. Operations

7. **A composite/atomic operation is ONE guarded tool**, not a multi-step the agent can half-complete. Where the domain needs a write that must happen together, make it a single tool with the guard *inside* (invisible to the agent). WayAI mechanism: `composed_tools` on a custom tool — run native side effects on success, gated by `success_path` ([`custom-tools.md`](custom-tools.md#composed_tools-side-effects-on-success)).

## D. Posture & descriptions

8. **Design for the FALLIBLE agent.** Assume it fills empties, reconstructs ids it should have copied, skips steps, and retries without self-correcting. The *surface* — not a paragraph of warnings — must be robust to that. (Instruction-side counterparts in [`prompt-principles.md`](prompt-principles.md): the "State discipline" principle and the fallible-model framing in its Meta line.)
9. **A field description is a NUDGE, not a fix.** Describe *kind + source + when-to-use*, with **placeholders, not real values** (a real example id anchors the agent and goes stale). If a description is load-bearing for correctness, you're on the weakest rung — move the constraint up to validation or surface curation.

## The thread to context & evals

Tool call/result is **ephemeral by default** in WayAI (`keep_in_history: false`) — good hygiene, but it means the curated surface is most of what the agent reliably acts on, and downstream consumers (e.g. the `message_evaluator`) are blind to tool I/O they weren't given. A clean surface is therefore also an **evals-reliability** lever. See [`prompt-principles.md`](prompt-principles.md) ("Context placement", state vs tool-history) and [`../evals.md`](../evals.md) (the false-PASS).

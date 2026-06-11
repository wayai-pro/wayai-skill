# Canonical Example

A complete, minimal hub: **Customer Support** with one pilot agent, a knowledge resource, two states, a kanban workflow, and one eval. Read this **once** before generating a new hub — it shows how the per-domain references fit together. The per-domain refs (`references/connections.md`, `references/agents/*`, `references/states.md`, `references/resources.md`, `references/evals.md`) describe each piece in isolation; this file shows the **wiring between them**.

This is **reference content** — it is not a template to copy. Use the shapes and cross-reference patterns; pick names and content that fit the user's actual hub.

## Layout

```
canonical-example/
├── hub.yaml                              # config + states + connections + resources declarations
├── agents/
│   ├── support-pilot.yaml                # agent config (connection ref, tools, resource link)
│   └── support-pilot.md                  # instructions w/ {{...}} placeholders
├── resources/
│   └── company-faq/                      # slugified resource folder
│       ├── refunds.md
│       └── billing.md
└── evals/
    └── refund-eligible.yaml              # one scenario testing the refund path
```

## Cross-references (this is the part you can't get from per-domain refs)

### 1. Agent → Connection
`agents/support-pilot.yaml` field `connection: openai-main` matches the entry in `hub.yaml` `connections:` with `name: openai-main`. On `wayai push`, that connection auto-creates from the first org credential matching `type: Agent` + `service: OpenAI` (auth type: API Key). OAuth connections (WhatsApp, Instagram, Google Calendar, MCP OAuth) must already exist in the UI; the agent file references them by name only — see `references/connections.md`.

### 2. Agent → State (two paths)
**Tool path:** `agents/support-pilot.yaml` `tools.native` lists `get_state`, `update_state`, `set_state_path`. At runtime the agent calls these with `state_name` matching a name in `hub.yaml` `states[]` (e.g., `ticket_intake`, `customer_profile`). Scope is derived from the state definition — the agent passes only the name.

**Placeholder path:** `agents/support-pilot.md` (and the `additional_context_template` in the YAML) reads state values via `{{state(<scope>, <name>)}}` — `{{state(conversation, ticket_intake)}}`, `{{state(user, customer_profile)}}`. The scope and name **must** match a `hub.yaml` `states[]` entry. A mismatch silently renders empty (placeholder rule from `references/agents/instructions.md`).

Note where each lives: stable identity in `instructions` (system prompt — cached), high-churn reads in `additional_context_template` (per-turn — outside the cache). This split is what makes Anthropic/OpenAI prompt-cache hits possible. See `references/agents/instructions.md#additional-context-cache-friendly`.

### 3. Agent → Resource
`agents/support-pilot.yaml` `resources:` lists `name: Company FAQ`, which matches `hub.yaml` `resources[].name: Company FAQ`. The folder under `resources/` is the **slugified** resource name (`Company FAQ` → `company-faq`). Files inside the folder are the actual knowledge content — the list of files is **not** in YAML; the filesystem is the source of truth (per `references/resources.md`).

The agent invokes the resource at runtime via the native tools `list_resource_files` (discover what's there) and `read_file` (read the content) — both declared in `tools.native`. The instructions reinforce this: look up the FAQ before promising anything.

### 4. Kanban slug invariant
`agents/support-pilot.md` instructs the agent to call `update_kanban_status` with slugs: `in_progress`, `waiting_for_customer`, `resolved`. These **must** match `hub.yaml` `hub.kanban_statuses[].slug` exactly — the slug is the stable identifier (`references/agents/native-tools.md#update_kanban_status`). The display `name` is for humans; the agent never sees it.

Allowed transitions: `hub.yaml` declares `allowed_next_statuses` per status. The native tool rejects transitions that violate the list or that originate from `isTerminalStatus: true`.

### 5. Followup → Status
`hub.yaml`'s `waiting_for_customer` status declares an inactivity followup (24 hours). When the agent moves the conversation into that status and the customer goes quiet, the platform fires the followup message automatically — no agent action needed. The followup's `instructions` field also supports `{{...}}` placeholders.

### 6. Delegation → Team
`agents/support-pilot.yaml` `tools.delegation` targets `Tier 2 Support`. Teams are NOT defined in `hub.yaml` — they live in the platform UI under **Hub → Users → Teams**. The name in `target:` is matched at runtime. A nonexistent target name fails at runtime with a `transfer_to_team` error.

### 7. Eval → Agent + Tools + State
`evals/refund-eligible.yaml` `agent: Support Pilot` matches the agent's `name` in `support-pilot.yaml`. The `expected.tool_calls` list only the deterministic outcome calls (`set_state_path`, `transfer_to_team`) — the FAQ-consultation step lives in `evaluator_instructions` instead, because the agent's runtime `resource_id` / `file_id` for `list_resource_files` / `read_file` are resolved at runtime and can't be statically asserted. All four tools are still declared in the agent's `tools.native` and `tools.delegation`; the eval just doesn't pin their exact call shape. The `expected` response and the agent's actual response (text + tool calls) are compared automatically; the `evaluator_instructions` only layer on the FAQ-consultation criterion that can't be statically asserted, as extra guidance for the hub's `message_evaluator` agent (a separate background-role agent, not shown here — see `references/agents/roles-and-settings.md`).

## What is intentionally NOT in this example
- **`hub_id`, `hub_environment`, per-entity `id`** — set by `wayai pull`. Never author them. New hubs ship without these fields; they appear on the first pull after `push`.
- **Channels other than `app`** — the `app` channel auto-creates with the hub. WhatsApp / Instagram / Email / Telegram require channel connections set up via the UI (OAuth) or by adding a Channel-type entry to `connections:` (API-key channels). See `references/connections.md`.
- **Custom tools, `composed_tools`, `response_format`, MCP / Google Calendar tools, skills, outbound, scheduling, branched preview hubs.** Each has a dedicated reference — load them when the user's hub requires them.
- **Multiple agents (specialist, advisor, copilot, evaluators).** This example has one pilot. Multi-agent patterns are covered in `references/agents/roles-and-settings.md` (`Delegation Flow`, `Role Reference`).

## When to consult this file
- **State 8 of cold-start** — generating a new hub from a user's description. Read the README once for wiring, then the per-domain refs for the specific shapes you need.
- **When unsure how two entities reference each other** — e.g. "how does an agent link to a resource?", "what's the right placeholder for state?". The cross-reference section above is the answer.
- **After producing a hub that doesn't behave as expected** — check this file's invariants (slug must match, scope+name must match, resource folder must be slugified, etc.) before debugging deeper.

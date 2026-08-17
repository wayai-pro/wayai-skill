# Toolsets and Actions (the MCP factory)

How a base publishes a **purpose-built MCP server** for an agent: Actions (the tool definitions), toolsets (the surface that composes them), the tokens that reach them, and the org vault behind them. Open this when you are about to author or debug a curated tool surface over base data — for the base ontology itself see [`records.md`](records.md) and [`querying.md`](querying.md); for the config-as-code round trip see [`config-as-code.md`](config-as-code.md).

An agent connected to a toolset sees `create_invoice`, `list_payments` — its own domain verbs. It never sees generic record CRUD, and no Data-platform concepts at all.

**Contents**

- [Actions](#actions)
- [Composite Actions](#composite-actions)
- [Toolsets](#toolsets)
- [The MCP connection URL](#the-mcp-connection-url)
- [Which base serves a slug](#which-base-serves-a-slug)
- [Promotion blocking](#promotion-blocking)
- [Per-Action grammar](#per-action-grammar)
- [API tokens](#api-tokens)
- [The org secret vault](#the-org-secret-vault)
- [Modeling principles for agent consumption](#modeling-principles-for-agent-consumption)
- [Tool design principles](#tool-design-principles)
- [Recipes](#recipes)

---

## Actions

An **Action** is *one record type + one operation* (`create` / `get` / `list` / `update` / `delete`) under an immutable slug `id`. The Action owns all guard and shaping config — `filterable_fields`, `expose_*`, `default_*` on reads; `writable_fields`, `precondition`, `binding` on writes — and **its `id` is the agent-facing tool name**. Name it for the job (`book_slot`, `find_patient_by_phone`), not for the table.

```bash
wayai actions upsert get_invoice    --base my-base --record-type invoices --operation get
wayai actions upsert list_invoices  --base my-base --record-type invoices --operation list
wayai actions upsert create_payment --base my-base --record-type payments --operation create
wayai actions upsert list_payments  --base my-base --record-type payments --operation list

wayai actions list   --base my-base
wayai actions get    get_invoice --base my-base
wayai actions delete get_invoice --base my-base [-y]
```

| Flag | Notes |
|------|-------|
| `--record-type <t>` | The record type the Action operates over (single-op; omit for a composite) |
| `--operation <op>` | `create` \| `get` \| `list` \| `update` \| `delete` |
| `--description <d>` | Agent-facing description |
| `--config <json>` | Full Action config; `@file.json` reads it from disk. Merged with the flags above when both are given |

A `list` Action with **no** `filterable_fields` generates a tool with **no parameters at all** (machinery is hidden by default) — right for a small catalog the agent reads whole. If the tool is meant to filter, declare the fields:

```bash
wayai actions upsert search_invoices --base my-base --record-type invoices --operation list \
  --description "Find a customer's invoices, optionally narrowed by status" \
  --config '{ "filterable_fields": [
    { "field": "customer_id", "match": "exact" },
    { "field": "status", "optional": true }
  ] }'
```

`search_invoices` is shaped the way [the push-time warnings](#tool-design-warnings-at-push) want: `customer_id` is the tool's question so it stays required, and it takes an exact match safely because the **native** `invoices` schema declares the root `customer_id` property as `x-fk` into `customers` — a made-up id is rejected by the foreign-key probe rather than silently matching nothing. The referencing record type must have `sources: []`; nested or composed `x-fk` and relationship-data declarations are rejected at config write. `status` is a genuine narrowing filter, so it is marked `optional`. An exact-match string with neither a closed set nor a foreign key behind it is the shape to avoid.

## Composite Actions

An Action can instead be a **composite** — an ordered list of write `steps[]` executed **atomically** as one MCP tool. All steps commit together or none do, so the agent gets a single semantic call (`place_order`) instead of a create-then-link-then-update sequence it could leave half-applied.

- Each step is one native write: a record `create`/`update`/`delete`, or a relationship `create`/`delete`. Give each a unique `key` — it namespaces that step's inputs on the generated tool. A composite carries `steps` **instead of** the top-level `record_type`/`operation`, so it is authored with `--config` only (the flags cover single-op).
- **Per-step guards, AND-combined.** A step may carry a `precondition` (the same compare-and-set Filter DSL as a single write). If *any* step's precondition fails, the whole Action is rejected (409) and nothing is written — so one call can be gated on the current state of several records at once.
- Steps must be **native** record types. A proxy- or source-backed record type cannot join the atomic write and is rejected when you save the Action.

```bash
wayai actions upsert place_order --base my-base --config '{
  "description": "Create an order and its first line atomically",
  "steps": [
    { "key": "order", "record_type": "orders",      "operation": "create", "writable_fields": [{ "field": "status" }, { "field": "total" }] },
    { "key": "line",  "record_type": "order_lines", "operation": "create" }
  ]
}'
```

Reference it from a toolset like any Action (`--action place_order`). The generated tool takes one input object per step key: record steps take `data` (plus `external_id` for an idempotent upsert, or `id` for delete); relationship steps take `source_record_type`/`target_record_type` plus, per endpoint, exactly one of the internal id (`source_id`/`target_id`) or the external key (`source_external_id`/`target_external_id`, with optional `source_external_source`/`target_external_source`), an optional relationship-level `external_id`/`external_source` (the link's own key — makes a re-run of the whole Action idempotent for that step), and `id` for delete. The agent never sees the preconditions.

**Cross-step references work through external ids.** Set an `external_id` on a create step and address the link step's endpoint with `source_external_id`/`target_external_id` — steps run in order inside one transaction, so the link resolves the record the same Action just created. Referencing a same-Action record by its *server-assigned* id remains impossible: the id does not exist until the step runs. A miss rejects the whole Action.

## Toolsets

A **toolset** composes Actions **by reference** and adds the surfaces that are not Actions: relationship tools, batch, and SQL.

```bash
wayai toolsets upsert invoicing-agent --base my-base \
  --name "Invoicing Agent" \
  --action get_invoice --action list_invoices \
  --action create_payment --action list_payments \
  --relationship "invoice_payment:list" \
  --batch "invoices:create,update" --batch "payments:create" --batch "invoice_payment:create" \
  --sql-query

wayai toolsets list   --base my-base
wayai toolsets get    invoicing-agent --base my-base [--resolved]   # --resolved inlines record-type schemas
wayai toolsets delete invoicing-agent --base my-base [-y]
wayai toolsets url    invoicing-agent
```

| Spec | Format |
|------|--------|
| `--action` | `<action_id>`, or `<action_id>=<surface_name>` to rename it in this toolset (repeatable) |
| `--relationship` | `rel_type:op1,op2` — operations `create`, `list`, `delete` (repeatable) |
| `--batch` | `record_type_or_rel:op1,op2` (repeatable) |
| `--sql-query` | Enables the `sql_query` tool |
| `--mint-token` | Also mints a toolset-bound token after the upsert (interactive convenience — not a credential-capture path; see [API tokens](#api-tokens)) |

For advanced control use `--config` with full JSON (or `--config @toolset.json`):

```bash
wayai toolsets upsert invoicing-agent --base my-base --config '{
  "name": "Invoicing Agent",
  "actions": [
    { "action": "search_invoices" },
    { "action": "get_invoice" },
    { "action": "create_payment" },
    { "action": "get_payment" },
    { "action": "list_payments", "description_override": "List the payments recorded against an invoice" }
  ],
  "relationships": [
    {
      "rel_type": "invoice_payment",
      "operations": ["create", "list"],
      "names": { "create": "assign_payment" },
      "description_override": "Link a payment to an invoice"
    }
  ],
  "sql_query": true
}'
```

A toolset `actions[]` entry accepts **only** `action`, `action_name`, and `description_override`. Any other key — a stray `record_type`, `operations`, `filterable_fields` — is rejected at config write: the shaping config lives on the Action, not on the reference.

Relationship tools and `batch` are declared inline on the toolset because they are not first-class Actions. A relationship entry is `{ rel_type, operations, name?, names?, description_override? }`; `batch` is `{ "enabled": true, "operations": { "invoices": ["create","update"] } }`.

**Toolsets and Actions are preview-only config.** Author them against a preview base and promote with the schema change they depend on. Config-as-code round-trips them under `wayai-ws/bases/<base>/` via `wayai pull bases/<base>` / `wayai push bases/<base>` — see [`config-as-code.md`](config-as-code.md).

## The MCP connection URL

```bash
wayai toolsets url invoicing-agent
# → https://data-mcp.wayai.pro/t/invoicing-agent/mcp
```

Streamable HTTP. Connect the agent with a token scoped to exactly one base — for least privilege, one **bound to the toolset** (see [API tokens](#api-tokens)). The agent sees only the tools you configured.

## Which base serves a slug

`https://data-mcp.wayai.pro/t/<slug>/mcp` resolves the toolset from the base your **token** is scoped to.

- **Production:** promote the toolset (and the Actions it references), then connect with a token scoped to the production base (`my-base`).
- **Preview (sandbox testing):** connect with a token scoped to the **preview base id** (`my-base--<preview-slug>`) to serve the not-yet-promoted toolset.

If the slug cannot be resolved for your token's base — unknown toolset, or one that exists only in a preview you are not scoped to — listing the tools returns a clear JSON-RPC error telling you to scope to the preview base id or promote. It never silently hands back an empty tool list. A tool call made *without* listing first reports the underlying not-found instead, so read the error from the listing step.

## Promotion blocking

`wayai bases promote <production-id> --from <preview-id>` is blocked if it would break a published toolset:

- removing a record type or relationship type the toolset exposes,
- dropping an Action a reference points at,
- removing a field its `filterable_fields`, `writable_fields`, `base_filter`, or `precondition` depend on,
- removing a named write binding an Action's `binding` selects.

So promote the Actions and the toolset **together with** the schema change. `--dry-run` lists any such conflicts first. Promotion is human-run.

---

## Per-Action grammar

Everything below is Action config (`wayai actions upsert <id> --config '{…}'`), not toolset config.

### Tool names

An Action's generated MCP tool name defaults to its **`id`**. A toolset reference can override two things per surface without touching the Action:

- **`action_name`** — rename this Action *in this toolset*: `{ "action": "search_invoices", "action_name": "find_invoices" }`. The `--action <id>=<surface_name>` flag form is the same thing.
- **`description_override`** — replace the Action's default description for this toolset.

Tool names must be unique across the toolset, so if you reference the same Action twice (or two Actions that would collide) give one a distinct `action_name`. **Relationship** tools are named by the inline `name`/`names` map on their entry: `name` sets the base noun (link / list / unlink over it), `names` overrides a specific operation (`{ "create": "assign_payment" }`). Every key inside `names` must be a valid relationship operation (`create`/`list`/`delete`), or the override is rejected at config write.

### Typed filter params (`filterable_fields`)

A read-shape knob on a **`list`** Action. Expose chosen fields as typed parameters on the generated tool, derived from the record-type schema — so the agent fills native arguments instead of writing a filter expression.

```bash
wayai actions upsert search_invoices --base my-base --record-type invoices --operation list --config '{
  "filterable_fields": [
    { "field": "customer", "match": "text" },
    { "field": "status", "optional": true },
    { "field": "issued_at" }
  ]
}'
```

Each field becomes a parameter shaped by its type: an `enum` or boolean field → an exact-match param; a number or date/date-time field → a range pair; a string field → a `_contains` substring param.

**`param` is a stem, not always the final param name.** Some matches suffix it:

| `match` | Generated param(s) |
|---|---|
| `exact`, `any_of`, `member` | `<param>` (verbatim) |
| `text` | `<param>_contains` |
| `range` | `<param>_min` / `<param>_max` (`_after` / `_before` for dates) |
| `search` | `<param>_search` |

So `{"field": "plan_name", "match": "text"}` exposes `plan_name_contains`, not `plan_name`.

Per field you may set:

| Key | Meaning |
|---|---|
| `field` | Required. Record-data path — `data.status` or a bare `status`; a nested path such as `address.city` is allowed |
| `param` | Rename the generated param **stem** (see table above) |
| `match` | `exact` \| `range` \| `text` \| `any_of` \| `member` \| `search`. `any_of` accepts a list and matches any; `member` exposes an **array** field as a single membership value |
| `enum` | Constrain the param to a fixed set, so the agent can only pick a valid value |
| `pattern` | A regex declaring the valid format of a structured string field (`^pat-[0-9]+$`), so a placeholder like `"null"`/`"undefined"` is rejected with an actionable error instead of silently matching nothing. Applies to `exact`/`any_of`/`member` on a plain string field; cannot be combined with `enum` |
| `optional` | Opts out of required-by-default (below) |
| `description` | Nudge text — name the field's kind, source, and when to use it; placeholder shapes, never real data values |

An array field auto-exposes as a `member` param. Object fields are rejected — expose a nested path instead. The generic `filter` escape hatch (for OR / nesting) is **not** on the tool unless `expose_filter: true`; when it is, typed params and `filter` are AND-combined.

#### The ranked `search` match

`match: search` exposes a **ranked similarity** param (`<field>_search`) compiling to the `search` operator. It is the **only** match that reaches `search`, and therefore the only way a field's `x-search` tuning (`mode`, `threshold`) affects an agent-facing tool — `match: text` is a plain `%contains%` (`ilike`): case-insensitive, but accent-**sensitive** and typo-intolerant.

- **Plain string fields only.** `enum`/`pattern` do not apply (the value is a fuzzy query, not a value to pick), and a field whose `x-search` sets `searchable: false` is rejected at config write.
- **Tuning is top-level-only.** `x-search` is read from top-level schema properties, so a `search` param on a nested path (`address.city`) works but uses the default blend — see [`querying.md`](querying.md).
- **Ordering.** When a search param is filled, results come back best-match-first by default; an explicit `sort` overrides that.
- **Native record types only.** Like every non-`eq` match, `search` is not forwardable to a proxy-backed source — such a call fails loudly rather than silently ignoring the filter.
- **Rank is not existence — pick a `threshold` deliberately.** Everything above the cutoff is returned ranked, so a value that *doesn't exist* can still return a confident-looking near-match of something else. This bites hardest with `mode: name` on datasets sharing a prefix token (hundreds of plans all starting `AMIL`), where scores compress high and stop discriminating. If your workload needs "no match" to be a reliable signal, raise `threshold` for that field, or keep a `match: exact`/`text` param alongside for confirmation.

#### Required by default (`optional`)

A declared field **is** the tool's question, so its generated param is **required**. `"optional": true` opts one out:

```json
{ "filterable_fields": [
  { "field": "patient_id", "match": "exact" },
  { "field": "status", "match": "exact", "optional": true }
] }
```

Why this direction: an optional parameter is one a weak model may fill with an invented placeholder (`null`, `none`, `sem_filtro`, `.*`). That compiles to a real condition matching zero rows — which, without the recovery hint below, reads as "nothing found" rather than an error, so the model retries with another guess instead of correcting, and can burn its whole iteration budget. Requiring the parameters that define the tool's question makes that failure unreachable.

- A **range** field's two bounds are always optional — `"optional": false` on a range is rejected at config write.
- A required param must carry a real value: blank or whitespace-only input counts as missing. On a **multi-select** (`any_of`) param an empty selection counts too — both the native `[]` and its JSON-encoded `"[]"` spelling — since either compiles to no condition at all. On a scalar param `"[]"` is just a value.
- **Several optional filters on one tool is a design smell.** It usually means the tool answers more than one question. Split it into per-intent Actions (`find_patient_by_phone` + `search_patient_by_name`) rather than widening one.

### Machinery params, hidden by default (`expose_*` / `default_*`)

Read-shape knobs on a **`list`** Action. The five machinery params — `filter`, `sort`, `limit`, `offset`, `fields` — are **not** on a generated tool unless you ask for them, so its inputSchema is pure typed semantics. Server-side translation of the typed params is unaffected, and so is the underlying REST API; this is about what the *agent* sees.

Opt one back in with `"expose_<param>": true`, and set the server-side value applied while a param stays hidden with `"default_sort"` / `"default_limit"` / `"default_fields"`:

```bash
wayai actions upsert search_invoices --base my-base --record-type invoices --operation list --config '{
  "default_limit": 50,
  "default_fields": "external_id,data.status",
  "filterable_fields": [ { "field": "status" } ]
}'
```

- `default_fields` takes the same shape as the `fields` param — a comma-separated string or an array.
- A hidden `limit` defaults to a generous page and surfaces a `"truncated"` flag in the response if it caps the result, so rows are never silently dropped. Set `default_limit` when you know the right page size, and `default_fields` to keep large records from filling the agent's context.
- An **empty result** where the agent itself narrowed the search carries an `"empty_result_hint"` naming the parameters it supplied (names only, never values) with the recovery that actually works: verify the value for a required parameter, drop it for an optional one, simplify a raw `filter`. This is what breaks the guess → zero rows → guess-again retry loop. An empty page the agent did **not** cause gets no hint: no filters supplied, only a `base_filter` it cannot see, or a page past the last row.
- Expose a param only when the agent genuinely needs to drive it. `"expose_filter": true` restores the raw Filter-DSL escape hatch for OR / nesting the typed params cannot express.
- `"agent_minimal": true` named the old opt-in preset for this behavior. It is still accepted (existing config keeps working) but does nothing, since hiding is now the default — omit it in new Actions.

### Curated write surface (`writable_fields`)

A write-shape knob on a **`create`**/**`update`** Action — the write-side mirror of `filterable_fields`. List exactly the fields the tool may set. The allowlist does two things at once:

- **Least-privilege, intent-scoped tools.** A field the agent sends that is not on the list is rejected with an actionable error. A front-desk tool can be barred from ever setting a price or an internal field, and one operation splits into intent tools — a `confirm` tool allowed to set only `status`, a `reschedule` tool allowed to set only the time — instead of relying on prose to fence them.
- **A rich, typed write schema.** The tool's `data` parameter is generated from the record-type schema for just those fields — their types, enums, formats, `required`, and descriptions — instead of a generic "any object" slot.

```bash
wayai actions upsert reschedule_appointment --base my-base --record-type appointments --operation update --config '{
  "writable_fields": [
    { "field": "start_time", "param": "when", "description": "New start time (ISO 8601)" }
  ]
}'
```

Per field you may set `param` (rename the key the agent sets in `data`; the tool maps it back to the real field) and `description`. Fields are **top-level only** — writes merge at the top level, so a partial update changes just the fields you send and leaves the rest untouched, while nested objects are replaced whole. The record-type schema stays the validation source of truth: `writable_fields` shapes *which* fields the tool exposes and *how they are described*, it does not re-validate values. Omit it to keep the generic any-field `data` slot.

### Baked-in read predicates (`base_filter`)

A read-shape knob on a **`list`** Action. `base_filter` is a Filter-DSL expression AND-merged into every call of that tool, **server-side and invisible to the agent** — it never appears in the generated `inputSchema`.

```bash
wayai actions upsert list_active_practitioners --base my-base --record-type practitioners --operation list --config '{
  "base_filter": { "field": "data.status", "op": "eq", "value": "active" },
  "filterable_fields": [ { "field": "specialty", "match": "exact", "enum": ["cardiology", "dermatology", "pediatrics"] } ]
}'
```

The agent sees one parameter (`specialty`) and cannot see, set, or widen the status predicate. This is the read-side twin of `precondition`: the alternative — an optional `status` param the model is supposed to know to fill — is exactly the shape that invites an invented value. Use it to carve a narrow tool out of a broad record type: `list_active_practitioners` and `list_open_tickets` are the same `list` Action shape with a different predicate baked in, and each reads to the model as a single-purpose tool.

- **Combines flat.** The baked predicate, the typed params, and (if exposed) the raw `filter` are AND-combined into ONE group rather than nested pairwise, so the merge costs exactly **one** nesting level no matter how many parts are present. Config write reserves that level — a `base_filter` is validated in its merged shape, so one that only fits un-merged is rejected up front. The level is not free: an agent filter already at the maximum depth is now one over, so a `base_filter` costs the raw escape hatch one level of headroom.
- **`list` only.** Rejected on a `get`, which fetches one record by key and has no filter to merge into.
- **Native record types, or a source-backed one whose list endpoint declares `local_filters`.** A plain source forwards only the filters it declares, so an invisible predicate there would either be dropped upstream (silently widening the result the tool promised to narrow) or fail every call with an error naming a filter the agent never sent. `local_filters` removes that risk by construction: it asserts the collection is bounded and returned whole, and the whole predicate is evaluated locally, exactly as for a native record type. `forward_filters` is **not** enough — it forwards a flat AND of equalities only, so a grouped or range-bearing predicate could never survive the trip. Rejected at config write, and re-checked at promote in case a promote adds a source, drops `local_filters` from the bound one, or renames the source a declared `external_source` names. See [`integrations.md`](integrations.md).

  On a `local_filters` source the predicate is only as faithful as the values it compares. A datetime predicate assumes the upstream field arrives as an ISO datetime — map it with `field_mapping`'s `date_format` if it does not, or the comparison degrades to string ordering in proxy mode while an eval run against seeded (canonicalized) data looked correct.

  **It also changes what each call fetches.** The predicate can only be applied to rows held locally, so an Action with a `base_filter` puts every call into local mode: the upstream fetch becomes the whole bounded collection (`max_rows + 1`, 501 by default) from offset 0, rather than the agent's own page — and paginating re-fetches that collection per page. Set `cache_ttl` on the source's list endpoint to amortize it; the local-mode cache key excludes the filter, so one fetch per TTL serves every filter value (including every distinct `$now`).

  **A multi-source record type must name its source.** The predicate is evaluated against one source, and auto-binding happens only when the record type has exactly one. With two or more, set `external_source` on the Action to say which — otherwise config write rejects it as unresolvable.
- **No `search`.** It would route every call off the read-after-write path; an invisible predicate must not silently change where the tool reads from.
- **Paths** address record data (`data.status`, or a bare `status`) and must resolve against the record-type schema — checked at config write and again at promote, since a dangling predicate would surface to the agent as "no results" with no way to recover. Bare paths are stored canonicalized, so `wayai pull bases/<base>` always writes `data.status`.
- **The field must be comparable by the operator.** A value comparison needs a field the schema shows to be scalar — a declared `string`/`number`/`integer`/`boolean`, or (with no declared type) an `enum`/`const` of scalar values, which is how a workflow status field is spelled. **Array** fields take `has` and nothing else (any other operator compares the whole array and matches inconsistently); conversely `has` requires an array. Everything else — objects, maps, `$ref`/`oneOf`-composed fields, a bare `{}` — is rejected with a message naming the fix: address a nested path, declare the field's type, or use `is_null`/`is_not_null`, which are presence checks and stay valid against any shape. The rule is stated as what is *allowed* rather than what is blocked, deliberately: JSON Schema has open-ended ways to express structure, so an unrecognized shape is refused at config write with an actionable error rather than becoming a predicate that silently matches nothing.
- **A filterless Action with a `base_filter` no longer forwards the raw `filter` verbatim** — merging requires parsing it first. Practically, a malformed `filter` is rejected by the translator rather than the engine; the error is the same class either way.

#### `$now` — a predicate that stays current

A literal date baked into a `base_filter` ages: `list_free_slots` written today keeps offering yesterday's slots tomorrow. The value `"$now"` is resolved server-side to **the instant of each call**, so "future only" stays true without anyone editing the config.

```bash
wayai actions upsert list_free_slots --base my-base --record-type slots --operation list --config '{
  "base_filter": { "and": [
    { "field": "data.status",    "op": "eq", "value": "free" },
    { "field": "data.starts_at", "op": "gt", "value": "$now" }
  ] }
}'
```

- **Where it is allowed.** A field declared `format: date-time`, with a range operator (`gt`, `gte`, `lt`, `lte`). A moving instant is a boundary, not a value, so `eq`/`in` are rejected — as is a `format: date` field, whose right boundary depends on a timezone the predicate does not carry.
- **One instant per call.** Every occurrence in one predicate resolves to the same value, so two conditions cannot straddle a tick and produce an empty window.
- **Only a `base_filter` resolves it.** A precondition is checked against a live record at the write chokepoint and a trigger filter runs at dispatch — neither is a query the tool translator ever sees. An unresolved `"$now"` would stay a literal string, and a range comparison against it degrades to text ordering, which **matches every row**. So it is **rejected** in a precondition (both in an Action's config and in the raw `precondition` a REST or batch write may carry), in a trigger filter, and in a typed datetime param (`starts_at_after`). It stays an inert literal only in an agent's raw `filter` argument. Within a `base_filter` it must be the condition's own value — not an array member, which the substitution never reaches — and on a date/date-time field a near-miss spelling like `"$Now"` is refused rather than silently compared as text.

**It is not a security boundary.** Like `precondition`, it shapes the generated tool, not the record type: a caller with raw REST access to the same base still reads unfiltered rows, and a toolset-bound token's *surface* check is at record type + operation granularity (filter shape has never been an access boundary). Scope data access with token grants; use `base_filter` to make the agent's job unambiguous, not to keep rows secret.

### Conditional writes (`precondition`)

A write-shape knob on a **`create`**/**`update`** Action — a Filter DSL expression checked against the record's **current** state before the write applies. If it does not hold, the write is rejected with a 409 and nothing changes. This turns a fragile read-then-write into one correct, race-free call: model a bookable slot as a record and make "book it" a guarded update, so two agents cannot both book the same slot.

```bash
wayai actions upsert book_slot --base my-base --record-type slots --operation update --config '{
  "precondition": { "field": "data.status", "op": "eq", "value": "free" }
}'
```

The guard is **invisible to the agent** — Action config, never a tool parameter — so the agent just calls `book_slot(...)` and booking integrity is enforced for it. Paths address the current record as `data.<field>`, `version`, or `id` (`{ "field": "version", "op": "is_null" }` = create-only-if-absent). One precondition per Action; the `search` operator is not allowed. Idempotent writes (retries that must not double-create) are already handled by upsert-by-`external_id` — the precondition is for cross-state guards, not retry safety.

#### Guarded and unguarded writes on one record type

A `precondition` is one-per-Action, so to expose both a guarded write and a plain write on the same record type, author **two Actions** with the same operation but distinct ids, then reference both:

```bash
wayai actions upsert book_slot   --base my-base --record-type slots --operation update --config '{
  "precondition": { "field": "data.status", "op": "eq", "value": "free" }
}'
wayai actions upsert manage_slot --base my-base --record-type slots --operation update
```

`book_slot` succeeds only on a slot that is still `free`; `manage_slot` updates the same slot unconditionally (an operator correction). Because each Action has its own id, both surface as distinct tools with no name collision.

### Named write bindings (`binding`)

A write-shape knob on a **`create`**/**`update`**/**`delete`** Action. When a proxy-backed record type's source declares **named write bindings** — a `create`/`update`/`delete` op given as a map of named endpoint variants, see [`integrations.md`](integrations.md) — a write Action picks which one it dispatches to:

```bash
wayai actions upsert book_trial  --base my-base --record-type bookings --operation create --config '{ "binding": "trial" }'
wayai actions upsert book_makeup --base my-base --record-type bookings --operation create --config '{ "binding": "makeup" }'
```

This is what keeps one canonical record type (`bookings`) serving a backend whose writes fan out to several endpoints. The agent never sees a `binding` parameter — it is injected server-side, exactly like `precondition`. Required when the bound op is a binding map with no `default`; rejected when the op is a single endpoint or names a binding the source does not declare (validated at config write, re-checked at promotion).

### External addressing and source topology

`external_id` and `external_source` are envelope parameters, not fields of `data`, and a generated tool publishes them only where the agent's value is actually used. `writable_fields` does not reach them — it scopes keys inside `data`.

- **`create` on a source-backed record type omits `external_id`.** The upstream owns identity: it assigns the id and the source's `id_path` captures it from the create response. A supplied value would not be an idempotency key there — the write path reads it as the upstream *address* and issues an **update** instead of a create, against a record the upstream has never seen. A native record type keeps the parameter, where upsert-by-`external_id` is the core primitive.
- **`create` omits `external_source` wherever the server already resolves it** — an Action that pins one, or a record type with exactly one source (auto-bind). Passing it there could never redirect the write; it was always overridden. It stays published on a sourceless record type, where it is a free-form tag scoping your own `external_id`. A multi-source record type must pin `external_source` on the Action, so the param is withheld there too; a published one can still be reached by drift, if you add a source to a record type whose Actions already exist.
- **`get`/`update`/`delete` are unchanged.** Both params stay published on every topology. `external_id` is the only address a proxy-backed row has — read it off the `list` tool rather than inventing one — and while `external_source` is overridden there just as on `create`, withholding it would prevent no misroute and only break a call that names it.

Both rules read config alone, never the base's `integrations` mode, so an `--integrations disabled` eval clone publishes an identical surface.

### Lenient list-tool arguments

Generated `list` tools are lenient about how structured arguments arrive: `filter` is always a JSON-encoded Filter DSL string, while `sort` and any multi-value (`any_of`) parameter accept **either** the native array **or** a JSON-encoded string of it — so an agent that serializes array arguments as strings still works. (`sort` is the same JSON array of `{"field","direction"}` terms described in [`querying.md`](querying.md).)

Generated tool schemas are otherwise **closed**: an argument not declared by the tool's `inputSchema` is rejected with the unknown name, a likely correction when one exists, and the valid parameter list. This includes stale typed-filter names after a rename and machinery parameters that are not exposed; they are never silently discarded into a broader call. A missing **required** filter param is rejected the same way, naming every missing parameter at once so one round trip carries the whole fix.

### Tool-design warnings at push

`wayai push bases/<base>` lints the `list` Actions in your config against the principles below and prints any suggestions to stderr. They are **advisory** — a rule encodes a heuristic about how a model behaves, and a heuristic that failed your push would be worse than the shape it objects to. Config validity is separate and still fails loudly.

| Rule | What it catches |
|---|---|
| `all-filters-required` | Every **value** filter is required and there is more than one, so the agent must supply all of them on every call. Usually means narrowing filters need `"optional": true`, or the Action is several intents fused together. This is the shape a config authored *before* required-by-default lands in. A `range` field is ignored here, so declaring a date window neither triggers the rule nor silences it. |
| `many-optional-filters` | More than two optional filter **params** on one tool — each is a parameter a weaker model may fill with an invented value. A `range` field's two bounds do not count: they are optional by construction, so an anchor plus a date window stays clean. |
| `unconstrained-string-filter` | A value-picking string param with no `enum` and no `pattern`, so a made-up value compiles to a real condition that matches nothing. A supported **root-property** `x-fk` on a native referencing record type is exempt: its valid values are whatever rows exist in the target record type, and the foreign-key probe rejects a bad one. A nested path such as `insurance.plan_id` stays flagged; nested/composed `x-fk` is rejected at config write, while grandfathered production config remains unenforced until corrected. |
| `filter-escape-hatch` | `expose_filter` enabled alongside declared filter fields — the agent can bypass them with a raw expression, which is the untyped surface the typed params exist to replace. Fires on any truthy value, since `expose_filter: yes` is a *string* in YAML and still opens the hatch. |

Warnings appear on `wayai push bases/<base>`, on `--dry-run`, and on a push with no changes — that last case matters, because a config written before these rules existed is exactly the one that never diffs again.

---

## API tokens

`rec_` tokens scope access to base data, distinct from the platform's own `way_` tokens. They can be restricted to specific bases, record types, environments, and permissions, and are hashed before storage — **the raw value is shown only once, on creation**.

```bash
# Full-access token
wayai bases tokens create --name "my-key" --grants '[{"scope":{"bases":["*"]},"permissions":["*"]}]'

# Scoped: read-only on one production base
wayai bases tokens create --name "client-a-reader" \
  --grants '[{"scope":{"bases":["client-a"],"environments":["production"]},"permissions":["read:records","read:record_types"]}]'

# Disposable probe token — auto-expires, no cleanup needed (--ttl takes s/m/h/d)
wayai bases tokens create --name "probe" --ttl 10m \
  --grants '[{"scope":{"bases":["*"]},"permissions":["*"]}]'

# Absolute expiry + a note about who holds the token
wayai bases tokens create --name "temp-key" \
  --grants '@grants.json' \
  --expires-at 2026-12-31T23:59:59Z \
  --description "CI pipeline for repo X"

wayai bases tokens list                # status, last_used_at, expires_at, description
wayai bases tokens revoke <token_id>   # permanent
```

`--grants` takes inline JSON or `@file.json`. `--ttl` and `--expires-at` are mutually exclusive. **Mint probe/test/diagnostic tokens with `--ttl`** so they expire on their own — a disposable token never needs a manual revoke. Use `--description` to record what holds a token, so future you knows what breaks if it is revoked.

**Toolset-bound (consumer) tokens.** For least privilege, mint a token whose authorization **is** the toolset's tool surface — exactly those record types and operations, relationships, batch, and SQL only if you enabled it, nothing else. A leaked one cannot reach past the tools you exposed, and you rotate or revoke one per agent.

```bash
wayai bases tokens create-for-toolset invoicing-agent --base my-base \
  [--name "<name>"] [--ttl 30d | --expires-at <iso>] [--description "..."]
```

`--name` defaults to `<slug> consumer`. In table output the command also prints the MCP URL. `wayai toolsets upsert --mint-token` mints one as an interactive convenience, but it is narrower: the token's name is fixed at `<slug> consumer` and it accepts no `--ttl`, `--expires-at` or `--description`. Its combined toolset-and-token output is also **not** an automated credential-capture path — use `bases tokens create-for-toolset --token-only` for that.

### Capturing the one-time secret

Prefer `--token-only`: it validates the token shape and emits only the one-time secret — no table, no JSON, no MCP URL around it. Capture it without printing it or enabling shell tracing:

```bash
TOKEN="$(wayai bases tokens create-for-toolset <slug> --base <base> --token-only)"
case "$TOKEN" in rec_*) ;; *) echo "Token creation failed" >&2; exit 1 ;; esac
```

If a workflow must consume `--json`, responses are already unwrapped: extract top-level **`.token`**, never `.data.token`, and fail closed rather than accepting `null`:

```bash
TOKEN="$(wayai bases tokens create-for-toolset <slug> --base <base> --json \
  | jq -er '.token | strings | select(startswith("rec_"))')"
```

`--token-only` is fail-closed by construction: a response without a well-formed `rec_` token errors instead of printing the string `undefined`. Nothing on this surface writes a token to disk or folds one into an error message, and `list`/`prune` read metadata only.

### Permissions and scope

**Permissions:** `read:records`, `write:records`, `read:record_types`, `write:record_types`, `read:relationships`, `write:relationships`, `read:relationship_types`, `write:relationship_types`, `read:attachments`, `write:attachments`, `read:files`, `write:files`, `read:file_types`, `write:file_types`, `read:inbound_webhooks`, `write:inbound_webhooks`, `read:triggers`, `write:triggers`, `read:toolsets`, `write:toolsets`, `read:actions`, `write:actions`, `read:seeds`, `write:seeds`, `read:bases`, `write:bases`, `read:audit` (read-only change history; admin-gated, not implied by other grants), `write:imports` (bulk historical import — separate from `write:records` because an import commits without firing triggers, and not implied by other grants), or `*` for all.

**Scope fields:** `bases` (required), `record_types` (optional — omit for all), `environments` (optional — `production`, `preview`, or omit for both).

Tokens enforce a privilege ceiling: you can only create tokens with equal or narrower scope than your own.

### Revoking and pruning

Revoking a token that was used in the last 7 days, or that is toolset-bound (a consumer token an MCP connection likely holds), is refused with a warning; `--force` overrides. Off a TTY the command prints the reason and exits 1 rather than auto-confirming. Do **not** `--force` (or prune with `--include-bound`) unless a human has confirmed nothing depends on the token — revocation is permanent and breaks live traffic immediately. Expired tokens revoke without friction.

```bash
wayai bases tokens prune                        # expired tokens only (preview)
wayai bases tokens prune --unused-since 30d     # + tokens with no activity for 30 days
wayai bases tokens prune --unused-since 30d --yes
```

`prune` selects by **staleness only** — expired, or unused past the window — never by name, and previews before revoking (`--yes` is required non-interactively). Expiry is authoritative: an **expired** token is pruned regardless of binding, so `--include-bound` does not gate that half. The bound-token skip (and `--include-bound`) applies to the *unused-past-the-window* branch only, which also always skips legacy tokens whose usage was never tracked — "stale" is unknowable for those; revoke them individually if truly unused.

## The org secret vault

Organization-level credential storage behind a base's external sources — API keys, signing secrets, or credential-grade blobs (a client certificate, keystore, service-account JSON). Values are encrypted at rest and always masked in responses. Reference them from source or trigger config (an executor's signing secret, see [`executors.md`](executors.md) and [`integrations.md`](integrations.md)), or have an executor read one at runtime.

**Secret values never travel in argv** — there is deliberately no `--value` flag. An argv element is readable from `/proc/<pid>/cmdline` on a default host and lands in shell history and CI job logs. A value arrives from a pipe, a masked prompt, or a file.

```bash
# From a pipe (recommended for CI)
printf '%s' "$STRIPE_KEY" | wayai bases secrets create --name stripe-key --value-stdin --tags billing

# From a masked interactive prompt
wayai bases secrets create --name stripe-key --value-prompt --tags billing

# A credential file, base64-encoded, with its MIME type and an expiry
wayai bases secrets create --name partner-cert --file ./partner.p12 \
  --content-type application/x-pkcs12 --expires-at 2027-01-01T00:00:00Z

wayai bases secrets list                          # values masked
wayai bases secrets list --expiring [--days 30]   # only secrets expiring/expired within the window
wayai bases secrets get <id>                      # metadata, value masked
wayai bases secrets rotate <id> --value-stdin [--expires-at <iso>]   # never auto-generates
wayai bases secrets delete <id> [-y]
```

`--file` cannot be combined with `--value-stdin` or `--value-prompt`: with `--file` silently winning, a CI job that piped the new credential and also named a stale file would store the stale one and report success. `delete` requires `--yes` when non-interactive.

Set `--expires-at` (ISO-8601) on credentials that lapse — yearly certificates, time-bound tokens — then `wayai bases secrets list --expiring` surfaces what needs renewing before it breaks an integration. Expiry is informational; a secret is never auto-disabled.

External source credentials are **not** carried over by `wayai bases promote` — a net-new inline source credential has to be set on production directly, and the promote prints which ones.

---

## Modeling principles for agent consumption

The grammar above is the *mechanics*. This is how to **combine** it so an agent-backed app is reliable. The mechanics make almost any shape possible; these principles pick the shape that makes a fallible LLM agent succeed. Domain-neutral — apply them whether you model invoices, appointments, or tickets. These shape the agent-facing tool surface; for how to **back** an entity (native, proxied, mirrored) see [`integrations.md`](integrations.md).

**The one rule: model it so the right thing is the only easy thing to express, and the wrong thing is impossible.** The gap between an agent that systematically drops steps and one that runs a clean full lifecycle is usually the *shape* of the schema and tools — not the prompt or the model.

1. **One intent = one atomic write.** An action that takes N separate writes is N chances to drop one and leave the system half-updated. Collapse a single user intent into a single write; when one intent genuinely spans multiple records, use a [composite Action](#composite-actions) or `wayai bases batch` so they all commit or all roll back.
2. **State lives on the entity — don't duplicate it.** Keep each piece of state in exactly one place, on the entity it describes. Mirroring a status onto a second record forces the agent to update both in sync, recreating the multi-write problem. Link or query the source of truth instead of copying its fields.
3. **Expose the job, nothing more.** Generated tools already hide the machinery (sort, limit, offset, field selection, the raw filter DSL). The agent fills meaning via typed `filterable_fields` on reads and `writable_fields` on writes — the tool sees only the fields its job sets — not plumbing.
4. **Guards belong in config, not arguments.** Enforce invariants server-side where the agent cannot forget them, not as instructions it must follow. A `precondition` makes a state-dependent write race-free and invisible; schema `required` + enums reject malformed input. The rejection message is the recovery channel — keep it legible and actionable so the agent self-corrects.
5. **Narrow the input space.** Every degree of freedom is a way to get it wrong. Constrain with enums, typed `filterable_fields`, stable `external_id` keys (idempotent upsert, no invented ids), and `x-fk` on writes (a reference must resolve to a real record).
6. **Model for the fallible agent, not the ideal one.** Agents skip steps, send empty strings for fields they did not fill, and invent ids. `required` rejects the missing field, enums reject the made-up value, `x-fk` rejects the invented reference, `precondition` rejects the out-of-order write — each turns a silent corruption into a clear, correctable error.
7. **Separate the operator surface from the agent surface.** Broad human/CI tokens design schemas and toolsets; toolset-bound tokens only call the curated tools and cannot reach past them. Name tools by action and cardinality (`book_slot`, `list_invoices`, `assign_payment`) so the name itself tells the agent what the tool does.

**Also:** keep data **canonical and self-describing** — declare datetime fields `format: date-time` and set the record type's `x-timezone` so every value carries its offset, and prefer readable enums over opaque codes, so a record can be reasoned about without outside context. And **optimize the common path deliberately** — make the single most frequent action one well-named tool call, accepting more friction on the rare one.

## Tool design principles

How to shape the tools an agent actually calls. The defaults above already implement these — the failure mode is *widening* a tool back out, so each rule is about what not to add. The cost of a loose surface is paid by the weakest model that will ever hold the token: an optional parameter is one it may fill with an invented placeholder (`null`, `none`, `sem_filtro`, `.*`), which compiles to a real filter that matches nothing, reads as "no results" rather than an error, and drives a retry loop.

1. **A tool is an intent, not an endpoint.** Name it for the job (`find_patient_by_phone`, `book_appointment`), and let it answer exactly one question. Two ways to look something up = two Actions, not one tool with two optional filters.
2. **Machinery stays hidden.** `filter`, `sort`, `limit`, `offset`, `fields` are off by default; the inputSchema is pure semantics. Opt one back in with `expose_<param>: true` only when the agent genuinely needs to drive it, and set `default_limit` / `default_fields` instead so the server decides.
3. **Declared parameters are required; optionality is a choice you defend.** A `filterable_fields` entry is the tool's question, so it is required by default. Add `optional: true` for a genuine narrowing filter — and if a tool accumulates several, that is the signal to split it by intent. (Range fields are always optional: their two bounds cannot be mandatory.)
4. **Every discrete string parameter gets a closed shape.** An `enum` (best declared on the record-type schema so writes validate too) or a `pattern` (one valid format, e.g. `^\+[1-9][0-9]{6,14}$`). An unconstrained optional string filter is where invented values come from.
5. **What the agent shouldn't decide is injected server-side.** A `base_filter` scopes what a read can see; a `precondition` guards a state-dependent write invisibly; `writable_fields` bounds what a write may touch; `external_source` / `binding` route it. None are agent-facing parameters.
6. **Rejections teach the next move.** The error message is the recovery channel (MCP clients drop `error.details`), so it must name the parameter and the fix. This is why a bad value must fail loudly rather than compile into a filter that returns zero rows.

**Counter-rule:** split by *intent*, never by combinatorics. Three clearly-named tools beat one five-parameter tool; fifteen near-identical variants are worse than either, because tool selection degrades too.

The same hierarchy — surface curation > value validation > prompt instruction — governs hub agent tools; see [`../agents/tool-principles.md`](../agents/tool-principles.md).

## Recipes

Principle → the feature that realizes it. The lookup from intent to implementation; each knob is detailed in [Per-Action grammar](#per-action-grammar).

| Intent | Feature |
|---|---|
| One tool = one intent; curate the surface | One Action per intent (its `id` is the tool name), even two Actions on one record type; a curated `filterable_fields` list |
| Hide machinery (filter / sort / limit / offset / fields) | Hidden by default; `expose_<param>: true` opts one back in, `default_*` sets the server-side value applied while it stays hidden |
| Make a filter optional | `filterable_fields[].optional: true` (declared fields are required by default; a `range` field's bounds already are, and `optional: false` on one is rejected) |
| Constrain a closed set | `filterable_fields[].enum`, or an `enum` on the record-type schema field — preferred, since writes validate against it too and the filter param derives it |
| Constrain a structured id or code | `filterable_fields[].pattern` (`^pat-[0-9]+$`); published to the tool schema and enforced at runtime |
| Require a value at all | Automatic: a declared `filterable_fields` entry is required unless `optional: true`, and a required param rejects blank / empty-selection input, so the model cannot fake compliance with `""` or `[]` |
| Fail loud and actionable, never silent | `x-fk` on write fields; the empty / `enum` / `pattern` rejections on read params (read/write parity) |
| Scope a tool to a subset without an agent-facing filter | `base_filter` on a `list` Action (invisible, always applied); `$now` where the boundary must stay current |
| Make a state-dependent write atomic and race-free | A `create`/`update` Action with a `precondition` (compare-and-set; invisible; 409 on conflict) |
| Make a multi-record intent one call | A [composite Action](#composite-actions) — ordered `steps[]`, one transaction, per-step preconditions |
| Bound what a write may touch | `writable_fields` on a `create`/`update` Action |
| Route one canonical record type's writes to several endpoints | `binding` on a write Action, selecting a named write binding of its source |
| Nudge the agent toward the right value | `filterable_fields[].description`: name the field's kind, source, and when to use it; placeholder shapes, never real data values |
| Check a tool surface against these principles | `wayai push bases/<base>` prints advisory tool-design warnings; never blocking |
| Least-privilege credential per agent | `wayai bases tokens create-for-toolset <slug> --base <base>` |

**One gotcha:** an `enum` invites the agent to *pick* a value — right for a required parameter it should always fill, wrong for an `optional: true` one it should usually skip (a fixed set nudges it to choose rather than omit). On an optional filter prefer a `pattern`, or a plain free param; better still, if the field deserves an enum, ask whether it is really the tool's intent and should be required.

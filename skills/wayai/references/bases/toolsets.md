# Toolsets and Actions (the MCP factory)

How a base publishes a **purpose-built MCP server** for an agent: Actions (the tool definitions), toolsets (the surface that composes them), the tokens that reach them, and the credentials behind them. Open this when you are about to author or debug a curated tool surface over base data — for the base ontology itself see [`records.md`](records.md) and [`querying.md`](querying.md); for the config-as-code round trip see [`config-as-code.md`](config-as-code.md).

An agent connected to a toolset sees `create_invoice`, `list_payments` — its own domain verbs. It never sees generic record CRUD, and no Data-platform concepts at all.

**Contents**

- [Actions](#actions)
- [Composite Actions](#composite-actions)
- [SQL Actions](#sql-actions)
- [Toolsets](#toolsets)
- [File tools](#file-tools)
- [Customer scope](#customer-scope)
- [Reviewed record types](#reviewed-record-types)
- [The MCP connection URL](#the-mcp-connection-url)
- [Which base serves a slug](#which-base-serves-a-slug)
- [Promotion blocking](#promotion-blocking)
- [Per-Action grammar](#per-action-grammar)
- [API tokens](#api-tokens)
- [Credentials behind a base's integrations](#credentials-behind-a-bases-integrations)
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
- **Per-step write shape.** A record step's `writable_fields` bounds its `data` exactly as on a single-op Action: a key outside the list is rejected, and a renamed `param` is the key the agent sends. An `update` step changes only the fields it sends when its `external_id` names an existing record, and creates the record otherwise. A relationship `delete` step removes only a relationship of its own `rel_type`.
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

## SQL Actions

A **SQL Action** is a read-only query you write once, with typed parameters the agent fills. It is one tool that can do what a `list` Action's filters over a single record type cannot: join record types, check a date against two fields of each row, negate a membership, aggregate, or add a column of notices the agent relays. The agent never sees or writes the query.

```json
{
  "description": "Free slots of a service, with the professional who holds each",
  "sql": "SELECT s.id AS slot_id, s.data.starts_at.:String AS starts_at, p.data.name.:String AS professional FROM records AS s JOIN records AS p ON p.external_id = s.data.professional_id.:String AND p.org_id = {org_id:String} AND p.base_id = {base_id:String} AND p.record_type = 'professionals' AND p.deleted = false WHERE s.org_id = {org_id:String} AND s.base_id = {base_id:String} AND s.record_type = 'slots' AND s.deleted = false AND s.archived = false AND s.data.status.:String = 'free' AND s.data.service_id.:String = {service_id:String} AND ({starts_after:Nullable(DateTime64(3, 'UTC'))} IS NULL OR parseDateTime64BestEffort(s.data.starts_at.:String) >= {starts_after:Nullable(DateTime64(3, 'UTC'))}) ORDER BY starts_at",
  "params": [
    { "name": "service_id", "type": "String", "description": "The service's external_id, from list_services", "pattern": "^svc-[a-z0-9-]+$" },
    { "name": "starts_after", "type": "DateTime64(3, 'UTC')", "optional": true, "description": "Earliest start" }
  ],
  "record_types": ["slots", "professionals"],
  "limit": 20
}
```

```bash
wayai actions upsert free_slots --base my-base --config @free_slots.json
```

In config-as-code it is `actions/free_slots.yaml`, the query as a block scalar (`sql: |`); see [`config-as-code.md`](config-as-code.md).

- **The query.** One `SELECT` (or `WITH … SELECT`) in the dialect of `wayai bases sql` ([querying.md](querying.md)), over the `records` table. Write it there first, with `--param` for each parameter, then paste it. Like any query there, it must scope by `{org_id:String}` and `{base_id:String}`, which the server binds on every call; add `deleted = false`, and `archived = false` when archived records should not count.
- **Parameters.** Every value the agent supplies is a typed parameter, `{name:Type}`, declared in `params` with one of these types: `String`, `Int64`, `Float64`, `Bool`, `Date`, `DateTime64(3, 'UTC')`, `Array(String)`, `Array(Int64)`. A parameter is required unless it sets `optional: true`; write an optional one as `Nullable(<type>)` in the query, where it is `NULL` when the agent omits it (an array parameter cannot be optional). `description`, `enum` and `pattern` mean what they mean on [`filterable_fields`](#typed-filter-params-filterable_fields), and `org_id` and `base_id` are reserved. A value is always bound as a value of its type, never pasted into the query text.
- **`record_types`.** The record types the query reads, and the only ones it can read: a row of any other type, of another base or of another organization is invisible to it, whatever the query says. Each must be a native record type of the base (not [source-backed](integrations.md)). The query reads no other table: not relationships, change history or usage.
- **`limit`.** The most rows a call answers, 1 to 200 (default 50). A call runs for at most 5 seconds.

Push checks the Action: its parameters against the query's placeholders, every declared record type, and the query itself, planned against the query engine, so an unknown column or function is refused (400) before anything is stored. When the check cannot run, the push is refused with a 503 and nothing is stored; push again.

Reference it from a toolset like any Action (`--action free_slots`, or `free_slots=<surface_name>` to rename it). The generated tool:

- takes the parameters as its arguments, each refused when it is blank, outside its `enum` or off its `pattern`; a list parameter also takes a JSON-encoded string of its list;
- answers `{ "data": [ …each row, with the query's columns… ], "meta": { "rows": n, "truncated": false } }`. When more rows matched than `limit`, `truncated` is `true` in `meta` and beside it, with a note telling the agent to narrow its parameters. An empty answer carries an `empty_result_hint` naming the parameters the agent supplied;
- has the Action's description, plus one line on the row limit and on freshness: a query reads records as the base's analytical copy holds them, which can be a moment behind the latest write. Keep writes guarded where it matters ([`precondition`](#conditional-writes-precondition)), as a booking that checks the slot is still free does;
- reads the query from the Action at each call, so a pushed change to it applies on the next call.

`wayai toolsets get --resolved` lists each under `sql_actions`: its tool name, description, parameters, record types and limit — never its query.

A SQL Action bills one operation per call. A token not bound to the toolset calls it with `read:records`, and [minting](#permissions-and-scope) a bound token covers the Action's record types. On a base of the `local` analytics tier, the tier eval clones default to, every call is refused with 409 `ANALYTICS_LOCAL`, and cloning such a base warns `ANALYTICS_LOCAL_SQL_ACTION`: evaluate an agent that holds one on a base created on the `standard` tier ([`querying.md`](querying.md#the-sql-surface)).

A [customer-scoped](#customer-scope) toolset may hold one only when every record type it declares is `shared`, and a [reviewed record type](#reviewed-record-types)'s reader may not hold one that declares that type: the query reads every row of each type it declares.

On a WayAI hub, a toolset holding a SQL Action is not offered to a customer-facing agent yet.

## Toolsets

A **toolset** composes Actions **by reference** and adds the surfaces that are not Actions: relationship tools, [file tools](#file-tools), batch, and SQL.

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
| `--batch` | `record_type_or_rel:op1,op2` (repeatable); the key names a relationship type when the toolset has a relationship tool for it, and a record type otherwise |
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

A toolset `actions[]` entry accepts **only** `action`, `action_name`, and `description_override`, whatever kind of Action it references (single-op, [composite](#composite-actions) or [SQL](#sql-actions)). Any other key — a stray `record_type`, `operations`, `filterable_fields` — is rejected at config write: the shaping config lives on the Action, not on the reference.

Relationship tools and `batch` are declared inline on the toolset because they are not first-class Actions. A relationship entry is `{ rel_type, operations, name?, names?, description_override? }`; `batch` is `{ "enabled": true, "operations": { "invoices": ["create","update"] } }`.

**What a `batch` key names.** A key names a relationship type when the toolset also has a relationship tool for that type (`--relationship`, any operation), and a record type otherwise, whatever the base defines by that id. Its operations are then only that kind's: `upsert_relationship` and `delete_relationship` on a relationship type's key, `upsert_record` and `delete_record` on a record type's. Any other operation on the key is refused (400) and the batch writes nothing. So to write links of a type through `batch`, give the toolset a relationship tool for it too, as `invoice_payment` has above. A [bound token](#api-tokens) is minted with `write:relationships` for such a key and `write:records` for the others, so what `batch` runs is what the token may call.

### Relationship `list` tools: one hop

A relationship `list` tool walks one hop from a record: given `record_type`, `record_id` and an optional `direction` (`source`, `target`, or `both`, the default), it answers each link of its relationship type at that record, newest first, as a row `{ link, node }`.

- **`link`**: its `id`, `rel_type`, `direction` (`outgoing` when the start is its source, `incoming` when it is its target), `created_at`, and `data`, a summary of the link's data (see `fields` below; up to 512 bytes), with `omitted`, the number of its fields left out.
- **`node`, a record**: its `record_type`, `id` and `external_id`. When the toolset has a `get` tool for that record type, it also carries `fields`, a summary of the record (its top-level text, number, true/false and null fields, each text cut to 200 characters, up to 1 KiB; a field is left out whole, never cut to fit), and `omitted`, the number of fields left out. Call the type's `get` for the whole record. A type the toolset only lists answers its record's identity alone. A record the base no longer holds (deleted or archived) answers `"held": false` with its type and id, and a record of a [source-backed](integrations.md) type answers its type and id.
- **`node`, a file**: its `file_type`, `path`, `version` and `title` (from its metadata, cut to 200 characters), and `get_tool`, the toolset's `get` file tool that reads it by `path` (absent when only a `list` tool's folder holds it).
- **Only what the toolset reads.** A link is answered only when its other end is a record of a type the toolset has a `get` or `list` tool for, or a live file in the folder of one of its `list` or `get` [file tools](#file-tools). Any other link is left out, so the graph an agent walks is the one its tools read. A hop starts from a record of such a type; any other start, a file included, is refused with the record types it can start from. A file node is a leaf.
- **Paging.** A page holds up to 25 rows and 64 KiB. While `meta.has_more` is true, call again with `meta.next_cursor` as `cursor`. A link written while you page does not move the pages. Each call reads a bounded number of links, so a page may hold fewer rows than it could, even none, while `has_more` is still true: keep paging.
- **Permissions.** A token not bound to the toolset calls a relationship `list` tool with `read:relationships`, plus `read:records` when the toolset reads a record type and `read:files` when it reads a folder.
- **Link tools are not confined this way.** A relationship `create` writes any end its relationship type allows.
- **A toolset removes only the files it can delete, and ties only those to a record.** Deleting a link of a `cascade` relationship type to a file removes the file, rewriting it so it no longer holds that file (a link `create` whose `external_id` it holds) does too, and so does deleting a record that holds such a link, once archival moves the link out ([files.md](files.md)). Writing such a link (a rewrite to another file or record, or a deleted link written back by its `id`) makes the record at its other end decide the file's lifecycle: the file goes when that record is deleted, through the plain API too, or archived. A toolset call that would do any of these — a relationship or record tool, a composite step or `batch` — is refused (403 `FORBIDDEN`, naming the file type) unless the toolset has a [file tool](#file-tools) with `delete` over that file's type and folder; a folder it only lists or reads does not count. Nothing is written, and a batch or composite rolls back whole. A live link written back with its own type and ends (its `data` changed, say) changes no file's lifecycle and is written. The answer is the same whether or not another owning link also holds the file. The plain API (a base token's record, relationship and batch routes) is not held to this.

### Navigating the graph

Every toolset with a relationship `list` tool also gets `describe_graph`, a generated tool that answers the map of what the toolset reaches, read from its current config on every call:

- **`record_types`**: each record type the toolset has a record tool, a composite Action step, a [SQL Action](#sql-actions) or a `batch` entry for (a `batch` key with no relationship tool for it, when the base defines a record type by that id): its `key` (`record:<id>`), `name`, `description` (cut to 200 characters), `tools` (each operation, or `composite`, `sql_action` or `batch`, to this toolset's tool name; a list when several share one), and `references`, its `x-fk` fields whose record type a hop can answer. On a [customer-scoped](#customer-scope) toolset each carries its `customer_scope` (`root`, `owned` or `shared`), and a [reviewed record type](#reviewed-record-types) the toolset only reads carries `"review": "approved_only"` (or `"refused"` when its review cannot be applied to this toolset, which then reads none of its records).
- **`relationship_types`**: each relationship type with a relationship tool, a composite step or a `batch` entry (a `batch` key counts as a relationship type when the toolset has a relationship tool for it; see [What a `batch` key names](#toolsets)), with `source` and `target`: the ends a hop of its `list` tool can answer, by key. A record type is an end when the toolset has a `get` or `list` tool for it, a file type (`file:<id>`) when one of its folders has a `list` or `get` tool, and `["*"]` marks a side the relationship type leaves open. A `create` tool may link other ends.
- **`file_types`**: each file type with file tools, and its folders (`path_prefix`, `""` for the whole type), each with its tools. A folder whose tools are only `get_url`, `put` or `put_url` is listed, but a hop answers none of its files, so it makes its file type no end.
- **`truncated`**: the map holds up to 64 KiB of whole entries; when it leaves some out, it says `true`.

Pass `detail` with a key from the map for one entry's fields: a record type's top-level fields (name, `type`, `format`, `enum` up to 20 values, `required`, `description`, and `references` when its `x-fk` names a record type a hop answers), a relationship type's data fields with its ends and `cascade`, or a file type's metadata fields with its folders. Up to 32 KiB of whole fields, with `truncated` as above. A record type the toolset reaches only through `batch` has no detail, and an unknown key is refused with the valid ones. Nothing outside the toolset's own tools appears in either answer.

The name `describe_graph` is reserved on every toolset, whether or not it has a relationship `list` tool: config write and promotion refuse a toolset that names another tool `describe_graph`. A toolset that took the name before it was reserved keeps that tool, and gets no `describe_graph`, until you rename it; a copy of its base (a new preview, a rollback) keeps it as it is. A call bills as one `get`. A token not bound to the toolset calls it with `read:toolsets`, `read:relationship_types` and `read:file_types`, and [minting](#permissions-and-scope) a token bound to a toolset with a relationship `list` tool takes those two reads too.

**Give `describe_graph` to every agent that holds a relationship `list` tool.** It tells the agent which links lead where, which tool reads each record type it meets, and where to start a hop, so it walks the graph instead of guessing. Assign it alongside the relationship tools.

**Toolsets and Actions are preview-only config.** Author them against a preview base and promote with the schema change they depend on. Config-as-code round-trips them under `wayai-ws/bases/<base>/` via `wayai pull bases/<base>` / `wayai push bases/<base>` — see [`config-as-code.md`](config-as-code.md).

## File tools

A toolset can give its agent the documents of a file type ([files.md](files.md)): a `files` entry per file type and folder, declared inline like a relationship tool. A toolset of file tools alone is valid (`"actions": []`), the usual shape for an agent that only reads documents.

```bash
wayai toolsets upsert support-agent --base my-base --config '{
  "name": "Support Agent",
  "actions": [],
  "files": [
    { "file_type": "policies", "path_prefix": "public", "operations": ["list", "get"] },
    { "file_type": "policies", "path_prefix": "drafts", "operations": ["put", "delete"], "name": "draft" }
  ]
}'
```

A file entry is `{ file_type, path_prefix?, operations, name?, names?, description_override? }`. Its operations are `list`, `get`, `put`, `delete`, `get_url` and `put_url`. Each operation becomes one tool named `<op>_<name>`, where `name` defaults to the file type (`list_policies`, `get_policies`, `put_draft`, `delete_draft`, `get_url_policies`); `names` overrides one operation's tool name. Two tools of a toolset can't share a name: an entry named `url_policies` with `get` and an entry for `policies` with `get_url` would both publish `get_url_policies`, and the toolset write is refused.

- **`path_prefix` is the tool's whole reach.** It is a folder (`public` reaches `public/eu/refunds.md`, never `public-old/…`); omit it for the whole file type. Every path a tool takes or returns is the file's full path, and a path outside the folder is refused. No file tool moves a file, and no other tool of the toolset removes a file outside every folder it has `delete` over, or links one so it goes with a record ([owning links](#relationship-list-tools-one-hop)).
- **`list`** — searches the folder tree; `path` narrows it to a folder beneath. `query` matches the file name or a `title` in the file's metadata (case-insensitive); `tags` matches files whose metadata `tags` array holds any of them; `metadata_filter` is a MongoDB-style filter over the metadata (`$eq` `$ne` `$gt` `$gte` `$lt` `$lte` `$in` `$nin` `$contains` `$and` `$or` `$not`), and a field the file type's metadata schema does not declare is refused. Declare `title` and `tags` in the metadata schema to search on them. Pages of 50 by default and up to 100 (`limit`, `offset`); a full page may have more after it, so page on with `offset`; a search that reads 5,000 files without filling its page is refused — search a narrower folder.
- **`get`** — the file with its `content` and `version`: UTF-8 text for a text file, base64 otherwise (`encoding` says which).
- **`put`** — `path`, `content`, optional `encoding` (`utf-8` default, or `base64`), `content_type`, `metadata`, and `version`. **With `version` it overwrites only while that version is still current** — the one the agent read; **without it, it only creates**, refusing a path that is taken. A write whose version is no longer current is refused (409) and changes nothing. A version counts content and moves: new content becomes a new version in the file's history, and so does a move to another path (a REST or CLI rename; no file tool moves a file), while a write that changes only metadata keeps the version (so it does not make another reader's version stale). A path's version numbers never start over: a file created at a path, or moved onto it, continues past the versions of any file deleted or moved away from there, so a version read from that earlier file is refused. A new file with no `content_type` gets one from its extension (`.md` → `text/markdown`), else `text/plain` (or `application/octet-stream` for base64); an existing file keeps its own, and keeps its metadata when `metadata` is omitted.
- **`delete`** — `path` and a **required `version`**: it removes the file only while that version is current.
- **`get_url`** — `path`. Answers a **download URL** for the file's current version, with `expires_at`, `version`, `size`, `content_type` and `content_sha256`. It is for code that fetches the file itself (a script, a sandbox), so the file can be any size: an HTTP `GET` of the URL returns that version's bytes, as an attachment, and `Range` requests work. The URL works for **5 minutes**, needs no credential, and serves that exact version even after a newer one lands. It stops working once the file is deleted or moved out of the tool's folder, or the toolset no longer has `get_url` over it.
- **`put_url`** — `path`, the **`content_sha256`** (64 lowercase hex characters) and **`size`** of the exact bytes to upload, and optional `content_type`, `metadata` and `version`, which work as on `put`: **with `version` the upload replaces only that version; without it, it only creates**. Answers a **single-use upload URL** with `method: "PUT"`, `expires_at`, `size`, `content_type`, `content_sha256` and the `version` it replaces. Send the declared bytes as one HTTP `PUT` with a `Content-Length` equal to `size`, starting within **5 minutes**; an upload has 120 seconds to finish. The URL writes one new version, once, of that file at that version, and only bytes whose hash and size match what was declared; a repeat of a finished upload answers the same result and writes nothing. Content identical to the file's current content is refused (it would write no version), and so is a stale version (409), a taken path for a create, and a size, content type or metadata the file type doesn't allow, all before any URL exists. At most 100,000,000 bytes. The new version is recorded under the caller that asked for the URL.
- **Size.** A file tool carries at most 1 MiB of content per call, in either direction. `get_url` and `put_url` carry no content, so their files can be larger.
- **Permissions.** A token bound to the toolset carries `read:files` for `list`/`get`/`get_url` and `write:files` for `put`/`delete`/`put_url`, so minting one requires those on the base. A token not bound to the toolset needs the same permission for each call.
- **A URL is a secret for its lifetime.** Anyone holding one can use it until it expires (a download) or is used (an upload), without a credential: hand it only to the code that uses it.

Give a customer-facing agent `list` and `get` only: a hub refuses a pilot, pilot specialist or pilot advisor a toolset with `put`, `delete` or `put_url` ([Base Toolset](../connections.md#base-toolset)). Keep those for curator or back-office toolsets. A file tool needs its file type on the base: a toolset write naming a missing one is refused, and so is a promotion that would leave one missing.

**`wayai toolsets upsert` replaces the whole toolset.** Pass `files` on every upsert of a toolset that has file tools — through `--config`, since the flag form has no file option. An upsert without `files` (the flag form included) removes them.

> **In config-as-code.** `wayai pull` writes a toolset's `files` into `toolsets/<id>.yaml`, and
> `push` applies them. Unlike `toolsets upsert`, a pushed toolset that omits `files` keeps the base's
> file tools; [config-as-code.md](config-as-code.md) says how to remove them.

### URL tools and hub agents

`get_url` and `put_url` serve an external agent that connects to the toolset with its own token. A hub agent edits a large or binary document by copying it into its [code sandbox](../agents/native-tools.md#code-sandbox-tools) instead: a Base Toolset connection leaves those tools out of its tool list, and a tool call WayAI makes for a hub agent is refused when its answer carries a signed file URL, whatever the connection (a generic MCP connection to the toolset's MCP URL or a REST tool included). The one exception is a hosted agent's own MCP servers (`mcp_servers`), which its vendor calls directly, so WayAI's refusal never applies to their answers (they still land in the conversation's record, a live URL included): point those only at a toolset without `get_url` or `put_url`.

## Customer scope

A **customer-scoped** toolset serves one customer per call. It is for an agent that talks to customers, where each call must stay inside the records of the customer it serves. Two declarations make it up.

**Each record type it reaches says what it is**, with `x-customer-scope` at the root of its schema ([records.md](records.md#customer-scope-x-customer-scope)):

- `{ "role": "root" }` — the customer record type itself (`customers`). A base has one.
- `{ "role": "owned", "owner": "customer_id" }` — each record belongs to one customer, named by `owner`: a top-level property that declares `x-fk` into the root record type ([records.md](records.md#referential-integrity-x-fk)).
- `{ "role": "shared" }` — no customer data: a catalog, a plan, a price list. Every call may reach it, as far as the toolset's tools allow.

A record type that declares none cannot be reached by a customer-scoped toolset. `root` and `owned` need a native record type; a source-backed one can only be `shared`.

**The toolset says it is scoped**, with `customer_scope`:

```bash
wayai toolsets upsert support-agent --base my-base --config '{
  "name": "Support Agent",
  "customer_scope": true,
  "actions": [
    { "action": "get_my_account" },
    { "action": "list_my_orders" },
    { "action": "list_products" }
  ],
  "files": [
    { "file_type": "policies", "path_prefix": "public", "operations": ["list", "get"] }
  ]
}'
```

A customer-scoped toolset is refused when it:

- exposes `sql_query` or `batch` — both reach any record;
- reaches, through an Action or a composite step, a record type that declares no role;
- holds a [SQL Action](#sql-actions) that declares a record type other than `shared`: its query reads every row of the type;
- creates or deletes the customer record (`root`): it may read and update it.

A base is refused a second `root`, and an `owned` type whose `owner` does not point at the root.

An `internal_write` [trigger](integrations.md#triggers) that fires on a record type a customer-scoped toolset can write (`create` or `update`, by an Action or a composite step) must locate its target through a field that declares `x-fk` into that target: `match.source_field` names one top-level field, `data.<field>`, whose `x-fk.record_type` is the trigger's `target` and whose key (`x-fk.key`, `external_id` by default) is its `match.target_key`. A customer-scoped write keeps every `x-fk` within its customer's reach, so such a trigger patches only a record the customer reaches. Matched on any other field (a plain string, the record's own `id`), it is refused, naming the trigger, the field, the record type and the toolsets that write it. A trigger fires on a record type when its `record_type_scope` names it or is omitted; an empty scope fires on none. A disabled trigger is judged when it is enabled.

Under the config as it stands, no trigger cascades past that first hop. The patch such an `internal_write` makes on its `target` is a cascade write, and a trigger with `skip_cascade_writes: false` would run on it as configured: matched on a plain field of the patched record, it reaches whatever record that field names. On a shared record, that field may hold what another customer's write put there. So on every record type such an `internal_write` can patch, every trigger that would fire on the patch must keep `skip_cascade_writes: true`, the default. It applies to a trigger of any action type, a webhook included, that fires on `record.updated` on that type. One with `false` is refused, naming it, the `internal_write` whose patch would set it off, the record types and the toolsets.

The rule reads the config as it stands, and an `async` patch already queued is applied as it was queued, to the record type it was queued for. A config change can leave no such `internal_write` patching that type any more, for example by removing or re-targeting the one that queued it. Once that happens, in the same write or a later one, the triggers on that type are no longer held to `skip_cascade_writes: true`, and the queued patch sets off whichever of them then fire on it.

- **Where each rule is checked:**
  - when the toolset, a record type or Action it reaches, or a trigger is written;
  - for the config a `wayai push` (as its files describe it) or a batch of record type writes leaves as a whole;
  - for the whole production config at [promotion](#promotion-blocking), which refuses any of these problems, including one production already has.
- **What is not checked when written:**
  - A single record type delete ([records.md](records.md#customer-scope-x-customer-scope) says why).
  - A preview clone copies its source as it is, problems included: a scoped toolset reaching a record type without a role carries over. It is refused only for a problem its source did not already have.
  - A rollback restores the snapshot as it was.
  - A relationship tool or a composite's relationship step names its endpoints per call, so the record types it reaches are not checked when the toolset is written.

A base left with any of these problems refuses the customer-scoped calls they affect, and its promotion is refused.

- **Its write tools leave the owner out.** The create and update tools of an `owned` record type do not offer its `owner` field: the owner of a record a customer-scoped call writes is the customer the call serves.
- **What it reaches.** `wayai toolsets get --resolved` reports `customer_scope` and `record_type_scopes`: what each record type the toolset can reach is — `root`, `owned` or `shared`, or `null` for one that declares none — including every record type its relationship tools and composite relationship steps can link (every record type, when a relationship type names none on an end).
- **Calls read and write within their customer.** A call reaches only what its customer reaches, and writes only there. See [Serving a customer](#serving-a-customer).
- **Turning it off.** A toolset write that omits `customer_scope` keeps the stored value — unlike `files` on `toolsets upsert` — so an older client writing a toolset back never turns a scope off. Send `"customer_scope": false` to turn it off.

### Serving a customer

Every call to a customer-scoped toolset names the customer it serves. A token bound to the toolset carries it: mint one per customer session, naming the customer under `toolset_binding.customer`. Name the customer record by its id, or by its `external_id` (with its `external_source`, if it has one):

```bash
curl -X POST "https://data.wayai.pro/v1/tokens" \
  -H "Authorization: Bearer $TOKEN" -H 'content-type: application/json' \
  --data '{"name": "support session",
           "toolset_binding": {"base_id": "my-base", "toolset_id": "support-agent",
                               "customer": {"external_id": "cus_123", "external_source": "crm"}}}'
```

A token naming a customer must be bound to a customer-scoped toolset. It expires within 24 hours: 24 hours from the mint when `expires_at` is omitted. A later `expires_at` is refused.

A WayAI hub's agents name their customer on their own: a [Base Toolset](../connections.md#base-toolset) connection's call to a customer-scoped toolset names the contact the conversation's WhatsApp phone, Instagram id or app account names, on a scheduled follow-up in that conversation too, and is refused on any other channel. An eval on a preview hub names the contact its scenario or journey declares ([`customer_contact_id`](../evals.md#name-the-customer-customer_contact_id--customer-scoped-toolsets)).

A WayAI contact's customer record is the root record whose `external_id` is the contact id and whose `external_source` is `wayai_contact`. That source is reserved. Only a person signed in to WayAI keys a record under it, or a seed fixture on a preview base, so an eval can seed a customer. Any other write that would key a record under it — a new record, or one re-keyed onto it — is refused, a tool call included. A single write answers `RESERVED_EXTERNAL_SOURCE` (403). A batch or an import answers `BATCH_OPERATION_FAILED` (403), naming the refused operation, with `RESERVED_EXTERNAL_SOURCE` as its `details.original_error.code`. Updating a record that already holds such a key, without changing the key, is not refused.

#### Linking a contact to its customer record

An organization admin signed in to WayAI links a contact to an existing record of the base's customer record type (the one `root` type), and unlinks it, with the CLI or the API:

```bash
wayai login                                   # a browser sign-in: an API token is refused
wayai contacts link-customer <contact_id> --base <base_id> --record <record_id>
wayai contacts unlink-customer <contact_id> --base <base_id>
# API: PUT    /api/contact-book/contacts/<contact_id>/customer?organization_id=<org_id>  {"base_id": "...", "record_id": "..."}
#      DELETE /api/contact-book/contacts/<contact_id>/customer?organization_id=<org_id>&base_id=<base_id>
```

- **Who.** A person signed in to WayAI who manages the organization's contact book and may write its data: an organization admin. An API token is refused, whatever its permissions, including one holding `contacts:write`. `wayai login --token` and `WAYAI_TOKEN` are tokens, so these commands need `wayai login` without `--token`.
- **Link** keys the record under the contact: `external_id` is the contact id and `external_source` is `wayai_contact`. The record must hold no external key, or the contact id under another source (an unlinked record: only the source changes). A record keyed under another `external_id`, such as a CRM's id, is refused with `customer_record_keyed_elsewhere`: re-keying it would orphan its owned records, whose `owner` holds the old key. Link a record holding no key instead, or create the customer record under the contact (`wayai records upsert <type> --base <base_id> --data <json|@file> --external-id <contact_id> --external-source wayai_contact`). A contact is linked to at most one record of a base; linking another is refused with `contact_customer_taken` until the first is unlinked. A base that declares no customer record type, or more than one, is refused with `customer_root_unavailable`.
- **Unlink** moves the record's `external_source` to `wayai_contact_unlinked` and keeps its `external_id`, the contact id. Conversations with the contact then name a customer no record holds, so their calls are refused. The record and its owned records stay as they are: an owned record's `owner` matches `external_id` under any source. Linking the same record again moves the source back.
- **Linking an unlinked contact to a different record** leaves two customer records holding the contact id: the unlinked one and the new one. The new one serves the contact, but its owned record types are unreachable, since an `owner` value two customer records hold belongs to neither. Re-key or delete the unlinked record to clear it. Unlinking the new one is refused with `customer_unlinked_key_taken` while the old one holds the unlinked key.
- A record changed between the read and the write is refused with `customer_record_changed`: run the command again.

What a call reaches:

- **`root`:** the customer's own record. `get` answers another customer's record as not found, and `list` returns only the customer's own.
- **`owned`:** the records whose `owner` holds the customer record's key — the key the owner's `x-fk` names (`external_id` by default). The same applies to `get` and `list` as for the root.
- **`shared`:** every record, as far as the tool allows.
- **File tools:** `list`, `get` and `get_url` as configured. `put`, `delete` and `put_url` are refused: under a customer scope, files are shared documents. A toolset switched to customer scope stops its outstanding upload URLs.

The rule applies to every filter an agent sends, archived records included.

**Relationship `list` tools.** These answer only links whose other end both the toolset reads ([one hop](#relationship-list-tools-one-hop)) and the customer reaches: its own record, a record it owns, a shared record, or a file. A node's `fields` are those of the record the scope was judged on. A walk through a shared record does not widen the scope. From a product, the orders linked to it come back only when they are the customer's. Such a call starts from a record the customer reaches; any other start is answered as not found. Owned records are matched as the base holds them now, so an archived order is not reached by a link. A link whose other end is a record type the relationship type no longer names on either end is left out too, so narrowing a relationship type's record types hides the links it no longer admits.

**Writes.** A record tool's `create`, `update` and `delete`, a relationship tool's `create` and `delete`, and every step of a composite write only inside the customer's reach:

- **An owned record is the customer's.** Its `owner` is set to the customer record's key, whether or not the call names it. A call naming another owner is refused.
- **Another customer's record is not there.** An `update` or `delete` of a record the customer does not reach is answered as not found. A `create` whose `external_id` such a record holds — or a link `create` whose `external_id` a link outside the reach holds — is refused with `CONFLICT` (409), and a composite `update` step whose `external_id` such a record holds is answered as not found. A free key creates in both cases, so either answer does tell the call that a record or link outside its reach holds the key, though nothing of it.
- **References stay inside.** Every `x-fk` a written record holds must name a record the customer reaches: its own, one it owns, or a shared one. A reference to any other record is refused exactly as a reference to a record that does not exist (400), on a shared record too.
- **The customer record is updated, never created or deleted.** A call writes only the customer's own: a write of any other customer record, or one that would create one, is answered as not found. Its `external_id`, its `external_source`, and every field an `owned` type's owner matches on (the owner's `x-fk.key`) never change, and its delete is refused.
- **Both ends of a link are inside.** A link joins only records the customer reaches, as a `list` reads them, and a call deletes or overwrites only such a link. Any other end, or a delete of any other link, is answered as one that does not exist. Deleting a link of a `cascade` relationship type to a file, or overwriting one (a link `create` whose `external_id` it holds), is refused, since dropping it removes the file unless another such link still holds it ([files.md](files.md)).
- **A composite is all or nothing.** A step refused under these rules rolls back every step of the call, which answers `BATCH_OPERATION_FAILED`, naming the refused step, with the refusal's code as its `details.original_error.code`.
- **A source-backed record type** is written only when it is `shared`.

**Triggers.** An `internal_write` [trigger](integrations.md#triggers) that such a write sets off is located through an `x-fk` into its target ([Customer scope](#customer-scope)), which the write has kept within reach, so it patches a record the customer reaches. A base that holds one located otherwise — stored before the rule, cloned from such a source, or kept by a push that `--prune` did not finish — refuses a customer-scoped write on that record type with `CUSTOMER_SCOPE_UNSUPPORTED`, naming the trigger. No trigger fires on that patch while the config holds the rule: every trigger on the target keeps `skip_cascade_writes: true`. An `async` patch queued across a config change is the exception ([Customer scope](#customer-scope)). A base that holds one with `false` is refused the write the same way, naming it and the `internal_write` whose patch would set it off.

A call is refused when:

| Code | Status | When |
|---|---|---|
| `CUSTOMER_SCOPE_REQUIRED` | 403 | The call names no customer. |
| `CUSTOMER_SCOPE_MISMATCH` | 403 | The call names a customer, and the toolset is not customer-scoped. |
| `CUSTOMER_NOT_FOUND` | 404 | No live record of the root type matches the customer. |
| `CUSTOMER_SCOPE_UNAVAILABLE` | 409 | The base declares no root record type, or more than one. |
| `CUSTOMER_SCOPE_UNSUPPORTED` | 403 | The call reaches a record type the customer cannot. That is a type that declares no role, or an `owned` type whose key this customer record does not hold (an empty value counts as none) — or holds in common with another customer record, under any `external_source`. A `list` is also refused for a customer whose key reads as a full date-time (`2026-01-01T00:00:00Z`): a filter compares those as instants, not as text. A write is also refused when it writes a file, names another owner, deletes the customer record or changes one of its keys, deletes or overwrites a `cascade` link to a file, writes a source-backed type that is not `shared`, or writes a record type an `internal_write` fires on that is not located through an `x-fk` into its target, or whose patch a trigger with `skip_cascade_writes: false` fires on. |

The customer record is read from the base's live records. Once its record type is declared `root`, archival keeps every record of the type there. A customer record archival had already moved out before then, by whichever rule moved it, is not found.

## Reviewed record types

A [reviewed record type](records.md#reviewed-records-x-review) holds drafts and approved records. Its `x-review.writers` names the toolsets that write its drafts; every other toolset that reaches it only reads its approved records.

**A writer** — a toolset its `writers` name:

- reads every record of the type, drafts included, so it can tell what is already there;
- writes drafts. Its create and update tools leave out the status and the stamp fields, and a create lands as `draft`, stamped with the toolset id in `drafted_by`. A tool call is never an approver, even a person's;
- is refused changing, deleting or approving an approved record (`REVIEW_APPROVER_REQUIRED`, 403). To correct one, it writes a new draft — a field naming the record it replaces lets the person approving it retire the old one;
- may not name the status or a stamp field in an Action's `writable_fields`: the server writes them. A create's `writable_fields` need not name the status, though it is `required`.

**A reader** — any other toolset that reaches the type through an Action, a composite step or a relationship tool:

- `list`, `get` and a relationship `list` answer only approved records whose `audience` holds the toolset's own id or `*` (every approved record, when the type declares no audience). Any other record is answered as not found, and a link to one is left out; a relationship `list` starting from one is answered as not found. The rule is not a tool parameter: an agent cannot see it or turn it off, and it applies to every filter the agent sends. A list of archived records of the type answers none; a `get` still reads an archived record that meets the rule;
- holds no write on the type: a create, update or delete Action, a composite step that writes it, or a `batch` that allows any operation on it under a key naming the record type ([What a `batch` key names](#toolsets): a key the toolset has a relationship tool for names that relationship type instead);
- holds no `sql_query`, which reads every row of the base, unless the toolset is a writer of every reviewed type in the base;
- holds no [SQL Action](#sql-actions) that declares the type: its query reads every row of the type, drafts included.

A toolset that breaks either list is refused when it, an Action it references, or the record type is written, and at [promotion](#promotion-blocking). A config restored without these checks — a rollback, or a preview clone of a source that already had the problem — still refuses each such call with `REVIEW_READER_REFUSED` (403). The same code refuses every reader of a type whose `x-review` is malformed, whose status or audience field is not one plain top-level name, or, when an audience is declared, a reader whose own toolset id reads as a date-time (`2026-01-01`).

A [customer-scoped](#customer-scope) reader of a `shared` reviewed type is held to both rules at once.

**What the resolve reports.** `wayai toolsets get --resolved` carries, under `record_types`, the schema (its `x-review` included) of each record type the toolset's Actions, composite record steps and SQL Actions reach, and of every reviewed record type it reaches any other way: at either end of a relationship tool's or a composite relationship step's relationship type (every reviewed type, when that type names none on an end), in a `batch` entry that names a record type, and every reviewed type for `sql_query`. So whoever holds the toolset can tell from `x-review.writers` which reviewed types it writes.

On a WayAI hub, a customer-facing agent never holds the tools of a writer of a type they reach, and may hold a reader with no customer scope when every record type its tools reach is reviewed and `shared` ([Base Toolset](../connections.md#base-toolset)).

For a worked example, with a writer toolset, a reader per agent kind and what a person approves, see [Recipe: notes for agent kinds](records.md#recipe-notes-for-agent-kinds).

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

- removing a record type, relationship type, or file type the toolset exposes,
- dropping an Action a reference points at,
- removing a field its `filterable_fields`, `writable_fields`, `base_filter`, or `precondition` depend on,
- removing a named write binding an Action's `binding` selects,
- a toolset naming a tool `describe_graph`, the [reserved](#navigating-the-graph) name,
- breaking a [customer scope](#customer-scope): a second `root`, an `owned` type whose `owner` no longer points at the root, a customer-scoped toolset left reaching a record type without a role or holding a SQL Action over a type that is not `shared`, an `internal_write` trigger on a type such a toolset writes that is not located through an `x-fk` into its target, or a trigger with `skip_cascade_writes: false` on a type such an `internal_write` patches.
- breaking a [reviewed record type](#reviewed-record-types): a malformed `x-review`, a toolset its `writers` do not name holding a write on the type or a SQL Action that declares it, `sql_query` on a toolset that is not a writer of every reviewed type, a writer's `writable_fields` naming the status or a stamp, or an `internal_write` trigger targeting the type.

So promote the Actions and the toolset **together with** the schema change. `--dry-run` lists any such conflicts first. Promotion is human-run.

---

## Per-Action grammar

Everything below is Action config (`wayai actions upsert <id> --config '{…}'`), not toolset config.

### Tool names

An Action's generated MCP tool name defaults to its **`id`**. A toolset reference can override two things per surface without touching the Action:

- **`action_name`** — rename this Action *in this toolset*: `{ "action": "search_invoices", "action_name": "find_invoices" }`. The `--action <id>=<surface_name>` flag form is the same thing.
- **`description_override`** — replace the Action's default description for this toolset.

Tool names must be unique across the toolset, so if you reference the same Action twice (or two Actions that would collide) give one a distinct `action_name`. No tool may be named `describe_graph`, which is [reserved](#navigating-the-graph). **Relationship** tools are named by the inline `name`/`names` map on their entry: `name` sets the base noun (link / list / unlink over it), `names` overrides a specific operation (`{ "create": "assign_payment" }`). Every key inside `names` must be a valid relationship operation (`create`/`list`/`delete`), or the override is rejected at config write.

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

**It binds this one tool, not the record type.** Every call of this `list` tool applies it — and nothing else does. A toolset-bound token can call only its toolset's tools, so what an agent can read is exactly what *any* of those tools returns: a tool that returns or reaches a record of the same type by key or by link (`get`, `update`, `delete`, a relationship `list`, a composite or `batch` write) returns rows the filter excludes, a [SQL Action](#sql-actions) that declares the type reads every row of it, `sql_query` reads every row of the base, and a base token on the plain API reads everything. To keep rows from an agent, `base_filter` alone is not enough: no tool in its toolset may be able to return them.

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

**Toolset-bound (consumer) tokens.** For least privilege, mint a token whose authorization **is** the toolset: it can call that toolset's tools through its MCP endpoint and nothing else. The plain API (records, relationships, batch, SQL) refuses it with a 403, even for a record type its toolset has a tool for; use a normal base token for the plain API. A leaked one cannot reach past the tools you exposed, and you rotate or revoke one per agent — so what such a token can do is exactly what those tools can.

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

**Permissions:** `read:records`, `write:records`, `read:record_types`, `write:record_types`, `read:relationships`, `write:relationships`, `read:relationship_types`, `write:relationship_types`, `read:attachments`, `write:attachments`, `read:files`, `write:files`, `read:file_types`, `write:file_types`, `read:inbound_webhooks`, `write:inbound_webhooks`, `read:triggers`, `write:triggers`, `read:toolsets`, `write:toolsets`, `read:actions`, `write:actions`, `read:seeds`, `write:seeds`, `read:bases`, `write:bases`, `read:audit` (read-only change history; admin-gated, not implied by other grants), `write:imports` (bulk historical import — separate from `write:records` because an import commits without firing triggers, and not implied by other grants), `read:base_credentials` / `write:base_credentials` (a base's credentials; metadata only on read — a stored value is never returned, and neither is implied by other grants), or `*` for all.

**Scope fields:** `bases` (required), `record_types` (optional — omit for all), `environments` (optional — `production`, `preview`, or omit for both).

Tokens enforce a privilege ceiling: you can only create tokens with equal or narrower scope than your own.

### Revoking and pruning

Revoking a token that was used in the last 7 days, or that is toolset-bound (a consumer token an MCP connection likely holds), is refused with a warning; `--force` overrides. Off a TTY the command prints the reason and exits 1 rather than auto-confirming. Do **not** `--force` (or prune with `--include-bound`) unless a human has confirmed nothing depends on the token — revocation is permanent and breaks live traffic immediately, a hub's Base Toolset connection holding it included, with one exception for older connections ([connections.md](../connections.md#base-toolset)). Expired tokens revoke without friction.

```bash
wayai bases tokens prune                        # expired tokens only (preview)
wayai bases tokens prune --unused-since 30d     # + tokens with no activity for 30 days
wayai bases tokens prune --unused-since 30d --yes
```

`prune` selects by **staleness only** — expired, or unused past the window — never by name, and previews before revoking (`--yes` is required non-interactively). Expiry is authoritative: an **expired** token is pruned regardless of binding, so `--include-bound` does not gate that half. The bound-token skip (and `--include-bound`) applies to the *unused-past-the-window* branch only, which also always skips legacy tokens whose usage was never tracked — "stale" is unknowable for those; revoke them individually if truly unused.

## Credentials behind a base's integrations

Credential storage behind a base's external sources — API keys, signing secrets, or credential-grade blobs (a client certificate, keystore, service-account JSON). Values are encrypted at rest and always masked in responses. Reference them from source or trigger config as `credential:<name>` (an executor's signing secret, see [`executors.md`](executors.md) and [`integrations.md`](integrations.md)), or have an executor read one at runtime.

**A credential belongs to one base.** A `credential:<name>` reference resolves against the credentials of the base serving the request, so the same configuration promoted or cloned elsewhere reads that base's own credential of that name. A preview is created with a copy of its origin's credentials, so cloned integrations keep working.

**Promotion publishes what you opt in.** Each credential carries a publish flag (on by default): promoting a preview copies its opted-in credentials onto production, overwriting the production credential of the same name. Turn the flag off for anything production owns — a live API key you set there directly — and the promotion leaves it untouched. Credentials linked from an organization credential are never published this way; they are re-linked on production instead. Whatever a promoted source references and production still lacks is listed by name in the promotion result, before and after cutover.

**Two ways a base gets one.** Create it **directly** on the base, with the commands below or in the base's Credentials tab. Or **link an organization credential** into the base: the organization credential stays the one place the secret is entered and rotated, the base gets a mirror of it, and rotating the organization credential reaches every base linked to it at once. Linking applies the same tag and environment rules that gate an organization credential's use anywhere else, and only single-value credential types project — an API key or a bearer token, not a username-and-password pair, because a base substitutes one value.

A linked credential is **unlinked**, never deleted, and never rotated on the base: rotating it there would write a value the organization credential does not know about, which the next organization-level rotation would silently overwrite.

**Linking is done by a person.** Creating or removing an organization link marks a credential as the organization's on someone's behalf, so it requires a signed-in session — the dashboard, or a CLI logged in with `wayai login`. An API token cannot do it. This also means a promotion run from CI cannot re-link: the promotion still applies, and its result names every organization-linked credential production is left without, so someone can link them once from the dashboard.

**Secret values never travel in argv** — there is deliberately no `--value` flag. An argv element is readable from `/proc/<pid>/cmdline` on a default host and lands in shell history and CI job logs. A value arrives from a pipe, a masked prompt, or a file.

```bash
# From a pipe (recommended for CI)
printf '%s' "$STRIPE_KEY" | wayai bases credentials create --base crm --name stripe-key --value-stdin

# From a masked interactive prompt
wayai bases credentials create --base crm --name stripe-key --value-prompt

# A credential file, base64-encoded, with its MIME type and an expiry
wayai bases credentials create --base crm --name partner-cert --file ./partner.p12 \
  --content-type application/x-pkcs12 --expires-at 2027-01-01T00:00:00Z

# Production owns this one — do not publish it when this base is promoted
wayai bases credentials create --base crm --name live-key --value-stdin --no-sync-to-production
wayai bases credentials set-sync <id> --base crm --no-sync-to-production   # or change it later
wayai bases credentials set-sync <id> --base crm                            # publish it again

wayai bases credentials list --base crm           # metadata; values are never returned
wayai bases credentials get <id> --base crm       # metadata for one
wayai bases credentials rotate <id> --base crm --value-stdin   # never auto-generates
wayai bases credentials delete <id> --base crm [-y]

# Link an organization credential into this base, and remove that link.
# Both require a signed-in person — an API token cannot create or remove an organization
# link, so these two are not available to CI.
wayai bases credentials link --base crm --org-credential <organization-credential-id> [--name stripe-key]
wayai bases credentials unlink <id> --base crm --org-credential <organization-credential-id>
```

`--file` cannot be combined with `--value-stdin` or `--value-prompt`: with `--file` silently winning, a CI job that piped the new credential and also named a stale file would store the stale one and report success. `delete` requires `--yes` when non-interactive, and refuses a linked credential — use `unlink`.

Set `--expires-at` (ISO-8601) on credentials that lapse — yearly certificates, time-bound tokens — so `list` shows what needs renewing before it breaks an integration. Expiry is informational; a credential is never auto-disabled.

An **inline** source credential is not carried over by `wayai bases promote` — set it on production directly. A `credential:<name>` reference resolves against the credentials of the base serving the request, and a promotion publishes the preview credentials you opted in (see above); either way the promote prints every credential production will still lack.

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

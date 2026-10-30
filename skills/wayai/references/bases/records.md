# Record types, records, relationships

Open this before authoring a record-type schema, writing or querying records, linking entities, or
loading data in bulk. Concepts are in [README.md](README.md); this is the command and schema grammar.

Record types, relationship types and file types are **config** — write them in a preview base, then
promote. Records and relationships are **data** — write them anywhere.

## Record types

```bash
wayai record-types list --base <base>
wayai record-types get <id> --base <base>
wayai record-types upsert <id> --base <base> --name <name> \
  [--schema <json|@file>] [--description <desc>] [--icon <name>] [--color <hex>] [--sources <json|@file>]
wayai record-types delete <id> --base <base> [-y]
wayai record-types history <id> --base <base> [--limit <n>] [--offset <n>] [--diff]
```

A record type is a JSON Schema. Only `--name` is required; for a schema longer than a line, write it
to a file and pass `--schema @path/to/schema.json`. `--icon`/`--color` are display hints. `--sources` declares external
sources — see [integrations.md](integrations.md).

Deleting a record type also removes all of its records and any relationships pointing at them, with
the files those relationships owned (a [`cascade`](files.md) link) that no other owning link still
holds; a record type recreated with the
same id starts empty.

## Records

```bash
wayai records upsert <record_type> --base <base> --data <json|@file> \
  [--id <uuid>] [--external-id <id>] [--external-source <src>]
wayai records get <record_type> <id> --base <base> [--external-source <src>]
wayai records query <record_type> --base <base> \
  [--filter <json|@file>] [--sort <json>] [--fields <list>] [--limit <n>] [--offset <n>] [--external-source <src>]
wayai records delete <record_type> <id> --base <base> [-y]
wayai records history <id> --base <base> [--limit <n>] [--offset <n>] [--diff]
```

`--external-id` is what makes a write idempotent: re-upserting the same external id updates the
existing record in place instead of creating a twin. Set it on every record with a natural key.

**Who wrote it.** Every record carries `created_by` and `updated_by`, and every relationship
`created_by`. The server writes them from who is calling, and ignores any value a write sends for them,
on every way of writing: a single write, a toolset's tool, a batch, a composite Action, an import, a
seed fixture. They hold ids only, never a name or an email: `user:<WayAI user id>` for a person,
`token:<id>` for an API token or an agent (the agent's id), `inbound_webhook` for an inbound webhook,
and `system` for a trigger. `created_by` names the writer that created it and never changes;
`updated_by` names the last writer that wrote its data; a delete and a cancel leave it as it was
(`history` names them). Both are empty on a record an external source serves live, and a record or
relationship written before the server wrote them keeps what that write sent.

`history` returns every version as a snapshot with the actor, operation and timestamp. **Admin-only**
— available to platform admins or a token granted `read:audit`; ordinary agent tokens are rejected.
The same subcommand exists on `relationships`, `record-types` and `relationship-types`.

Each version also carries `toolset_id`: the toolset whose tool made the change, and empty for every
other writer (a person, an API token, an import, a trigger), and for changes recorded before toolsets
were tracked. That is how you read one agent kind's changes apart from everyone else's. The base-wide change feed, `GET /v1/<base>/audit`, takes the same
grant and narrows by `entity_type`, `record_type`, `actor_id`, `operation`, `from`/`to`, `toolset_id`
(the changes made through one toolset's tools) and `source` (how the change came in: `api`, `cli`,
`mcp`, `import`, `seed`, `inbound_webhook`, `cascade`, `promote`, `archival` or `snapshot`).

There is no `records list`: listing IS `records query` with no `--filter`.

### Filtering & search

`query` takes a Filter DSL expression — a condition `{field, op, value}` or an `and`/`or` group of
them. Operators: `eq`, `neq`, `gt`, `gte`, `lt`, `lte`, `in`, `not_in`, `like`, `ilike`, `is_null`,
`is_not_null`, `has`, and **`search`**. This DSL is the only accepted form — Mongo-style shorthand
(`{"data.field":"value"}`, `{"field":{"$in":[...]}}`) is **not** supported.

`search` matches a field by **approximate** value — use it when you have a near-correct string but
not the exact stored one (a model name, a place, a SKU with a possible typo). Results come back
ranked by closeness, each carrying a `_search_score` (0–1, higher = closer). Every field is searchable
by default; no setup needed. Exact conditions and `search` compose — put the exact legs alongside the
`search` leg so they narrow the set and `search` ranks within it:

```jsonc
// "vehicles whose model is close to 'honda civic', active ones only"
{ "and": [
  { "field": "data.status",    "op": "eq",     "value": "active" },
  { "field": "data.car_model", "op": "search", "value": "honda civic" }
] }
```

To **tune** how a field is matched — `x-search` modes (`name`/`text`/`fuzzy`), `threshold`, and
`searchable: false` — see [querying.md](querying.md). To expose `search` to an **agent** through a
toolset, declare `match: search` on a `list` Action's `filterable_fields`; that is the only match that
reaches this operator ([toolsets.md](toolsets.md)).

**Consistency.** Listing and filtering active records is read-after-write consistent — a record you
just wrote appears in the very next query. Two shapes instead read from recently-synced data, so a
just-written record may take a moment to appear: `search` queries, and filters comparing the built-in
`created_at`/`updated_at` timestamps to a value written **without a timezone offset**. Requesting a
projected subset of fields never moves a query onto that path. Give any `created_at`/`updated_at`
bound an explicit offset (`2026-08-01T00:00:00Z`) to stay on the consistent path.

### Sorting

`--sort` takes a **JSON array** of terms, each `{"field":"data.<field>","direction":"asc"|"desc"}`:

```bash
--sort '[{"field":"data.created_at","direction":"desc"}]'
```

Multiple terms sort lexically (first term primary); `direction` defaults to `asc`. The colon shorthand
`data.name:asc` is **not** accepted. Omit `--sort` when using `search` so the relevance ranking
survives.

### Datetimes

Declare datetime fields in the schema with `"format": "date-time"` (or `"format": "date"` for
date-only). Values are stored in one canonical form: a naive value (no offset) gets the record type's
timezone attached, so `2026-06-11T13:00:00` is stored and returned as `2026-06-11T13:00:00-03:00` —
the same shape from a query and a single-record read, with no UTC math required of you. The timezone
resolves record type `x-timezone` → base `settings.timezone` → UTC.

**Filtering is instant-aware:** `eq`/`neq`/`in`/`not_in` and the range operators match datetimes by
point in time, so the exact representation you pass (`T` vs space, with or without offset) does not
change the result. Use `eq` for an exact datetime, never `like` (plain substring). Raw
`wayai bases sql` is **not** rewritten this way — prefer the Filter DSL for datetime conditions.
Timezone configuration is in [querying.md](querying.md).

### Partial vs full updates

A full upsert writes the **whole** record — send every field. To change just one field without
re-sending the rest, use a **partial update**: the `patch` operation on the MCP record tool, an
`update` Action exposed through a toolset (its tool name is whatever `id` you gave the Action —
[toolsets.md](toolsets.md)), or REST `PATCH .../records/<record_type>/<id>`. A
partial update merges the provided fields over the stored record (top-level merge; nested objects are
replaced wholesale), targets an existing record by id or `external_id`, and the merged result must
still satisfy the schema.

For a record type backed by an **external source**, the provided fields are forwarded upstream, which
decides how the update is applied — the merge-and-keep-the-rest guarantee applies to records stored in
the base.

### Cancellation & archival

Records are **updatable by default** — re-upserting the same `external_id` updates in place, and you
can advance state freely (`scheduled → confirmed`). A record type can opt into stricter archival via
its schema's `x-archival.mode`:

| Mode | Behavior |
|---|---|
| `immutable` | Write-once — any update is rejected |
| `on_attribute` | Updatable until a configured terminal value |

A record can also be **cancelled** — a first-class state distinct from delete — via the MCP record
tool's `cancel` operation or REST `POST .../records/<record_type>/<id>/cancel`. There is no CLI
`cancel` subcommand. A cancelled record can no longer be updated.

Records may be **archived** automatically (on reaching a terminal/immutable state, or after long
inactivity). An archived record stays readable and can still be cancelled, but cannot be deleted, and
re-upserting the same `external_id` creates a **new** record rather than updating the archived one.

**List queries return active records by default.** To reach archived rows through a query, add a
condition on `archived` (`{"field":"archived","op":"eq","value":true}`) — your condition then decides
which tier you see. Relationship traversal has no filter override; archived links are reachable only
via raw SQL, which applies no default at all (add `archived = false` yourself; cancelled rows carry
`cancelled = true`).

### Referential integrity (`x-fk`)

An immediate child of the schema's root `properties` can be declared a **foreign key** into another
record type with an `x-fk` hint. Every native write then checks the value points at a real record — a
missing reference is rejected with an actionable error instead of landing as silent bad data.

```jsonc
"patient_id":      { "type": "string", "x-fk": { "record_type": "patients" } },                     // matches patients by external_id (default)
"professional_id": { "type": "string", "x-fk": { "record_type": "professionals", "key": "code" } }, // matches a named field on the target
"note_patient_id": { "type": "string", "x-fk": { "record_type": "patients", "key": "id" } }         // matches the target's system id
```

- `key` is the field on the **target** the value must match — `external_id` by default, the target's
  system `id`, or any top-level field (a `code`).
- Use `key: "id"` when the target is a native record with no external id. The system id is assigned at
  creation, so create the target first and read its `id` back — which cannot be done in one batch pass
  or a seed fixture (both resolve by `external_id`), so use `external_id`/data keys there.
- Checked on every write (create, full upsert, partial update, batch). Empty/absent values are
  skipped — mark the field `required` if it must be present.
- **The restriction is on the referencing type**, which must be native (`sources: []`). The target may
  itself use external sources; the local existence probe is skipped for such a target because its rows
  live upstream.
- Nested/composed declarations (`insurance.plan_id`), declarations on a source-backed referencing
  type, and declarations in a relationship type's `data_schema` are rejected at config write with the
  exact schema pointer. Hoist the reference to a root property, drop `x-fk`, or make the referencing
  type native. Existing production config keeps serving while it is corrected in a linked preview; a
  promotion dry run reports blockers across the full resulting config and a real promotion stays
  blocked until they are fixed.
- In a **batch**, write the target before the record referencing it (the batch is atomic — a bad
  reference rejects the whole batch). That ordering helps `external_id`/data keys only.
- Use foreign keys for **reference data** (patients, plans, products), not high-volume event logs.

### Customer scope (`x-customer-scope`)

A record type can say what it is to a [customer-scoped toolset](toolsets.md#customer-scope) — one that serves a single customer per call — with `x-customer-scope` at the root of its schema:

```jsonc
// customers — the customer record type
{ "type": "object", "properties": { "name": { "type": "string" } }, "x-customer-scope": { "role": "root" } }
// orders — each order belongs to the customer its customer_id names
{ "type": "object",
  "properties": { "customer_id": { "type": "string", "x-fk": { "record_type": "customers" } }, "total": { "type": "number" } },
  "x-customer-scope": { "role": "owned", "owner": "customer_id" } }
// products — no customer data
{ "type": "object", "properties": { "title": { "type": "string" } }, "x-customer-scope": { "role": "shared" } }
```

- `root` — the customer record type. A base has at most one.
- `owned` — `owner` names a top-level property declaring `x-fk` into the root record type.
- `shared` — holds no customer data.
- `root` and `owned` need a native record type (`sources: []`); a source-backed type can only be `shared`.
- A record type that declares none is unreachable from a customer-scoped toolset.
- A WayAI hub's call to a customer-scoped toolset is [customer-facing](access.md#customer-facing-calls): it reaches a record type only when the type also declares person entries in `enforce` or `x-access: [{ shared: true }]`, so declare one on each type it should reach (`{ shared: true }` on a `shared` catalog).

Checked at config write and at promotion: the declaration's shape on the record type itself, and across the base, one root and every owner pointing at it.

- A single write or a batch that would leave two roots, or an owner pointing elsewhere, is refused.
- A preview clone copies its source as it is, problems included, so it is refused only for a problem the source did not already have. A re-clone that brings the origin's root back beside the root the preview moved elsewhere is one example.
- A `wayai push` is judged on the config its files describe: what it would remove counts as gone. One push can therefore move the root to another record type, or remove a root together with the types it owns, when it runs with `--prune`. Without `--prune` the removed types stay on the base.
- A single delete is not checked, since `--prune` deletes one record type at a time.
- A base left with two roots or an owner without one refuses the customer-scoped calls that depend on it, and a promotion refuses it.

### Access dimensions (`x-access`)

A record type can hold records that only some people, hubs and agents may reach, by naming a `type: string` field (or a list of strings, [List fields](access.md#list-fields)) that holds a value of one of the organization's access dimensions:

```jsonc
{ "type": "object",
  "properties": { "class": { "type": "string" } },
  "x-access": [ { "field": "class", "dimension": "class", "mode": "enforce" } ] }
```

`mode` is `enforce` (filters the type for calls bound to a toolset; switch back to `report` to undo) or `report` (counts what enforce would hide or refuse, filters nothing), for a type that already has live users. Assign values before declaring `enforce`: a hub or person with none reaches nothing on the type. A native record type only. `x-access` also takes person entries (`{ "field": "guardian_ids", "person": true, "owner": true, "mode": "report" }`), naming the field that holds each record's people: in `enforce`, a hub toolset call reads only the records naming each person it is for and writes them only as those people allow ([Person writes](access.md#person-writes)), a customer-facing call included, and a token bound to a toolset reaches none unless it is [bound to persons](access.md#tokens-bound-to-persons) ([Person entries](access.md#person-entries)). A customer-facing call reaches only such types and those declaring `x-access: [{ shared: true }]` ([Customer-facing calls](access.md#customer-facing-calls)). The declaration, the organization's dimensions, who is assigned which values and what an enforcing type refuses are in [access.md](access.md).

### Reviewed records (`x-review`)

A record type can hold records that a person approves before agents read them, such as knowledge notes: agents write drafts, a person approves them, and only approved records reach the agents that read them. Declare it with `x-review` at the root of the schema:

```jsonc
{ "type": "object",
  "properties": {
    "title":       { "type": "string" },
    "body":        { "type": "string" },
    "status":      { "type": "string", "enum": ["draft", "approved", "retired"] },
    "audience":    { "type": "array", "items": { "type": "string" } },
    "approved_by": { "type": "string" },
    "approved_at": { "type": "string", "format": "date-time" },
    "drafted_by":  { "type": "string" }
  },
  "required": ["title", "status", "audience"],
  "x-review": {
    "field": "status",
    "audience": "audience",
    "writers": ["notes-curator"],
    "stamps": { "approved_by": "approved_by", "approved_at": "approved_at", "drafted_by": "drafted_by" }
  } }
```

- `field` names the status: a top-level property listed in `required`, whose `enum` is all strings and holds `draft` and `approved`. Any other value (`retired`) means not approved.
- `audience` (optional) names a top-level array of strings listed in `required`: the toolset ids that read each record, or `*` for every reader. An id that names no toolset matches nothing.
- `writers` lists the toolset ids that write drafts ([toolsets.md](toolsets.md#reviewed-record-types)). An id that names no toolset is inert, so the record type can be written before its toolsets.
- `stamps` (optional) names the fields the server writes: `approved_by` and `drafted_by` (strings) and `approved_at` (`format: date-time`). None of them may be listed in `required`.
- The record type is native (`sources: []`) and stays updatable: no `x-archival` mode other than `mutable`, and no terminal `x-workflow` state, so a person can always edit an approved record.

**Who approves.** Only a person signed in to WayAI: the console, the CLI after `wayai login` (not `--token` or `WAYAI_TOKEN`), and that person's REST writes, batches and imports. A seed fixture on a preview base approves too, so a preview can hold approved records. Every other writer writes drafts: a toolset call (a person's included), an API token, an agent, an inbound webhook, a trigger. Such a writer:

- creates a draft when it sends no status;
- is refused setting `approved`, moving a record out of `approved`, changing an approved record (its data, or its `external_id` or `external_source`), and deleting or cancelling one. A full replace that leaves out the status would move the record to `draft`, so it is refused too;
- is refused cancelling an archived record of the type, whose status cannot be read;
- may write again what an approved record already holds: a write that changes nothing passes, so a nightly import re-asserting the same rows keeps working.

A deleted record written again counts as a new record. A person may do anything; editing an approved record keeps it approved.

A refused write answers `REVIEW_APPROVER_REQUIRED` (403), naming the record type. A batch, a composite or an import answers `BATCH_OPERATION_FAILED` (403), naming the refused operation, with `REVIEW_APPROVER_REQUIRED` as its `details.original_error.code`. An [import rollback](#bulk-import) by any other writer keeps the records that are approved now, and its REST answer reports them under `kept.records`; a person's rollback of the same import removes them. Before a rollback runs, its plan, asked for its counts (`GET /v1/<base>/import/<import_id>/rollback-plan?counts=true`), reports the same `kept.records` for the caller beside `matched_records`, so `matched_records` minus it is what the rollback would remove now. Neither answer carries `kept` when the rollback keeps nothing. `wayai bases import rollback` asks to delete exactly that difference, saying how many approved records stay, and asks nothing when it would delete none; once done, it prints the count the rollback kept.

**Stamps.** The server writes the fields `stamps` names, and ignores any value a write sends for them. They hold ids only, never a name or an email:

- `approved_by`: `user:<WayAI user id>`, or `seed:<fixture>` for a seed fixture. `approved_at`: when. Both are written when a person approves a record or edits an approved one, and removed when the record leaves `approved`.
- `drafted_by`: written whenever a writer that is not an approver changes a record: the toolset id for a toolset call, `token:<token id>` for an API token, `inbound_webhook` or `system` (a trigger) otherwise. A person's writes, and any write that changes nothing, leave it as it was.

They are ordinary fields: `records get` and `records query` return them, and a filter can select on them (`{"field":"data.approved_by","op":"eq","value":"user:..."}`).

**Not held to the rule:** deleting the record type itself, which is a config change, and the external key a trigger's `external_write` links back onto a record.

**Checked at config write and at promotion**, across the base ([toolsets.md](toolsets.md#reviewed-record-types)): the declaration's shape; a toolset its `writers` do not name holding a create, update or delete on the type (an Action, a composite step or a `batch`), or a [SQL Action](toolsets.md#sql-actions) that declares the type; `sql_query` on a toolset that is not a writer of every reviewed type; a writer's `writable_fields` naming the status or a stamp; and an `internal_write` [trigger](integrations.md#triggers) targeting the type. `x-review` promotes like any other schema key. Records do not promote: production's are approved in production, and a preview gets approved records from seed fixtures. A reviewed type's records stay in the base's active records whatever their age, so an approved record never drops out of a list.

#### Recipe: notes for agent kinds

Notes are the short procedures and recurring issues an agent kind follows, such as "a red LOS light means the fiber signal is cut: check the green connector before anything else". A back-office or curator agent writes them as drafts, a person approves them, and each agent reads only the approved notes meant for its kind. A note's `audience` names the toolsets that read it: for each kind, the id of the toolset that holds its notes tools (step 3).

**A note never holds anything about one customer:** no name, document number, phone, address, order or invoice. That is what lets a customer-facing agent read notes on every channel, including the ones that name no customer, without a [customer scope](toolsets.md#customer-scope). Facts about a customer stay in records a customer-scoped toolset reads.

**1. The record type**, `record-types/notes.yaml`, declared `shared`, open to customer-facing calls (`x-access: [{ shared: true }]`: without it a customer-facing agent's call finds no note, [access.md](access.md#customer-facing-calls)) and reviewed:

```yaml
id: notes
name: Notes
json_schema:
  type: object
  properties:
    title:
      type: string
      minLength: 1
      maxLength: 200
      x-search: { mode: text, threshold: 0 }
    body:
      type: string
      minLength: 1
      maxLength: 4000
      description: The procedure. Never a customer's name, document, phone, address or invoice.
      x-search: { mode: text, threshold: 0 }
    status: { type: string, enum: [draft, approved, retired] }
    audience:
      type: array
      minItems: 1
      items: { type: string, enum: ["*", support, notes-curator] }   # ids of the toolsets holding the notes tools, or * for all
    tags: { type: array, items: { type: string } }
    supersedes:                       # the approved note a corrected draft replaces
      type: string
      x-fk: { record_type: notes, key: id }
    approved_by: { type: string }
    approved_at: { type: string, format: date-time }
    drafted_by: { type: string }
  required: [title, body, status, audience]
  x-customer-scope: { role: shared }
  x-access:
    - { shared: true }
  x-review:
    field: status
    audience: audience
    writers: [notes-curator]
    stamps: { approved_by: approved_by, approved_at: approved_at, drafted_by: drafted_by }
sources: []
```

An `enum` on the audience items keeps a writer from naming a kind that does not exist, which would leave the note unread. Adding an agent kind then means adding the id of the toolset that holds its notes tools there. The writer's own id, `notes-curator`, marks a note for the back office alone: the writer reads every note, and no reader's id matches it.

`mode: text` with `threshold: 0` makes a search keep every note the reader may see and rank them, best match first: a few words score low against a long title or body, and a higher threshold would hide a note that holds them ([Search tuning](querying.md#search-tuning)). The tool still answers one page, the Action's `default_limit` (step 2).

**2. The Actions.** A reader's search and a writer's search are separate Actions, so each returns the fields its kind needs:

```yaml
# actions/search_notes.yaml: readers
id: search_notes
record_type: notes
operation: list
description: Search the approved notes for you (procedures and recurring issues) by title or text.
filterable_fields:
  - { field: title, match: search, optional: true }
  - { field: body, match: search, optional: true }
default_limit: 50
default_fields: id,data.title,data.tags
```

A search answers titles, not bodies, so a page of 50 costs the agent little; it reads the note it needs with `get_note`. Keep each kind's notes within that page: a note ranked past it never comes back, and the answer then says `truncated: true`. A kind that will hold more notes than a page needs `expose_offset: true` on `search_notes`, with an instruction to page on while the answer is `truncated`.

- `search_all_notes`: the same `list`, for the writer, whose `default_fields` add `data.status`, `data.audience`, `data.supersedes`, `data.drafted_by` and `data.approved_at`.
- `get_note`: a `get` on `notes`.
- `draft_note` (`create`) and `revise_note` (`update`), whose `writable_fields` are `title`, `body`, `audience`, `tags` and `supersedes`. The status and the stamps stay out: the server writes them.

**3. The toolsets.**

- **The writer**, `notes-curator`, the one toolset `x-review.writers` names: `search_all_notes`, `get_note`, `draft_note` and `revise_note`. It writes and reads as [Reviewed record types](toolsets.md#reviewed-record-types) says; to correct an approved note, it writes a new draft whose `supersedes` names it.
- **A reader per kind**, holding `search_notes` and `get_note`. Every call answers only the approved notes whose `audience` holds that toolset's id or `*`.
  - A kind whose toolset is not [customer-scoped](toolsets.md#customer-scope), such as one that reads only documents, adds the two tools to that toolset.
  - A kind whose toolset is customer-scoped gets a second toolset for its notes, with no customer scope, and its `audience` id is that second toolset's. Every call to a customer-scoped toolset names a customer, so on a channel that names none, notes on it would be refused.

On a hub, connect each toolset with a [Base Toolset](../connections.md#base-toolset) connection, whose token is bound to that toolset on the base of the hub's environment; in `hub.yaml`, the connection names the Bearer Token org credential that holds it. Give the writer's tools to an agent the team works with, such as the hub's copilot, and the reader's to the agents of its kind: a customer-facing agent may hold them, and never the writer's ([Base Toolset](../connections.md#base-toolset), *Customer-facing agents*). After adding the notes tools to a toolset the hub already connects, sync the connection's tools and assign them to the agents ([Giving agents its tools](../connections.md#base-toolset)).

**4. The instructions.** Tell a reader where notes fit beside its documents, for example:

```markdown
- For a procedure or a recurring issue (a fault, an outage, how to fix something), search the notes with
  `search_notes` by the subject's words. It answers the closest titles first: read the note whose title
  is about the subject with `get_note`. If no title is, and the answer isn't marked `truncated`, there is no
  such note. A person approved every note you get: follow it as written.
- Notes never hold customer data. A customer's details come only from <where they live>.
- Before saying you don't have some information, look in the documents and also in the notes with
  `search_notes`. If neither has it, say you don't have that information.
```

Without that last line, an agent asked something its documents don't cover tends to answer from them alone, without searching the notes.

Tell the writer what a note is and what it never holds: one procedure or recurring issue per note, the audience it is for, no customer data, `search_all_notes` before writing so it doesn't duplicate a note, and a new draft with `supersedes` to correct an approved one.

**5. What a person approves.** Signed in to WayAI (the console, or the CLI after `wayai login`), a person reads the drafts and approves each one that is right, holds no customer data, and names the right audience:

```bash
wayai records query notes --base <base> --filter '{"field":"data.status","op":"eq","value":"draft"}'
wayai records get notes <id> --base <base>
wayai records upsert notes --base <base> --id <id> --data @note.json   # the whole note, with "status": "approved"
```

`records upsert` replaces the whole record, so send every field, with the status changed. Approving a draft that `supersedes` a note means retiring that note too (`"status": "retired"`). The server stamps `approved_by` and `approved_at`.

**6. Preview and production.** The record type, the Actions and the toolsets are config: write them on a preview, then promote. Notes are data and don't promote: production's notes are written and approved in production. A preview holds notes from a [seed fixture](config-as-code.md#seed-fixtures), which may write `approved`, so an eval can show what each kind reads: a reader answers from an approved note of its own kind, and doesn't know what a draft, a retired note or another kind's note says.

### Field constraints

The schema's own keywords are enforced on every write — they are not documentation. Reach for these
before anything more elaborate:

```jsonc
"status":  { "type": "string", "enum": ["draft", "sent", "paid"] },
"email":   { "type": "string", "format": "email" },
"cpf":     { "type": "string", "pattern": "^\\d{11}$" },
"title":   { "type": "string", "minLength": 1, "maxLength": 200 },
"quantity":{ "type": "integer", "minimum": 1 }
```

`enum`, `pattern`, `minLength`/`maxLength`, `minimum`/`maximum`, and asserted `format` (`email`,
`uri`, `uuid`, `date`, `date-time`, …) all reject a bad value at write time with an error naming the
field. `required` at the schema root is what makes a field mandatory.

One bound worth knowing: **`pattern` is only evaluated on values up to 1024 characters.** A longer
value skips *that* assertion and is written — every other constraint on it still applies. Patterns
guard short identifier-shaped fields in practice, and the alternative (rejecting long values) would
start failing writes on data that was legal when it was stored. Don't use `pattern` as a length cap;
use `maxLength`, which has no such bound.

Keep patterns simple. Nested quantifiers over overlapping alternatives — `^(a+)+$`, `^(\w+\s?)*$`
and their shape — can take exponentially long to reject a near-miss value, and that time is charged
to your base's writes. Prefer a flat character-class pattern and move anything more involved into a
custom field rule below, which runs under a compute budget.

### Custom field rules (`x-validate`)

When a rule can't be written as a constraint keyword — a check digit, a comparison between two
fields, a value that must never change once set — declare it as `x-validate` on the field it belongs
to:

```jsonc
"end_date": {
  "type": "string", "format": "date",
  "x-validate": {
    "code": "({ value, data }) => value >= data.start_date ? { valid: true } : { valid: false, message: 'end_date must be on or after start_date' }"
  }
}
```

The `code` is a JavaScript **function expression** returning `{ valid, message? }`:

| Argument   | What it is |
|------------|------------|
| `value`    | This field's value. `undefined` when the field is absent (use `required` to make it mandatory). |
| `data`     | The whole record as it will be stored — after a partial update is merged. Cross-field rules read this. |
| `previous` | The record's data before this write; `null` when creating. Immutable-once-set rules read this. |

```jsonc
// check digit — the case `pattern` cannot express
"cpf": {
  "type": "string", "pattern": "^\\d{11}$",
  "x-validate": { "code": "({ value }) => { const d = String(value).replace(/\\D/g, ''); const dv = (n) => { let s = 0; for (let i = 0; i < n; i++) s += Number(d[i]) * (n + 1 - i); const r = (s * 10) % 11; return r === 10 ? 0 : r; }; return d.length === 11 && !/^(\\d)\\1{10}$/.test(d) && dv(9) === Number(d[9]) && dv(10) === Number(d[10]) ? { valid: true } : { valid: false, message: 'cpf check digits are invalid' }; }" }
}

// conditional requirement
"cnpj": { "type": "string", "x-validate": { "code": "({ value, data }) => data.person_type !== 'company' || value ? { valid: true } : { valid: false, message: 'cnpj is required when person_type is company' }" } }

// immutable once set
"invoice_number": { "type": "string", "x-validate": { "code": "({ value, previous }) => !previous || previous.invoice_number === undefined || previous.invoice_number === value ? { valid: true } : { valid: false, message: 'invoice_number cannot be changed once issued' }" } }
```

Rules of the road:

- **Your `message` is the error the caller sees**, so write it as an instruction: say what is wrong
  and what a correct value looks like. That string is how an agent corrects itself on the retry.
- **Return `{ valid: true }` explicitly.** Anything else — throwing, returning nothing, returning a
  non-object — rejects the write. The rule fails closed on purpose: a validator that quietly stops
  asserting is worse than one you never wrote.
- **Keep it a pure function of its three inputs.** It runs in a secure sandbox with no network, no
  storage, and no access to other records — and with strict time and memory budgets. A rule that runs
  long or allocates heavily is stopped and the write is rejected, naming the field and the budget.
  Cross-record checks belong in `x-fk`; anything needing a lookup or a call out belongs in a trigger.
- **Determinism matters.** Every write re-runs every rule, and promoting a config replays the new
  rules against a sample of existing production records. A rule whose answer changes over time — one
  comparing a stored date against "today", say — will eventually block a promotion or start failing
  writes on records nobody touched. Compare fields to each other, not to the clock.
- Declare rules on **root-level fields only** — a nested declaration is rejected at config write —
  at most 16 per record type, each up to 32 KB of code. Note that 16 is a cap on how many rules a
  type may declare, not a compute allowance: a record write also has a **combined** budget across all
  of its rules, so 16 individually-expensive rules can still be rejected together. Ordinary rules
  cost a tiny fraction of it.
- The code is parsed when you write the config, so a syntax error is rejected at
  `record-types upsert` rather than on the first record write.
- Rules run on **every** write — create, full upsert, partial update, batch, bulk import, seed
  fixtures, and writes made by triggers. One request's rules share a single compute budget, including
  the writes its triggers cascade into; if a large batch is rejected for exceeding it, send fewer
  operations per request.
- A **seed fixture** restore is the one exception to `previous`: it is putting a record *back*, so
  rules see `previous` as `null`. Value and cross-field rules still apply; an immutable-once-set rule
  does not block a `seed reset`.
- A record type backed by **external sources** is validated by the upstream system, so `x-validate`
  (like the rest of the schema) does not run against writes that go there.
- Relationship types accept the same declarations in their `data_schema`, with the same contract.

## Relationship types

A relationship type defines a `rel_type` — its **id is the type**, and there is no separate name.
It must exist before any relationship of that type. Config: preview, then promote.

```bash
wayai relationship-types upsert <rel_type> --base <base> \
  [--description <text>] [--schema <json>] [--source-record-types <json>] [--target-record-types <json>]
wayai relationship-types list --base <base>
wayai relationship-types get <rel_type> --base <base>
wayai relationship-types delete <rel_type> --base <base> [-y]
wayai relationship-types history <id> --base <base> [--limit <n>] [--offset <n>] [--diff]
```

`--schema` (optional) validates relationship metadata; omit it to allow any metadata.
`--source-record-types`/`--target-record-types` (optional, JSON arrays) restrict which record types
the type may connect. Deleting a relationship type also removes every relationship of that type, and
for a `cascade` type the files those relationships owned that no other owning link still holds
([files.md](files.md)).

## Relationships

```bash
wayai relationships upsert --base <base> --type <rel_type> \
  [--source <record_type/id>       | --source-external <record_type/external_id> [--source-external-source <name>]] \
  [--target <record_type/id>       | --target-external <record_type/external_id> [--target-external-source <name>]] \
  [--id <id>] [--external-id <key> [--external-source <name>]] [--data <json>]
wayai relationships get <id> --base <base> [--rel-type <type> [--external-source <name>]]
wayai relationships delete <id> --base <base> [--rel-type <type> [--external-source <name>]] [-y]
wayai relationships history <id> --base <base> [--limit <n>] [--offset <n>] [--diff]
wayai query-relationships <record_type> <id> --base <base> \
  [--type <type>] [--direction outgoing|incoming|both] [--limit <n>] [--offset <n>]
```

Each endpoint takes **exactly one** of `--source`/`--source-external` (same for target). The two modes
differ on missing records: an internal id is stored as given (linking before the record exists is
allowed), while external addressing **requires the record to exist** and resolves active records only
— the write fails with a clear error otherwise. External addressing is the convenient one right after
an `external_id` upsert, when you never kept the returned ids.

A relationship can also carry its **own** external key: `--external-id` on upsert makes creation
idempotent (a retried upsert with the same key updates the first row instead of duplicating it), and
`get`/`delete` accept that key in place of the id when `--rel-type` scopes it.

`--data` is validated against the relationship type's schema. `query-relationships` traverses from one
record; it takes no filter — narrow with `--type` and `--direction`.

## Batch (atomic)

Up to 1,000 operations, all-or-nothing:

```bash
wayai bases batch --base <base> --operations '[
  {"type":"upsert_record","record_type":"invoices","external_id":"inv-1","data":{"customer":"A","amount":100}},
  {"type":"upsert_record","record_type":"customers","external_id":"cust-1","data":{"name":"A"}},
  {"type":"upsert_relationship","rel_type":"belongs_to","source_record_type":"invoices","source_external_id":"inv-1","target_record_type":"customers","target_external_id":"cust-1"}
]'
```

Operation types: `upsert_record`, `delete_record`, `upsert_relationship`, `delete_relationship`,
`upsert_record_type`, `delete_record_type`. A relationship op addresses each endpoint by internal id
(`source_id`/`target_id`) or external key (`source_external_id` + optional `source_external_source`) —
exactly one per endpoint. An `upsert_record` with `"merge": true` changes only the fields its `data`
sends when its `external_id` names a live record, instead of replacing the record (it creates one
as usual when there is none). A relationship op
that names an existing relationship by `id` acts only on one of its own type: an
`upsert_relationship` fails on an `id` held by another `rel_type` (change a relationship's type with
`wayai relationships upsert`), and a `delete_relationship` given a `rel_type` fails on a
relationship of any other type.

Ops run **in order inside one transaction**, so the example above creates two records and links them
in a single atomic call: the link resolves the external ids the earlier ops just wrote, and a
resolution miss rolls the whole batch back. Add `--json` for the full per-operation payload; the
default table output prints one summary line per op. A single failing operation rolls the **whole**
batch back and exits non-zero naming the offending index — no partial writes land.

## Bulk import

For a historical backfill, import is the purpose-built path: it writes like ordinary upserts but
**skips trigger fan-out** (no webhooks, no internal/external write cascades fire for backfilled rows),
which is why it needs the dedicated `write:imports` permission on top of `write:records`.

```bash
wayai bases import run <record_type> --file patients.ndjson --base <base>  # one JSON object per line
wayai bases import list --base <base> [--limit <n>] [--offset <n>]         # sessions, newest first
wayai bases import rollback <import_id> --base <base> [--yes]              # undo
```

The CLI streams the file (size is not bounded by memory) and chunks it. It retries a *transient*
failure in place; a run that dies is resumed by hand.

- **Resume.** `run` prints the `request_key` it generated — keep it. If the run dies partway, it
  prints the exact flags to resume with, and you re-run using them: `--request-key <key>
  --start-chunk <n>` and **the same `--chunk-size`**. Chunk numbers are derived from the chunk size,
  so resuming under a different one points index N at different rows; the CLI requires the flag
  explicitly whenever `--start-chunk` is used. Re-run the *same* command with every other flag
  unchanged — a too-high `--start-chunk` skips rows silently, and dropping `--base` resumes into a
  different base entirely. `--chunk-size` is ≤ 1,000; `--no-finalize` leaves the session open for a
  later resume.
- **Idempotent by `external_id`.** A row whose content already matches is skipped without a write, so
  re-running a full refresh only rewrites actual changes. One exception: a matching row not written to
  in ~45 days is rewritten anyway (version bump) to keep periodically re-imported records counted as
  active rather than drifting into archival — so refreshes sparser than that rewrite every row.
- **Every validation and archival gate applies** exactly as on normal writes: import into an
  `immutable` record type is create-only. Native record types only — a source-backed type cannot be
  bulk-written.
- **Sessions expire, and the month boundary is handled for you.** A session idle for 24h has its slot
  reclaimed. A session also cannot outlive the UTC month it was granted in, because its allowance
  snapshot *is* that month's grant — but the CLI recovers that one in place: it begins a fresh
  session, re-sends the in-flight chunk, and carries on. Already-written rows hit the unchanged-row
  skip, so the boundary costs one extra `begin`, not a re-import. **What it does change is undo:** the
  logical run now spans **two `import_id`s**, and the CLI says so on the spot
  (`this run now spans two import_ids; undo needs both`). Roll back each one — a single
  `rollback <import_id>` leaves the other half of the import live.

**Undo.** `wayai bases import rollback <import_id>` soft-deletes only the rows the import **created** —
a row it merely overwrote stays, keeping the imported values (there is no version-revert). Links
pointing at deleted rows go too, so nothing dangles, and so does a file such a link owned (a
[`cascade`](files.md) link), even one attached after the import, unless another owning link still holds it. It works in pages and the CLI loops until
complete; retrying is always safe, since pages already swept stay swept. **In a terminal** it shows
how many live records it will delete (those [kept](#reviewed-records-x-review) left out) and asks to
confirm, or asks nothing when that is none. **Off a TTY — a script, CI, or an agent — a missing
`--yes` is REFUSED, never assumed**: the prompt would have no one to answer it, so the command errors
out rather than sweeping unattended. Pass `--yes` deliberately when you mean it. Like the import
itself it **fires no triggers**, so a trigger that syncs deletions elsewhere will not see them. Rows
already moved to archival storage are out of reach, so the practical undo window is roughly the
archival cadence. **Re-importing with corrected data is usually the better fix** — rows upsert by
`external_id`, so a corrected re-run overwrites in place.

Import rows are metered against a plan entitlement (`import_rows_included`), separate from the record
cap; rows past the allowance are metered per request, so send rows in reasonable batches — smaller
chunks never cost less. Only rows actually **written** count: skipped rows and replayed chunks are
free, with the ~45-day rewrite above as the one exception. Beginning and finalizing a session are
free; each rollback page is metered.

## Output and input conventions

- `--json` (or `--output json`) on any Data command for machine-readable output; the default is a
  table.
- Any `--data`, `--schema`, `--sources`, `--operations` or `--filter` flag accepts inline JSON
  (`'{"key":"value"}'`) or a file reference (`@path/to/file.json`).
- **Confirmation is not one convention — check the command before scripting it.** Three shapes:
  - `bases credentials delete`, `bases tokens prune` and `bases import rollback` **refuse** without a
    confirmation flag when stdin/stdout are not TTYs, exiting non-zero. Pass it from a script, CI, or
    an agent — but note the spelling differs: the first two take `-y`/`--yes`, while
    **`bases import rollback` accepts only the long `--yes`** (`-y` is rejected as an unknown
    option, so the rollback never runs).
  - `bases tokens revoke` takes **`--force`, not `--yes`**, and revoking is otherwise unguarded: an
    ordinary token is revoked permanently by the first call. Only a token the server reports as live
    (recently used, or toolset-bound) is held back, and off a TTY that case prints the reason and
    exits 1 rather than prompting. See [toolsets.md](toolsets.md).
  - The ontology deletes (`records delete`, `record-types delete`, `relationship-types delete`,
    `relationships delete`) prompt unconditionally, so off a TTY they **hang and then exit 0 having
    deleted nothing** — a green run that did not do the thing.

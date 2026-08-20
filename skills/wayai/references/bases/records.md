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

Deleting a record type also removes all of its records and any relationships pointing at them; a
record type recreated with the same id starts empty.

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

`history` returns every version as a snapshot with the actor, operation and timestamp. **Admin-only**
— available to platform admins or a token granted `read:audit`; ordinary agent tokens are rejected.
The same subcommand exists on `relationships`, `record-types` and `relationship-types`.

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
the type may connect. Deleting a relationship type also removes every relationship of that type.

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
exactly one per endpoint.

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
pointing at deleted rows go too, so nothing dangles. It works in pages and the CLI loops until
complete; retrying is always safe, since pages already swept stay swept. **In a terminal** it shows
the live matched count and asks to confirm. **Off a TTY — a script, CI, or an agent — a missing
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

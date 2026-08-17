# Querying & search

Deep how-to for reading a base: the `wayai bases sql` surface and its mandatory scoping, a worked SQL example gallery, `search`-field tuning (`x-search`), and datetime/timezone configuration. Open it when a read needs more than a single filter, when approximate matching returns the wrong things, or when datetimes come back in an unexpected zone. The entity model, the Filter DSL operators, and the day-to-day command shapes live in [`../../SKILL.md`](../../SKILL.md) and [`records.md`](records.md).

## Table of Contents

- [The SQL surface](#the-sql-surface)
- [Scoping predicates](#scoping-predicates)
- [SQL query examples](#sql-query-examples)
- [Search tuning](#search-tuning)
- [Datetime configuration](#datetime-configuration)

## The SQL surface

```bash
wayai bases sql "<query>" --base <base> [--param key=value ...] [--file query.sql]
```

Read-only SELECT over one base's own data — filtering, aggregation, JOINs, CTEs, array operations. Queries always see the latest version of each row; the server handles deduplication.

- The statement comes from the positional argument **or** `--file <path>` — an empty file and a missing statement are both refused, so a query is never silently run as blank.
- `--param key=value` binds a named placeholder (`{status:String}`). It is repeatable and also accepts several pairs after one flag. The split is on the **first** `=` only, so a value may itself contain `=`.
- The base comes from `--base` or `WAYAI_BASE`. `--json` is declared on the `bases` namespace and is accepted on either side of the subcommand.

Distinct from `wayai analytics sql`, which reads conversation analytics (`conversation`, `message`) — see [`../analytics.md`](../analytics.md). Different store, different tables, different questions; nothing you learn about one transfers to the other.

**Tables**: `records`, `relationships`, `record_types`, `relationship_types`, `bases`, `operations_log`, `organization`, `audit_log`.

**Accessing JSON fields**: use `data.field.:Type` subcolumn syntax for the native JSON type, or `CAST(data.field, 'Type')` when you need a type-safe conversion (e.g. integers stored as `Int64` vs `Float64`).

The `organization` table exposes org-level metadata. Its `plan` and `status` columns are **not maintained and carry no billing meaning** — entitlement belongs to the platform that manages the account, not here. Depending on when the org was created they hold either an empty string or a stale default; never read them as an entitlement. Being org-level, `organization` also has no per-base row, so this base-scoped surface cannot answer it — and the CLI ships no org-wide SQL command.

The `audit_log` table is the change history behind `wayai records history` (and the `history` subcommand on record types, relationships and relationship types) — every version of every entity, with the actor. Any query referencing it, JOINs included, is admin-gated the same way: platform admins, or a token granted `read:audit` (see [`README.md`](README.md)). It and `operations_log` are **append-only**: neither has a `deleted` column, and both reject a `deleted = false` predicate.

**Eval clones**: a preview cloned from another preview runs on the `local` analytics tier, where SQL over `records`, `relationships` and `audit_log` is refused outright rather than answering empty. Every other table still answers. The tier is fixed at creation and there is no CLI flag for it — the only way to choose `standard` on such a clone is the config-as-code auto-create branch, which is also your one chance to set it (see [`config-as-code.md`](config-as-code.md)). Plan eval assertions around exact `data.*` filters instead, and give any `created_at`/`updated_at` bound an explicit offset.

## Scoping predicates

Every query must carry BOTH `org_id = {org_id:String}` AND `base_id = {base_id:String}`, plus `deleted = false`. Both scoping predicates are mandatory — a base id is unique per-org, not globally, so `org_id` is what isolates your data — and both placeholders are bound server-side from your authenticated org and base, never from `--param`.

Three tables have no `base_id` column, so bind `{base_id:String}` to the column that actually identifies the base:

| Table | Base-scoping predicate |
|---|---|
| `bases` | `id = {base_id:String}` |
| `operations_log` | `data.base_id.:String = {base_id:String}` |
| `organization` | none — it is org-level, with no per-base row |

Keep `org_id = {org_id:String}` in all three cases. Drop `deleted = false` only on the append-only `audit_log` and `operations_log`, which reject it.

The `records` table also carries `archived` and `cancelled` boolean columns — add `archived = false` to see active records only.

Note the contrast with Filter-DSL list queries (`wayai records query`, and the query tools a toolset generates): those return **active records by default**, and an explicit filter condition on `archived` overrides that default — your predicate decides which tier you see. Raw SQL applies no such default; you state the tiers yourself. It is also the only way to read archived rows through relationship traversal, since `wayai query-relationships` takes no filter at all (only `--type`, `--direction`, `--limit`, `--offset`).

## SQL query examples

```bash
# Simple query
wayai bases sql "SELECT data.invoice_number.:String as num, data.status.:String as status FROM records WHERE org_id = {org_id:String} AND base_id = {base_id:String} AND record_type = 'invoices' AND deleted = false" --base my-base

# Aggregation
wayai bases sql "SELECT data.status.:String as status, count() as cnt FROM records WHERE org_id = {org_id:String} AND base_id = {base_id:String} AND record_type = 'invoices' AND deleted = false GROUP BY status" --base my-base

# Array aggregation (sum embedded line items)
wayai bases sql "SELECT data.invoice_number.:String as num, arraySum(CAST(data.line_items[].amount, 'Array(Float64)')) as total FROM records WHERE org_id = {org_id:String} AND base_id = {base_id:String} AND record_type = 'invoices' AND deleted = false" --base my-base

# Explode array elements with ARRAY JOIN
wayai bases sql "SELECT item.description.:String as item, sum(CAST(item.amount, 'Float64')) as revenue FROM records ARRAY JOIN data.line_items[] as item WHERE org_id = {org_id:String} AND base_id = {base_id:String} AND record_type = 'invoices' AND deleted = false GROUP BY item ORDER BY revenue DESC" --base my-base

# CTE joining records with relationships
wayai bases sql "WITH inv AS (SELECT id, data.invoice_number.:String as num, arraySum(CAST(data.line_items[].amount, 'Array(Float64)')) as total FROM records WHERE org_id = {org_id:String} AND base_id = {base_id:String} AND record_type = 'invoices' AND deleted = false), pay AS (SELECT target_id, sum(CAST(data.allocated, 'Float64')) as paid FROM relationships WHERE org_id = {org_id:String} AND base_id = {base_id:String} AND rel_type = 'payment_for' AND deleted = false GROUP BY target_id) SELECT inv.num, inv.total, coalesce(pay.paid, 0) as paid, inv.total - coalesce(pay.paid, 0) as balance FROM inv LEFT JOIN pay ON pay.target_id = inv.id ORDER BY balance DESC" --base my-base

# With parameters
wayai bases sql "SELECT * FROM records WHERE org_id = {org_id:String} AND base_id = {base_id:String} AND data.status.:String = {status:String} AND deleted = false" --base my-base --param status=issued

# Fuzzy / approximate text match, ranked by closeness (power-user form of `records query --filter {op:search}`).
# Fold case + accents on BOTH sides so "São" matches "sao"; jaroWinklerSimilarity suits short names.
wayai bases sql "SELECT *, jaroWinklerSimilarity(lowerUTF8(data.car_model.:String), lowerUTF8({q:String})) AS score FROM records WHERE org_id = {org_id:String} AND base_id = {base_id:String} AND record_type = 'vehicles' AND deleted = false AND score >= 0.85 ORDER BY score DESC LIMIT 10" --base my-base --param q='honda civic'
```

Long statements belong in a file: `wayai bases sql --file balances.sql --base my-base --param status=issued`.

## Search tuning

The Filter DSL `search` operator matches a field by **approximate** value — use it when you have a near-correct string but not the exact stored one (e.g. a model name, a place, a plan name, a SKU with a possible typo). Results come back **ranked by closeness**, each carrying a `_search_score` (0–1, higher = closer):

```jsonc
// "find vehicles whose model is close to 'honda civic', only active ones"
{ "and": [
  { "field": "data.status",    "op": "eq",     "value": "active" },
  { "field": "data.car_model", "op": "search", "value": "honda civic" }
] }
```

Pass it with `wayai records query <record_type> --base <base> --filter '<json|@file>'`. Every field is searchable by default — `search` needs no setup. Exact filters and `search` compose in one query: put exact conditions alongside the `search` leg so the exact ones narrow the set and `search` ranks within it. To **tune** how a field is matched, set an optional `x-search` hint on that field in the record type schema:

```jsonc
"car_model": { "type": "string", "x-search": { "mode": "name" } }
```

- `mode: "name"` — short proper nouns (people, brands, plan/model names); favors close, prefix-aligned matches.
- `mode: "text"` — free-text / multi-word fields; matches on shared word fragments.
- `mode: "fuzzy"` — codes / SKUs / IDs where typos dominate.
- `threshold` (0–1) — minimum closeness to return; `searchable: false` — forbid searching a field (e.g. a large free-text blob you never want scanned).

Tuning is optional — unset fields use a sensible default. `x-search` hints apply to **top-level fields**; `search` still works on nested paths (e.g. `data.address.city`) using the default behavior, they just aren't individually tuned. Search runs against the latest synced data, so a record written a moment earlier may take a brief moment to appear.

**Pick `threshold` deliberately.** Everything above the cutoff is returned, ranked — so a value that doesn't exist can still come back as a confident near-match of something else. `mode: "name"` compresses scores high when many values share a prefix token (hundreds of plans all starting `AMIL`), which is exactly when a default cutoff under-discriminates. If "no match" needs to be a trustworthy signal in your workload, raise `threshold` on that field.

**Exposing search to an agent:** these hints only affect a toolset's generated tool if the `list` Action declares `match: search` on that field in `filterable_fields` — the only match that compiles to the `search` operator ([`toolsets.md`](toolsets.md)). A `searchable: false` field is rejected there at config-write.

Omit `--sort` when using `search` to keep the relevance ranking.

## Datetime configuration

Declare datetime fields in the record type schema with `"format": "date-time"` (or `"format": "date"` for date-only). A base stores them in a single canonical form: a naive value (no offset) gets the record type's timezone attached, so `2026-06-11T13:00:00` is stored and returned as `2026-06-11T13:00:00-03:00` — the same shape from a list query and a single-record read, with no UTC math required of you.

Set the timezone with `"x-timezone": "America/Sao_Paulo"` on the record type schema, or set a base-wide default that every record type without its own `x-timezone` inherits. Resolution order: record type `x-timezone` → base `settings.timezone` → UTC.

```bash
wayai bases create <base-id> --name "My Base" --timezone America/Sao_Paulo
wayai bases update <base-id> --timezone America/Sao_Paulo
```

The value must be a valid IANA timezone (invalid names are rejected server-side). `--timezone` **alone** on `update` is a read-modify-write — it merges into the existing `settings` and preserves keys you never mentioned. `--settings <json|@file>` **replaces** the whole `settings` object, so send it complete; passing both applies `--timezone` on top of the object you supplied. Omitted top-level fields (`name`, `description`, `tags`) are left unchanged, and `update` refuses an invocation that carries no flags at all. Base settings tracked in the workspace folder are covered in [`config-as-code.md`](config-as-code.md).

**Filtering is instant-aware:** `eq`/`neq`/`in`/`not_in` and the range operators match datetimes by point-in-time, so the exact representation you pass (`T` vs space, with or without offset) doesn't change the result — `{ "field": "data.starts_at", "op": "eq", "value": "2026-06-11T13:00:00" }` matches the stored value regardless of how it was written. Use `eq` for an exact datetime, not `like` (which is plain substring matching). Raw `wayai bases sql` is **not** rewritten this way — match the canonical stored form, or prefer `wayai records query --filter` for datetime conditions.

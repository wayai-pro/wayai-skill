# Integrations

How a base reaches an external system: **external sources** (back a record_type with an upstream API), **inbound webhooks** (data in), **triggers** (data out), and **provider adapters**. Open this when you are backing a record_type with something outside the base, or wiring change notifications in either direction — and read [Canonical-first modeling](#canonical-first-modeling) *before* picking an edge.

An **executor** is a small stateless HTTP service you deploy for work a base cannot do itself (custom logic, mTLS, SOAP, heavy processing); a base signs every request it dispatches to one. Everything about executors — the contract, the SDK, where to run them — is in [executors.md](executors.md). Most integrations need none.

## Contents

- [Canonical-first modeling](#canonical-first-modeling)
  - [Pattern catalog](#pattern-catalog)
  - [Identity keys](#identity-keys)
- [External sources](#external-sources)
  - [What a source declares](#what-a-source-declares)
  - [`field_mapping` rules](#field_mapping-rules)
    - [Split one field into several upstream params](#split-one-field-into-several-upstream-params)
    - [Destructure a composite field](#destructure-a-composite-field)
    - [Compose a canonical field from several upstream fields](#compose-a-canonical-field-from-several-upstream-fields)
    - [Per-operation override](#per-operation-override)
  - [Per-operation endpoints](#per-operation-endpoints)
  - [Named write bindings](#named-write-bindings)
  - [`id_path`](#id_path)
  - [`forward_filters`](#forward_filters)
  - [`local_filters`](#local_filters)
  - [Request body shaping](#request-body-shaping)
  - [`current` and `prior` tokens](#current-and-prior-tokens)
  - [Other source options](#other-source-options)
  - [URL and SSRF rules](#url-and-ssrf-rules)
  - [Worked example: a legacy upstream](#worked-example-a-legacy-upstream)
- [Inbound webhooks](#inbound-webhooks)
- [Triggers](#triggers)
- [Provider adapters](#provider-adapters)

---

## Canonical-first modeling

**The one rule: model the entity canonically first — agnostic to any integration — then decide, per operation, how it is backed.** The record_type IS the entity: a clean schema with business-named properties, sensible enums and defaults — never the upstream payload's shape. Integration lives at declarative edges, never smeared into the schema or the tool surface. A consumer (an agent, a query) should not be able to tell whether a field is base-native or mirrored from a legacy system.

- **Model the canonical first.** Derive the schema and tool surface from the domain and the agents' real journeys, not from any backend's payload. Put each business rule at the most reliable layer it fits — schema (enums, `required`, defaults) > trigger (reactive invariant) > tool constraint (`writable_fields`/`precondition`) > prompt — and never encode an enforceable data invariant as a prompt rule. Right-grain it: model only what the journeys touch and extend additively.
- **Swap-test.** If re-backing an entity (legacy → native → a different system) would force a change to its schema or to the *semantic* tool surface — tool names, `data` fields, filter params — the model leaked the integration; fix the model. Only an integration-agnostic model is swappable: the same canonical contract can be served natively in one base and proxied to a legacy API in another by swapping only the source's `field_mapping`. What re-backing *does* change is the envelope addressing params (`external_id`/`external_source`), which the source takes over — machinery moving server-side, not a leak. But a prompt or harness that passes `external_id` on create is coupled to the backing choice, so keep it out of the agent's instructions. Backfill catalog gaps the legacy system lacks with native record_types in the same base, exposed through the same toolset.
- **Offline evals are a free payoff — the swap-test from the other side.** The preview-only integrations toggle (`wayai bases create-preview <origin-id> --name evals --integrations disabled`, or `wayai bases update <id> --integrations disabled`) works *precisely because* the model is integration-agnostic: where the swap-test *re-backs* an entity without touching its surface, the toggle *un-backs* it — every external integration edge goes inert and the same canonical schema, tools, and field mappings keep serving from seeded local data. Model canonical-first and deterministic, production-safe agent evals come free. See [README.md](README.md) for base/preview lifecycle.
- **Native vs proxy is a per-record_type choice.** A record_type with no sources is native (all CRUD local); a record_type with a source is proxy-backed — every write goes upstream and reads resolve against the upstream (it stores nothing locally). The source's `list`/`get`/`create`/`update`/`delete` blocks pick *which* operations the upstream supports; an operation the source omits is **unsupported**, not silently native. You don't split native and proxy across the operations of one record_type — you mix them across record_types: **one toolset freely composes native-backed and proxy-backed tools** (a proxied `invoices` beside native `plans`/`activities` you backfilled).
- **Don't route native-vs-proxy per tool or per field.** `writable_fields` restricts *which* fields a tool writes, never *where*. (Two proxy tools on the same op MAY select different [named write bindings](#named-write-bindings), but that only picks which upstream *endpoint* a write hits; it stays all-proxy.) A "native read + proxy write on one record_type" mix is a read-consistency hole — the native read won't reflect the proxy write until something syncs it — and once you add that sync, native-write + an `external_write` trigger is strictly better. The two coherent shapes are **all-proxy** or **all-native + sync**; the per-tool mix is the incoherent middle.
- **Prefer the native mirror; reach for proxy only when forced.** A native record_type kept in sync gives local query/join/filter, speed, transactional writes, audit, and availability decoupled from the upstream. Choose proxy-on-read only when staleness is unacceptable, a write must be synchronously confirmed by the upstream system of record, or the dataset is too large or cold to mirror.
- **Hybrid entities (mirrored fields + a locally-owned overlay) live in ONE record_type.** A field the upstream lacks (a workflow `status`, a tag, a score) belongs on the same record as the mirrored fields — integrated query/filter demands it (you can't list `where status=X` if `status` lives in another record_type). Splitting an entity to separate ownership trades a real query capability for bookkeeping; don't. Exception: an overlay that is itself a richer entity with its own lifecycle and history → a related record_type + relationship. A scalar attribute is never a reason to split.
- **`field_mapping` is one bidirectional contract, reused at every edge.** The same source contract powers proxy reads, write-through, inbound-webhook hydration (`source_binding`), and `external_write` out. Define the translation once; the edges reuse it.
- **Reference webhooks: hydrate via the source's `get`, never follow the payload's callback.** Most mature webhooks are reference-style ("record X changed, go fetch it") — small payload, no stale snapshot, self-correcting ordering. Resolve them by extracting the id and fetching through the bound source's `get`. A payload-supplied callback URL is an SSRF vector and is ignored.

### Pattern catalog

Pick the edge by what the upstream offers and what you need:

| Pattern | Source / edge shape | Reads | Writes | Use when |
|---|---|---|---|---|
| **Native** | `sources: []` | local | local | the base owns the data |
| **Proxy-read** | source `list`/`get` | live upstream | — | freshness-critical or too-big-to-mirror reference data |
| **Write-through** | source `create`/`update` (+ `get`/`list` to read) | live upstream | → upstream (id returned) | upstream is system of record; you want synchronous confirmation, no local mirror |
| **Push-synced mirror** | hydrating inbound webhook + bound source `get` | local | local | upstream pushes change notifications; you want local query of a fresh mirror |
| **Bidirectional mirror** | hydration in (`merge`) + `external_write` trigger out (`watched_fields`) | local | local (→ upstream via trigger) | a hybrid entity synced both ways, all-native tools |

The **bidirectional mirror** is the idiomatic home for a hybrid entity: one native record_type, all-native tools, sync as declarative edges. Use hydration `"merge": true` so a re-sync refreshes only the mapping-owned fields and preserves your overlay, and `external_write` `"watched_fields": "mapped"` so an overlay-only change doesn't push a no-op upstream. The hydration write target must be **native** (no source) — the out-edge is the `external_write` trigger, not a proxy write on the same record_type — and the skip-on-inbound default keeps a re-sync from re-firing the trigger (no loop).

### Identity keys

Catalog / reference entities (plans, activities, locations) → key by a stable **slug** that doubles as `external_id`, so cross-record_type `x-fk` references resolve. People / relations (members, bookings, enrollments) → key by the base's internal **uuid**. See [records.md](records.md) for external addressing.

---

## External sources

A record_type can declare one or more **external sources**. When a record operation carries an `external_source` matching a configured source name, the base proxies that read/write to the external API (or an executor) instead of its own store — so an agent reads and writes `stripe`-backed invoices through the same `wayai records` / MCP calls, with no separate integration code.

Sources are part of the record_type config — **preview only**, promoted like any schema change:

```bash
wayai record-types upsert invoices --base crm--preview --name "Invoices" \
  --schema @schema.json --sources @sources.json
```

They also round-trip through config-as-code (`sources:` on the record_type file under `wayai-ws/bases/<base>/record-types/`, `wayai pull bases/<base>` / `wayai push bases/<base>` — see [config-as-code.md](config-as-code.md)).

### What a source declares

- `name` — must equal the record's `external_source`.
- `auth` — header + value template; the secret inline or a `credential:<name>` reference. An **inline** secret is environment-local and promotion never copies it, so a newly promoted source carrying one stays unconfigured until you set it on production — requests fail with a clear "no usable credential configured" error until you do. A `credential:<name>` reference resolves against the credentials of *the base serving the request*; promotion publishes the preview credentials you opted in and reports every referenced credential production will still lack. A preview is created with a copy of its origin's credentials, so cloned integrations keep working. (`vault:<name>` is the deprecated spelling of the same reference and resolves identically.)
- `field_mapping` — optional; omit for identity passthrough. See [below](#field_mapping-rules).
- `get` / `list` / `create` / `update` / `delete` — [per-operation endpoints](#per-operation-endpoints), optionally as [named write bindings](#named-write-bindings).
- [`id_path`](#id_path) — where the `external_id` lives in each raw upstream record.
- [`forward_filters`](#forward_filters) / [`local_filters`](#local_filters) — the two mutually exclusive filtering stances for `list`.
- `static_body`, `body_template`, `request_encoding` — [request body shaping](#request-body-shaping).
- `injections`, `executor_secrets`, `cache_ttl` + `stale_if_error`, `signing`, `timeout_ms`, `breaker` — see [other source options](#other-source-options).

### `field_mapping` rules

**Optional** — omit for identity passthrough (the upstream object is stored as the record data verbatim, and writes send your data unchanged). When set, it maps between your canonical schema and the upstream's field names in both directions:

- **`to_external`** — the write direction, canonical → upstream.
- **`to_record`** — the read direction, upstream → canonical.

A plain rename in `to_external` **auto-inverts on read** (upstream `state` → canonical `status`), which covers most mappings, so `to_record` is optional and usually omitted. A *rich* rule does not auto-invert — it is skipped by the auto-inversion — so a mapping that needs one on read has to spell the read direction out.

> **`to_record` REPLACES the read direction; it does not add to it.** The moment the key is present the auto-inversion of `to_external` is not used **at all**, so `to_record` must list **every** field you want mapped on read — including the plain renames that would otherwise have inverted for free. A `to_record` holding only the one rich field silently drops every other field from every read: proxied `get`/`list` return records missing them, and an inbound webhook with `merge` off full-replaces the stored record with the reduced shape. Prefer keeping mappings plain (auto-inversion), or [`computed`](#compose-a-canonical-field-from-several-upstream-fields) for derived values, precisely to avoid owning this list.

```json
{
  "to_external": { "status": "state", "due": { "path": "dueDate", "date_format": "dd/MM/yyyy" } },
  "to_record": {
    "state": "status",
    "dueDate": { "path": "due", "date_format": "dd/MM/yyyy" }
  }
}
```

Note `"state": "status"` is restated even though it would have auto-inverted on its own — omit it and the canonical `status` stops being populated. A `to_record` key is the upstream field name and its value (or rich-rule `path`) names the canonical field, the mirror of `to_external`. Because it is read-only it rejects the write-side-only forms (`targets`, `parts`, `target`) at config time rather than ignoring them.

A value is either a simple rename (`"status": "invoice_status"`) or a rich rule `{ path, values, default, transform, date_format, array_mode, target }`:

- `transform` — `lowercase` / `uppercase` / `trim` / `to_string` / `to_number` / `to_boolean` / `iso_date`.
- `values` — an enum map, **always canonical-keyed** (`{ "<your value>": "<upstream value>" }`, e.g. `{ "active": "ACTIVE" }`). The *same* map serves both directions: forward (write) looks up by key, reverse (read) matches by value and returns the key. **The orientation does not flip in a read-direction rule** even though everything else in such a rule reads upstream → canonical: a `values` map stays canonical-keyed and is reverse-matched. As a convenience the reverse path also accepts the intuitive upstream-keyed form when the canonical-keyed match misses — but write the canonical-keyed form to stay unambiguous (reverse-match wins on any key/value overlap). A fully backwards or mistyped map otherwise passes the raw upstream value through unchanged rather than erroring, so confirm by reading a record back after configuring.
- `default` — value used when the field is missing.
- `array_mode` — `first` / `last` / `flatten` for array-valued paths.
- `date_format` — bidirectional date/time reshaping between the external wire format and the base's canonical form. Tokens `yyyy MM dd HH mm ss` plus literal separators (`/ - : . space T`), e.g. `"date_format": "dd/MM/yyyy"`. The canonical side is **ISO-8601 by default** (`yyyy-MM-dd` for a date-only pattern, `yyyy-MM-ddTHH:mm:ss` when a time is present); set **`record_format`** on the same rule to truncate or reshape the canonical side instead — same token vocabulary, e.g. `{ "date_format": "HH:mm:ss", "record_format": "HH:mm" }` stores `09:30` from an upstream `09:30:00`. `record_format` is ignored unless `date_format` is set. A value that doesn't match `date_format` passes through unchanged. Applied after `transform`.
- `target` (`to_external`, write side; default `body`) — route a mapped param to the request **query string** (`"target": "query"`) instead of the body on `create`/`update`. Value maps, transforms and defaults still apply — the write-direction counterpart of `forward_filters.target`. Use it for an upstream whose create/update params are query params, e.g. `"activity_id": { "path": "idActivity", "values": { "lash-lift": "42" }, "target": "query" }` sends `?idActivity=42`. A query-target name colliding with a param already in the endpoint's `url` is rejected at config time; each split `targets` entry can pick its own `target`.

#### Split one field into several upstream params

(`to_external` only) Give the rule `targets` — an array of rules — instead of a single `path`, to fan ONE canonical field out to multiple upstream params, each with its own `path`/`transform`/`date_format`/`target`. The headline use is an upstream wanting separate `date` + `time` params fed from a single ISO datetime:

```json
"starts_at": { "targets": [
  { "path": "data", "date_format": "dd/MM/yyyy" },
  { "path": "hora", "date_format": "HH:mm" }
] }
```

A split field can't be a `forward_filters` field (no single upstream name).

#### Destructure a composite field

(`to_external` only) Give the rule a `separator` plus a `parts` array to split ONE composite canonical value and route each part to its own upstream param, each independently value-mapped, `date_format`'d and `target`'d. This is the write-side inverse of a composite [`id_path`](#id_path): a write tool references the entity by its composite id — `session_id = "<activity>::<date>"` — but the upstream action endpoint needs the decomposed parts as separate, independently-translated params:

```json
"session_id": { "separator": "::", "parts": [
  { "path": "idActivity",   "values": { "yoga": "A1" }, "target": "query" },
  { "path": "activityDate", "date_format": "dd/MM/yyyy", "target": "query" }
] }
```

Parts are **positional by default** (segment `i` → `parts[i]`, so declare them in the order the read-side composite id concatenates); a part with no matching segment emits its `default` or is omitted, and extra trailing segments are dropped. **Selective extraction:** give each part an explicit `segment` index to consume a chosen — possibly non-contiguous — segment, declaring only the parts you need. So ONE canonical id carrying a superset of identifiers (`activityCode::configId::date::time`) can serve N write actions that each need a different subset: a trial-booking endpoint keyed by `(activityCode, date)` uses segments 0+2, a makeup-booking endpoint keyed by `(configId, date)` uses segments 1+2 — one record_type, no ignored params, no per-action duplication. Per rule it's all-or-nothing: declare `segment` on **every** part or none; declared indices must be unique. Unlike a split (same value to every target), a destructure routes a *different* part to each. This keeps the agent-facing tool integration-agnostic: it references the entity by the same canonical composite id whether the backend is native or a proxied legacy endpoint. A destructure field can't be a `forward_filters` field.

#### Compose a canonical field from several upstream fields

`computed` (read side) builds a canonical `data.*` field from a `{{token}}` template over the **raw upstream** record — the inverse of a split, and the same template grammar as a composite `id_path`. Use it to rebuild one field from several upstream fields, or to derive a key from siblings:

```json
"computed": {
  "starts_at": { "template": "{{data}}T{{hora}}", "sources": {
    "data": { "date_format": "dd/MM/yyyy" },
    "hora": { "date_format": "HH:mm" }
  } },
  "slot_id": { "template": "{{professional_id}}::{{data}}::{{hora}}" }
}
```

`sources` reshapes each token with the read-direction rule vocabulary (`values` reverse-maps, `transform`, `date_format`, `default`) minus `path` and `array_mode` — the token name *is* the upstream dot-path. If any token is absent or null the whole field is omitted — never half-built (a `sources.default` fills an **absent** token only; an explicit upstream `null` still omits). Tokens reference the upstream's own field names; there is no chaining over other computed or renamed fields. A token may also read a **forwarded list filter** via the reserved `{{filter.<field>}}` namespace — see [`forward_filters`](#forward_filters) — to backfill a scope key or date the upstream omits from its list rows (e.g. `"starts_at": { "template": "{{filter.date}}T{{hora_atendimento}}" }` when a list-by-date upstream returns only the time). `{{filter.*}}` populates on `list` only, a composite `id_path` may use it too, and each referenced field must be in `list.forward_filters.fields` (validated at config-write).

#### Per-operation override

The source-level `field_mapping` applies to every op, but an RPC/legacy upstream may use *different* param names for the same field across verbs (create takes `entity_id`, update takes `id_entity`). Put a `field_mapping` on an individual `get`/`list`/`create`/`update` endpoint to override the source-level one for that op — **full replacement**, so set every field it needs; it falls back to the source-level mapping when unset. Not allowed on `delete` (no body — set its param names in the URL template).

### Per-operation endpoints

Each of `get` / `list` / `create` / `update` / `delete` has its own `url` and `method` (GET/POST/PUT/PATCH/DELETE), so **RPC-style verb paths and all-POST upstreams work directly** — `"list": { "url": ".../listar_x", "method": "POST" }`, `"create": { "url": ".../incluir_x", "method": "POST" }`.

URL tokens: `{{external_id}}`, `{{query.*}}`, `{{data.*}}`, `{{current.*}}`/`{{prior.*}}` (update/delete — see [below](#current-and-prior-tokens)), `{{auth.org_id}}`, `{{auth.base_id}}`.

Envelope unwrapping:

- `response_path` / `total_path` — extract records and count from an envelope (`"response_path": "dados"` pulls the array out of `{ …, "dados": [...] }`).
- `success_path` / `message_path` — for upstreams that return HTTP 200 even on logical failure (`{ "success": false, "message": "…" }`). When `success_path` resolves falsy the call surfaces as an error carrying the `message_path` text instead of being mistaken for data. Dot-paths, like `response_path` — not JSONPath.

### Named write bindings

(`create` / `update` / `delete` only) When one canonical entity's writes fan out, in the backend, to **several** endpoints with different URLs, params or id semantics — a "book trial" and a "book makeup" that are both *creates* of the same booking — declare the write op as a **map of named variants** instead of a single endpoint:

```json
"create": {
  "trial":  { "url": ".../experimental-class", "method": "POST", "id_path": "idActivitySession" },
  "makeup": { "url": ".../enroll",             "method": "POST", "id_path": "{{data.member_id}}::{{data.session_id}}" }
}
```

Each binding is a full endpoint (its own `url`/`method`/`field_mapping`/`id_path`/`body_template`/`response_path`/`headers`) and **shares the source's transport** (`auth`/`injections`/`signing`/`breaker`/`cache_ttl`/`request_encoding`/`static_body`). The caller selects one **by name** per write — at the toolset layer via an Action's `binding` (`wayai actions upsert … --config`, see [toolsets.md](toolsets.md)) — so the canonical record_type stays ONE record_type instead of shattering into one-per-endpoint. A `default` binding is used when no name is selected (e.g. an `external_write` trigger's dispatch, which selects none — so a source used as an `external_write` target must give the targeted op a single endpoint or a `default` binding). Reads (`get`/`list`) don't fan out — they stay a single endpoint. The single-endpoint form is the default; selecting a binding name the op doesn't declare (or naming one on a single-endpoint op) is rejected at config-write and again at promotion.

### `id_path`

Dot-path to the `external_id` within each **raw** upstream record, for upstreams keyed on a domain field rather than `id`/`_id` (e.g. `"id_path": "codigo"`, `"identifiers.cpf"`). Applies to LIST (per row) and CREATE (from the response). When unset, the id is taken from the first present conventional key (`id`/`_id`/`Id`/`ID`/`uuid`/`key`). **Set this when your upstream has no conventional id field** — otherwise a LIST whose rows all lack a resolvable `external_id` fails loud with a `422 EXTERNAL_CONFIG` naming the fix and the row keys it saw (a partial drop returns the resolvable rows plus a `meta.dropped_no_id` count), and CREATE fails to identify the new record.

- **Composite (list-only).** For an upstream whose identity is composed of several fields with no single natural id, `id_path` may instead be a template joining fields — `"id_path": "{{entity}}::{{date}}::{{time}}"`. A composite id is **list-only**: it is rejected at config-write when `get`/`create`/`update`/`delete` is also configured, since it can't address a single row by id.
- **Create-only override.** For an RPC-style upstream that returns the new record's server-assigned id on CREATE under a *different* field than the one keying LIST rows, set a create-only `id_path` on the `create` endpoint (`"create": { …, "id_path": "created_ref" }`). It overrides the source-level `id_path` for the create read-back and falls back to it when unset.
- **Request-templated create id.** For an action/RPC write endpoint that returns **no addressable entity** (an empty body, or a bare `{ "result": "ok" }`), make the create-only `id_path` a request template over the create inputs — `"create": { …, "id_path": "{{data.member_id}}::{{data.enrollment_id}}::{{data.date}}" }` — and the base derives the `external_id` from the request data. Identical inputs map to the same id, so a repeated identical action upserts the same record rather than duplicating. Its tokens must be `{{data.<field>}}` naming declared record_type fields, and the source must be create-only (a list is fine): `get`/`update`/`delete` are rejected at config-write.

### `forward_filters`

(on the `list` endpoint) Opt in to forwarding the agent's filter conditions to the upstream so a proxied list can pass a search key, a date, a parent id. `fields` is the allowlist of canonical field names that may be forwarded (each translated to the upstream's name/value via `field_mapping`); `target` (`query` for a GET, `body` for a POST/PUT/PATCH list — inferred from the method when omitted) chooses where they go:

```json
"list": { "url": ".../search", "method": "GET", "forward_filters": { "fields": ["email"] } }
```

Without it (or [`local_filters`](#local_filters)), a proxied query **rejects** any `--filter` / MCP `filter`. It currently forwards exact-match (`eq`) conditions only — a single `eq` or a flat AND of `eq`s; any other operator or shape is rejected with an actionable error rather than silently returning unfiltered data. A forwarded value is also exposed to the read mapping as `{{filter.<field>}}` (in a `computed` field or a composite `id_path`), so a row can carry the filter value the upstream leaves out of its list rows.

### `local_filters`

(on the `list` endpoint) The other filtering stance: when the upstream has **no filter params at all** — an endpoint that just returns all ~64 practitioners of a clinic — no `forward_filters` can ever help. Declaring `local_filters` asserts the collection is **bounded and returned whole in one page**; the base then fetches it unfiltered and applies the agent's full Filter DSL **and sort** in memory. Every operator except `search` works (`has`, ranges, `in`, `like`/`ilike` with accent-folding), pagination is post-filter, and typed params behave exactly as against a native record_type. `{ "list": { "url": ".../professionals", "method": "GET", "local_filters": {} } }`; optional `max_rows` (default 500, ceiling 999) bounds the fetch.

Rules and sharp edges:

- **Mutually exclusive with `forward_filters`** — pick one stance per list endpoint. An upstream that *requires* an input param still gets it via a `{{query.*}}` token in the `url` (that hatch reaches REST/CLI list params, not toolset tools, which pass no free-form query params).
- **Fail-loud on truncation.** A fetch that returns — or reports via `total_path` — more than `max_rows` rows errors instead of silently filtering a partial collection. One edge is undetectable: an upstream that caps its page size below the fetch **and** declares no `total_path`. Only declare `local_filters` on endpoints that genuinely return the whole collection in one page.
- **Set `cache_ttl` alongside it.** With a read-through cache the whole collection is fetched once per TTL and every filter value is served from that one cached page; without it, every filtered list refetches the collection upstream.
- **`search` is rejected** at config-write, at promotion, and at request time: ranked search has no in-memory evaluator. Use `"match": "text"` (substring, accent-insensitive) on typed params instead. A deliberate, documented residual — `"match": "search"` works against a native store and is refused through a `local_filters` source.
- **Envelope timestamps are fetch-time.** A proxied row's `created_at`/`updated_at` are when the base fetched it (and `version` is always 1), so filtering or sorting on them is meaningless — target `data.*` fields. A filter-only request preserves the upstream's row order; ordering only changes when you pass an explicit `sort`. See [querying.md](querying.md) for the Filter DSL.

### Request body shaping

- **Static constants** need no special field: bake constant **query** params into the `url` (`".../listar?clinica=35"`), and put constant non-secret **headers** in `endpoint.headers` (`{ "X-Tenant": "35" }`). For a constant **body** param use `static_body`; for a secret use `injections`.
- `request_encoding` — `json` (default) or `form` to send `application/x-www-form-urlencoded` bodies for legacy form-post upstreams.
- `static_body` — constant, non-secret params merged into every body-bearing request (including POST-based reads), e.g. a fixed tenant id the agent never supplies. Agent data and `injections` win on a key collision.
- `body_template` (per `create`/`update`/`delete` endpoint) — a map of body field → templated string, interpolated against the same per-op tokens as that endpoint's `url` and merged into the request body. Use it to put the record id — or any addressable value — **into** the body for RPC/legacy upstreams that key the update/cancel verb by id in the body rather than the URL: `"update": { …, "body_template": { "agendamento_id": "{{external_id}}" } }`. Body-bearing ops only (create/update, and a `delete` whose method carries a body); rejected on get/list and on any GET/DELETE-method endpoint. Unlike `static_body` (constants, never templated) its values are filled per request; on a key collision agent data is overlaid by `body_template`, and `injections` win over both. Values are filled as **strings** — for a numeric or boolean field an upstream is strict about, send it through `field_mapping` or `static_body` instead.

### `current` and `prior` tokens

`{{current.*}}` / `{{prior.*}}` (in an `update`/`delete` `url`, `headers`, or `body_template`) resolve to the record's **current stored field values from the upstream**, for upstreams that need existing state on a write, not just the id plus new values: optimistic concurrency (`?ifmatch={{prior.version}}`), "move"-style updates taking the OLD value alongside the NEW (a reschedule sending the current `data_agendada` plus `nova_data`), or a cancel keyed by current field values.

```json
"update": { "…": "…", "body_template": {
  "agendamento_id": "{{external_id}}",
  "data_agendada":  "{{current.data_agendada}}",
  "nova_data":      "{{data.data_agendada}}"
} }
```

The base sources them with a pre-write read through the source's **`get`** endpoint, so a `get` is **required** when you reference one (rejected at config-write otherwise) — and the values are the **raw upstream record** (the upstream's own field names and wire format, echoed back verbatim). `prior` is an alias of `current` (a proxied write has no local merge). Update/delete only; referencing them costs one extra read per write, skipped entirely when unused.

### Other source options

- `injections` — extra per-request credentials placed into a header or body field, for upstreams needing their own credential.
- `executor_secrets` — named credentials an executor pulls at dispatch, for a binary or large per-tenant credential it can't take inline (e.g. an mTLS client certificate). Each `{ name, secret_ref }` declares a `credential:<name>` reference; each pull is short-lived, single-use, and resolves against the calling base's own credentials. Placeholders are not allowed in a `credential:` reference — a plain name already means "this base's credential". See [executors.md](executors.md).
- `cache_ttl` (seconds) + `stale_if_error` — read-through caching, optionally serving the last-known value on a transient upstream failure.
- `signing` — opt into HMAC request signing when the source points at an executor; omit for third-party APIs. See [Inbound webhooks](#inbound-webhooks) for the scheme (it is the same one in both directions).
- `timeout_ms` — per-request upstream timeout (default 10s, max 30s); a timeout surfaces as a transient error so `stale_if_error` can serve a cached value.
- `breaker` — per-source circuit breaker `{ failure_threshold, cooldown_ms }`. After `failure_threshold` consecutive transient failures (default 5) the source opens and short-circuits calls for `cooldown_ms` (default 30s) before a single half-open probe. Always on; this only tunes it.

### URL and SSRF rules

URLs must be absolute `https` — or `http` only when the source sets `"allow_insecure_http": true`, for a legacy/on-prem upstream without TLS — and tokens may appear only in the **path or query, never the host**. An SSRF guard rejects otherwise. A callback URL arriving inside a webhook payload is never followed (see [Inbound webhooks](#inbound-webhooks)).

### Worked example: a legacy upstream

A non-REST upstream — all-POST verb paths, a form body, a `{ success, dados }` envelope, a constant tenant id, `dd/MM/yyyy` dates — configured as a **direct** source, no executor:

```json
{
  "name": "legacy",
  "auth": { "header": "Authorization", "value_template": "Bearer {{secret}}", "secret_ref": "credential:legacy-key" },
  "request_encoding": "form",
  "static_body": { "clinica": "35" },
  "success_path": "success",
  "message_path": "message",
  "id_path": "codigo",
  "list":   { "url": "https://api.legacy.example/listar_agendamentos", "method": "POST", "response_path": "dados", "total_path": "total" },
  "create": { "url": "https://api.legacy.example/incluir_agendamento",  "method": "POST", "response_path": "dados" },
  "field_mapping": {
    "to_external": {
      "date": { "path": "data", "date_format": "dd/MM/yyyy" },
      "patient": "paciente"
    },
    "computed": {
      "date": { "template": "{{data}}", "sources": { "data": { "date_format": "dd/MM/yyyy" } } }
    }
  }
}
```

`patient` → `paciente` is a plain rename, so it auto-inverts on read. `date` is a rich rule and does not, so the read direction is supplied here by a `computed` field over the raw upstream `data` key — which, unlike a `to_record` block, adds to the auto-inversion instead of replacing it.

---

## Inbound webhooks

External systems push data into a base through inbound webhooks; each provides a unique ingest URL.

```bash
wayai inbound-webhooks create --base <base> --name <name> --secret-stdin \
  [--id <id>] [--record-type-scope <comma-separated>] \
  [--field-mapping <json>] [--source-binding <json>] \
  [--ingest-auth <json>] [--hydration <json>]
wayai inbound-webhooks list --base <base>
wayai inbound-webhooks get <id> --base <base>
wayai inbound-webhooks deliveries --base <base> [--status <pending|delivered|failed|dead>] [--inbound-webhook <id>]
wayai inbound-webhooks delete <id> --base <base> [-y]
```

The shared secret is **read from stdin or a masked prompt, never an argument** — `--secret-stdin` for CI, or omit it to be prompted. An argv element is readable from `/proc/<pid>/cmdline`, shell history and CI job logs, and whoever reads it can sign forged deliveries into the base.

**Authentication.** By default the sender signs each ingest request and the base verifies an HMAC over id + timestamp + method + path + body. The delivery also carries a **timestamp** the receiver checks for freshness and an **idempotency key** (falling back to the message id) used for dedupe: a duplicate replays the first response instead of writing again. **This is one scheme in both directions** — the same signature and timestamp that triggers and proxied executor calls use, so one signer works both ways. A receiver must verify it with the published SDK rather than hand-rolling it — [executors.md](executors.md#the-sdk-and-the-wire) carries the package, the header names, and the exact canonical string the signature covers. A legacy body-only HMAC is still accepted for older senders. For senders that authenticate with a **static per-account header** instead of signing each request, `--ingest-auth '{"type":"static_header","header":"X-Account-Key"}'` compares that header's value against the shared secret in constant time.

`--record-type-scope` restricts which record_types the webhook may write to (omit for all). Inbound webhooks can only be created and deleted in **preview** bases; promote with `wayai bases promote <production-id> --from <preview-id>`.

**Translate the received payload before write.** By default the payload is stored as-is. To store **canonical** records straight from an external system's raw shape, attach a mapping — the same `field_mapping` contract external sources use (renames, value maps, date reformatting, `computed`), applied in the inbound direction:

- `--source-binding '{"record_type":"<id>","source":"<name>"}'` reuses an existing source's `field_mapping`, so the same translation that proxies that record_type's reads and writes also canonicalizes inbound deliveries — one contract, both directions. **Prefer this form.**
- `--field-mapping '{"to_external":{"status":"state"}}'` is an inline mapping for a purely-native record_type with no source; `to_external` renames auto-invert on read, `to_record` spells the read direction out (replacing that auto-inversion entirely — see the warning under [`field_mapping` rules](#field_mapping-rules)), and a `computed` block covers derived fields.

The two are mutually exclusive. The mapping is validated when the webhook is created and again at promotion (a `source_binding` must still resolve to a source that has a `field_mapping`). This makes an executor-free **native-mirror sync** complete: pair an inbound webhook's read-direction mapping with an `external_write` trigger's `to_external` to keep a native record_type in sync with a plain-HTTP upstream in both directions, reusing a single source contract.

**Hydrating webhooks (notification + fetch).** Many systems send *reference-style* webhooks — "record X changed, here is its id" — instead of the full record (Stripe recommends re-fetching by id; Shopify, Salesforce, HubSpot and legacy CRMs do the same). Point such a webhook at a record_type's read source and the base **fetches the full record itself** on each delivery, then stores the canonical record:

```bash
wayai inbound-webhooks create --base <base> --name orders-ref --secret-stdin \
  --record-type-scope orders \
  --source-binding '{"record_type":"orders","source":"shop_api"}' \
  --ingest-auth '{"type":"static_header","header":"X-Account-Key"}' \
  --hydration '{"id_path":"record_id","event_path":"event","event_map":{"ResourceDeleted":"delete"}}'
```

On a delivery the base reads the record id from the thin body (`id_path`), calls the bound source's read endpoint to pull the full record, canonicalizes it through the source's `field_mapping`, and upserts it by `external_id` — the inbound twin of a proxied read. The event field (`event_path` + `event_map`, with an optional `default_op`) routes to upsert or delete; a delete skips the fetch and removes the mirrored record, and everything else defaults to upsert. The fetch and write happen **asynchronously** with automatic retry and backoff, so the sender gets an immediate accept — check progress with `wayai inbound-webhooks deliveries`. Requires `--source-binding` (it provides the read endpoint) and a single native `--record-type-scope`.

**Security:** the base only ever fetches the *source-configured* URL with the extracted id. A callback URL inside the payload is ignored, never followed. Reference-style senders typically pair with `--ingest-auth`, since they don't sign each delivery.

**Keep your own fields on re-sync — `merge`.** By default each re-sync replaces the whole record, so any field you added locally (a workflow status, tag, score, assignment, internal note) is overwritten. Add `"merge": true` to `--hydration` and a re-sync refreshes **only** the fields the source's mapping owns and **preserves** everything else on the record — so one record_type holds both the upstream mirror and your local enrichment, and you can still list and filter the entity by your own field (`status=qualified`). The first sync still creates the record from the mapped fields; a field the upstream later clears is reflected; the merged record is still schema-validated. Example: `--hydration '{"id_path":"record_id","merge":true}'`.

---

## Triggers

Triggers fire automatically when records or files change. A trigger's action is one of:

- **webhook** — an HMAC-signed HTTP POST to an external system (`--url` plus a signing secret).
- **internal_write** — the base updates a *referenced* record for you, with no external service. On a matching write it locates a target record and applies a guarded patch, with an optional compare-and-set `precondition` so the update only lands when the target is still in the expected state. Ideal for "when A changes, update related B" — booking an appointment flips its slot to busy only if it was free. `record.created`/`record.updated` only. Two modes: **`async`** applies the patch just after the triggering write (eventual; a precondition miss is recorded but the triggering write still stands), and **`transactional`** applies it atomically *with* the triggering write — if the `precondition` misses, the whole write is rejected and rolled back. Use `transactional` when the two writes must succeed or fail together.
- **external_write** — the base writes the changed record *out* to an external system by reusing an external source's write path (its field translation, endpoint, body shape and auth), with **no executor**. Use it to keep a native record_type mirrored to a legacy API. The action names the source; `record_type` is the record_type that *defines* the source (a native mirror points at a sibling source-backed record_type, and defaults to the triggering record_type if omitted), and `op` defaults from the event (created→create, updated→update, deleted→delete) and is overridable. On a **create**, the base reads the upstream-assigned id from the response and links it back onto the triggering record as its `external_id` — so the native record points at its upstream counterpart and retries don't duplicate; opt out with `"link_external_id": false`. Updates and deletes are dispatched only for records already linked to that source. By default every matching update is pushed upstream; `watched_fields` scopes **update** dispatch to writes that actually changed an upstream-mapped field — `"mapped"` watches exactly the fields the source maps out (so a write touching only a locally-owned field skips the redundant no-op push), or pass an explicit array of data-field names. Creates and deletes always dispatch. Delivery is async, signed, retried and dead-lettered like a webhook. `record.created`/`record.updated`/`record.deleted` only.

```bash
# webhook — the signing secret comes from --secret-stdin or --secret-prompt, never argv
wayai triggers create --base <base> --name <name> --events <comma-separated> \
  --url <url> (--secret-stdin | --secret-prompt) \
  [--id <id>] [--record-type-scope <comma-separated>] [--filter <json>] [--skip-cascade-writes <bool>]

# internal_write
wayai triggers create --base <base> --name slot-busy --events record.created --record-type-scope appointments \
  --action '{"type":"internal_write","target":"slots","match":{"source_field":"data.slot_id"},"patch":{"status":"busy"},"precondition":{"field":"data.status","op":"eq","value":"free"},"mode":"async"}'

# external_write — mirror a native record_type's writes out through a source's write path
wayai triggers create --base <base> --name mirror-out --events record.created,record.updated --record-type-scope appointments \
  --action '{"type":"external_write","record_type":"upstream_appointments","source":"legacy","op":"create","watched_fields":"mapped"}'

wayai triggers list --base <base>
wayai triggers get <id> --base <base>
wayai triggers deliveries --base <base> [--status <pending|delivered|failed|dead>] [--trigger-id <id>]
wayai triggers delete <id> --base <base> [-y]
```

`--events` is a **comma-separated list**, not a JSON array. Valid events: `record.created`, `record.updated`, `record.deleted`, `record.cancelled`, `relationship.created`, `relationship.updated`, `relationship.deleted`, `file.created`, `file.updated`, `file.deleted`. Bare names like `create`/`update` silently never match. File events fire when a file is written (created / any content-or-metadata change / deleted); a file trigger's `--record-type-scope` names **file types** and its `--filter` matches the file's metadata (see [files.md](files.md)). File events are **webhook-only** — `internal_write` and `external_write` are record-only.

`--record-type-scope` limits firing to specific record_types (omit for all). An `internal_write` target record_type stays available as long as a trigger references it. For `external_write` the referenced source is validated at create time (the named record_type, source, and write endpoint for each op must exist) — and referencing a source from a trigger never turns the triggering record_type into a proxy; it stays native.

`--filter '<json>'` fires only on records matching a condition, using the same Filter DSL queries use, evaluated against the record (or relationship) being written: `--filter '{"field":"data.status","op":"eq","value":"paid"}'`. Combine conditions with `and`/`or` groups. A malformed filter is rejected at create time.

**Delivery.** Webhook and `external_write` deliveries are HMAC-signed with the same scheme described under [Inbound webhooks](#inbound-webhooks) — a request signature plus a timestamp the receiver checks for freshness and an idempotency key (falling back to the message id) for dedupe — so one signer serves both directions. Verify with the published SDK, never by hand ([executors.md](executors.md)). Failed attempts are retried with backoff and dead-lettered after repeated failure; inspect with `wayai triggers deliveries`.

**Loop guards.** By default writes that arrive from an inbound webhook don't re-fire triggers, and `--skip-cascade-writes <bool>` (default `true`) keeps a trigger from firing on writes an `internal_write` cascade produced. Both round-trip in config-as-code as `skip_inbound_webhook_writes` and `skip_cascade_writes` on the trigger file. Triggers can only be created and deleted in **preview** bases.

---

## Provider adapters

Move tool definitions between LLM providers and a base's record types. Supported providers: `openai`, `anthropic`, `google`, `mcp`.

```bash
# Import tool definitions as record_types
wayai bases providers import openai --base <base> \
  --tools '[{"type":"function","function":{"name":"create_invoice","parameters":{"type":"object","properties":{"customer":{"type":"string"},"amount":{"type":"number"}},"required":["customer"]}}}]'
wayai bases providers import anthropic --base <base> --tools '[{"name":"create_invoice","input_schema":{"type":"object","properties":{"customer":{"type":"string"}}}}]'
wayai bases providers import google    --base <base> --tools '[{"name":"create_invoice","parameters":{"type":"object","properties":{"customer":{"type":"string"}}}}]'
wayai bases providers import mcp       --base <base> --tools '[{"name":"create_invoice","inputSchema":{"type":"object","properties":{"customer":{"type":"string"}}}}]'
wayai bases providers import openai    --base <base> --tools @tools.json

# Export record types as tool definitions
wayai bases providers export openai    --base <base>
wayai bases providers export anthropic --base <base> --record-types invoices,customers
wayai bases providers export mcp       --base <base> --to tools.json

# Turn a provider-native tool call into a record
wayai bases providers import-call openai invoices --base <base> \
  --data '{"arguments":{"customer":"Acme","amount":5000}}' \
  --external-id call_abc123 --external-source openai
wayai bases providers import-call anthropic invoices --base <base> --data '{"input":{"customer":"Acme","amount":5000}}'
wayai bases providers import-call google    invoices --base <base> --data '{"args":{"customer":"Acme","amount":5000}}'
```

`--tools` and `--data` accept inline JSON or `@filename`. `--to <file>` (not `--output`, which selects the output format on the namespace) writes the export to disk. Imported record_types are ordinary record_types — re-model them canonically before shipping them to agents rather than leaving the provider's payload shape in place; see [Canonical-first modeling](#canonical-first-modeling) and [toolsets.md](toolsets.md).

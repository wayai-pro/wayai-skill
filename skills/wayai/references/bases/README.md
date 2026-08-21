# Bases — the Data surface

Open this the moment a task involves a **base**: designing a schema, reading or writing records,
linking entities, storing files, wiring an integration, or building an agent-facing toolset. It is
the complete concept map for the Data surface; the sibling files carry the field-level grammar.

A **base** is an org-level data container — the system of record behind your hubs. Hubs hold
conversations; bases hold the structured data those conversations act on. They are separate entities
with separate commands: `wayai publish` promotes a **hub**, `wayai bases promote` promotes a **base**,
and `wayai pull`/`wayai push` route by workspace subtree (`hubs/` vs `bases/`). Never substitute one
for the other.

## The object model

```
Organization                       ← account boundary; base tokens
└── Base (production | preview)    ← one per app, domain, or tenant
    ├── CONFIG — edit in a PREVIEW, ship with PROMOTE (human-run):
    │     record types (+ their external sources) · relationship types
    │     · triggers · inbound webhooks · Actions · toolsets · seed fixture definitions
    │     · file types (config too, but NOT in the config-as-code subtree — see files.md)
    └── DATA — read/write on ANY base, production included:
          records · relationships · files · attachments · batch writes
          (seed apply/reset/clear and seed leases run only on previews)
```

**The one rule that routes every command:** config writes are rejected on a production base — make
them in a **preview** (`wayai bases create-preview`), then have the user run `wayai bases promote`.
Ordinary data writes work anywhere; seed-fixture and seed-lease operations are preview-only. If a
write unexpectedly fails with a preview/403 error, check whether it is config or eval-state work
aimed at production.

### Every primitive, one line each

Two schema→instance pairs anchor the model: **record type → record** and **relationship type →
relationship**. Files repeat the pattern (**file type → file**).

| Primitive | What it is | Commands / reference |
|---|---|---|
| **Base** | Top-level data container. `production` (config changes only via promote; metadata like name/tags/timezone stays editable) or `preview` (`<origin>--<slug>`, a config-editable clone that promotes back to its origin) | `wayai bases` |
| **Record type** | JSON Schema defining a kind of record — created at runtime, no migrations. Carries schema hints (`x-fk` foreign keys, `x-search` tuning, `x-archival` mode, `x-timezone`) and optionally declares **external sources**. Display config is a top-level `ui` key *beside* `json_schema`, not a schema hint, and is config-as-code only | `wayai record-types` — [records.md](records.md) |
| **Record** | JSON row conforming to its record type. System-generated `id` plus your `external_id`/`external_source` for **idempotent upsert**. Can be **cancelled** (first-class state) or **archived** (terminal/inactive) | `wayai records` — [records.md](records.md) |
| **Filter DSL** | The one condition language — `{field, op, value}` plus `and`/`or` groups — used by `records query`, trigger `--filter`, and Action `precondition` | [records.md](records.md) |
| **Relationship type** | Config defining a `rel_type` (its id IS the type) with an optional metadata schema and source/target record-type allowlists. Must exist before any relationship of that type | `wayai relationship-types` — [records.md](records.md) |
| **Relationship** | Typed, directed link between two records (an endpoint can also be a file), with validated metadata; traversable in both directions | `wayai relationships`, `wayai query-relationships` — [records.md](records.md) |
| **File type** | Bucket config for files: max size, allowed content types, optional metadata schema | `wayai file-types` — [files.md](files.md) |
| **File** | Versioned, path-addressed content (PDFs, images, documents). Per-version history and diff, S3-mountable; compare-and-set writes exist on the storage surface but are not reachable from the CLI — [files.md](files.md) says why | `wayai files` — [files.md](files.md) |
| **Attachment** | Simple record-scoped upload — Files are the richer, versioned evolution | `wayai attachments` — [files.md](files.md) |
| **External source** | Declared **on a record type**: backs its CRUD operations with an external API through one bidirectional `field_mapping` contract | [integrations.md](integrations.md) |
| **Trigger** | Fires on record/relationship/file events. Its action: signed **webhook** out, **internal_write** (guarded patch to a related record), or **external_write** (push the change upstream through a source) | `wayai triggers` — [integrations.md](integrations.md) |
| **Inbound webhook** | Signed ingest URL an external system pushes to — payload stored raw, translated via a mapping, or **hydrated** (thin "id changed" notification → the base fetches the full record through a bound source) | `wayai inbound-webhooks` — [integrations.md](integrations.md) |
| **Executor** | A small HTTP service **you** deploy for what the platform can't call directly (custom logic, mTLS, SOAP, binary creds); receives signed dispatches | [executors.md](executors.md) |
| **Action** | One record type + one operation, shaped and guarded (`filterable_fields`, `writable_fields`, `precondition`, `binding`) — or a **composite** of atomic multi-record steps. Its id is the agent-facing tool name | `wayai actions` — [toolsets.md](toolsets.md) |
| **Toolset** | A curated MCP server composed of Action references (plus relationship/batch/SQL tools), served at `https://data-mcp.wayai.pro/t/<slug>/mcp` | `wayai toolsets` — [toolsets.md](toolsets.md) |
| **Seed fixture / lease** | Named, reversible records+relationships baseline for hermetic agent evals. Definitions are config; `apply`/`reset`/`clear` are preview-only data operations. An actor-bound lease exclusively resets, owns, and clears fixture state for one eval run | `wayai seed` — [config-as-code.md](config-as-code.md) |
| **Base API token** | `rec_…` credential: grant-scoped (bases × record types × environments × permissions) or **toolset-bound** (its authorization IS one toolset's tool surface) | `wayai bases tokens` — [toolsets.md](toolsets.md) |
| **Credential** | Encrypted credential held by a base (string or file blob), referenced from that base's source/trigger config as `credential:<name>`, pullable by an executor at dispatch | [toolsets.md](toolsets.md) |

**Identity rules.** Config slugs are permanent ids — the `id` of a base, record type, relationship
type, Action, and toolset *is* its identity and is **immutable**. There is no rename: to "rename" one
you create a new entity and migrate its data. Choose clear, stable, lowercase-slug ids up front
(`patients`, `treated_by`). Display names and descriptions stay editable. Always set `external_id` on
records that have a natural key — retries and re-imports then upsert instead of duplicating.

**Consistency & limits.** Single-record `get`s and eligible active-record list queries reflect writes
immediately; `sql`, `search`, and filters on the built-in `created_at`/`updated_at` timestamps written
without a timezone offset may lag a write by a moment. Record/relationship JSON is capped at ~1 MiB —
large or binary content belongs in Files.

## Agent guidelines for base work

- Schema work (record types, relationship types, file types, inbound webhooks, triggers, Actions,
  toolsets, seed fixtures) only happens in **preview** bases. Create or select one before changing
  schema.
- **Promotion is human-only.** When the schema is ready, surface the exact `wayai bases promote`
  command and wait for the user to run it. Recommend `--dry-run` first.
- Base tokens are shown only **once on creation**. Tell the user to copy and store one before doing
  anything else.
- Once the config spans more than a couple of entities, prefer config-as-code
  (`wayai pull bases/<base>` / `wayai push bases/<base>`) over one-off commands, so changes are
  reviewable files. See [config-as-code.md](config-as-code.md).
- Never auto-commit. Show `git diff` and wait for approval.
- `wayai <command> --help` is the authoritative flag list at runtime. If a command, flag, or URL
  appears in neither these references nor `--help`, assume it does not exist — never guess one.

## Task → reference map

| You need to… | Use | Reference |
|---|---|---|
| Store & validate structured business data | record type schema + records | [records.md](records.md) |
| Change one field without resending the record | partial update (`patch`) | [records.md](records.md) |
| Prevent duplicates on retries and re-imports | upsert by `--external-id` | [records.md](records.md) |
| Link entities; traverse a graph | relationship type + relationships | [records.md](records.md) |
| Make a reference resolve to a real record | `x-fk` on the schema field | [records.md](records.md) |
| Find by approximate name/text | Filter DSL `search` operator | [records.md](records.md), tuning in [querying.md](querying.md) |
| Reports, aggregations, joins | `wayai bases sql` | [querying.md](querying.md) |
| Several writes, all-or-nothing | `wayai bases batch` | [records.md](records.md) |
| Load historical data in bulk (backfill) | `wayai bases import run` (NDJSON; streams, resumes) | [records.md](records.md) |
| Store PDFs / images / versioned documents | file type + files; mount over S3 | [files.md](files.md) |
| Notify an external system on data change | trigger → webhook | [integrations.md](integrations.md) |
| "When A changes, update related B" | trigger → `internal_write` | [integrations.md](integrations.md) |
| Mirror native records out to an upstream API | trigger → `external_write` reusing a source | [integrations.md](integrations.md) |
| Accept pushes from outside | inbound webhook (+ optional mapping / hydration) | [integrations.md](integrations.md) |
| Read/write an external API as if it were native | external sources on the record type | [integrations.md](integrations.md) |
| Integrate something the platform can't call directly | executor | [executors.md](executors.md) |
| Give an LLM agent safe, domain-named tools | Actions + toolset + toolset-bound token | [toolsets.md](toolsets.md) |
| Make a state-dependent write race-free | Action `precondition` (compare-and-set) | [toolsets.md](toolsets.md) |
| Least-privilege machine/CI access | scoped tokens (`bases tokens create-for-toolset`) | [toolsets.md](toolsets.md) |
| Store integration credentials | base credentials (`credential:<name>` references) | [toolsets.md](toolsets.md) |
| Manage config in git / review in PRs | `wayai pull bases/<base>` / `wayai push bases/<base>` | [config-as-code.md](config-as-code.md) |
| Deterministic agent evals against known data | a preview with integrations disabled + seed fixtures | [config-as-code.md](config-as-code.md), and [`../evals.md`](../evals.md) for the hub side |
| Import/export provider tool definitions | provider adapters | [integrations.md](integrations.md) |
| See who changed what, and when | `history` subcommands (admin-gated) | [records.md](records.md) |
| Shape schemas/tools so agents succeed | modeling + tool-design principles | [toolsets.md](toolsets.md) |
| Decide native vs proxy vs mirror backing | the pattern catalog | [integrations.md](integrations.md) |

## Getting a base for the first time

You are already authenticated — one `wayai login` covers hubs and bases, and `wayai status --json`
reports it. There is no second CLI, no second login, and no second workspace.

```bash
wayai bases create <base-id> --name "<Display Name>" [--tags a,b] [--timezone <IANA>]
```

`--environment production` is the default, and what your plan bounds is **capacity** — records,
operations, import rows — not which environment you may create.

`--tags` names the organization's own tags — the same list hubs and credentials draw from, created
under Settings → Organization → Tags. A name that is not in that list is rejected rather than
created, so tags mean the same thing wherever they appear.

**Create production first for anything you intend to keep.** A base's environment cannot be changed
while it exists, and there is no step that links an origin-less preview up to production later — so
the durable name should be claimed by the production base at creation; then take a linked preview off
it for config work and promote from there. (The slug is recoverable if you get this wrong, but only
by destroying the preview's data and starting the base over on the same id — see below.)

```bash
wayai bases create <base-id> --name "<Display Name>"          # production — claims the durable id
wayai bases create-preview <base-id> --name "<Display Name> (dev)"
# …edit record types, triggers, toolsets in the preview…
wayai bases promote <base-id> --from <preview-id>
```

Create a base **directly** as a preview only when it is ephemeral — an eval run or a throwaway
sandbox under a derived, unique id:

```bash
wayai bases create <eval-run-id> --name "Eval run" --environment preview
```

A preview created that way has **no production origin**, so it cannot be promoted directly, and
`environment` cannot be changed on a base that exists. That is the trap to avoid for a durable base:
there is no later step that links it up. The way back is to destroy the preview's data — delete it
with purge — and, once that deletion completes, create the same id again as production: a freed id
is a blank slate and may be created in either environment. Nothing carries over, so it is a fresh
start on the slug rather than a promotion; see [config-as-code.md](config-as-code.md).

Then define the first record type in a preview, write test records, and iterate:

```bash
wayai bases create-preview <base-id> --name dev
wayai record-types upsert <slug> --base <base-id>--dev --name "<Display Name>" --schema @schema.json
wayai records upsert <slug> --base <base-id>--dev --data '{...}'
```

Set `WAYAI_BASE=<id>` in the shell to stop passing `--base` on every command. **`wayai use bases/<id>`
is not an alternative** — the worktree scope is a routing tripwire for `pull`/`push` only, and every
other command still resolves its target from `--base` or `WAYAI_BASE` and errors out without one.

When the schema is right, surface the promotion for the user to run:

```bash
wayai bases promote <base-id> --from <base-id>--dev --dry-run
```

## Connecting a hub to a base

Two paths work today, and neither is the same thing as the hub's own config:

- **Agent tools** — build a toolset ([toolsets.md](toolsets.md)), mint a toolset-bound `rec_` token,
  and add it to the hub as an ordinary **MCP Server** connection (Streamable HTTP, Bearer Token)
  pointed at `https://data-mcp.wayai.pro/t/<slug>/mcp`. The agent then sees your domain tools
  (`book_appointment`, `list_invoices`), not generic data operations. See
  [`../connections.md`](../connections.md) for the connection mechanics.
- **Eval seeding** — an eval set or journey names a `fixture:` and a `target_base:`, and the hub's
  seed connection supplies the API origin and a `write:seeds` token. See [`../evals.md`](../evals.md).

A hub's own knowledge/skill **resources** are a different surface and stay hub-local — they are not
base files. See [`../resources.md`](../resources.md).

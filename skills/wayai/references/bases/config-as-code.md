# Base config as code, environments, and seed fixtures

Open this when managing a base's config in git, moving config between preview and production, or
setting up hermetic data for agent evals. Concepts are in [README.md](README.md).

## The workspace subtree

Seven config entity kinds — record types, relationship types, inbound webhooks, triggers, toolsets,
Actions, seed fixtures — live as version-controlled YAML under `wayai-ws/bases/<base>/`, one file per
entity, so changes review cleanly in a pull request.

> **File types are NOT in this subtree.** They are config in every other sense (preview-only writes,
> promoted with the base), but `pull` never writes them, `push` never applies them, and `--prune`
> never touches them. Manage them with `wayai file-types` ([files.md](files.md)) and do not assume a
> committed base folder captures them — it does not.

```
wayai-ws/
├── wayai.yaml                        # OPTIONAL repo defaults — declare AT MOST ONE of
│                                     #   default_hub: / default_base:
├── hubs/<hub>/                       # hub config-as-code (a different subtree, a different verb set)
└── bases/
    └── <base-id>/
        ├── base.yaml                 # base identity; `wayai use bases/<folder>` reads it
        ├── record-types/<id>.yaml
        ├── relationship-types/<id>.yaml
        ├── inbound-webhooks/<id>.yaml
        ├── triggers/<id>.yaml
        ├── toolsets/<id>.yaml
        ├── actions/<id>.yaml
        └── seeds/<id>.yaml
```

```bash
wayai pull bases/<base>              # server config → files (also mirrors the linked production, read-only)
wayai push bases/<base> [--dry-run]  # apply local files to the PREVIEW base (--dry-run shows the diff only)
wayai push bases/<base> --prune      # also delete entities on the server that your files no longer declare
wayai use bases/<base>               # scope THIS git worktree to a base (--add widens instead of replacing)
wayai unbind bases/<base>            # drop that base from this worktree's scope (bare `wayai unbind` clears everything)
```

### `pull` / `push` route by subtree, and refuse a mixed invocation

`pull` and `push` are **one verb each**. They resolve to exactly one subtree, in this order: an
explicit `--hub`/`--base` flag → an explicit positional (`hubs/support`, `bases/crm`) → the folder you
are `cd`'d into → `wayai-ws/wayai.yaml`'s `default_hub`/`default_base` → the sole candidate folder
across both subtrees.

An invocation that names targets in **both** subtrees is **refused, never merged** — non-zero exit,
before any network call and before any file write. So in a workspace holding both a hub folder and a
base folder, a bare `wayai pull` is refused as ambiguous: name the subtree
(`wayai pull bases/crm`, `wayai push hubs/support`) or declare **one** default in
`wayai-ws/wayai.yaml`. Passing both `--hub` and `--base` is refused the same way.

Per-subtree refusals survive unchanged: `push` against `hubs/` targets preview hubs (production is
`wayai publish`), and `push` against `bases/` targets preview bases (production is
`wayai bases promote`).

### Rules that bite

- **Previews are the only edit target.** `push` writes preview bases only — config is never edited
  directly in production. Pulling a preview also writes the linked production config beside it as a
  **read-only, git-ignored mirror** so you can see the live config while iterating; `push` refuses that
  folder before any network round-trip, and bare auto-selection ignores it. Pulling a production base
  directly writes only that read-only mirror.
- **Auto-create a preview.** Scaffold `wayai-ws/bases/<name>/base.yaml` with
  `origin_base_id: <production-id>` and **no** `base_id`; `wayai push bases/<name>` creates the preview
  from that origin, writes the new id back, and applies your files. `--dry-run` **stops before any
  request** on that branch and says what a real push would create, rather than creating it — a dry run
  must not mint a base, and a CI pipeline re-running one would otherwise orphan a fresh base each time.
- **`base.yaml` describes the base; it does not reconfigure it.** Alongside `base_id`/`origin_base_id`,
  a pull records `name`, `description`, `environment`, `integrations` and `analytics` there so the
  folder documents what it targets. For a folder that already has a `base_id` those are purely
  descriptive — editing them changes nothing and the next pull overwrites them. Treat a surprising
  value as a signal to re-check the base, not to edit the file. **The auto-create branch is the
  exception**: there `name`, `description` and `analytics` are all forwarded to the base `push`
  creates — and `name` is load-bearing, because it becomes the new preview's id suffix
  (`<origin>--<name>`). `analytics` is the one you cannot change afterwards. No CLI flag sets the
  analytics tier, so that branch is the *only* place you can choose it — and it is a one-shot choice:
  the tier is fixed at creation, and a base minted on the wrong one can only be replaced, never
  corrected. Declare `analytics: standard` before the first push if the origin is a preview (whose
  clones default to `local`) and you will want SQL, history or traversal on it
  ([querying.md](querying.md)).
- **The worktree scope is a routing guard.** Each git checkout carries one untracked scope file
  (`<git-dir>/wayai-scope`) holding a set of hubs and a set of bases; the base axis is what applies
  here. `pull`/`push` refuse to run against a base outside a non-empty set, and the axis is seeded
  automatically on the first successful pull/push into a checkout that has none. It is a routing
  tripwire for those two verbs only — no other command reads it. `wayai status` reports both axes.
  **If pull/push errors with a scope refusal, stop and ask the user before doing anything else** —
  it usually means the prompt was meant for a different worktree. A worktree legitimately serving
  several bases widens with `wayai use --add bases/<id>`, but do not run that, `wayai use`, or
  `wayai unbind` without explicit user instruction in the current session; widening the scope is as
  much a routing decision as clearing it. Full semantics: SKILL.md § Worktree scope.
- **Secrets are never written to files.** Inbound-webhook, trigger and external-source secrets are
  stripped on pull. On push, a newly added inbound webhook or trigger gets a fresh secret, printed
  once — save it. Existing secrets are left untouched. Manage inbound-webhook and trigger values with
  `wayai inbound-webhooks` and `wayai triggers`; an external source's credential is a credential of
  the base itself, managed through its credential API.
- **Deletions are opt-in.** Because deleting a record type also removes its records, `push` is
  additive by default: an entity on the server that your files no longer declare shows in the diff as
  a removal but is **kept** unless you pass `--prune`. The corollary bites when copying config between
  folders: an entity file you forgot to copy is one the push simply will not create, so check the diff
  rather than assuming a clean apply means a complete one.
- **Renaming a record type is re-seed, not in-place.** A record type's `id` is its identity and must
  equal its filename (`record-types/<id>.yaml`), so a rename means changing **both** the filename and
  the inner `id:` together (changing only one errors; or delete the inner `id:` so it defaults to the
  filename). Either way it is a **create-new + orphan-old**: the diff shows `+ <new>` / `- <old>`,
  push creates a new **empty** record type, and the old one keeps all its records. To rename and keep
  the data: push to create the new empty type, re-seed its records (`wayai records upsert`, or
  `wayai bases batch`), then `push --prune` to delete the orphan — which cascades its records. The
  same filename-is-`id` rule applies to every config entity.
- **Push prints advisory tool-design warnings** for the toolsets it applies (see
  [toolsets.md](toolsets.md)). They go to stderr and never block a push.
- **Record base context in `AGENTS.md`.** The CLI does **not** seed one for a base folder — create
  `wayai-ws/bases/<base>/AGENTS.md` (plus a one-line `CLAUDE.md` holding `@AGENTS.md`) yourself and
  keep it current. See [`../agents-md-template.md`](../agents-md-template.md).

## Environments: preview, production, promote

```bash
wayai bases create-preview <origin-id> --name <slug> [--description <d>] [--integrations enabled|disabled] [--create-only]
wayai bases list-previews <origin-id>
wayai bases update <preview-id> --integrations enabled|disabled
wayai bases promote <production-id> --from <preview-id> [--dry-run] [--record-types <ids>] [--triggers <ids>] [--inbound-webhooks <ids>]
wayai bases promotions <production-id>
wayai bases rollback <production-id> --promotion <promotion-id>
```

The preview's id is `<origin-id>--<slug>`; use that as `--base` for every schema command. The origin
may be a production base **or another preview** — cloning a preview links back to the preview it was
cloned from, so it promotes *through* that origin rather than straight to production.

Re-running `create-preview` re-applies config onto an existing preview and leaves its data alone.
Pass `--create-only` to fail instead — which is what you want for a throwaway per-run base, where
landing on someone else's would corrupt both runs.

**Promotion is human-only.** Surface the command and let the user run it; recommend `--dry-run`
first. Promote selectively with `--record-types`/`--triggers`/`--inbound-webhooks` (omit to promote
everything). `promotions` lists prior promotions and `rollback` reverts one by id. Promotion is
blocked if it would break a published toolset — a dry run lists the conflicts. An **inline** source secret does not travel with a
promotion — it is environment-local, so a newly-promoted source carrying one stays unconfigured
until you set it on production. A `credential:<name>` reference resolves against the credentials of
the base serving the request, and promotion publishes the preview credentials you opted in
(overwriting production's row of that name), leaving opted-out and organization-linked ones alone. A
promotion (including `--dry-run`) reports every referenced credential production will still lack.

**Delete vs purge.** `wayai bases delete <id>` is a tombstone: the base stops being listed, but its
records, relationships, file metadata and config are retained, and recreating the id restores them.
`--purge` (preview bases only) destroys them permanently, and also starts a background deletion of
everything the base leaves outside itself — its analytical rows and its uploaded file contents — so
those stop occupying storage rather than lingering. Until that deletion finishes, the base's records
keep counting toward the organization's record limit, because they still exist.

A purged id is **retired while that deletion runs** — recreating it is refused, because the previous
base's rows are still out there and would otherwise be served to the new one — and **frees itself
once the deletion completes**. The deletion is asynchronous, so a run that needs a base right now
should derive a fresh id rather than wait for the old one to come back.

A freed id comes back as a **blank slate**: the new base takes its own name, description, settings
and tags, and it may be created in either environment. That last part is what makes the id worth
reclaiming rather than replacing — a preview you prototyped on can become the durable production
base under the same slug, once its data is gone. Nothing about the previous base carries over, so
state everything the new one needs; in particular, if you want the environment to change, say so
explicitly, because a recreate that omits it keeps the previous one rather than taking the usual
default.

Tokens do not survive the recycle. A token scoped to a purged base stops working on that id: when the
id is later reused, the new base is a different base as far as the token is concerned, and requests
carrying the old token are answered as though it were not there. Mint a token for the new base after
creating it — never reuse the previous run's.

### Eval mode (`--integrations disabled`)

A preview can run with its external integrations **disabled** (the default is `enabled`; production
bases are always `enabled`). When disabled, every external edge goes inert — record-type sources,
outbound `external_write` triggers, inbound-webhook hydration — and the preview serves its own
**seeded** data through the exact same schema, tools and field mappings, without calling the external
systems.

That makes a preview a deterministic, production-safe target for agent evals: exercise the agent's
policy against the same canonical surface without polluting a production-only upstream or hitting live
rate limits and PII. Seed it by writing records while disabled (writes to source-backed record types
land locally), flip to `enabled` to test the real integration, and re-clone from the origin to stay
drift-free.

This works *precisely because* the model is integration-agnostic — it is the canonical-first rule in
[integrations.md](integrations.md) seen from the other side, not a separate feature.

## Seed fixtures

A **fixture** is a named, reversible set of records and relationships you apply to a **preview** base
to give agent evals a clean, re-runnable starting state. The definition is config; the apply/reset/
clear verbs are preview-only data operations.

```yaml
# wayai-ws/bases/<preview-base>/seeds/demo.yaml
id: demo
description: Overdue-bills demo baseline
records:                       # author foreign-key targets BEFORE the records that reference them
  - record_type: customers
    external_id: cust-1
    data: { name: Ada }
  - record_type: bills
    external_id: bill-1
    data: { customer_id: cust-1, status: overdue, amount: 120 }
relationships:                 # endpoints reference records by external_id
  - rel_type: owes
    source: { record_type: customers, external_id: cust-1 }
    target: { record_type: bills, external_id: bill-1 }
exclusive_record_types: [customers, bills]   # optional: reset also removes non-declared records of these types
```

```bash
wayai push bases/<preview>                  # store the fixture with the rest of the config
wayai seed apply <name> --base <preview>    # write the fixture's records/relationships
wayai seed reset <name> --base <preview>    # restore the declared baseline (fixes rows a run mutated)
wayai seed clear <name> --base <preview>    # delete every row the fixture owns (teardown)
wayai seed list --base <preview>
wayai seed get <name> --base <preview>
```

- **Idempotent by `external_id`.** `apply` upserts each record by its external id, so re-applying
  updates in place and never creates a duplicate.
- **`reset` restores the baseline.** It re-applies the declared records in place (fixing any a run
  mutated — a status a trigger flipped, say) and removes fixture-owned rows the fixture no longer
  declares. Use it between runs for a guaranteed-clean start.
- **`exclusive_record_types` makes `reset` authoritative for whole record types.** Records an agent
  *creates* during a run carry no ownership marker, so a plain reset leaves them behind and they
  accumulate. List the record types your evals write to and reset also removes every record of those
  types the fixture did not declare — regardless of who created it, including other fixtures' rows —
  plus relationships pointing at removed records. A record referenced by one of the fixture's declared
  relationships always survives. Opt-in and reset-only; choose exclusive types deliberately on a
  shared preview, because the fixture becomes the single source of truth for them.
- **`clear` is teardown.** It deletes exactly the rows the fixture owns and nothing else — ownership
  is recorded via a reserved marker, so teardown can never touch non-seed data.
- **Preview-only.** `apply`/`reset`/`clear` refuse production. The *definition* promotes with the base
  like any other config; the data ops stay preview-scoped.
- **No trigger side-effects.** Applying or resetting a fixture is baseline setup and fires no
  triggers, so restoring data cannot cascade.

### Seed leases

For concurrent or outcome-unknown eval runs, use an actor-bound **lease** instead of calling `reset`
and `clear` separately.

```bash
wayai seed lease acquire <fixture> --base <preview> --lease-id <uuidv7> --owner-ref <safe-run-id>
wayai seed lease status --base <preview> [--lease-id <uuidv7>]
wayai seed lease release <fixture> --base <preview> --lease-id <uuidv7> --owner-ref <safe-run-id>
```

Acquire atomically resets the fixture **and** locks the whole preview's fixture-data operations;
release atomically clears the fixture-owned data and unlocks it. Generate a fresh UUIDv7 once and
persist the immutable tuple until release. `owner_ref` is only a safe run label and never grants
authority: it must be 1–200 characters, start with a letter or digit, and otherwise use only letters,
digits, or `._:@/+~=-` (no whitespace).

Only the same verified identity may retry, inspect, or release the tuple. Exact retries are no-ops,
released ids are never reusable, and another authorized actor learns only that the base is locked. If
an acquire times out with an unknown outcome, retry the **same** tuple. While any lease is active,
ordinary `apply`/`reset`/`clear` for *every* fixture on that base are refused; fixture-definition
config CRUD stays available. The permanent ledger is capped at 1,000 ids per preview (warnings begin
at 800 and 950) — replace the disposable preview when exhausted.

**The hub side drives this for you.** An eval set or journey that declares `fixture:` and
`target_base:` has the platform acquire the lease before the session and release it after, so a
mutating eval starts from a known baseline every run. You rarely call these verbs by hand — see
[`../evals.md`](../evals.md).

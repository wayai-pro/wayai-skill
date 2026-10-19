# Files and attachments

Open this before storing PDFs, images, generated reports, or any large/binary content in a base.
Concepts are in [README.md](README.md).

Record and relationship JSON is for structured data and is capped at ~1 MiB — inlining base64 blobs is
rejected. Binary content goes in **Files** (versioned, path-addressed, mountable) or, for the simple
case, **attachments** (record-scoped uploads). Files are the richer evolution; prefer them for
anything you will version, diff, describe with metadata, or mount.

Files are where an organization's documents live. A hub's or an organization's **knowledge
resources** move into an organization's knowledge base as file types with
`wayai knowledge migrate` ([below](#moving-knowledge-resources-into-a-base)); **skills** stay
resources ([`../resources.md`](../resources.md)).

## File types

A file type is a bucket that configures storage policy — max size, allowed content types — plus an
optional metadata schema. It is **config**: write it in a preview base, then promote.

- **Promote** adds new file types to production and updates changed ones. A changed policy or
  metadata schema applies to later writes; existing files are not re-checked.
  `wayai bases promote --file-types <ids>` promotes only the file types it names.
- **Files never move with config.** They are data: a promotion or a new preview carries the file
  type, never its files. Sync the files themselves with `wayai files pull` and `push`
  ([below](#syncing-files-as-a-folder)), on any base.
- **Removing a file type through promote** happens only when production holds no live file of it.
  Otherwise the promotion is blocked and `--dry-run` names the file type: recreate it in the preview
  to keep it, or delete its files on production first.
- **Rollback** restores the file types production had before that promotion. A file type the
  promotion added is kept if it has gained files since, and the rollback names it.
- **`create-preview`** copies its origin's file types into the new preview.

Unlike every other config entity, a file type is **not** part of the config-as-code subtree yet — `pull`
does not write it, `push` does not apply it, `--prune` does not remove it. Manage one with these
commands, or through the base's config API, which carries a `file_types` section
([config-as-code.md](config-as-code.md)).

```bash
wayai file-types upsert <id> --base <base> --name <name> \
  [--description <desc>] [--max-size <bytes>] [--allowed-content-types <json|@file>] [--metadata-schema <json|@file>]
wayai file-types list --base <base>
wayai file-types get <id> --base <base>
wayai file-types delete <id> --base <base> [-y]
```

## Files

```bash
wayai files put <file_type> <path> --base <base> --file <local> [--content-type <mime>] \
  [--if-match <hash>] [--label <text>] [--metadata <json|@file>]
wayai files get <file_type> <path> --base <base> [--to <local>] [--meta] [--version <n>]
wayai files list <file_type> --base <base> [--prefix <path>] [--depth <n>] [--limit <n>] [--offset <n>]
wayai files mv <file_type> <from> <to> --base <base>
wayai files rm <file_type> <path> --base <base> [--if-match <hash>] [-y]
wayai files history <file_type> <path> --base <base> [--limit <n>] [--offset <n>]
wayai files diff <file_type> <path> --base <base> --from <n> --to <n>
wayai files pull <file_type> --base <base> [--folder <path>] [--dir <local>] [--overwrite] [-y]
wayai files push <file_type> --base <base> [--folder <path>] [--dir <local>] [--dry-run] [--prune] [--label <text>] [-y]
wayai files mount <file_type> --base <base> [--mode <read_only|read_write>] [--path <prefix>] [--expires-in <seconds>]
```

A file is addressed by a **mutable path**; `mv` moves only the address, never the bytes. Paths may
contain folders (`q3/summary.pdf`); `--prefix` and `--depth` navigate them on `list`.

**Content versions.** Every content change is retained as an immutable version. Read an earlier one
with `wayai files get --version <n>` (reads return the latest by default), list them with
`wayai files history`, and compare two with `wayai files diff --from <n> --to <n>` — you get a
metadata delta plus a line-by-line content diff for text files. Re-tagging a file (metadata only)
does not create a version. Moving it with `mv` does: the same bytes become a new version at the new
path. A path's version numbers never start over. A file created at a path where another was deleted
or moved away from is a new file with its own history, but its version numbers continue past the
earlier file's; a file moved onto such a path does the same. So a version read from one file never
matches another file later at its path.

**Labels, metadata, and compare-and-set on upload.** An upload — `wayai files put`, or a REST `PUT`
of the file's path — takes three optional settings, each a request header:

- `--label <text>` (`X-Data-Label`) — a free-string label for the content version the upload
  creates. The CLI takes printable ASCII only, since a header carries nothing else intact.
- `--metadata <json|@file>` (`X-Data-Metadata`) — the file's structured metadata, a JSON object,
  checked against the file type's metadata schema. Without it the file keeps the metadata it has.
- `--if-match <hash>` (`If-Match`) — compare-and-set: the file's content hash, which a download
  returns as its `ETag` (`--meta` shows it as `content_sha256`). The upload is refused with a 409 if
  the file changed since, or no longer exists.

A REST upload can instead ask to create the file only: with `If-None-Match: *` it is refused with a
412, and the file at the path is left as it is, if one already exists there. It does not combine
with `If-Match`, and takes no value but `*`. `wayai files put` does not send it yet.

Without them, `put` is an unconditional overwrite. Read the label and metadata back with `--meta`
and `diff`.

**Compare-and-set on delete.** `wayai files rm --if-match <hash>` (a REST delete's `If-Match`)
removes the file only if its content hash still matches, and is refused with a 409 otherwise.
Without it a delete is unconditional.

**Searching.** The REST list takes three optional search parameters, which search the whole folder
tree under `prefix` (they do not combine with `depth`):

- `query` — case-insensitive text in the file name or in a string `title` in the file's metadata.
- `tags` — repeatable; matches files whose metadata `tags` array holds any of them.
- `metadata_filter` — a MongoDB-style filter over the metadata, as JSON (`$eq` `$ne` `$gt` `$gte`
  `$lt` `$lte` `$in` `$nin` `$contains` `$and` `$or` `$not`). A field the file type's metadata
  schema does not declare is refused.

`title` and `tags` are ordinary metadata: declare them in the file type's metadata schema to use them.
A search that reads 5,000 files without filling its page is refused — search a narrower prefix.
`wayai files list` does not declare these parameters yet.

**Mounting a file type as a filesystem.** `wayai files mount <file_type>` mints an S3-compatible
credential, so any harness or tool that speaks S3 (s3fs, cloud storage mount SDKs) can mount the
bucket as a local folder and read files with ordinary filesystem calls — no per-tool integration. It
prints an endpoint, bucket name, and access keys; **the secret is shown once**. Point your S3 client at
them with path-style addressing.

- `--mode read_only` (default) or `--mode read_write` sets the whole-bucket mode. Minting a write
  credential requires write access to files.
- `--path <prefix>` confines the credential to a sub-tree and is **repeatable**. Suffix a prefix with
  `:read_only` or `:read_write` to set its mode individually (`--path drafts/:read_write`); an
  unsuffixed prefix takes `--mode`, defaulting to read-only.
- `--expires-in <seconds>` sets the lifetime (default 30 days, max 90). Prefer a short one.

A `read_write` mount can write files back, and those writes are conflict-safe and versioned: each
change is checked against the version you started from (a create requires the path be new; an update
requires the file be unchanged since you read it), so a stale overwrite is refused rather than
silently clobbering a concurrent change. Every write is a new version with full history. A delete
may carry `If-Match` too, and is refused with 412 if the file changed since you read it.

**Linking files to records.** A relationship can point at a file — its endpoint kind is `file`. If the
link's relationship type is declared **`cascade`**, the link owns the file, and dropping the file's
last owning link removes it: whether that link is deleted on its own, in a batch, or with its
relationship type or the record type at its other end; moved to archival storage after its record is
deleted, or with its record when the record is archived; or rewritten by an upsert so it no longer
owns the file. While another owning link still holds the file, dropping one keeps it. A non-cascade
link is a plain reference, so a file shared across records survives.

A relationship type's `cascade` flag cannot change while the type has live links: an upsert of the
type, a config push, a preview re-clone, a promotion or a rollback that would change it is refused
and changes nothing, since it would turn every existing link from owning to reference or back. Take
the links off the type first: delete them, change the flag, and re-create them, or move them to a new
relationship type with the flag you want. Deleting a file's last owning link removes the file, so give
a file you keep another owning link before you delete its old ones.

**File tools for agents.** A toolset can give its agent `list`, `get`, `put` and `delete` tools over
one file type and folder. A tool's `put` and `delete` check the file's **version**, not its content
hash, so their rules differ from the mount's — see [File tools](toolsets.md#file-tools). A hub agent
connected to the toolset, and holding both `list_files` and `read_file`, reads these documents at
`data/<file-type>/<path>` with those two tools, and writes them there with `write_file` and `edit_file`
through the toolset's `put`, when it holds those too ([native-tools.md](../agents/native-tools.md#resource--file-tools)).

## Syncing files as a folder

`wayai files pull` writes a file type — or one folder of it, with `--folder` — into a local folder,
and `wayai files push` sends that folder's changes back, showing them first. This keeps documents
(policies, manuals, price sheets) as files in git, reviewed as a diff before they reach a base.

```bash
wayai files pull policies --base crm                 # → ./policies/
wayai files pull policies --base crm --folder hr     # → ./policies/hr/, paths relative to hr/
wayai files push policies --base crm --dry-run       # the diff only
wayai files push policies --base crm                 # the diff, a question, then the push
```

- **These are data commands.** They write the base `--base` names, production included: files are
  data, and promote never carries them. Config is different — `wayai push bases/<base>` still
  writes previews only. Name the same `--folder` (and `--dir`, if you gave one) on every pull and
  push of a folder.
- **Pull records what it saw.** `.wayai-files.json`, at the folder's root, names the base, file
  type and folder, and each file's version and content hash. Commit it with the files: push checks
  against it from any checkout, and refuses a folder pulled from another base, file type or folder.
- **A change on the base is a conflict, not an overwrite.** A file the base changed, deleted or gained
  since your pull — an agent's edit, say — is a **conflict**: listed, and not written. Each update
  and delete also carries `If-Match` with the content hash your pull recorded, so one the base
  changes while the push runs is refused too. The rest of the push goes through, and the command
  exits non-zero (so does a `--dry-run` that lists a conflict).
- **The one gap: a new file.** Push does not yet upload a new file as create-only
  (`If-None-Match: *`), so a file new in your folder is checked against the listing push reads first. A file created at the same path in the
  moment between that read and the upload is replaced — its content stays in the file's history
  (`wayai files history`).
- **Resolving a conflict.** Commit your copy, `wayai files pull … --overwrite` to take the base's,
  merge the two with git, and push again.
- **Pull keeps your local changes.** It takes the base's changes over files you have not changed,
  removes files the base deleted that you have not changed, and keeps every file you changed or
  deleted, at the version you pulled, so a later push of it is still checked — an edit made while
  the pull runs included. `--overwrite` replaces your changes with the base's copies; it asks
  first (`-y` skips the question).
- **Deletions are opt-in.** A file you delete locally stays on the base unless you push with
  `--prune`, and prune deletes only files your last pull saw and the base has not changed since,
  under the same check. A file you never pulled is never deleted.
- **Each version names who pushed it.** Push writes through the same file routes as `put`, under
  your login. `--label <text>` labels every version it creates — a commit hash, say.
- **Content, not metadata.** Push uploads content only: a changed file keeps its metadata and
  content type, and a new file starts with no metadata and its extension's content type
  (`.md` is `text/markdown`; unknown extensions are `application/octet-stream`). Set metadata with
  `wayai files put --metadata`.
- **Left out on both sides:** `.wayai-files.json` itself, `.git` and `.DS_Store`, however a base
  path spells them (`.GIT` is `.git` on macOS and Windows). A symlink in the folder is refused, and
  so is a git repository inside it: sync into a folder of its own — a folder inside a repository
  is fine, a repository's root is not.
- **One name, one file.** macOS and Windows treat `Leave.md` and `leave.md` as one file, so pull
  refuses a base holding both, and a name that differs from another only in case — a base file
  against a local file or folder, or a new local file against any path of the file type, inside
  your folder or not — is a conflict, never written. So is one path as both a file and a folder
  (`notes` and `notes/a.md`), whichever side holds which. Rename or remove one of the two to settle
  it; `--overwrite` cannot. A case-only rename
  on the base pulls cleanly once nothing else is left under the old name; one in your folder pushes
  with `--prune`. On Windows, pull and push also refuse names that can stand for another file:
  8.3 short-name shapes (`GIT~1`, `NOTES~2.MD`) and names with `:`.
- **Without a terminal**, a push or an `--overwrite` pull needs `-y`; `--dry-run` never asks.
- **One request per file.** Pull and push pace themselves below the data request budget your other
  commands and the web console share (120 requests a minute today), and wait out that budget when
  it is spent: a large file type takes minutes.

## Moving knowledge resources into a base

`wayai knowledge migrate` moves knowledge resources into the organization's knowledge base, the
base its agents' documents live in: first the organization's, then each hub's own.

```bash
wayai knowledge migrate --org <org_id> --base <knowledge base> --dry-run   # what it would write, and why it would stop
wayai knowledge migrate --org <org_id> --base <knowledge base>             # the organization's resources
wayai knowledge migrate --org <org_id> --base <knowledge base> --hub <hub_id>   # then a hub's own
```

- **One file type per resource.** An organization resource becomes the file type named by its
  slug (`policies`); a hub resource becomes `<hub-slug>--<resource-slug>` (`support--faq`), the hub
  part taken from the hub's name when it first moves. The file type's schema names the resource it
  came from, so the id is kept after a rename of the resource or its hub: a later run finds it. A
  hub's copies of organization resources are the organization's, and move with them. Skills stay
  resources.
- **Paths are kept.** A file agents read at `resources/<slug>/<path>` moves to
  `data/<file type>/<path>`. A folder with a description gains an `_index.md` holding it.
- **Metadata is kept.** The resource's folder metadata schema is folded into its file metadata
  schema, beside `title` and `tags`, and each file carries its folders' values (the nearest folder
  winning), its own over them, and its title and tags. When a file type is first created, a field
  that would refuse a value a file holds is widened to accept it, and a field no schema declared is
  declared, so no value is lost and a filter on any of them is answered. The dry run lists each
  such field.
- **Schema refs: plain local ones only.** A schema moves with its `$ref`s only in the form
  `#/$defs/<name>` or `#/definitions/<name>`, where `<name>` needs no escaping (no `~`, `/`, `%`,
  brackets or other characters a URI encodes). Any other `$ref` (`#`, a URI, a relative ref, a
  pointer into `properties` or below a definition), any `$id` (or a draft-4 `id`, read as one),
  `$anchor` or `$recursiveRef` — in an `x-` annotation too — a definition name that needs
  escaping, and a definition both schemas name with different content stop the run, naming the
  resource and the field: simplify the schema by hand (inline what the ref names, or rename
  the definition and the refs to it), then run again. A definition both schemas hold alike is kept
  once.
- **Where it writes.** `--base` names the knowledge base: its production base, or the preview its
  file types are written in. The file types are written in the preview and promoted, with no other
  config section; like every promotion, it also publishes the preview's credentials opted in to
  production. The files are written in production only: the preview gets none of them until you
  refresh it ([below](#refreshing-a-previews-knowledge)).
- **A long id gets an alias.** A file type id past 57 characters is named by a short alias (its
  first 50 characters and 6 hex digits), so a toolset's tools for it (`delete_<name>`) fit 64
  characters: name the toolset's file tool by it.
- **Org first.** `--hub` moves the hub's own resources, read from its production hub, and refuses
  until the organization's have moved.
- **It stops before writing** on: two resources whose file types would take one id (two hubs whose
  names give the same slug, each with a resource of the same name), an id production holds for
  another resource (a hub named as another hub was when it moved) or that this command did not
  write, a folder description whose `_index.md` a file already holds, two described folders sharing
  a path, a file whose metadata title or tags disagree with its own, a resource schema whose refs
  are not plain local ones (above), a schema a file type refuses (an `x-validate` anywhere but a
  field's top level, where the move drops it with the field's other `x-` annotations, or a field
  whose schema is `null`), a new file its file type would refuse (a content-type or size limit, or
  its metadata under the whole schema — a moved file type keeps an admin's later limits and
  constraints), and a production file that differs
  from its source. That last one was edited after it moved, so it is never overwritten: remove
  it from production to move it again. Separately, the promotion is first run as a dry run: one
  that would change anything but these file types, or report a failing check, stops with the file
  types written in the preview and nothing in production.
- **Run it again freely.** What has moved is not written again; a run with nothing left to move
  writes nothing. A file type already in production is never changed by a later run — its name,
  description, limits and schema stay as they are, an edit made after the move included. A new
  file its schema refuses stops the run, naming the field and the value: widen the file type
  yourself (in the preview, then promote it), or change the file's metadata. Each version it writes
  is labelled `wayai knowledge migrate`.
- **Without a terminal** it needs `-y`; `--dry-run` never asks. It paces itself below the request
  budgets it shares with your other commands and the web console, like `wayai files push`.

## Refreshing a preview's knowledge

A preview is created with its production base's config, never its files. To test agents against
production's documents on the preview, copy them in:

```bash
wayai knowledge refresh-preview --org <org_id> --base <knowledge base> [-y]
```

- **`--base`** names the production base (its one preview is refreshed) or the preview itself.
- **What it does.** For every file type production holds, the preview's files become production's:
  each path, its content, content type and metadata. A file the preview changed is replaced, and a
  file only the preview holds is removed. Nothing else on the preview changes, and production is
  only read. The preview must already hold each of production's file types: create the preview
  again to bring its config up to date (it keeps its data), then refresh.
- **When.** Nothing runs it for you: run it after creating the preview, and whenever the preview
  should be current again. Creating a hub preview never copies files, since every hub preview of
  the organization shares the one preview base and a copy would overwrite the others' edits.
- **It reports each page** it copied: how many files it wrote, removed and left alone. A refresh
  of a preview that already equals production changes nothing, and says so.
- **Cost.** Each file written or removed counts as a file write when an API token runs it; signed
  in as a person, a refresh isn't counted. A file already equal is free, so refreshing a current
  preview costs nothing.
- **No file events.** A refresh fires none of the preview's file triggers.
- **In the console,** a preview's card on the organization's **Bases** tab has **Refresh preview
  knowledge**, which does the same and shows each page's result, or why it was refused.
- **The call behind it** is `POST /v1/{preview}/files/snapshot` with `{ "cursor"?: string,
  "file_types"?: string[] }`, one page per request. It answers
  `{ "data": { "next_cursor", "written", "removed", "unchanged" } }`; send `next_cursor` back until
  it is `null`. A `409` means a preview file changed while the page was copied: send the same cursor
  again. It needs file write access on the preview and file read access on its production base,
  and is refused on a production base and on a preview not created from one.

## Cutting a hub over to the knowledge base

Once a hub's knowledge has moved, `wayai knowledge cut-over` switches its preview's agents from
their knowledge resources to the knowledge base. It changes the preview only, whose agents read the
knowledge base's preview: [refresh it](#refreshing-a-previews-knowledge) first, since the migration
writes production only.

```bash
wayai knowledge cut-over --org <org_id> --base <knowledge base> --hub <preview hub_id> --dry-run   # every change, and why it would stop
wayai knowledge cut-over --org <org_id> --base <knowledge base> --hub <preview hub_id>
```

- **Access stays per agent.** Each distinct set of knowledge the hub's agents read (their enabled
  knowledge links) gets one read-only toolset on the knowledge base, `knowledge-<hash>`, with
  `list` and `get` on each of its file types: the organization's and the hub's own, never another
  hub's. It is written in the preview base and promoted, with no other config section. The hub
  reaches each through a [Base Toolset](../connections.md#base-toolset) connection of the same
  name, holding a new token bound to it on the preview base.
- **Each agent is given** `list_files`, `read_file` and its toolset's `list_<file type>` and
  `get_<file type>` tools (a long file type's alias in place of its id), so its documents are at
  `data/<file type>/<path>`.
- **Paths are rewritten.** `resources/<slug>/…` of a moved resource becomes `data/<file type>/…` in
  agents' instructions and additional context, and in `run_agent` automations' instructions. A
  skill's paths are left alone.
- **The sources.** Each move is recorded first: the hub's own moved resources, which become
  read-only ([resources.md](../resources.md)), and each organization resource whose link it
  detaches. A move the organization's registry refuses (a file type id another resource or hub
  holds) stops the run there, before any toolset, connection, tool, text or link is written; the
  moves recorded ahead of it stay (those resources are read-only, still read through their links).
  Once the claim is resolved, a run skips the hub's own recorded moves, and sends an organization
  resource's again while its link is enabled, which changes nothing. The agents' knowledge links
  are disabled last, once they hold their toolset's tools.
- **It stops before writing** on: a resource placeholder that reads knowledge an agent is about to
  lose (`{{files()}}`, `{{files('kb')}}` or `{{resources()}}`, and `{{resource_content()}}` or
  `{{file_content()}}` naming such a resource), which would render empty: move that text into the
  agent's instructions or a skill first; an agent on a hosted harness that reads knowledge (it
  takes no Base Toolset's tools yet, so it would lose it); an agent reading knowledge that has not
  moved; preview
  knowledge resources that differ from production's (sync or discard those changes first: a
  preview's resource is paired with production's, which moved, by its name, so two knowledge
  resources of one name stop it too); and a hub file type whose hub part two production hubs'
  names share, whose move can't be recorded until all but one are renamed; and a preview base that
  lacks a file production holds for one of its file types (refresh it first).
- **Run it again freely.** It writes only what differs (an organization resource's move is sent
  again while its link is enabled, which changes nothing), and a re-run after a run cut short
  finishes it on the same toolsets. A re-run of a finished cut-over writes nothing.
- **Production** is never touched: once the preview passes its evals, publish it, then mint a
  token for each toolset on the production base and save it on production's connection. The run
  prints both commands.
- **Without a terminal** it needs `-y`; `--dry-run` never asks.

## Attachments

Record-scoped uploads — the simpler, unversioned surface. Reach for Files instead when you want
versioning, metadata schemas, or an S3 mount.

```bash
wayai attachments upload <record_type> <id> --base <base> --filename <name> [--content-type <mime>] [--file <path>]
wayai attachments url <record_type> <id> --base <base> --filename <name>
wayai attachments list <record_type> <id> --base <base> [--prefix <path>]
wayai attachments delete <record_type> <id> <attachment-id> --base <base> [-y]
```

`upload` with `--file` uploads the local file directly; without `--file` it returns a presigned URL you
`PUT` the bytes to. `url` returns a **download** URL for an existing attachment. Filenames may contain
paths for folder structure (`docs/guide.md`), and `--prefix` filters `list` to one.

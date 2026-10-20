# Files and attachments

Open this before storing PDFs, images, generated reports, or any large/binary content in a base.
Concepts are in [README.md](README.md).

Record and relationship JSON is for structured data and is capped at ~1 MiB — inlining base64 blobs is
rejected. Binary content goes in **Files** (versioned, path-addressed, mountable) or, for the simple
case, **attachments** (record-scoped uploads). Files are the richer evolution; prefer them for
anything you will version, diff, describe with metadata, or mount.

Files are where an organization's documents live: its knowledge base, whose files agents read and
edit at `data/<file type>/<path>` through a [Base Toolset](../connections.md#base-toolset)
connection. **Skills** stay in the hub ([`../skills.md`](../skills.md)).

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

Like every other config entity, a file type is part of the config-as-code subtree: `wayai pull bases/<base>`
writes it to `file-types/<id>.yaml`, and `wayai push` applies it to a preview
([config-as-code.md](config-as-code.md)). Or manage one with these commands.

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
one file type and folder, and `get_url` and `put_url` tools that answer a short-lived download URL and
a single-use upload URL, for code that moves a file of any size itself. A tool's `put` and `delete` check the file's **version**, not its content
hash, so their rules differ from the mount's — see [File tools](toolsets.md#file-tools). A hub agent
connected to the toolset, and holding both `list_files` and `read_file`, reads these documents at
`data/<file-type>/<path>` with those two tools, and writes them there with `write_file` and `edit_file`
through the toolset's `put`, when it holds those too ([native-tools.md](../agents/native-tools.md#resource--file-tools)).
A toolset's relationship `list` tool answers a file linked to a record as a node with its `path` and
`get_tool`, the toolset's `get` tool whose folder holds it, when the file lies in the folder of one of
its `list` or `get` file tools; a link to any other file is left out
([one hop](toolsets.md#relationship-list-tools-one-hop)).

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
- **A new file is created, never replaced.** Push uploads a file new in your folder as create-only
  (`If-None-Match: *`), so a file created at the same path after your pull — even in the moment
  between push's listing and its upload — is a conflict, not overwritten.
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

Knowledge resources are retired, and `wayai knowledge migrate`, which moved them here, with them.
Add documents to the knowledge base as files: `wayai files push` ([above](#syncing-files-as-a-folder)),
or `wayai files put`.

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

`wayai knowledge cut-over`, which switched a hub's agents from their knowledge resources to the
knowledge base, is retired with them. An agent reads the knowledge base through a toolset of `list`
and `get` file tools ([toolsets.md](toolsets.md#file-tools)) on a
[Base Toolset](../connections.md#base-toolset) connection, holding `list_files` and `read_file`.

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

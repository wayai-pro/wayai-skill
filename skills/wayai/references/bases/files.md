# Files and attachments

Open this before storing PDFs, images, generated reports, or any large/binary content in a base.
Concepts are in [README.md](README.md).

Record and relationship JSON is for structured data and is capped at ~1 MiB — inlining base64 blobs is
rejected. Binary content goes in **Files** (versioned, path-addressed, mountable) or, for the simple
case, **attachments** (record-scoped uploads). Files are the richer evolution; prefer them for
anything you will version, diff, describe with metadata, or mount.

These are a different surface from a hub's knowledge/skill **resources**, which stay hub-local and
sync through hub config-as-code ([`../resources.md`](../resources.md)). The two are deliberately
separate.

## File types

A file type is a bucket that configures storage policy — max size, allowed content types — plus an
optional metadata schema. It is **config**: write it in a preview base, then promote.

- **Promote** adds new file types to production and updates changed ones. A changed policy or
  metadata schema applies to later writes; existing files are not re-checked.
- **Files never move with config.** They are data: a promotion or a new preview carries the file
  type, never its files.
- **Removing a file type through promote** happens only when production holds no live file of it.
  Otherwise the promotion is blocked and `--dry-run` names the file type: recreate it in the preview
  to keep it, or delete its files on production first.
- **Rollback** restores the file types production had before that promotion. A file type the
  promotion added is kept if it has gained files since, and the rollback names it.
- **`create-preview`** copies its origin's file types into the new preview.

Unlike every other config entity, a file type is **not** part of the config-as-code subtree — `pull`
does not write it, `push` does not apply it, `--prune` does not remove it
([config-as-code.md](config-as-code.md)). These commands are the only way to manage one.

```bash
wayai file-types upsert <id> --base <base> --name <name> \
  [--description <desc>] [--max-size <bytes>] [--allowed-content-types <json|@file>] [--metadata-schema <json|@file>]
wayai file-types list --base <base>
wayai file-types get <id> --base <base>
wayai file-types delete <id> --base <base> [-y]
```

## Files

```bash
wayai files put <file_type> <path> --base <base> --file <local> [--content-type <mime>]
wayai files get <file_type> <path> --base <base> [--to <local>] [--meta] [--version <n>]
wayai files list <file_type> --base <base> [--prefix <path>] [--depth <n>] [--limit <n>] [--offset <n>]
wayai files mv <file_type> <from> <to> --base <base>
wayai files rm <file_type> <path> --base <base> [-y]
wayai files history <file_type> <path> --base <base> [--limit <n>] [--offset <n>]
wayai files diff <file_type> <path> --base <base> --from <n> --to <n>
wayai files mount <file_type> --base <base> [--mode <read_only|read_write>] [--path <prefix>] [--expires-in <seconds>]
```

A file is addressed by a **mutable path**; `mv` moves only the address, never the bytes. Paths may
contain folders (`q3/summary.pdf`); `--prefix` and `--depth` navigate them on `list`.

**Content versions.** Every content change is retained as an immutable version. Read an earlier one
with `wayai files get --version <n>` (reads return the latest by default), list them with
`wayai files history`, and compare two with `wayai files diff --from <n> --to <n>` — you get a
metadata delta plus a line-by-line content diff for text files. Renaming or re-tagging a file —
metadata only, no new bytes — does not create a version. A file created at a path where one was
deleted is a new file with its own history, but its version numbers continue past the deleted
file's rather than starting over at 1.

**Labels, metadata, and compare-and-set on upload.** A REST upload (`PUT` of a file's path) takes
three optional request headers:

- `X-Data-Label` — a free-string label for the content version the upload creates.
- `X-Data-Metadata` — the file's structured metadata, as JSON.
- `If-Match` — compare-and-set: the file's content hash, which a download returns as its `ETag`
  (`--meta` shows it as `content_sha256`). The upload is refused with a 409 if the file changed
  since, or no longer exists.

> **Not on the CLI yet.** `wayai files put` does **not** declare `--label`, `--metadata`, or
> `--if-match`: it is an unconditional overwrite that sets no label or metadata. Read them back with
> `--meta` and `diff`; to write them, use the REST upload.

**Compare-and-set on delete.** A REST delete takes the same optional `If-Match`: the file is removed
only if its content hash still matches, and the delete is refused with a 409 otherwise. Without it a
delete is unconditional. (`wayai files rm` does not declare `--if-match` either.)

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
link's relationship type is declared **`cascade`**, deleting the link (or the owning relationship)
removes the file; a non-cascade link is a plain reference, so a file shared across records survives.

**File tools for agents.** A toolset can give its agent `list`, `get`, `put` and `delete` tools over
one file type and folder. A tool's `put` and `delete` check the file's **version**, not its content
hash, so their rules differ from the mount's — see [File tools](toolsets.md#file-tools).

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

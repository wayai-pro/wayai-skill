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
metadata only, no new bytes — does not create a version.

> **Not reachable from the CLI yet: labels, metadata, and compare-and-set on upload.** The storage
> surface takes a version label, structured metadata, and an `if-match` content hash as request
> headers, but the backend proxy forwards a fixed header allowlist and drops those three. `wayai
> files put` therefore does **not** declare `--label`, `--metadata`, or `--if-match`: declaring them
> would promise a compare-and-set that silently does not happen — a concurrent overwrite the user
> asked to be refused would be applied instead, and the label and metadata would vanish with no
> error. Read them back (`--meta`, `diff`) but do not plan a workflow around writing them from the
> CLI today.

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
silently clobbering a concurrent change. Every write is a new version with full history.

**Linking files to records.** A relationship can point at a file — its endpoint kind is `file`. If the
link's relationship type is declared **`cascade`**, deleting the link (or the owning relationship)
removes the file; a non-cascade link is a plain reference, so a file shared across records survives.

**Agents do not get file tools from a toolset.** A toolset composes Actions (record-type operations),
relationship tools, batch and SQL — there is no file or file-type tool among them
([toolsets.md](toolsets.md)). Files are reached by the CLI, the REST surface, or an S3 mount.

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

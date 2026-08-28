# `AGENTS.md` memory files

A workspace folder's `AGENTS.md` is **memory** — the context its config files
can't capture. Two of them, same shape, different owner:

- **Per-hub `wayai-ws/hubs/<hub>/AGENTS.md`** (+ a one-line `CLAUDE.md` that
  imports it) — seeded as a scaffold on `wayai pull` / `wayai push`, and
  **absence-guarded**: a file you already have is never overwritten. See
  [Hub-Folder Memory](../SKILL.md#hub-folder-memory) for what belongs in it.
- **Per-base `wayai-ws/bases/<base>/AGENTS.md`** — the same pattern, yours to
  create. See [Base folders](#base-folders) below.

Neither is ever synced to the platform. A **repo-root** `AGENTS.md` is a
different thing and is not part of this pattern — if one is present it is an
older CLI's bootstrap; see
[Retiring the old root `AGENTS.md`](../SKILL.md#retiring-the-old-root-agentsmd).

## Base folders

A base folder deserves the same memory a hub folder gets, but **the CLI does not
seed it** — `wayai pull bases/<base>` writes only `base.yaml` and the entity
files. When you work in `wayai-ws/bases/<base>/`, create `AGENTS.md` (and a
one-line `CLAUDE.md` holding `@AGENTS.md`) yourself if they are missing, in the
same shape as the per-hub pair.

Use it for what the base's config files can't capture: the base's purpose, key
decisions and *why*, the business rules and terminology behind the schema,
integration quirks. Keep it about *why / what is different* — surface mechanics
live in the skill. Details:
[`references/bases/config-as-code.md`](bases/config-as-code.md).

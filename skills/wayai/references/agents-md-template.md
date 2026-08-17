# `AGENTS.md` templates

The WayAI CLI seeds two kinds of agent-context file, each **absence-guarded** — a
file you already have is never overwritten:

- **Repo-root `AGENTS.md`** (+ a one-line `CLAUDE.md` that imports it) — a session
  *bootstrap*, seeded on `wayai init` / `wayai pull` / `wayai push`. Canonical
  content in the block below.
- **Per-hub `wayai-ws/hubs/<hub>/AGENTS.md`** (+ `CLAUDE.md`) — hub-specific
  *memory*, seeded as a scaffold on pull/push. See
  [Hub-Folder Memory](../SKILL.md#hub-folder-memory) for what belongs in it.

A third file follows the same pattern but is **yours to create**: a base folder's
`AGENTS.md`. See [Base folders](#base-folders) below.

## Keep the root file fresh

The block below is the source of truth for the repo-root `AGENTS.md`. **At the
start of a session, if the repo-root `AGENTS.md` is missing or has drifted from
this block, (re)write it to match.** The CLI seed is a bootstrap, not a frozen
artifact — this reconcile step is what stops it going stale (the failure mode
that got the original auto-write removed).

One skill and one CLI cover everything — hubs, agents, channels **and** bases.
An older repo may still carry a root `AGENTS.md` that installs a second skill and
tells you to load both every session; that is the drift this reconcile removes.
Base work loads [`references/bases/`](bases/README.md) on demand instead.

<!-- BEGIN root-agents-md — drift-guarded byte-for-byte against ROOT_AGENTS_MD in apps/cli/src/lib/workspace-files.ts -->
```text
# Working with WayAI

WayAI is the layer that talks to people across channels, plus the data layer
(Bases) behind it. One CLI, one skill, one login.

## At the start of every session

1. Check for updates first, and update whatever is not current:
   - Skill: `wayai status` → if newer, `mkdir -p .claude && npx skills add wayai-pro/wayai-skill -y`
   - CLI: `wayai update`
2. Load the WayAI skill at the start of every session — every time, even for a
   quick task. It holds the real instructions for everything else.

More detail if you need it: https://www.wayai.pro/docs/get-started

## When helping the person you are working with

- They may be new to WayAI — use plain language, not jargon.
- Don't explain things they didn't ask about. Only if they seem stuck or ask
  what something means, give a one-line answer, then move on — don't turn
  replies into tutorials.
- Keep answers short and to the point.
```
<!-- END root-agents-md -->

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

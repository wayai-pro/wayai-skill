# `AGENTS.md` templates

The WayAI CLI seeds two kinds of agent-context file, each **absence-guarded** — a
file you already have is never overwritten:

- **Repo-root `AGENTS.md`** (+ a one-line `CLAUDE.md` that imports it) — a session
  *bootstrap*, seeded on `wayai init` / `wayai pull` / `wayai push`. Canonical
  content in the block below.
- **Per-hub `wayai-ws/hubs/<hub>/AGENTS.md`** (+ `CLAUDE.md`) — hub-specific
  *memory*, seeded as a scaffold on pull/push. See
  [Hub-Folder Memory](../SKILL.md#hub-folder-memory) for what belongs in it.

## Keep the root file fresh

The block below is the source of truth for the repo-root `AGENTS.md`. **At the
start of a session, if the repo-root `AGENTS.md` is missing or has drifted from
this block, (re)write it to match.** The CLI seed is a bootstrap, not a frozen
artifact — this reconcile step is what stops it going stale (the failure mode
that got the original auto-write removed).

<!-- BEGIN root-agents-md — drift-guarded byte-for-byte against ROOT_AGENTS_MD in apps/cli/src/lib/workspace-files.ts -->
```text
# Working with WayAI and Rekor

WayAI is the layer that talks to people across channels; Rekor is the data
layer (system of record) behind it. They are used together.

## At the start of every session

1. Check for updates first. Make sure both skills and both CLIs are current,
   and update whatever is not:
   - WayAI skill: `wayai status` → if newer, `mkdir -p .claude && npx skills add wayai-pro/wayai-skill -y`
   - Rekor skill: `rekor status` → if newer, `mkdir -p .claude && npx skills add wayai-pro/rekor-skill -y`
   - CLIs: `wayai update` and `rekor update`
2. Load both the WayAI and Rekor skills at the start of every session — every
   time, even for a quick task. They hold the real instructions for everything
   else.

More detail if you need it:
- WayAI: https://www.wayai.pro/docs/get-started
- Rekor: https://rekor.pro/en/docs/get-started

## When helping the person you are working with

- They may be new to WayAI and Rekor — use plain language, not jargon.
- Don't explain things they didn't ask about. Only if they seem stuck or ask
  what something means, give a one-line answer, then move on — don't turn
  replies into tutorials.
- Keep answers short and to the point.
```
<!-- END root-agents-md -->

## Rekor bases

A Rekor *base* folder gets the same treatment from the `rekor` CLI + Rekor skill:
create the base folder's `AGENTS.md` / `CLAUDE.md` when missing, and record the
context the base's settings can't capture. If you're working in a Rekor base and
those files are absent, create them the same way — the Rekor skill covers the
details.

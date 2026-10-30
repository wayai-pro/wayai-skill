# Skills

Skills are versioned agent capability packages attached to agents. A skill's files are stored on the filesystem and synced bidirectionally to the platform via `wayai push` / `wayai pull`.

**Documents live in the knowledge base, not here.** An organization's policies, manuals and price sheets are files in its knowledge base, which agents read and edit at `data/<file type>/<path>` through a [Base Toolset](connections.md#base-toolset) connection ([`bases/files.md`](bases/files.md)). Knowledge resources are retired: a hub or an organization holds skills only.

## Table of Contents
- [Directory Structure](#directory-structure)
- [`skills` Block in `hub.yaml`](#skills-block-in-hubyaml)
- [Agent ↔ Skill Linkage](#agent--skill-linkage)
- [Folders Pulled Before Skills (`resources/`)](#folders-pulled-before-skills-resources)
- [Text vs Binary Files](#text-vs-binary-files)
- [Skill Files](#skill-files)
- [Skill Execution Modes](#skill-execution-modes)
- [Syncing Skills to Providers](#syncing-skills-to-providers)
- [Authoring Skills](#authoring-skills)

---

## Directory Structure

Skill files live under `wayai-ws/hubs/<hub>/skills/`:

```
wayai-ws/hubs/<hub>/skills/
└── order-management/            # One folder per skill, named by its skill_name
    ├── SKILL.md                 # Skill entry point (YAML frontmatter required)
    ├── references/
    │   └── api-docs.md          # Text file — editable, diffable
    └── assets/
        └── flow.png             # Binary file — uploaded as asset
```

Folder name = the skill's `skill_name` (its name, slugified, when it has none), the same `skills/<skill_name>/` path an agent reads it at. Two skills that map to one folder (one's `skill_name` the other's, or its name slugified) are refused by `wayai pull`, `push`, `diff`, `publish`, `create` and `replicate` (and the org commands), naming both, before anything is created: give one a different `skill_name`. Subfolders inside a skill are preserved (relative paths kept on push). Renaming a skill's `skill_name` in `hub.yaml` means renaming its folder too.

A skill directory, every path component leading to it, and every file and subfolder inside it must be **real, not a symlink**, even one pointing elsewhere inside the workspace. If the hub folder or `skills/` itself is a symlink, every `wayai pull` and `wayai push` stops with an error. The same goes for a symlinked skill folder, subfolder, or file inside `skills/`: `wayai push` (and `diff`/`publish`) stops with an error, and so does `wayai pull` into an existing preview hub folder, which reads the folder first to show the diff. Where pull writes without reading first (a production mirror, or a folder it could not read that you agreed to replace), it skips that entry with a warning instead. Either way a pre-planted link can't redirect a write outside the skill folder or send an outside file to the platform.

Organization skills (`wayai org pull` / `push` / `diff`) use the same layout under `wayai-ws/org/`: `skills.yaml` lists them, with `environment`, `tags` and `folders` beside the fields below, and each one's files live under `skills/<skill_name>/`.

---

## `skills` Block in `hub.yaml`

Skills are declared in `hub.yaml` under `skills:`. The list of files is **not** in YAML — the filesystem is the source of truth for a skill's content.

```yaml
skills:
  - id: "skill-uuid"              # set by pull
    name: Order Management
    skill_name: order-management  # auto-derived from name if omitted; names the folder
    description: Handles order queries
    # enabled: true               # default — omitted
```

A skill takes no `type`: an entry naming one other than `skill` (a retired `type: knowledge`, say) is refused.

**Default omission:** `enabled: true` is the default — it's omitted from pulled YAML and from your edits.

**Push is declarative.** `wayai push` sends the hub's whole `skills:` list, an empty one included, so a skill you remove from `hub.yaml` is deleted, the last one too. `wayai org push` does the same with `org/skills.yaml`.

---

## Agent ↔ Skill Linkage

Skills are attached to agents in `agents/<slug>.yaml` under `skills:` (parallel to `tools:`):

```yaml
# agents/sales-agent.yaml
name: Sales Agent
skills:
  - name: Order Management
    skill_id: "skill-uuid"          # set by pull; one the hub does not hold gives way to `name`
    priority: 0
    use_native_integration: false   # see Execution Modes below
```

Higher `priority` means higher precedence when multiple skills apply. Agents never write a skill: they write documents in a base through its Base Toolset's file tools (`write_file`, `edit_file`).

---

## Folders Pulled Before Skills (`resources/`)

A CLI older than this layout wrote skills as `resources:` (in `hub.yaml` and each agent file, linking by `resource_id`) with their files under `resources/<slugified name>/`, and an organization's as `org/resources.yaml` plus `org/resources/<slug>/`. The CLI no longer pushes that layout:

- `wayai push`, `diff` and `publish` refuse a hub folder still holding `resources:` in `hub.yaml` or an agent file, or a `resources/` folder, naming `wayai pull`, before sending anything. `wayai org push` and `org diff` refuse `org/resources.yaml` or `org/resources/` the same way, naming `wayai org pull`.
- `wayai pull` (and `wayai org pull`) converts it: it reads the folder as its skills to list what the pull changes, as for any pull, then writes `skills:` and `skills/<skill_name>/`, the agents' links by `skill_id`, and removes the `resources/<slug>` folders named after the skills it wrote (and `org/resources.yaml`).
- Anything else in `resources/` (a knowledge backup, a folder of a skill renamed since your last pull, a loose file), or anything already in such a folder's `skills/` (which the pull writes from the platform without comparing it), stops the pull before it writes anything, naming it. Move it out of the hub folder (a backup belongs outside the workspace), then pull again. A read-only production mirror is the CLI's own copy: its refresh removes all of its `resources/`.
- An organization folder holding both `skills.yaml` and `resources.yaml` is refused: keep `skills.yaml`.
- A CI job that pushes a hub folder pulls once with the new CLI and commits the result.

This layout needs `@wayai/cli` 0.3.234 or later (`wayai update`). An older CLI does not read a folder in this layout. What it gets from the platform: a `wayai push`, `diff` or `publish` of a folder in the old layout carries `resources`, a `400` naming the minimum CLI; on a folder in this layout it sends no skills at all, so the push applies the rest and leaves every skill and agent skill link as it is. Its `wayai pull` over an existing folder in the old layout gets the same `400` on the comparison it makes first, so it cannot list what the pull changes (depending on its version, it then asks before replacing your local files, or replaces them unasked): update before you pull. Its `wayai org pull`, `push` and `diff` and its `wayai sync-skills` call paths the platform no longer serves (`404`). Update the CLI, then pull once to convert the folder.

---

## Text vs Binary Files

| Class | Examples | Push Behavior | Pull Behavior |
|-------|----------|---------------|---------------|
| Text | `.md`, `.txt`, `.json`, `.yaml`, `.html`, `.css`, `.js`, `.ts` | Inlined for upload; editable by AI agents; diffable in git | Restored as plaintext |
| Binary | `.pdf`, `.png`, `.jpg`, `.gif`, `.mp4`, `.zip`, `.woff2` | Base64-encoded for upload | Downloaded via signed URLs |

**Rules:**
- Detection uses a deny-list of known binary extensions — unknown extensions default to text
- **Size limit:** 10 MB per file. `wayai push`, `diff` and `publish` skip a larger file with a warning. `wayai pull` cannot compare a folder holding one: it asks before replacing your local files, and `pull -y` stops without writing anything. Move the file out of `skills/` or shrink it, then pull again
- **Change detection:** SHA-256 hash comparison — only changed files are uploaded on push

---

## Skill Files

Each skill must contain a `SKILL.md` with YAML frontmatter (`name`, `description` required) at the skill's root.

```
wayai-ws/hubs/<hub>/skills/order-management/
├── SKILL.md              # Required — YAML frontmatter + Markdown body
└── references/           # Optional supporting docs (loaded on demand)
    └── api-docs.md
```

`SKILL.md` frontmatter:

```yaml
---
name: order-management
description: |
  Handle order queries, status checks, and cancellations. Use when the user
  mentions an order ID or asks about delivery / refunds / cancellation.
---

# Order Management

[skill body — workflows, examples, references to ./references/*.md]
```

The `description` is the trigger mechanism — it determines whether the agent invokes the skill. Make it clear about both **what the skill does** and **when to use it**.

**Where the agent reads it.** An agent reads a skill's files at `skills/<skill_name>/…` — `skills/order-management/references/api-docs.md` above, the same path as in the workspace folder — with `list_files`, `read_file` and `{{file_content(skills/…)}}`. Text agents act on (agents' instructions, additional context, settings' system message and monitor notes; experiment-arm overrides of those; automations' instructions; kanban follow-ups' and statuses' instructions; custom tools' instructions and descriptions; and skill and folder descriptions) that cites a skill under `resources/` is refused on save and on `wayai push`, naming the `skills/` path to write ([native-tools.md](agents/native-tools.md)). An organization skill's descriptions are copied into each hub that links it with such a citation written as that hub's `skills/` path.

---

## Skill Execution Modes

Configured per agent-skill link via `use_native_integration`:

### Tool-based (default)

- `use_native_integration: false`
- Skill content is injected as a tool the agent can call
- Works with **all LLM providers** (OpenAI, Anthropic, Google, OpenRouter, etc.)
- Faster execution — no container overhead
- Best for most use cases

### Native integration (Anthropic, OpenAI)

- `use_native_integration: true`
- Skill is uploaded to the agent's provider and loaded inside a managed container:
  - **Anthropic** — `container.skills` + `code_execution_20250825` tool
  - **OpenAI** — `shell` tool with `container_auto` environment + `skill_reference` entries
- Requires an **Anthropic or OpenAI connection** on the hub
- Requires **syncing** skills to the provider (auto-sync on push, or manual)
- Best for skills that need sandboxed code execution; pay for container cold-start latency and a tool-call op per turn

### When to use which

| Scenario | Mode |
|----------|------|
| General agent capability | Tool-based (default) |
| Skill must work across non-skill-capable providers (Google, OpenRouter, xAI) | Tool-based |
| Sandboxed code execution required | Native integration |
| Hub uses Anthropic and/or OpenAI agents and wants native skill features | Native integration |

---

## Syncing Skills to Providers

When a skill is created and the hub has an Anthropic or OpenAI connection, skills are **automatically synced** (uploaded) to those providers on `wayai push`. Manual re-sync is needed when:
- The initial auto-sync failed (network error, rate limit)
- An Anthropic or OpenAI connection was added **after** the skill was created
- Skills were updated and need to be re-uploaded

Trigger a manual re-sync:

```bash
wayai sync-skills                              # all skills, all skill-capable connections (Anthropic, OpenAI)
wayai sync-skills --connection-id <uuid>       # scope to one connection
```

---

## Authoring Skills

1. Create the skill folder under `wayai-ws/hubs/<hub>/skills/<skill-name>/`
2. Write `SKILL.md` with YAML frontmatter (`name`, `description`)
3. Add `references/*.md` for deep-dive content the skill body points to
4. Declare the skill in `hub.yaml` under `skills:`, with `skill_name: <skill-name>` (or a name that slugifies to it)
5. Link to one or more agents in `agents/<slug>.yaml` under `skills:`
6. `wayai push` — uploads the skill, syncs to Anthropic and/or OpenAI if those connections exist
7. (Optional) `wayai sync-skills` if auto-sync didn't run or you added an Anthropic/OpenAI connection later

Treat skill content like the conceptual layer of a feature — the entry `SKILL.md` describes *what* and *when*, with body short enough to fit always-on, and references handle *deep how*.

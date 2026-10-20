# Resources

Resources are skills: versioned agent capability packages attached to agents. Resource files are stored on the filesystem and synced bidirectionally to the platform via `wayai push` / `wayai pull`.

**Documents live in the knowledge base, not here.** An organization's policies, manuals and price sheets are files in its knowledge base, which agents read and edit at `data/<file type>/<path>` through a [Base Toolset](connections.md#base-toolset) connection ([`bases/files.md`](bases/files.md)). Knowledge resources are retired: a hub or an organization holds skills only.

## Table of Contents
- [Resource Types](#resource-types)
- [Directory Structure](#directory-structure)
- [`resources` Block in `hub.yaml`](#resources-block-in-hubyaml)
- [Agent ↔ Resource Linkage](#agent--resource-linkage)
- [Text vs Binary Files](#text-vs-binary-files)
- [Skill Resources](#skill-resources)
- [Skill Execution Modes](#skill-execution-modes)
- [Syncing Skills to Providers](#syncing-skills-to-providers)
- [Authoring Skills](#authoring-skills)

---

## Resource Types

| Type | Purpose | Entry Point | Execution |
|------|---------|-------------|-----------|
| `skill` | Versioned agent capability package | `SKILL.md` with YAML frontmatter | Tool-based or native integration |

Every resource is a skill, so `type` can be left out. `wayai push` (and `wayai org push`) refuses an entry still saying `type: knowledge`, from a folder kept since knowledge was retired, naming the knowledge base, rather than push its documents as a skill.

---

## Directory Structure

Resource files live under `wayai-ws/hubs/<hub>/resources/`:

```
wayai-ws/hubs/<hub>/resources/
└── order-management/            # Skill resource (slugified name)
    ├── SKILL.md                 # Skill entry point (YAML frontmatter required)
    ├── references/
    │   └── api-docs.md          # Text file — editable, diffable
    └── assets/
        └── flow.png             # Binary file — uploaded as asset
```

Folder name = slugified resource name. Subfolders inside a resource are preserved (relative paths kept on push).

A resource directory, every path component leading to it, and every file and subfolder inside it must be **real, not a symlink**, even one pointing elsewhere inside the workspace. If the hub folder or `resources/` itself is a symlink, every `wayai pull` and `wayai push` stops with an error. The same goes for a symlinked resource folder, subfolder, or file inside `resources/`: `wayai push` (and `diff`/`publish`) stops with an error, and so does `wayai pull` into an existing preview hub folder, which reads the folder first to show the diff. Where pull writes without reading first (a production mirror, or a folder it could not read that you agreed to replace), it skips that entry with a warning instead. Either way a pre-planted link can't redirect a write outside the resource folder or send an outside file to the platform.

---

## `resources` Block in `hub.yaml`

Resources are declared in `hub.yaml` under `resources:`. The list of files is **not** in YAML — the filesystem is the source of truth for resource content.

```yaml
resources:
  - id: "skill-uuid"              # set by pull
    name: Order Management
    type: skill                   # written by pull; may be left out
    skill_name: order-management  # auto-derived from name if omitted
    description: Handles order queries
    # enabled: true               # default — omitted
```

**Default omission:** `enabled: true` is the default — it's omitted from pulled YAML and from your edits.

---

## Agent ↔ Resource Linkage

Resources are attached to agents in `agents/<slug>.yaml` under `resources:` (parallel to `tools:`):

```yaml
# agents/sales-agent.yaml
name: Sales Agent
resources:
  - name: Order Management
    resource_id: "skill-uuid"
    priority: 0
    use_native_integration: false   # see Execution Modes below
```

Higher `priority` means higher precedence when multiple resources apply. A link's `write_enabled` is still accepted but does nothing: agents never write a skill. They write documents in a base through its Base Toolset's file tools (`write_file`, `edit_file`).

---

## Text vs Binary Files

| Class | Examples | Push Behavior | Pull Behavior |
|-------|----------|---------------|---------------|
| Text | `.md`, `.txt`, `.json`, `.yaml`, `.html`, `.css`, `.js`, `.ts` | Inlined for upload; editable by AI agents; diffable in git | Restored as plaintext |
| Binary | `.pdf`, `.png`, `.jpg`, `.gif`, `.mp4`, `.zip`, `.woff2` | Base64-encoded for upload | Downloaded via signed URLs |

**Rules:**
- Detection uses a deny-list of known binary extensions — unknown extensions default to text
- **Size limit:** 10 MB per file. `wayai push`, `diff` and `publish` skip a larger file with a warning. `wayai pull` cannot compare a folder holding one: it asks before replacing your local files, and `pull -y` stops without writing anything. Move the file out of `resources/` or shrink it, then pull again
- **Change detection:** SHA-256 hash comparison — only changed files are uploaded on push

---

## Skill Resources

Skills are versioned agent capability packages. Each skill must contain a `SKILL.md` with YAML frontmatter (`name`, `description` required) at the resource root.

```
wayai-ws/hubs/<hub>/resources/order-management/
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

**Where the agent reads it.** An agent reads a skill's files at `skills/<skill_name>/…` — `skills/order-management/references/api-docs.md` above — with `list_files`, `read_file` and `{{file_content(skills/…)}}`. The workspace folder stays `resources/<slugified-name>/` for now. Text agents act on (agents' instructions, additional context, settings' system message and monitor notes; experiment-arm overrides of those; automations' instructions; kanban follow-ups' and statuses' instructions; custom tools' instructions and descriptions; and resource and folder descriptions) that cites a skill under `resources/` is refused on save and on `wayai push`, naming the `skills/` path to write ([native-tools.md](agents/native-tools.md)). An organization resource's descriptions are copied into each hub that links it with such a citation written as that hub's `skills/` path.

---

## Skill Execution Modes

Configured per agent-resource link via `use_native_integration`:

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
- An Anthropic or OpenAI connection was added **after** the skill resource was created
- Skills were updated and need to be re-uploaded

Trigger a manual re-sync:

```bash
wayai sync-skills                              # all skills, all skill-capable connections (Anthropic, OpenAI)
wayai sync-skills --connection-id <uuid>       # scope to one connection
```

---

## Authoring Skills

1. Create the skill folder under `wayai-ws/hubs/<hub>/resources/<skill-name>/`
2. Write `SKILL.md` with YAML frontmatter (`name`, `description`)
3. Add `references/*.md` for deep-dive content the skill body points to
4. Declare the resource in `hub.yaml` with `type: skill`
5. Link to one or more agents in `agents/<slug>.yaml` under `resources:`
6. `wayai push` — uploads the skill, syncs to Anthropic and/or OpenAI if those connections exist
7. (Optional) `wayai sync-skills` if auto-sync didn't run or you added an Anthropic/OpenAI connection later

Treat skill content like the conceptual layer of a feature — the entry `SKILL.md` describes *what* and *when*, with body short enough to fit always-on, and references handle *deep how*.

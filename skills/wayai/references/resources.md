# Resources

Resources are knowledge bases (documents) or skills (versioned agent capability packages) attached to agents. Resource files are stored on the filesystem and synced bidirectionally to the platform via `wayai push` / `wayai pull`.

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
| `knowledge` (default) | Document collections (FAQ, product docs, manuals) | Any file(s) | Always injected as context |
| `skill` | Versioned agent capability package | `SKILL.md` with YAML frontmatter | Tool-based or native integration |

`knowledge` is the default — `type` can be omitted in YAML when the resource is a knowledge base.

---

## Directory Structure

Resource files live under `workspace/<hub>/resources/`:

```
workspace/<hub>/resources/
├── product-catalog/             # Knowledge resource (slugified name)
│   ├── pricing.md               # Text file — editable, diffable
│   ├── catalog.pdf              # Binary file — uploaded as asset
│   └── images/
│       └── logo.png
└── order-management/            # Skill resource
    ├── SKILL.md                 # Skill entry point (YAML frontmatter required)
    └── references/
        └── api-docs.md
```

Folder name = slugified resource name. Subfolders inside a resource are preserved (relative paths kept on push).

---

## `resources` Block in `hub.yaml`

Resources are declared in `hub.yaml` under `resources:`. The list of files is **not** in YAML — the filesystem is the source of truth for resource content.

```yaml
resources:
  - id: "resource-uuid"           # set by pull
    name: Product Catalog
    # type: knowledge             # default — omitted
    description: Product docs
    # enabled: true               # default — omitted
    # user_browsable: false       # default — omitted

  - id: "skill-uuid"
    name: Order Management
    type: skill
    skill_name: order-management  # auto-derived from name if omitted
    description: Handles order queries
```

**Default omission:** `enabled: true`, `user_browsable: false`, and `type: knowledge` are defaults — they're omitted from pulled YAML and from your edits.

---

## Agent ↔ Resource Linkage

Resources are attached to agents in `agents/<slug>.yaml` under `resources:` (parallel to `tools:`):

```yaml
# agents/sales-agent.yaml
name: Sales Agent
resources:
  - name: Product Catalog
    resource_id: "resource-uuid"
    priority: 0

  - name: Order Management
    resource_id: "skill-uuid"
    priority: 1
    use_native_integration: false   # skill-only — see Execution Modes below
```

Higher `priority` means higher precedence when multiple resources apply.

---

## Text vs Binary Files

| Class | Examples | Push Behavior | Pull Behavior |
|-------|----------|---------------|---------------|
| Text | `.md`, `.txt`, `.json`, `.yaml`, `.html`, `.css`, `.js`, `.ts` | Inlined for upload; editable by AI agents; diffable in git | Restored as plaintext |
| Binary | `.pdf`, `.png`, `.jpg`, `.gif`, `.mp4`, `.zip`, `.woff2` | Base64-encoded for upload | Downloaded via signed URLs |

**Rules:**
- Detection uses a deny-list of known binary extensions — unknown extensions default to text
- **Size limit:** 10 MB per file. Files exceeding this are skipped with a warning
- **Change detection:** SHA-256 hash comparison — only changed files are uploaded on push

---

## Skill Resources

Skills are versioned agent capability packages. Each skill must contain a `SKILL.md` with YAML frontmatter (`name`, `description` required) at the resource root.

```
workspace/<hub>/resources/order-management/
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
| Skill must work across non-skill-capable providers (Google, OpenRouter) | Tool-based |
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

1. Create the skill folder under `workspace/<hub>/resources/<skill-name>/`
2. Write `SKILL.md` with YAML frontmatter (`name`, `description`)
3. Add `references/*.md` for deep-dive content the skill body points to
4. Declare the resource in `hub.yaml` with `type: skill`
5. Link to one or more agents in `agents/<slug>.yaml` under `resources:`
6. `wayai push` — uploads the skill, syncs to Anthropic and/or OpenAI if those connections exist
7. (Optional) `wayai sync-skills` if auto-sync didn't run or you added an Anthropic/OpenAI connection later

Treat skill content like the conceptual layer of a feature — the entry `SKILL.md` describes *what* and *when*, with body short enough to fit always-on, and references handle *deep how*.

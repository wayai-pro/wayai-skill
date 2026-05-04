# Hub Templates

Ready-made hub configurations and agent instruction packs. Copy a template into `workspace/<hub>/`, replace placeholders, then `wayai push` to create the hub.

## Table of Contents
- [Available Templates](#available-templates)
- [Workflow](#workflow)
- [Template Types](#template-types)
- [Schema Version](#schema-version)
- [Directory Structure](#directory-structure)
- [Hub File Format (`hub.md`)](#hub-file-format-hubmd)
- [Agent Instructions Format](#agent-instructions-format)
- [Placeholders](#placeholders)
- [Customization Markers](#customization-markers)

---

## Available Templates

Pick the language that matches the user — placeholders and prose differ between `en/` and `pt/`, but YAML structure (connector IDs, tool IDs, roles) is identical.

### English (`en/`)

| Template | Type | Description | Hub | Agent Instructions |
|----------|------|-------------|-----|-------------------|
| Pizzeria - Orders | vertical | Order intake via WhatsApp | `assets/templates/en/vertical/pizzeria/orders/hub.md` | `assets/templates/en/vertical/pizzeria/orders/server-instructions.md` |
| Dental - Scheduling | vertical | Dental appointment scheduling | `assets/templates/en/vertical/dental/scheduling/hub.md` | `assets/templates/en/vertical/dental/scheduling/receptionist-instructions.md` |
| Swimming - Customer Service | vertical | Visit booking for swimming academies | `assets/templates/en/vertical/swimming/customer-service/hub.md` | `assets/templates/en/vertical/swimming/customer-service/agent-instructions.md` |
| SDR - Simple | horizontal | Inbound lead qualification | `assets/templates/en/horizontal/sdr/simple/hub.md` | `assets/templates/en/horizontal/sdr/simple/sdr-instructions.md` |

### Portuguese (`pt/`)

| Template | Type | Description | Hub | Agent Instructions |
|----------|------|-------------|-----|-------------------|
| Pizzaria - Pedidos | vertical | Atendimento de pedidos via WhatsApp | `assets/templates/pt/vertical/pizzaria/pedidos/hub.md` | `assets/templates/pt/vertical/pizzaria/pedidos/atendente-instructions.md` |
| Odonto - Agendamento | vertical | Agendamento de consultas odontológicas | `assets/templates/pt/vertical/odonto/agendamento/hub.md` | `assets/templates/pt/vertical/odonto/agendamento/recepcionista-instructions.md` |
| Natação - Atendimento | vertical | Agendamento de visitas em academias | `assets/templates/pt/vertical/natacao/atendimento/hub.md` | `assets/templates/pt/vertical/natacao/atendimento/atendente-instructions.md` |
| SDR - Simples | horizontal | Qualificação de leads inbound | `assets/templates/pt/horizontal/sdr/simples/hub.md` | `assets/templates/pt/horizontal/sdr/simples/sdr-instructions.md` |

Templates ship inside this skill at `assets/templates/`.

---

## Workflow

1. Find the matching template in the table above
2. Read the template files using the paths shown
3. Copy the `hub.md` and instruction `.md` files into `workspace/<hub>/`
4. Replace `{NOME_EMPRESA}`, `{NOME_CLINICA}`, etc. (see [Placeholders](#placeholders))
5. Customize `{CUSTOMIZE: ...}` sections in the instruction files (see [Customization Markers](#customization-markers))
6. Add the connections the template requires to `hub.yaml` (auto-created from org credentials during push)
7. `wayai push` — creates agents, tools, states, and connections on the hub

---

## Template Types

- **`vertical`** — Industry-specific (pizzaria, odonto, natação, etc.)
- **`horizontal`** — Cross-industry functions (SDR, support, etc.)

---

## Schema Version

**Current schema:** 2.1

**ID Reference Notes:**
- `connector_id` — production UUIDs identifying connector types (OpenAI, WhatsApp, etc.). These are fixed IDs from the WayAI database, not placeholders.
- `tool_native_id` — production UUIDs identifying specific native tools. Also fixed IDs from the database.

These IDs are baked into the templates because the template system pre-dates name-based connector resolution. When using templates, treat them as opaque — don't edit.

---

## Directory Structure

Each template folder contains a hub config and agent instruction files (workspace format):

```
{lang}/{type}/{category}/{variant}/
├── hub.md                           # Hub config + agents config in YAML frontmatter
└── {agent-slug}-instructions.md     # Agent instructions only
```

**Example:** `pt/vertical/pizzaria/pedidos/`

```
├── hub.md                           # Hub settings + agent config (model, tools)
└── atendente-instructions.md        # Agent instructions
```

The structure is intentionally flat (no nested folders) — matches `download_workspace()` output, makes diff/review easy.

---

## Hub File Format (`hub.md`)

YAML frontmatter with hub settings and agents config, plus a Markdown body for human reference.

### Frontmatter Fields

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Hub name (use `{NOME_EMPRESA}` placeholder) |
| `description` | string | Brief description |
| `ai_mode` | string | `Pilot+Copilot`, `Pilot`, `Copilot`, `Turned Off` |
| `hub_type` | string | `chat` or `task` |
| `kanban_statuses` | array | Workflow stages with followup config |
| `tags` | array of slug names | Org tags attached to the hub. Names must already exist in the org (Settings → Organization → Tags). Gates which org credentials this hub can resolve — see [connections.md](connections.md#organization-tags) |
| `agents` | array | Agent configurations (see below) |
| `connections` | array | Required connections (referenced by name + connector_id) |

### Agent Configuration (in `hub.md`)

Each agent in the `agents` array:

| Field | Type | Description |
|-------|------|-------------|
| `agent_name` | string | Agent display name |
| `agent_role` | string | `Pilot`, `Copilot`, `Specialist for Pilot`, etc. |
| `model` | string | LLM model (e.g., `gpt-5.4-mini`) |
| `instructions_file` | string | Path to instructions file (`{agent-slug}-instructions.md`) |
| `tools.native` | array | Native tools grouped by connector |
| `tools.custom` | array | Custom tool definitions |

### Example

```yaml
---
name: "{NOME_EMPRESA}"
description: "Atendimento de pedidos para pizzaria"
ai_mode: Pilot+Copilot
hub_type: chat
connections:
  - connector_name: WhatsApp
    connector_id: "uuid-from-database"
    connector_type: Channel
agents:
  - agent_name: "Atendente"
    agent_role: Pilot
    model: gpt-5.4-mini
    instructions_file: atendente-instructions.md
    tools:
      native:
        - connector_name: Wayai Conversation
          connector_id: "uuid-from-database"
          tools:
            - tool_name: Close Conversation
              tool_native_id: "uuid-from-database"
---

# Pizzaria - Pedidos

Template para atendimento de pedidos de pizzaria via WhatsApp.

## Casos de Uso
...

## Conexões Necessárias
| Conexão | Tipo | Finalidade |
|---------|------|------------|
| WhatsApp Business | whatsapp | Atendimento ao cliente |
| OpenAI ou OpenRouter | agent | LLM para os agentes |

## Agents
| Agent | Role | Instructions |
|-------|------|--------------|
| Atendente | Pilot | `atendente-instructions.md` |

## Checklist de Customização
- [ ] Substituir `{NOME_EMPRESA}` pelo nome da pizzaria
- [ ] Adicionar cardápio completo com preços
```

---

## Agent Instructions Format

Located at `{agent-slug}-instructions.md` (e.g., `atendente-instructions.md`). Contains minimal frontmatter + the full agent prompt.

### Frontmatter

```yaml
---
agent_name: "Atendente"
---
```

### Body Structure

```markdown
---
agent_name: "Atendente"
---

# Instructions

[Opening statement with {NOME_EMPRESA} placeholder]

## Seus Objetivos
1. Objective 1
2. Objective 2

## {CUSTOMIZE: Section Name}
[Content requiring customization]

## Fluxo de Atendimento
1. Step 1
2. Step 2

## Tom de Voz
- Style guideline 1
- Style guideline 2

## Exemplo de Conversa
[Example dialogue]
```

### Common Body Sections

- **Seus Objetivos** — agent objectives
- **{CUSTOMIZE: Section}** — customizable content sections
- **Fluxo de Atendimento** — workflow steps
- **Tom de Voz** — tone and style guidelines
- **Exemplo de Conversa** — example conversation

---

## Placeholders

Replace these when customizing templates:

| Placeholder (PT) | Placeholder (EN) | Description | Example |
|-------------|-------------|-------------|---------|
| `{NOME_EMPRESA}` | `{COMPANY_NAME}` | Company / business name | "Pizzaria do João" / "John's Pizza" |
| `{NOME_CLINICA}` | `{CLINIC_NAME}` | Clinic name | "Clínica OdontoSorriso" / "Smile Dental" |
| `{NOME_ACADEMIA}` | `{ACADEMY_NAME}` | Academy name | "Academia AquaFit" / "AquaFit Academy" |
| `{NOME_PRODUTO}` | `{PRODUCT_NAME}` | Product name | "CRM Pro" |
| `{ENDERECO}` | `{ADDRESS}` | Street address | "Rua X, 123" / "123 Main St" |
| `R$ XX` / `$XX` | `$XX` | Price placeholder | "R$ 45,00" / "$45" |
| `[lista]` / `[list]` | `[list]` | List to be filled | Neighborhoods, services |

---

## Customization Markers

Sections marked `{CUSTOMIZE: ...}` require business-specific content. Examples:

- `{CUSTOMIZE: Cardápio}` — Menu and prices
- `{CUSTOMIZE: Informações de Entrega}` — Delivery info
- `{CUSTOMIZE: Serviços}` — Available services
- `{CUSTOMIZE: Horários}` — Business hours
- `{CUSTOMIZE: Modalidades e Horários}` — Class schedules
- `{CUSTOMIZE: Planos e Preços}` — Plans and pricing

When a template ships with `{CUSTOMIZE: X}` sections, the user-supplied content for that section replaces the marker entirely (including the marker line itself).

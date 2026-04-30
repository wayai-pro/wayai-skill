# Outbound

Outbound entities enable proactive messaging from a hub — sending a message before the user does, on a schedule, to a curated list of contacts. Defined in `hub.yaml` and managed via `wayai push`.

## Table of Contents
- [Outbound Contacts](#outbound-contacts)
- [Outbound Lists](#outbound-lists)
- [Outbound Schedules](#outbound-schedules)
- [Execution Modes](#execution-modes)
- [Channel Rules](#channel-rules)
- [Practical Limits](#practical-limits)

---

## Outbound Contacts

Hub-scoped contacts. At least one channel identifier (`phone`, `email`, or `instagram_sid`) is required.

```yaml
# hub.yaml
outbound_contacts:
  - name: "John Doe"
    phone: "+5511999999999"       # E.164 format (WhatsApp / SMS)
    email: "john@example.com"     # Email (Resend)
    instagram_sid: "123456789"    # Instagram Scoped ID
    tags: ["vip", "active"]
    enabled: true                 # default: true
```

Tags are free-form strings used to filter contacts when assembling lists or for downstream segmentation.

---

## Outbound Lists

Named collections of contacts. Contacts are referenced by name (slug-stable):

```yaml
outbound_lists:
  - name: "VIP Customers"
    description: "High-value customers for weekly check-ins"
    contacts:
      - "John Doe"
      - "Jane Smith"
```

Lists are static — they enumerate specific contacts. There's no dynamic tag-based query at the YAML layer; build that filtering in the agent or upstream automation.

---

## Outbound Schedules

Define when and how to message a list. The schedule fires at the cron expression in the configured timezone.

```yaml
outbound_schedules:
  - name: "Weekly Check-in"
    cron_expression: "0 9 * * 1"        # every Monday at 9 AM
    timezone: "America/Sao_Paulo"
    list: "VIP Customers"                # name of an outbound_lists entry
    channel_type: app                    # app | whatsapp | instagram | email
    execution_mode: agent_trigger        # direct_message | agent_trigger
    agent: "Support Agent"               # required when execution_mode = agent_trigger
    trigger_message: "Start a weekly check-in with {{user_name}}."
```

`cron_expression` uses standard 5-field cron syntax (minute hour day-of-month month day-of-week). All times are interpreted in the schedule's `timezone`.

---

## Execution Modes

| Mode | Behavior | Use Case |
|------|----------|----------|
| `direct_message` | Sends a template (WhatsApp) or free text (app/email) directly to each contact | Marketing blast, transactional template, simple announcement |
| `agent_trigger` | Inserts a system message into a (new or existing) conversation that triggers the agent to respond. The agent runs naturally — uses tools, reads state, follows instructions | Personalized check-ins, contextual nudges, conversational follow-ups |

For `agent_trigger`, the `trigger_message` is what the agent sees as a system instruction (not what the user sees). Placeholder support follows agent instruction conventions — see [agents/instructions.md](agents/instructions.md).

---

## Channel Rules

Different channels have different delivery constraints, primarily around messaging windows for OTT messaging providers (WhatsApp, Instagram require an active conversation window for free-form text).

| Channel | Direct Message | Agent Trigger | Requires 24h Window |
|---------|---------------|---------------|---------------------|
| App | Free text | System msg → agent | No |
| WhatsApp | Approved template | Agent (if 24h open); approved template fallback | Yes |
| Instagram | Free text (if 24h open) | Agent (if 24h open); skip if window closed | Yes |
| Email | Email template | System msg → agent | No |

For WhatsApp and Instagram, the platform automatically falls back to the appropriate delivery method when the 24-hour customer service window has closed. For WhatsApp, that means an approved message template. For Instagram, the message is skipped (no template equivalent on Instagram outbound).

---

## Practical Limits

Contacts declared inline in `hub.yaml` are practical up to **~500**. Beyond that:
- YAML diff/round-trip becomes slow and noisy
- Lists with hundreds of contacts make `wayai push` review unwieldy

For larger lists, use CSV import via the platform UI or the API directly. Schedules and lists themselves stay in `hub.yaml` — only the contact rows move out.

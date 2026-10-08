# Automations

An automation starts work in a hub on its own: on a schedule, for every contact on one of the hub's contact lists, it sends a message or runs an agent. Automations live in `hub.yaml` under `automations:` and are managed via `wayai push`; they are switched on from the app. The contacts and lists they target are the `outbound_contacts:` / `outbound_lists:` blocks ([Contacts & Lists](#contacts--lists)).

## Table of Contents
- [Shape](#shape)
- [What a Hub Can Save](#what-a-hub-can-save)
- [Examples](#examples)
- [What Each Fire Does](#what-each-fire-does)
- [References](#references)
- [Pushing `automations:`](#pushing-automations)
- [Enabling and Run Now](#enabling-and-run-now)
- [Message Templates](#message-templates)
- [Limits](#limits)
- [Contacts & Lists](#contacts--lists)

---

## Shape

One automation is a **trigger** (when it fires), a **target** (who each fire is for) and an **action** (what it does for each target).

```yaml
# hub.yaml
automations:
  - id: "automation-uuid"            # set by pull; never author it
    name: "Weekly Check-in"          # unique per hub
    description: "Monday check-in with VIP customers"   # optional
    enabled: false                   # stored as written; see "Enabling and Run Now"
    trigger:
      type: schedule
      cron: "0 9 * * 1"              # 5-field cron: minute hour day-of-month month day-of-week
      timezone: "America/Sao_Paulo"  # IANA; default UTC
    target:
      type: contact_list
      list: "VIP Customers"          # an outbound_lists entry, by name (or list_id)
    action:
      type: run_agent                # run_agent | send_message
      channel: whatsapp              # whatsapp | instagram | email | app
      agent: "Support Agent"         # run_agent only (or agent_id)
      instructions: "Check in on the customer's last order."  # run_agent only
      # connection / connection_id   — send_message on whatsapp, run_agent on email
      # template / template_id       — send_message on whatsapp
      # text                         — send_message on app
```

Every key not shown above is refused, never ignored — at the entry level and inside `trigger`, `target` and `action`.

---

## What a Hub Can Save

Every save — a push, the app, a hub-type change, a publish — is validated, and each refusal names the automation it is about.

| Hub type | Target | Action | Channel | Fields the action takes |
|----------|--------|--------|---------|-------------------------|
| `chat` | `contact_list` | `send_message` | `whatsapp` | `connection` + `template` (a WhatsApp template of that connection) |
| `chat` | `contact_list` | `send_message` | `app` | `text` |
| `chat` or `task` | `contact_list` | `run_agent` | `whatsapp`, `instagram`, `app` | `agent` + `instructions` |
| `chat` or `task` | `contact_list` | `run_agent` | `email` | `agent` + `instructions` + `connection` (required) |

- **Trigger:** `schedule` only, with a valid 5-field `cron` and an IANA `timezone` (default `UTC`).
- **Target:** `contact_list` only, naming one of the hub's own lists.
- **A field the chosen action does not take is refused** — e.g. `text` on a WhatsApp send, or `connection` on a `run_agent` that is not on email.
- **Task hubs run agents only:** `send_message` is refused there. A hub-type change that would make a stored automation invalid is refused, naming it.
- **Not available yet, refused at save:** `event` and `webhook` triggers, the `none` target, gates, and `send_message` on `email` or `instagram`.

---

## Examples

**WhatsApp template send** (chat hub):

```yaml
automations:
  - name: "Appointment Reminder"
    trigger: { type: schedule, cron: "0 8 * * *", timezone: "America/Sao_Paulo" }
    target: { type: contact_list, list: "Tomorrow's Appointments" }
    action:
      type: send_message
      channel: whatsapp
      connection: "WhatsApp Main"        # the WhatsApp connection's display name
      template: "appointment_reminder"   # a template of that connection
```

**App text** (chat hub):

```yaml
automations:
  - name: "Monthly Announcement"
    trigger: { type: schedule, cron: "0 10 1 * *" }    # timezone defaults to UTC
    target: { type: contact_list, list: "Members" }
    action:
      type: send_message
      channel: app
      text: "Our new opening hours start next week."
```

This saves and fires, but reaches no one yet — see [What Each Fire Does](#what-each-fire-does).

**Run an agent by email** (chat or task hub):

```yaml
automations:
  - name: "Renewal Follow-up"
    trigger: { type: schedule, cron: "30 9 * * 1-5", timezone: "Europe/Lisbon" }
    target: { type: contact_list, list: "Renewals Due" }
    action:
      type: run_agent
      channel: email
      connection: "Support Email"        # the email connection the conversation runs on
      agent: "Renewals Agent"
      instructions: "Remind the customer their plan renews this month and offer to answer questions."
```

---

## What Each Fire Does

A fire runs once per **enabled** contact on the list. A contact without the channel's identifier — `phone` for WhatsApp, `instagram_sid` for Instagram, `email` for email — is skipped. Each fire records a run with its totals (targets, succeeded, failed, skipped) in the automation's run history.

| Action | Channel | Per contact |
|--------|---------|-------------|
| `send_message` | `whatsapp` | Sends the template to the contact's `phone` and records it in the contact's WhatsApp conversation |
| `send_message` | `app` | Skipped: a list's contacts are not linked to app users yet, so there is no app conversation to post into |
| `run_agent` | `email` | The agent receives `instructions` as a system message (the contact never sees it) in the contact's email conversation and replies by email. Needs an enabled email channel on the action's connection, with a verified sending domain |
| `run_agent` | `whatsapp`, `instagram` | The agent takes a turn on `instructions` in a system conversation that is not addressed to the contact, so its reply does not reach them yet |
| `run_agent` | `app` | Skipped, as for `send_message` on `app` |

- `agent` must name one of the hub's agents at save; a fire does not yet use it to choose which agent takes the turn.
- A `send_message` fire checks the org's operations quota first. When the org is over its quota, every contact is skipped and the skipped fire is not sent again: the automation's next fire is its first scheduled one after the quota resets, or after a day if that is sooner.

---

## References

`list`, `connection`, `template` and `agent` take a **name**; each has an id twin (`list_id`, `connection_id`, `template_id`, `agent_id`). Give either or both — **the id wins** when both are present. `wayai pull` writes both.

- `connection` is the connection's display name; `template` is matched among the templates of the action's connection. A template name that matches more than one template is refused — name it by `template_id`.
- A reference to an agent or list the same push creates or renames resolves on that push, by the name the push gives it. A push whose agents are refused writes none of them, so an automation naming one of them is refused too.
- A reference to anything the hub does not hold is refused, naming the automation.

Automations are matched to the hub's stored ones by `id`, else by `name` (a stored automation another entry names by `id` is that entry's): renaming an entry that carries its `id` renames the automation, while renaming one without it creates a new automation and deletes the old.

---

## Pushing `automations:`

| `hub.yaml` | Push result |
|------------|-------------|
| No `automations:` key (or a key with no value) | **Nothing changes** — the hub keeps its automations |
| `automations: []` | **Deletes every automation** |
| A list of entries | The hub's automations become exactly that list: missing entries are deleted, new ones created, changed ones updated |

While the key is absent, the stored automations still protect what they use — a connection an automation sends through is never deleted as unreferenced. A refused entry is reported, naming the automation, and is not written — its stored automation, if any, stays as it was, and an entry that would take its name is refused too; the rest of the push still applies.

The old `outbound_schedules:` key is no longer read — the CLI warns and pushes none of its entries. Rewrite each schedule as an automation.

---

## Enabling and Run Now

- **An automation is created disabled.** Enabling it — the toggle in the hub's **Automations** tab — arms its schedule and shows the next run; disabling it cancels the pending fire.
- **`enabled:` in `hub.yaml` is stored as written but arms nothing**, and does not cancel a fire already scheduled. Arm and disarm with the toggle; an automation pushed with `enabled: true` reads as enabled with no next run until it is toggled off and on.
- **Run now** fires the automation once, immediately, whether or not it is enabled. It is refused while a run of the same automation is still in progress.
- A preview hub's automation delivers for real once enabled — to the contacts on that hub's lists.
- Publishing copies automations to production as written; it arms nothing.
- Deleting an automation cancels its pending fire.

---

## Message Templates

WhatsApp message templates belong to a WhatsApp connection and are managed from that connection's page in the app (create, submit to Meta for approval, check status, send a test). The same templates are sent by `send_message` on WhatsApp and by kanban follow-ups outside the 24-hour window ([kanban.md](kanban.md)).

A template is sent only once Meta has approved it, and an automation sends it as-is — no template variables are filled in, so choose a template whose body needs none.

---

## Limits

| Limit | Value |
|-------|-------|
| Automations per hub | 200 |
| `name` | 100 characters, unique per hub |
| `description` | 1,000 characters |
| `instructions` | 10,000 characters |
| `text` | 4,096 characters |
| `cron` | 512 characters |

---

## Contacts & Lists

The contacts and lists an automation targets are hub-scoped `hub.yaml` blocks.

**Contacts** — at least one channel identifier (`phone`, `email`, or `instagram_sid`) is required:

```yaml
outbound_contacts:
  - name: "John Doe"
    phone: "+5511999999999"       # E.164 (WhatsApp)
    email: "john@example.com"     # email
    instagram_sid: "123456789"    # Instagram Scoped ID
    tags: ["vip", "active"]       # free-form
    enabled: true                 # default true; a disabled contact is left out of every fire
```

**Lists** — named, static collections of contacts, referenced by contact name:

```yaml
outbound_lists:
  - name: "VIP Customers"
    description: "High-value customers for weekly check-ins"
    contacts:
      - "John Doe"
      - "Jane Smith"
```

Lists enumerate specific contacts; tags do not select contacts into a list.

**Unlike `automations:`, an absent `outbound_contacts:` or `outbound_lists:` key deletes every contact or list** on push — with one exception: a push never deletes a list that an automation it keeps still targets (with `automations:` absent, every stored automation counts). That delete is refused, naming the automations, and the rest of the push still applies. The same holds for a list left out of a shorter `outbound_lists:`. To delete such a list, move its automations to another list or delete them in the same push. Keep both blocks in `hub.yaml` while any automation targets a list.

Inline contacts are practical up to **~500** — beyond that, `hub.yaml` diffs and push reviews become slow and noisy. For larger lists, import contacts via the app or the API; lists and automations stay in `hub.yaml`.

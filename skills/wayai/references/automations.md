# Automations

An automation starts work in a hub on its own: on a schedule, for every contact of one of the organization's lists that the hub can see, it sends a message or runs an agent. Automations live in `hub.yaml` under `automations:` and are managed via `wayai push` or the app; an enabled automation runs on its schedule ([Enabling, Pausing and Run Now](#enabling-pausing-and-run-now)). The contacts and lists they target are organization data, kept in the organization's contact book — not in `hub.yaml` ([The Contact Book](#the-contact-book)).

## Table of Contents
- [Shape](#shape)
- [What a Hub Can Save](#what-a-hub-can-save)
- [Examples](#examples)
- [What Each Fire Does](#what-each-fire-does)
- [References](#references)
- [Pushing `automations:`](#pushing-automations)
- [Enabling, Pausing and Run Now](#enabling-pausing-and-run-now)
- [Message Templates](#message-templates)
- [Limits](#limits)
- [The Contact Book](#the-contact-book)

---

## Shape

One automation is a **trigger** (when it fires), a **target** (who each fire is for) and an **action** (what it does for each target).

```yaml
# hub.yaml
automations:
  - id: "automation-uuid"            # set by pull; never author it
    name: "Weekly Check-in"          # unique per hub
    description: "Monday check-in with VIP customers"   # optional
    purpose: marketing               # marketing | operational; default marketing — see "Suppressions"
    enabled: false                   # true = runs on its schedule; see "Enabling, Pausing and Run Now"
    trigger:
      type: schedule
      cron: "0 9 * * 1"              # 5-field cron: minute hour day-of-month month day-of-week
      timezone: "America/Sao_Paulo"  # IANA; default UTC
    target:
      type: contact_list
      org_list: "vip-customers"      # an org list of the organization's contact book, by name
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
- **Target:** `contact_list` only, naming an org list by `org_list` — a list name: lowercase letters, digits and hyphens, at most 64 characters. The name is not looked up at save: a fire that finds no list of that name records `list_not_found` ([What Each Fire Does](#what-each-fire-does)).
- **A field the chosen action does not take is refused** — e.g. `text` on a WhatsApp send, or `connection` on a `run_agent` that is not on email.
- **Task hubs run agents only:** `send_message` is refused there. A hub-type change that would make a stored automation invalid is refused, naming it.
- **Purpose:** `marketing` (the default when `purpose` is left out) or `operational` — any other value is refused. It decides which suppressions stop the automation's sends ([Suppressions](#suppressions)): use `operational` only for messages a person needs whatever they opted out of, such as an appointment reminder.
- **Not available yet, refused at save:** `event` and `webhook` triggers, the `none` target, gates, and `send_message` on `email` or `instagram`.

---

## Examples

**WhatsApp template send** (chat hub):

```yaml
automations:
  - name: "Appointment Reminder"
    purpose: operational                 # still sent to those who stopped only marketing
    trigger: { type: schedule, cron: "0 8 * * *", timezone: "America/Sao_Paulo" }
    target: { type: contact_list, org_list: "tomorrows-appointments" }
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
    target: { type: contact_list, org_list: "members" }
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
    target: { type: contact_list, org_list: "renewals-due" }
    action:
      type: run_agent
      channel: email
      connection: "Support Email"        # the email connection the conversation runs on
      agent: "Renewals Agent"
      instructions: "Remind the customer their plan renews this month and offer to answer questions."
```

---

## What Each Fire Does

A fire runs once per contact it reaches: each member of the org list that the firing hub can see ([Visibility](#visibility)). A contact without the channel's identifier — `phone` for WhatsApp, `instagram_sid` for Instagram, `email` for email — is skipped. Each fire records a run with its totals (targets, succeeded, failed, skipped) in the automation's run history.

| Action | Channel | Per contact |
|--------|---------|-------------|
| `send_message` | `whatsapp` | Sends the template to the contact's `phone` and records it in the contact's WhatsApp conversation |
| `send_message` | `app` | Skipped: a fire does not reach contacts in the app yet, so there is no app conversation to post into |
| `run_agent` | `email` | The agent receives `instructions` as a system message (the contact never sees it) in the contact's email conversation and replies by email. Needs an enabled email channel on the action's connection, with a verified sending domain |
| `run_agent` | `whatsapp`, `instagram` | The agent takes a turn on `instructions` in a system conversation that is not addressed to the contact, so its reply does not reach them yet |
| `run_agent` | `app` | Skipped, as for `send_message` on `app` |

- **A suppressed contact is skipped** (`suppressed`), before anything is sent or counted against the quota: on WhatsApp its `phone` is checked, on email its `email`, on Instagram its `instagram_sid`, and on the app every identity it holds. A `marketing` automation is stopped by a suppression of either scope, an `operational` one only by an `all` suppression ([Suppressions](#suppressions)). If the suppressions cannot be read, the fire sends to no one: every contact is recorded failed, and the next fire is the schedule's.
- `agent` must name one of the hub's agents at save; a fire does not yet use it to choose which agent takes the turn.
- **A list a fire cannot use reaches no one.** A fire whose `org_list` names no list of the contact book (deleted, renamed, or never created) records `list_not_found`; one whose list holds more contacts the hub sees than the platform's per-fire cap records `list_over_cap`. Either records a failed run with no targets and raises a warning on the hub's Status & Notices (`wayai alerts` lists it). Neither is sent again: the next fire is the schedule's.
- After the suppression check, a `send_message` fire checks the org's operations quota for the contacts it did not stop. When the org is over its quota, every one of them is skipped and the skipped fire is not sent again: the automation's next fire is its first scheduled one after the quota resets, or after a day if that is sooner.

---

## References

`connection`, `template` and `agent` take a **name**; each has an id twin (`connection_id`, `template_id`, `agent_id`). Give either or both — **the id wins** when both are present. `wayai pull` writes both.

- `connection` is the connection's display name; `template` is matched among the templates of the action's connection. A template name that matches more than one template is refused — name it by `template_id`.
- A reference to an agent the same push creates or renames resolves on that push, by the name the push gives it. A push whose agents are refused writes none of them, so an automation naming one of them is refused too.
- A reference to anything the hub does not hold is refused, naming the automation.
- `org_list` is a name only, with no id twin: an org list belongs to the organization, so a push stores the name as written and never checks it against the contact book, and publishing copies it unchanged.

Automations are matched to the hub's stored ones by `id`, else by `name` (a stored automation another entry names by `id` is that entry's): renaming an entry that carries its `id` renames the automation, while renaming one without it creates a new automation and deletes the old.

---

## Pushing `automations:`

| `hub.yaml` | Push result |
|------------|-------------|
| No `automations:` key (or a key with no value) | **Nothing changes** — the hub keeps its automations |
| `automations: []` | **Deletes every automation** |
| A list of entries | The hub's automations become exactly that list: missing entries are deleted, new ones created, changed ones updated |

While the key is absent, the stored automations still protect what they use — a connection an automation sends through is never deleted as unreferenced. A refused entry is reported, naming the automation, and is not written — its stored automation, if any, stays as it was, and an entry that would take its name is refused too; the rest of the push still applies.

The old `outbound_schedules:` key is no longer read — the CLI warns and pushes none of its entries. Rewrite each schedule as an automation. The same holds for the hub contact and list blocks the contact book replaced: recreate those contacts and lists in the contact book ([The Contact Book](#the-contact-book)).

---

## Enabling, Pausing and Run Now

An automation runs on its schedule while it is **enabled, not paused, and able to run** — in every hub, preview or production. Whatever changes one of those — a push, a save in the app, the enable toggle, a publish, pause or resume — reschedules it at once, so `enabled:` in `hub.yaml` means what it says.

- **Enabled** is config: `enabled:` in `hub.yaml` or the toggle in the hub's **Automations** tab. An automation created in the app starts disabled. Disabling it cancels its next run.
- **Paused** is not config: **Pause** in the Automations tab stops an automation at once, and **Resume** lets it run again. Publishing never changes it, so a production automation stays paused through later publishes.
- **Able to run** means it still passes the rules it was saved under — its agent, connection and template exist, and a `run_agent` automation's agent is enabled. One that is enabled and not paused but cannot run is skipped at each scheduled run: the Automations tab shows why, and the hub's **Status** tab shows an alert until it runs again, or is paused, disabled or fixed. Its org list is read at each run instead: a run that finds none of that name records `list_not_found` and raises its own hub alert.
- **Run now** runs the automation once, immediately, whether or not it is enabled. It is refused while the automation is paused, while a run of it is still in progress, and when it cannot run (the reason is in the refusal).
- **Production** hubs change only by publishing: there an automation is read-only in the app, except **Pause**, **Resume** and **Run now**, which need a hub admin. Publishing copies automations as written, `org_list` included — an enabled one starts running on production on its own schedule, reaching the members of that list production sees — and never copies a preview's pause, next run or run history.
- A preview hub's automation delivers for real — to the contacts of its org list that the preview hub sees: only those whose environment is `preview` or `all` ([Visibility](#visibility)).
- A preview automation whose connections — its own, or its agent's tools' — hold the same credential as production shows a warning in the Automations tab: a run from that preview reaches the same systems production does.
- Deleting an automation cancels its next run.

---

## Message Templates

WhatsApp message templates belong to a WhatsApp connection and are managed from that connection's page in the app (create, submit to Meta for approval, check status, send a test). The same templates are sent by `send_message` on WhatsApp and by kanban follow-ups outside the 24-hour window ([kanban.md](kanban.md)).

A template is sent only once Meta has approved it, and an automation sends it as-is — no template variables are filled in, so choose a template whose body needs none.

Publishing does not copy message templates yet, so a published WhatsApp `send_message` automation cannot run on production: it is skipped, with the reason shown, until its template exists there.

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

## The Contact Book

Contacts and the lists automations target belong to the **organization**, not to a hub. They are kept in the organization's contact book, managed in the organization's settings (**Contacts** tab) or through its API (`/api/contact-book`). They are not part of `hub.yaml`, and publishing or syncing a hub never copies them.

**Who manages it.** Organization admins, or a token holding the `contacts:write` permission over the whole organization (every hub, both environments), read and write every contact and org list. A hub admin can view the contacts their hub sees, but not which lists they are on.

**Contacts.** A contact has a name and at least one channel identifier — `phone` (an international number, stored as E.164, e.g. `+5511999999999`), `email` or `instagram_sid` — plus optional metadata and labels. Labels are free-form and stored lowercased: at most 20 per contact, each up to 64 characters, and **never holding a comma** — the API answers `400` to a contact, an import batch or a list filter carrying one.

**Linking an account.** An admin can link a contact to a WayAI account, chosen among the accounts that already hold a hub user in one of the organization's hubs and whose hub user carries the email asked for — so a link reveals nothing about any other address. Deleting the account removes its links; the contacts stay.

### Visibility

Which hubs see a contact is chosen explicitly when it is created or imported; there is no default:

- **All hubs**, stated outright, or
- **Scope tags** — organization tags ([connections.md](connections.md#organization-tags)): a hub sees the contact if it carries one of them.

A contact also has an **environment**: `production` (the default), `preview` or `all`. A hub sees only contacts of its own environment or of `all`, so a preview hub sees only `preview` and `all` contacts.

### Org lists

An org list has a name — lowercase letters, digits and hyphens, e.g. `vip-customers` — which is what an automation's `org_list` names. Its members are:

- the contacts **pinned** to it, and
- when its filter is not empty, every contact the filter matches: one carrying one of its labels (when it names labels) and one of its scope tags (when it names scope tags).

A list with an empty filter holds only its pins. Each fire reaches the members the firing hub sees.

**Deleting or renaming a list an automation names is allowed.** The automation's next fire records `list_not_found` and raises a warning on the hub ([What Each Fire Does](#what-each-fire-does)); point the automation at another list, or delete it.

### Suppressions

A suppression stops automations from sending to one **phone**, **email address** or **Instagram id**. It belongs to the identity, not to a contact: deleting the contact leaves it in force, and it applies to any contact that holds that identity later. The book keeps no phone, address or id for it — only a keyed fingerprint, unique to the organization — so the list of suppressions shows each one's type and scope, never whose it is; a contact's page shows the suppressions held against its own identities. An email address is matched as its mailbox: letter case, a `+tag` and, for Gmail, dots do not make another address.

Each suppression has a **scope**:

| Scope | Stops |
|-------|-------|
| `marketing` | automations whose `purpose` is `marketing` |
| `all` | every automation, `operational` ones included |

Suppressions are only ever widened: suppressing an identity already held at `marketing` with `all` makes it `all`, and suppressing one held at `all` with `marketing` leaves it `all`. To narrow one, remove it and add it again.

**Who writes them.** The organization's contact-book managers, with either scope, in the Contacts tab (Suppressions, or a contact's page) or the API: `POST /api/contact-book/suppressions` with `{ organization_id, identity: { kind: phone | email | instagram, value }, scope }`; `GET /api/contact-book/suppressions?organization_id=<org_id>` to list (paged by `cursor` and `limit`, newest first); `DELETE /api/contact-book/suppressions/<suppression_id>?organization_id=<org_id>` to remove; `GET /api/contact-book/contacts/<contact_id>/suppressions?organization_id=<org_id>` for one contact's. And the person themselves, through an email's one-click unsubscribe link, which suppresses their address with `all` — no sign-in, and their mail app's unsubscribe button does the same. Automation emails do not carry that link yet: until they do, record an email opt-out as a suppression yourself.

Suppressions are checked only on an automation's sends; an agent's replies in a conversation the person is having are never stopped.

### CSV import

Contacts import from a CSV file in batches of at most **500 rows**. Each batch states its visibility (required: all hubs, or scope tags), its environment, and optionally labels added to every row. In the file, a labels column separates labels with `;` or `|`, or with commas inside a quoted cell. A row whose phone, email or Instagram id another contact (or an earlier row of the same batch) already holds is not imported and is reported as a duplicate.

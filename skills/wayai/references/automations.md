# Automations

An automation starts work in a hub on its own: on a schedule, for every contact of a list that the hub can see, it sends a message or runs an agent — or, on a task hub, it opens tasks for an agent: one per run, or one per contact of a list ([Task Hubs](#task-hubs)), on a schedule or each time something happens in the hub ([Event Triggers](#event-triggers)). The list is one of the hub's own ([Hub Lists](#hub-lists), in `hub.yaml` under `contact_lists:`) or one of the organization's. Automations live in `hub.yaml` under `automations:` and are managed via `wayai push` or the app; an enabled automation runs on its schedule ([Enabling, Pausing and Run Now](#enabling-pausing-and-run-now)). The contacts themselves, and the organization's lists, are organization data, kept in the organization's contact book — not in `hub.yaml` ([The Contact Book](#the-contact-book)).

## Table of Contents
- [Shape](#shape)
- [What a Hub Can Save](#what-a-hub-can-save)
- [Examples](#examples)
- [What Each Fire Does](#what-each-fire-does)
- [Task Hubs](#task-hubs)
- [Event Triggers](#event-triggers)
- [References](#references)
- [Pushing `automations:`](#pushing-automations)
- [Enabling, Pausing and Run Now](#enabling-pausing-and-run-now)
- [Message Templates](#message-templates)
- [Limits](#limits)
- [Hub Lists](#hub-lists)
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
      type: schedule                 # schedule | event (task hubs only)
      cron: "0 9 * * 1"              # 5-field cron: minute hour day-of-month month day-of-week
      timezone: "America/Sao_Paulo"  # IANA; default UTC
      # event: conversation.closed   # an event trigger's event, with no cron or timezone — see "Event Triggers"
    target:
      type: contact_list             # contact_list | none (task hubs only: one task per run)
      list: "vip-customers"          # one of this hub's lists (`contact_lists:`), by name — or:
      # org_list: "vip-customers"    # an org list of the organization's contact book, by name
    action:
      type: run_agent                # run_agent | send_message
      channel: whatsapp              # whatsapp | instagram | email | app; left out on a task hub
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
| `chat` | `contact_list` | `run_agent` | `whatsapp`, `instagram`, `app` | `agent` + `instructions` |
| `chat` | `contact_list` | `run_agent` | `email` | `agent` + `instructions` + `connection` (required) |
| `task` | `none` — one task per run | `run_agent` | none | `agent` + `instructions` |
| `task` | `contact_list` — one task per contact | `run_agent` | none | `agent` + `instructions` |

- **Trigger:** `schedule`, with a valid 5-field `cron` and an IANA `timezone` (default `UTC`) — or, on a task hub, `event`, naming one `event` (`conversation.flagged` or `conversation.closed`) and no `cron` or `timezone`, with the `none` target ([Event Triggers](#event-triggers)). An `event` trigger is refused on a chat hub, and with a `contact_list` target.
- **Target:** `contact_list`, naming exactly one list — `list` (one of the hub's own lists) or `org_list` (an org list), never both. Either is a list name: lowercase letters, digits and hyphens, at most 64 characters. A `list` must name one of the hub's lists at save — on a push, one of the lists the push leaves ([Hub Lists](#hub-lists)). An `org_list` is not looked up at save: a fire that finds no list of that name records `list_not_found` ([What Each Fire Does](#what-each-fire-does)). On a task hub the target may also be `none` (`target: { type: none }`, naming no list): each run opens one task. `none` is refused on a chat hub.
- **A field the chosen action does not take is refused** — e.g. `text` on a WhatsApp send, or `connection` on a `run_agent` that is not on email.
- **Task hubs run agents only, with no channel:** `send_message`, a `channel` and a `connection` are refused there — each run opens tasks in the hub, which reach no one through a channel. The `agent` must be a `pilot` or `pilot_specialist` agent, since each task runs it ([Task Hubs](#task-hubs)). A hub-type change that would make a stored automation invalid is refused, naming it — so a hub whose automations name a channel cannot become a task hub, nor a task hub's become a chat hub, until they are rewritten.
- **Purpose:** `marketing` (the default when `purpose` is left out) or `operational` — any other value is refused. It decides which suppressions stop the automation's sends ([Suppressions](#suppressions)): use `operational` only for messages a person needs whatever they opted out of, such as an appointment reminder.
- **Not available yet, refused at save:** `webhook` triggers, `event` triggers on a chat hub, gates, and `send_message` on `email` or `instagram`.

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

**A hub list of this hub's own** (chat hub):

```yaml
contact_lists:
  - name: "lapsed-vips"
    filter:
      labels: ["vip"]
      scope_tags: ["sao-paulo"]          # organization tag names
automations:
  - name: "Win-back"
    trigger: { type: schedule, cron: "0 10 * * 1", timezone: "America/Sao_Paulo" }
    target: { type: contact_list, list: "lapsed-vips" }
    action:
      type: send_message
      channel: whatsapp
      connection: "WhatsApp Main"
      template: "we_miss_you"
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

**A daily task for an agent** (task hub):

```yaml
automations:
  - name: "Daily Review"
    trigger: { type: schedule, cron: "0 7 * * 1-5", timezone: "America/Sao_Paulo" }
    target: { type: none }               # one task per run
    action:
      type: run_agent                    # no channel on a task hub
      agent: "Ops Agent"
      instructions: "Review yesterday's open orders and list the ones that need a person today."
```

**A task per contact** (task hub):

```yaml
automations:
  - name: "Renewal Prep"
    purpose: operational
    trigger: { type: schedule, cron: "0 8 1 * *" }
    target: { type: contact_list, org_list: "renewals-due" }   # one task per contact
    action:
      type: run_agent
      agent: "Account Agent"
      instructions: "Prepare a renewal summary for this customer from their record and our order history."
```

**A task when a conversation closes** (task hub):

```yaml
automations:
  - name: "Close Review"
    enabled: true
    trigger: { type: event, event: conversation.closed }   # no cron, no timezone
    target: { type: none }               # one task per closed conversation
    action:
      type: run_agent
      agent: "QA Agent"
      instructions: "Review the closed conversation in your data and note anything the team should follow up on."
```

**Run an agent by email** (chat hub):

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

The table is a chat hub's fire; a task hub's opens tasks instead ([Task Hubs](#task-hubs)). The notes under it hold for both.

A fire runs once per contact it reaches: each member of its list — a hub list or an org list — that the firing hub can see ([Visibility](#visibility)), read when the fire runs. A contact without the channel's identifier — `phone` for WhatsApp, `instagram_sid` for Instagram, `email` for email — is skipped. Each fire records a run with its totals (targets, succeeded, failed, skipped) in the automation's run history.

| Action | Channel | Per contact |
|--------|---------|-------------|
| `send_message` | `whatsapp` | Sends the template to the contact's `phone` and records it in the contact's WhatsApp conversation |
| `send_message` | `app` | Skipped: a fire does not reach contacts in the app yet, so there is no app conversation to post into |
| `run_agent` | `email` | The agent receives `instructions` as a system message (the contact never sees it) in the contact's email conversation and replies by email. Needs an enabled email channel on the action's connection, with a verified sending domain |
| `run_agent` | `whatsapp`, `instagram` | The agent takes a turn on `instructions` in a system conversation that is not addressed to the contact, so its reply does not reach them yet |
| `run_agent` | `app` | Skipped, as for `send_message` on `app` |

- **A suppressed contact is skipped** (`suppressed`), before anything is sent or counted against the quota: on WhatsApp its `phone` is checked, on email its `email`, on Instagram its `instagram_sid`, and on the app every identity it holds. A `marketing` automation is stopped by a suppression of either scope, an `operational` one only by an `all` suppression ([Suppressions](#suppressions)). If the suppressions cannot be read, the fire sends to no one: every contact is recorded failed, and the next fire is the schedule's.
- `agent` must name one of the hub's agents at save; on a chat hub, a fire does not yet use it to choose which agent takes the turn.
- `instructions` are sent as written: a `{{…}}` placeholder in them is not filled in, and reaches the agent as typed. Placeholders work in an agent's own instructions and `additional_context_template` ([agents/instructions.md](agents/instructions.md)).
- **A list a fire cannot use reaches no one.** A fire whose `org_list` names no list of the contact book (deleted, renamed, or never created), or whose `list` names none of the hub's lists, records `list_not_found`; one whose list holds more contacts the hub sees than the platform's per-fire cap records `list_over_cap`. Either records a failed run with no targets and raises a warning on the hub's Status & Notices (`wayai alerts` lists it). Neither is sent again: the next fire is the schedule's.
- After the suppression check, a `send_message` fire — and a task hub's fire ([Task Hubs](#task-hubs)) — checks the org's operations quota for the targets it did not stop. When the org is over its quota, every one of them is skipped (`operations_quota_exceeded`) and the skipped fire is not sent again: a scheduled automation's next fire is its first scheduled one after the quota resets, or after a day if that is sooner. If the quota cannot be checked, every target is recorded failed (`operations_quota_check_failed`), nothing is sent, and a scheduled automation's next fire is a day later at the soonest. An event automation's next fire is its next event, either way ([Event Triggers](#event-triggers)).

---

## Task Hubs

On a task hub, each run of an automation opens **tasks**: one for a `none` target, or one per contact of its list that the hub sees and no suppression stops (every identity the contact holds is checked, as on the app). Each task:

- **is a conversation of its own in the hub's task list**, titled by the automation (and the contact's name, for a list), with **no person behind it**: it grants no one access to the hub, and the team answers and reviews it like any other task;
- **is run by the automation's `agent`**, which takes its first turn — so the agent must be a `pilot` or `pilot_specialist` agent. On a hub whose AI mode puts new conversations with the team, the task waits for the team instead;
- **receives `instructions` as a system message, exactly as written** — placeholders are not filled in;
- **receives a contact's record as data, never as instructions** — for a list, the task opens with the contact's record from the contact book (its id, name, phone, email, Instagram id, labels and metadata), fenced and marked as data, ahead of the instructions and outside them. A contact's values never reach the instructions, so a name or a metadata field cannot pass for an order to the agent. Write the instructions about "this contact" and let the agent read the record;
- **reads and writes the automation's own user state**: every task of one automation shares one set of user-scope states (`{{state(user, …)}}` in the agent's instructions reads it, `update_state` writes it), and no other automation sees it. Tasks of one run that write the same state at once keep the last write. Keep per-contact notes in the task or the contact, not in user state.

Each run records one target per task in the run history, with the task's conversation (and its contact, for a list). A run checks the org's operations quota before it opens any task: over the quota, no task opens and nothing is billed ([What Each Fire Does](#what-each-fire-does)). Each task opened is one turn initiation — one operation — plus its turn's time, billed to the hub's organization.

---

## Event Triggers

On a task hub, an automation can run each time something happens in the hub instead of on a schedule: `trigger: { type: event, event: <event> }`, with `target: { type: none }` — each event opens one task, about the event's conversation.

| `event` | Runs when |
|---------|-----------|
| `conversation.flagged` | One of the hub's conversations is flagged — by its monitor or its evaluator — while it was not flagged before. A flag stays on the conversation (reopening keeps it), so a conversation runs this at most once |
| `conversation.closed` | One of the hub's conversations closes — and again each time a reopened one closes again |

- **Each event runs each automation of the hub that listens for it once**, at once — every enabled, unpaused automation of that event that can run. Each run opens one task, as any task hub's run for a `none` target does ([Task Hubs](#task-hubs)).
- **An automation never runs on the events of the tasks it opened**, so one that reviews each closed conversation does not review its own reviews.
- **Each task receives the event as data, never as instructions** — its first message holds the event (`event`, its id and when it happened) and the conversation it is about (its id, title, status, kanban status, outcome, flag source, channel, when it started and ended), fenced and marked as data, before the instructions. Write the instructions about "the conversation in your data".
- **The same event never runs an automation twice.** An event is one conversation's one change — its flag, or one close — so a conversation flagged again and again, or a close the hub hears of twice, runs nothing more.
- **Automations that start each other stop.** A task an automation opens is one step deeper than the conversation whose event started it (a scheduled run's task is one step deep). An event from a conversation three steps deep runs nothing: the automation's Automations tab shows why, and the hub's **Status** tab shows an alert until it next runs. So two automations that answer each other's tasks' closes run three deep, then stop.
- **Every environment runs its own events:** a preview hub's conversations run that preview's automations, as production's run production's.
- **Eval runs' conversations run nothing.**
- An evaluator's flag that the hub's conversation list never shows — it could not be recorded there — runs nothing either.
- **An event is not run again later.** A run the org's operations quota stops is recorded skipped; the automation's next run is on its next event. An automation disabled or paused when the event happens does not run for it, even once resumed.
- An event automation has no next run to show. **Run now** runs it once with no event: its task receives no event data.

---

## References

`connection`, `template` and `agent` take a **name**; each has an id twin (`connection_id`, `template_id`, `agent_id`). Give either or both — **the id wins** when both are present. `wayai pull` writes both.

- `connection` is the connection's display name; `template` is matched among the templates of the action's connection. A template name that matches more than one template is refused — name it by `template_id`.
- A reference to an agent the same push creates or renames resolves on that push, by the name the push gives it. A push whose agents are refused writes none of them, so an automation naming one of them is refused too.
- A reference to anything the hub does not hold is refused, naming the automation.
- `org_list` is a name only, with no id twin: an org list belongs to the organization, so a push stores the name as written and never checks it against the contact book, and publishing copies it unchanged.
- `list` is a name only too, checked against the hub's lists: a push checks it against the lists the push leaves (the stored ones when `contact_lists:` is absent), so a list and the automation naming it can be created in the same push. Renaming a hub list renames it in every automation that names it, unless another list takes the old name in the same change; a hub list an automation names cannot be deleted ([Hub Lists](#hub-lists)).

Automations are matched to the hub's stored ones by `id`, else by `name` (a stored automation another entry names by `id` is that entry's): renaming an entry that carries its `id` renames the automation, while renaming one without it creates a new automation and deletes the old.

---

## Pushing `automations:`

| `hub.yaml` | Push result |
|------------|-------------|
| No `automations:` key (or a key with no value) | **Nothing changes** — the hub keeps its automations |
| `automations: []` | **Deletes every automation** |
| A list of entries | The hub's automations become exactly that list: missing entries are deleted, new ones created, changed ones updated |

While the key is absent, the stored automations still protect what they use — a connection an automation sends through is never deleted as unreferenced. A refused entry is reported, naming the automation, and is not written — its stored automation, if any, stays as it was, and an entry that would take its name is refused too; the rest of the push still applies.

`contact_lists:` follows the same rule: no key (or a key with no value) leaves the hub's lists as they are, `contact_lists: []` deletes every list, and a list of entries is the set the hub keeps. While `automations:` is absent, the stored automations still protect the lists they name: a push that would delete one is refused for that list, naming the automations.

The old `outbound_schedules:` key is no longer read — the CLI warns and pushes none of its entries. Rewrite each schedule as an automation. The same holds for the hub contact and list blocks the contact book replaced: recreate those contacts in the contact book ([The Contact Book](#the-contact-book)), and each list as a hub list under `contact_lists:` ([Hub Lists](#hub-lists)) or an organization list.

---

## Enabling, Pausing and Run Now

An automation runs on its schedule — or, with an event trigger, on each of its events ([Event Triggers](#event-triggers)) — while it is **enabled, not paused, and able to run** — in every hub, preview or production. Whatever changes one of those — a push, a save in the app, the enable toggle, a publish, pause or resume — reschedules it at once, so `enabled:` in `hub.yaml` means what it says.

- **Enabled** is config: `enabled:` in `hub.yaml` or the toggle in the hub's **Automations** tab. An automation created in the app starts disabled. Disabling it cancels its next run.
- **Paused** is not config: **Pause** in the Automations tab stops an automation at once, and **Resume** lets it run again. Publishing never changes it, so a production automation stays paused through later publishes.
- **Able to run** means it still passes the rules it was saved under — its agent, connection and template exist, and a `run_agent` automation's agent is enabled (on a task hub, also still a `pilot` or `pilot_specialist`). One that is enabled and not paused but cannot run is skipped at each scheduled run: the Automations tab shows why, and the hub's **Status** tab shows an alert until it runs again, or is paused, disabled or fixed. Its list — a hub list or an org list — is read at each run instead: a run that finds none of that name records `list_not_found` and raises its own hub alert.
- **Run now** runs the automation once, immediately, whether or not it is enabled. It is refused while the automation is paused, while a run of it is still in progress, and when it cannot run (the reason is in the refusal).
- **Production** hubs change only by publishing: there an automation is read-only in the app, except **Pause**, **Resume** and **Run now**, which need a hub admin. Publishing copies automations as written, `list` and `org_list` included, with the hub's lists beside them — an enabled one starts running on production on its own schedule, reaching the members of that list production sees — and never copies a preview's pause, next run or run history. A publish naming a hub list the preview does not hold is refused, naming the automation.
- A preview hub's automation delivers for real — to the contacts of its list that the preview hub sees: only those whose environment is `preview` or `all` ([Visibility](#visibility)).
- A preview automation whose connections — its own, or its agent's tools' — hold the same credential as production shows a warning in the Automations tab: a run from that preview reaches the same systems production does.
- Deleting an automation cancels its next run.

---

## Message Templates

WhatsApp message templates belong to a WhatsApp connection and are managed from that connection's page in the app (create, submit to Meta for approval, check status, send a test). The same templates are sent by `send_message` on WhatsApp and by kanban follow-ups outside the 24-hour window ([kanban.md](kanban.md)).

A template is sent only once Meta has approved it, and an automation sends it as-is — no template variables are filled in, so choose a template whose body needs none.

Templates are managed on the **preview** hub you publish from. Publishing and syncing copy them to production under the **same id**, so an automation's `template_id` and a follow-up's `template_whatsapp_id` name the same template on both. On production they are read-only: listed and sent, a test send included, never created, edited, submitted or deleted. A sibling or CI branch preview gets no copy: its WhatsApp connection names the same account, so a delete or an edit there would act on the template production sends. Meta approves a template once for the WhatsApp account, which the production connection shares with the preview's — so after Meta approves (or pauses) a template, check its status on the preview and **sync**, so production's copy says so. Deleting a template on the preview deletes it at Meta at once, so production's sends of it fail from then on, and the next sync removes production's copy.

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
| Hub lists per hub | 100 |

---

## Hub Lists

A hub list is one of the hub's own segments of the organization's contact book: a **filter** over the contacts' labels and scope tags, which an automation targets by `list`. It is hub config — in `hub.yaml` under `contact_lists:`, or the **Lists** sub-tab of the hub's **Automations** tab — and is copied by publish and sync like the rest of it. It holds no contacts of its own and has no pins: its members are read from the contact book at each fire.

```yaml
# hub.yaml
contact_lists:
  - id: "list-uuid"                # set by pull; never author it
    name: "lapsed-vips"            # unique per hub; lowercase letters, digits and hyphens
    description: "VIPs with no order this quarter"   # optional
    filter:
      labels: ["vip", "lapsed"]    # contact labels (lowercased); a contact needs one of them
      scope_tags: ["sao-paulo"]    # organization tag names; a contact needs one of them
```

- **Members:** a contact carrying one of the filter's labels (when it names labels) and one of its scope tags (when it names scope tags) — the org-list rule without pins. A filter must name at least one label or scope tag.
- **Never past the hub's visibility:** each fire reaches only the members the firing hub sees, by its tags and environment as they are when it fires ([Visibility](#visibility)) — an organization tag removed from the hub stops reaching the contacts scoped to it, and a preview hub never reaches a `production` contact.
- **Renaming** a list renames it in every automation that targets it, unless another list takes the old name in the same change (a swap of two names, or a new list under the old one): an automation keeps the name it states. **Deleting** a list an automation targets is refused, naming the automations; point them at another list first. In one push, a list an automation still targets is deleted after the automations are written, so its name and its place are free only on the next push.
- **Scope tags** are named by their organization tag names; an unknown name is refused. An organization tag cannot be deleted while a hub list's filter names it. If a list still names a tag the organization no longer has, pull writes the tag's id in its place. A push refuses that id and keeps the stored list, because leaving the tag out would reach more contacts; remove the id from `scope_tags` deliberately to push the list again. The app's Lists sub-tab offers the hub's own tags and those its lists already name; `hub.yaml` takes any of the organization's tags.
- Managed with `hub:write` on a preview hub; a production hub's lists change only by publishing.
- Every key not shown above is refused, at the entry level and inside `filter`. Limits: 100 lists per hub, 20 labels and 20 scope tags per filter, a 1,000-character description.

---

## The Contact Book

Contacts and the organization's lists belong to the **organization**, not to a hub. They are kept in the organization's contact book, managed in the organization's settings (**Contacts** tab) or through its API (`/api/contact-book`). They are not part of `hub.yaml`, and publishing or syncing a hub never copies them. A hub's own lists ([Hub Lists](#hub-lists)) are the exception: they are hub config, in `hub.yaml` under `contact_lists:` and copied by publish and sync — and they hold only a filter over the book, never a contact.

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

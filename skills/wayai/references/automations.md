# Automations

An automation starts work in a hub on its own: on a schedule, for every contact of a list that the hub can see, it sends a message or runs an agent — or, on a task hub, it opens tasks for an agent: one per run, or one per contact of a list ([Task Hubs](#task-hubs)), on a schedule, each time something happens in the hub ([Event Triggers](#event-triggers)), or on each delivery an external system sends to the automation's own URL ([Webhook Triggers](#webhook-triggers)). The list is one of the hub's own ([Hub Lists](#hub-lists), in `hub.yaml` under `contact_lists:`) or one of the organization's. Automations live in `hub.yaml` under `automations:` and are managed via `wayai push` or the app; an enabled automation runs on its schedule ([Enabling, Pausing and Run Now](#enabling-pausing-and-run-now)). The contacts themselves, and the organization's lists, are organization data, kept in the organization's contact book — not in `hub.yaml` ([The Contact Book](#the-contact-book)).

## Table of Contents
- [Shape](#shape)
- [What a Hub Can Save](#what-a-hub-can-save)
- [Examples](#examples)
- [What Each Fire Does](#what-each-fire-does)
- [Task Hubs](#task-hubs)
- [Event Triggers](#event-triggers)
- [Webhook Triggers](#webhook-triggers)
- [Gates](#gates)
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
      type: schedule                 # schedule | event | webhook (event and webhook: task hubs only)
      cron: "0 9 * * 1"              # 5-field cron: minute hour day-of-month month day-of-week
      timezone: "America/Sao_Paulo"  # IANA; default UTC
      # event: conversation.closed   # an event trigger's event, with no cron or timezone — see "Event Triggers"
      # auth: hmac                   # webhook only: hmac | static_header — see "Webhook Triggers"
      # header: "X-Account-Key"      # webhook, static_header only: the header that carries the secret
    target:
      type: contact_list             # contact_list | none (task hubs only: one task per run)
      list: "vip-customers"          # one of this hub's lists (`contact_lists:`), by name — or:
      # org_list: "vip-customers"    # an org list of the organization's contact book, by name
    action:
      type: run_agent                # run_agent | send_message
      channel: whatsapp              # whatsapp | instagram | email | app; left out on a task hub
      agent: "Support Agent"         # run_agent only (or agent_id); a pilot or pilot_specialist
      instructions: "Check in on the customer's last order."  # run_agent only
      connection: "Sales WhatsApp"   # the channel's connection: every channel but app (or connection_id)
      template: "check_in"           # whatsapp: the template sent, or a run_agent's opener (or template_id)
      # text                         — send_message on email and app
    gate:                            # optional — decides each run, or each contact of a list, before it starts; see "Gates"
      monitor: "Holiday Check"       # a monitor on trigger automation_fire, by agent name
```

Every key not shown above is refused, never ignored — at the entry level and inside `trigger`, `target`, `action` and `gate`.

---

## What a Hub Can Save

Every save — a push, the app, a hub-type change, a publish — is validated, and each refusal names the automation it is about.

| Hub type | Target | Action | Channel | Fields the action takes |
|----------|--------|--------|---------|-------------------------|
| `chat` | `contact_list` | `send_message` | `whatsapp` | `connection` + `template` (a WhatsApp template of that connection) |
| `chat` | `contact_list` | `send_message` | `email` | `connection` (an email connection) + `text` |
| `chat` | `contact_list` | `send_message` | `app` | `text` |
| `chat` | `contact_list` | `run_agent` | `whatsapp` | `agent` + `instructions` + `connection` + `template` (the opener, a WhatsApp template of that connection) |
| `chat` | `contact_list` | `run_agent` | `instagram`, `email` | `agent` + `instructions` + `connection` |
| `chat` | `contact_list` | `run_agent` | `app` | `agent` + `instructions` |
| `task` | `none` — one task per run | `run_agent` | none | `agent` + `instructions` |
| `task` | `contact_list` — one task per contact | `run_agent` | none | `agent` + `instructions` |

- **Trigger:** `schedule`, with a valid 5-field `cron` and an IANA `timezone` (default `UTC`) — or, on a task hub, `event`, naming one `event` (`conversation.flagged` or `conversation.closed`) and no `cron` or `timezone`, with the `none` target ([Event Triggers](#event-triggers)) — or, on a task hub, `webhook`, with an `auth` of `hmac` or `static_header` (the latter naming its `header`) and no `cron` or `timezone` ([Webhook Triggers](#webhook-triggers)). An `event` trigger is refused on a chat hub, and with a `contact_list` target. A webhook trigger is refused on a chat hub, and a hub-type change that would leave one on a chat hub is refused, naming it.
- **Target:** `contact_list`, naming exactly one list — `list` (one of the hub's own lists) or `org_list` (an org list), never both. Either is a list name: lowercase letters, digits and hyphens, at most 64 characters. A `list` must name one of the hub's lists at save — on a push, one of the lists the push leaves ([Hub Lists](#hub-lists)). An `org_list` is not looked up at save: a fire that finds no list of that name records `list_not_found` ([What Each Fire Does](#what-each-fire-does)). On a task hub the target may also be `none` (`target: { type: none }`, naming no list): each run opens one task. `none` is refused on a chat hub.
- **A field the chosen action does not take is refused** — e.g. `text` on a WhatsApp send, `connection` on the app, or `template` on Instagram.
- **A `run_agent`'s `agent` must be a `pilot` or `pilot_specialist` agent**, on every hub: the named agent takes the turn — in a contact's conversation on a chat hub, in each task on a task hub — and no other role takes it.
- **Task hubs run agents only, with no channel:** `send_message`, a `channel` and a `connection` are refused there — each run opens tasks in the hub, which reach no one through a channel ([Task Hubs](#task-hubs)). A hub-type change that would make a stored automation invalid is refused, naming it — so a hub whose automations name a channel cannot become a task hub, nor a task hub's become a chat hub, until they are rewritten.
- **Purpose:** `marketing` (the default when `purpose` is left out) or `operational` — any other value is refused. It decides which suppressions stop the automation's sends ([Suppressions](#suppressions)): use `operational` only for messages a person needs whatever they opted out of, such as an appointment reminder.
- **Gate:** `gate.monitor` must name one of the hub's monitors whose model is a decisions model, on `trigger: automation_fire` (a fire gate), or — for a `contact_list` target — `trigger: automation_item` (an item gate) ([Gates](#gates)): on a push, one of the agents the push leaves; on a publish, one of the preview's. Any other agent is refused, naming the automation. Leave `gate` out (or set it to `null`) for every run to fire.
- **Not available yet, refused at save:** `event` and `webhook` triggers on a chat hub, and `send_message` on `instagram`.

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

Each fire posts the text to the members linked to a WayAI account that holds access to the hub; the others are skipped — see [What Each Fire Does](#what-each-fire-does).

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

**Run an agent on WhatsApp** (chat hub) — the template opens a conversation whose 24-hour window is closed, and the agent answers the contact's reply:

```yaml
automations:
  - name: "Renewal Offer"
    trigger: { type: schedule, cron: "0 10 * * 2" }
    target: { type: contact_list, list: "renewals-due" }
    action:
      type: run_agent
      channel: whatsapp
      connection: "Sales WhatsApp"
      template: "renewal_hello"          # approved; sent first when the window is closed
      agent: "Renewals Agent"
      instructions: "Offer the customer this month's renewal discount and answer their questions."
```

**Send an email** (chat hub) — a marketing email ends with an unsubscribe link:

```yaml
automations:
  - name: "Monthly Newsletter"
    trigger: { type: schedule, cron: "0 9 1 * *" }
    target: { type: contact_list, org_list: "newsletter" }
    action:
      type: send_message
      channel: email
      connection: "Support Email"        # the email connection it is sent from
      text: "**This month at Acme:** new plans, and a guide to the new dashboard."
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

A fire runs once per contact it reaches: each member of its list — a hub list or an org list — that the firing hub can see ([Visibility](#visibility)). The list is read **a page of 100 contacts at a time** while the run goes on, so a list of any size the contact book holds is reached, and each contact's target shows in the run history as soon as its page is done; the run reads `running` until its last page. A contact without the channel's identifier — `phone` for WhatsApp, `instagram_sid` for Instagram, `email` for email, a linked WayAI account for the app — is skipped. Each fire records a run with its totals (targets, succeeded, failed, skipped) in the automation's run history. Each contact is started at most once per run, even when the run is interrupted and picks up where it stopped: a contact whose start was cut off part-way is recorded failed (`interrupted`) rather than started again.

Every send lands in **the contact's own conversation on that channel** — the one open with them, or a new one — so their replies arrive in it. It goes out through an enabled channel of the action's connection (an email one needs a verified sending domain).

| Action | Channel | Per contact |
|--------|---------|-------------|
| `send_message` | `whatsapp` | Sends the template to the contact's `phone` |
| `send_message` | `email` | Emails the `text` (Markdown) from the connection's address. A `marketing` email ends with an unsubscribe link and carries one-click unsubscribe headers, which suppress the address with `all` ([Suppressions](#suppressions)); an `operational` one carries neither |
| `send_message` | `app` | Posts the `text` to the WayAI account the contact is linked to ([The Contact Book](#the-contact-book)), as an agent's reply reaches them: unread in their conversation list, with a push when the app is closed |
| `run_agent` | `whatsapp` | The window is that of the conversation the send lands in. When the contact wrote in it in the last 24 hours, the `agent` takes a turn on `instructions` and replies on WhatsApp. Otherwise the `template` is sent first, and the agent answers when the contact replies, with the instructions in its context: WhatsApp refuses any other message outside the window. A new conversation holds no message of the contact's, so the template opens it — for a contact who never wrote, and for one whose previous conversation ended, even if they wrote within WhatsApp's 24 hours |
| `run_agent` | `instagram` | Instagram has no templates. A contact who has not written to the hub in the last 24 hours, in any of their conversations (an ended one included), is skipped (`outside_messaging_window`); for any other, the `agent` takes a turn on `instructions` and replies on Instagram |
| `run_agent` | `email`, `app` | The `agent` takes a turn on `instructions` and replies by email, or in the app |

- **A suppressed contact is skipped** (`suppressed`), before anything is sent or counted against the quota: on WhatsApp its `phone` is checked, on email its `email`, on Instagram its `instagram_sid`, and on the app every identity it holds. A `marketing` automation is stopped by a suppression of either scope, an `operational` one only by an `all` suppression ([Suppressions](#suppressions)). If the suppressions cannot be read for a page, that page sends to no one: its contacts are recorded failed (`suppression_check_failed`), and the next page is checked again.
- **A contact the hub blocked is never sent to** (`contact_blocked`): nothing is sent or recorded in a conversation.
- **The contact book approves its contacts.** On a hub that asks permission before talking to a new contact, a contact an automation reaches is approved by its contact-book entry: no access request is raised for the team, and the contact's replies reach the agent rather than waiting for approval. A contact whose access request was already waiting is approved too, and their conversation stays with the team, as the team's own approval leaves it.
- **The app reaches existing access only.** A contact reaches the app through the WayAI account it is linked to, and only when that account already holds enabled access to the firing hub — otherwise it is skipped (`missing_user_link`, `missing_hub_user`). An automation never grants anyone access, on any channel.
- **`run_agent` runs the named `agent`**, and it stays the conversation's agent, so the contact's replies reach it too. The `instructions` are a system message the contact never sees, kept in the conversation's context for its later turns — a retry, a transfer to another agent, the contact's replies. A contact whose conversation the team holds — one whose access request the automation just approved included — is skipped (`not_agent_track`), and so is every contact on a hub whose AI mode has no pilot (`mode_has_no_pilot`): nothing is sent or charged, and the conversation stays the team's.
- **WhatsApp's marketing opt-out is honoured.** When WhatsApp refuses a marketing message because the person stopped marketing messages from the business, the phone gets a `marketing` suppression: the contact is skipped (`whatsapp_marketing_opt_out`), later `marketing` automations skip them, and `operational` ones still reach them.
- `instructions` are sent as written: a `{{…}}` placeholder in them is not filled in, and reaches the agent as typed. Placeholders work in an agent's own instructions and `additional_context_template` ([agents/instructions.md](agents/instructions.md)).
- **A list a fire cannot use reaches no one.** A fire whose `org_list` names no list of the contact book (deleted, renamed, or never created), or whose `list` names none of the hub's lists, records `list_not_found`, a failed run with no targets, and raises a warning on the hub's Status & Notices (`wayai alerts` lists it). A list deleted while a run reads it ends that run `list_not_found` too, keeping the contacts it reached. A contact book that cannot be read is asked again for about 40 minutes; then the run ends `list_unavailable`. None of them is sent again: the next fire is the schedule's. (Runs from before lists were read a page at a time may show `list_over_cap`, a per-fire size cap that no longer applies.)
- After the suppression check, every page checks the org's operations quota before any of its contacts is started. When the org is over its quota, that page's contacts are skipped (`operations_quota_exceeded`), no later page is read, and the run ends `quota_exceeded`: a scheduled automation's next fire is its first scheduled one after the quota resets, or after a day if that is sooner. If the quota cannot be checked, that page's contacts are recorded failed (`operations_quota_check_failed`), nothing is sent to them, the next page checks again, and a scheduled automation's next fire waits a day. An event automation's next fire is its next event, either way ([Event Triggers](#event-triggers)).
- **Pausing stops a run that is still reading its list** before its next page: it ends `run_stopped`, keeping the contacts it reached. So does deleting the automation, or a change that leaves it unable to run.
- **Runs of a schedule do not pile up.** A scheduled fire that comes while the automation's previous run is still reading its list is skipped, and the schedule's next fire runs as usual. Run now is refused while a run is still going. A run still shown running two days after it started holds back neither ([Enabling, Pausing and Run Now](#enabling-pausing-and-run-now)).

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
- **An event is not run again later** — except a run whose gate could not decide, which is asked again a few times ([Gates](#gates)). A run the org's operations quota stops is recorded skipped; the automation's next run is on its next event. An automation disabled or paused when the event happens does not run for it, even once resumed.
- An event automation has no next run to show. **Run now** runs it once with no event: its task receives no event data.

## Webhook Triggers

On a task hub, an automation can run on each **delivery** an external system sends it, instead of on a schedule. Each accepted delivery is one run: it opens one task (a `none` target) or one per contact of its list, exactly as a scheduled run does ([Task Hubs](#task-hubs)), and every task opens with the **delivery's body, exactly as received**, fenced and marked as data, ahead of the instructions and outside them (with a list, the contact's record follows it). Write the instructions about "this delivery" and let the agent read it. No header of the delivery reaches the task.

```yaml
automations:
  - name: "Order Paid"
    enabled: true
    trigger: { type: webhook, auth: hmac }
    target: { type: none }
    action:
      type: run_agent
      agent: "Ops Agent"
      instructions: "An order was paid. Check its items are in stock and tell the team what to ship."
```

**The URL and the secret.** Each automation receives its deliveries at its own URL — `POST <api origin>/webhooks/automations/<hub_id>/<automation_id>` — and verifies them with its own **secret**. The secret is shown **once**: in the answer to creating the automation in the app or through the API (`POST /api/automations` answers `webhook: { url, secret }`; `GET /api/automations/<automation_id>?hub_id=<hub_id>` answers the `webhook_url` again, never the secret), and in the answer to each **rotate** (the app's **Rotate secret**, or `POST /api/automations/<automation_id>/webhook-secret/rotate?hub_id=<hub_id>`). No read returns it again, and no pull writes it. A rotate replaces it at once: the old secret stops verifying, so update the sender in the same step.

- **Each hub has its own.** A preview hub and its production hub hold different automation ids, so different URLs — and a publish never copies a secret: a webhook automation reaches production with **none**, and an admin of the production hub rotates it there to get one (on production, rotating needs a hub admin; on a preview, `hub:write`). A push creates none either. An enabled webhook automation with no secret takes no delivery, and the hub's **Status** tab shows an alert until one is rotated for it.
- Switching an automation to another trigger drops its secret; switching it back to `webhook` needs a rotate.

**`auth: hmac`** — the scheme a base's [inbound webhooks](bases/integrations.md#inbound-webhooks) use, so one signer works for both: the published SDK's `signRequest` ([bases/executors.md](bases/executors.md#the-sdk-and-the-wire)) produces exactly these headers.

| Header | Value |
|---|---|
| `X-Data-Signature` | `v1,<lowercase-hex HMAC-SHA256>` under the secret over `["v1", id, timestamp, method, path, body].join("\n")` — `path` the URL's path (and query), `body` the exact body sent. A space-separated list of `v1,<hex>` entries is accepted when any one matches |
| `X-Data-Timestamp` | Unix **seconds**, within **300 seconds** of now either way |
| `X-Data-Id` | The delivery's id, bound into the signature. Required |
| `X-Data-Idempotency-Key` | Optional: the intent the delivery carries, kept the same across the sender's retries |

`X-Data-Id` and `X-Data-Idempotency-Key` are each at most **256 characters**; a longer one is refused `401`.

**Repeats.** A delivery is answered as a repeat, and runs nothing, when in the last **10 minutes** a delivery was accepted with the same `X-Data-Id`, or with the same `X-Data-Idempotency-Key` **and the same body** (the key is not signed, so it counts only with the body the signature covers). Past that window a signed delivery's timestamp is stale anyway, but a retry signed afresh under the same id or key is a new delivery and runs again. Give each delivery a stable `X-Data-Id`, or keep its `X-Data-Idempotency-Key` and body across retries — `signRequest` mints a fresh id per call unless it is given one.

**`auth: static_header`** — for a sender that cannot sign: it sends the secret itself in the trigger's `header` (e.g. `X-Account-Key: <secret>`), compared in constant time. Nothing is signed, so there is no time window: a repeat is recognized by its `X-Data-Id`, or its `X-Data-Idempotency-Key` with the same body, within 10 minutes, as above; a delivery carrying neither always runs. The header must be the sender's own — not one the network sets (`Host`, `Content-*`, `CF-*`, `X-Forwarded-*`, …) nor a signing header.

**Answers.** `202 { ok: true, duplicate: false }` when the delivery is accepted (its task opens moments later); `202 { ok: true, duplicate: true }` for a repeat, which runs nothing; `401` when it does not verify (a wrong or stale signature, a missing or wrong header, an id or key over 256 characters); `404` for a URL that takes no delivery (no such automation, not a webhook trigger, or no secret yet); `409` while the automation is disabled, paused or unable to run; `413` for a body over 64 KiB; `429` past a rate limit. An accepted delivery runs only while the automation still may: one paused before its task opens is dropped, not run later. With a [gate](#gates), each accepted delivery's run asks it first: a run it skips opens no task, and one whose gate failed for a reason that may pass is asked again with the same body.

**Limits.** A body of at most **64 KiB**; at most **60 accepted deliveries a minute per automation**, beyond a per-address limit on the URL. Each run is billed as a scheduled run of the same automation is — one operation per task opened, behind the org's operations quota.

---

## Gates

A gate is a decisions monitor that answers its own questions and lets its rules decide. An automation names one gate, of one of two kinds, set by its monitor's trigger:

- a **fire gate** (`trigger: automation_fire`) decides each run **before it starts**: **run** it, **run it with another agent**, or **skip** it. A skipped run reads no list and sends nothing.
- an **item gate** (`trigger: automation_item`, a contact-list target only) decides **each contact** of a run before anything is started for that contact: **start** it, **start it with another agent**, or **skip** it ([Item Gates](#item-gates)).

```yaml
# agents/holiday-check.yaml — the gate's monitor
name: Holiday Check
role: monitor
connection: OpenRouter              # a connection that serves decisions models
settings:
  model: jev-latest                 # a decisions model
instructions: |
  The company is closed on public holidays in Brazil. Judge the run's date.
response_format:
  type: json_schema
  schema_name: gate
  schema_json:
    type: object
    properties:
      holiday: { type: boolean, description: "Is the run's local date a public holiday in Brazil?" }
      urgency: { type: string, enum: [low, high], description: "How urgent is this automation's work?" }
monitor_config:
  trigger: automation_fire
  rules:
    - when: [{ variable: holiday, operator: "=", value: true }, { variable: holiday_confidence, operator: ">=", value: 0.8 }]
      action: { kind: skip }
    - when: [{ variable: urgency, operator: "=", value: high }]
      action: { kind: run_with_agent, agent: "Senior Agent" }
  fallback: { kind: run }
```

```yaml
# hub.yaml
automations:
  - name: "Daily Review"
    trigger: { type: schedule, cron: "0 7 * * *", timezone: "America/Sao_Paulo" }
    target: { type: none }
    action: { type: run_agent, agent: "Ops Agent", instructions: "Review yesterday's open orders." }
    gate: { monitor: "Holiday Check" }
```

- **The gate's monitor** is a `monitor` on `trigger: automation_fire` ([agents/roles-and-settings.md](agents/roles-and-settings.md#an-automations-fire-gate--automation_fire)) — or `automation_item` for an item gate — bound to a **decisions model** — what a save checks — on a connection that serves one (OpenRouter today), which each run checks. A hub may hold one per automation; any number of automations may share one.
- **What it sees:** its own instructions, and the run as data: the automation (name, description, purpose, trigger, target, the action's type, channel and agent), the run's time in UTC and in the automation's timezone (local date, time and weekday), and what fired it (`schedule`, `event`, `webhook`, `run_now` or `retry`). An event's run — and its retries — also sees the event and its conversation, as its tasks receive them ([Event Triggers](#event-triggers)), and a webhook delivery's run the delivery's body, exactly as received, and when it was received ([Webhook Triggers](#webhook-triggers)), as `trigger_data`: data, never inside the instructions.
- **What it decides:** the first rule whose conditions all hold, else `fallback`; with no `fallback`, the run is **skipped**. Its conditions read its answer's fields and their `{field}_confidence` directly — a gate needs no `evaluation_variables`.
  - `run` — the run fires as configured.
  - `run_with_agent` — the run fires with the named agent (by agent name) in place of the automation's own. Only a `run_agent` automation can; the agent must exist, be enabled, and be a `pilot` or `pilot_specialist`, as the automation's own must. Otherwise the run does not fire, records `gate_agent_invalid`, and the automation's alert names why.
  - `skip` — the run does not fire.
- **Each run records the decision** in the run history, with its agent and how sure the gate was: the lowest confidence among the answers the deciding rule read (every answer, when no rule matched).
- **A gate that cannot decide does not run.** A provider failure or a gate slower than 5 seconds records the run `gate_failed` and nothing is sent. A provider's `4xx` (a revoked key, a model the provider refuses) also raises the connection's alert on the hub's Status & Notices. A scheduled run that failed is not retried: the next run is the schedule's.
- **An event's or a webhook delivery's run whose gate failed for a reason that may pass** — a timeout, an unreachable provider, a `429` or a `5xx` — is asked again with the same event or the same delivery's body: a minute after it failed, then 5 minutes after the second failure, then 15 after the third. A fourth failure drops the event or the delivery: the automation's Automations tab shows why, and the hub's **Status** tab shows an alert until it next runs. Any other `4xx` is not asked again. **Run now** is never asked again.
- **A gate that can no longer run** — its monitor deleted, disabled, renamed, moved off `automation_fire` or off a decisions model, or on a connection with no decisions endpoint — skips each run with the reason shown in the Automations tab and an alert on the Status tab, as an automation whose agent is gone does ([Enabling, Pausing and Run Now](#enabling-pausing-and-run-now)). Fix the monitor and the next run fires.
- **Run now** asks the gate too.
- **Cost.** Gate decisions bill the organization one operation for every so many decisions, a number the platform sets; a gate that failed bills nothing. A run the gate lets through also bills its own work as any run does. Each decision is a model call on the monitor's connection, billed by your provider.

### Item Gates

An item gate is asked once for **each contact** of a run, before anything is started for that contact — a send, a turn, a task — so a contact it skips costs no send and opens no conversation.

```yaml
# agents/renewal-fit.yaml — the item gate's monitor
name: Renewal Fit
role: monitor
connection: OpenRouter
settings:
  model: jev-latest
instructions: |
  Decide whether this customer should get this month's renewal offer, from their labels and metadata.
response_format:
  type: json_schema
  schema_name: item
  schema_json:
    type: object
    properties:
      offer: { type: string, enum: [send, senior, skip], description: "What should this customer get?" }
monitor_config:
  trigger: automation_item
  rules:
    - when: [{ variable: offer, operator: "=", value: skip }]
      action: { kind: skip }
    - when: [{ variable: offer, operator: "=", value: senior }]
      action: { kind: start_with_agent, agent: "Senior Agent" }
  fallback: { kind: start }
```

```yaml
# hub.yaml
automations:
  - name: "Renewal Offer"
    trigger: { type: schedule, cron: "0 10 * * 2" }
    target: { type: contact_list, org_list: "renewals-due" }
    action: { type: run_agent, channel: app, agent: "Renewals Agent", instructions: "Offer this month's renewal discount." }
    gate: { monitor: "Renewal Fit" }
```

- **Only on a contact list.** An automation whose target is `none` takes a fire gate only; naming an item gate's monitor there is refused, naming the automation.
- **What it sees:** what a fire gate sees ([above](#gates)), and the contact as data: its id, name, labels and metadata, and the list it was read from. Never its phone, email or Instagram id.
- **What it decides**, for that contact: `start` — its work starts as configured; `start_with_agent` — it starts with the named agent in place of the automation's own (a `run_agent` automation only, the agent held to the same rules as `run_with_agent`; otherwise that contact is recorded failed, `gate_agent_invalid`, and the automation's alert names why); `skip` — nothing is started for it (`gate_skipped`). With no rule matching and no `fallback`, the contact is skipped.
- **Each contact's target records the decision**, its agent and how sure the gate was. The run itself records no gate decision.
- **A gate that cannot decide starts nothing for that contact** (`gate_failed`); a provider's `4xx` also raises the connection's alert. A contact whose gate failed is not asked again in that run: the schedule's next run asks again.
- **Cost.** Each contact's decision counts toward the same billing as a fire gate's — one operation for every so many decisions, a gate that failed billing nothing — and a contact it starts bills its own work.

---

## References

`connection`, `template` and `agent` take a **name**; each has an id twin (`connection_id`, `template_id`, `agent_id`). Give either or both — **the id wins** when both are present. `wayai pull` writes both.

- `connection` is the connection's display name; `template` is matched among the templates of the action's connection. A template name that matches more than one template is refused — name it by `template_id`.
- A reference to an agent the same push creates or renames resolves on that push, by the name the push gives it. A push whose agents are refused writes none of them, so an automation naming one of them is refused too.
- A reference to anything the hub does not hold is refused, naming the automation.
- `gate.monitor` is an agent name only, with no id twin: publishing copies it unchanged, so it names the production copy of the monitor. A push checks it against the agents the push leaves, so a gate and its monitor can be created in the same push.
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
- A **webhook** automation has no schedule: while it is enabled, not paused and able to run, it takes deliveries ([Webhook Triggers](#webhook-triggers)); disabling or pausing it refuses them at once.
- **Able to run** means it still passes the rules it was saved under — its agent, connection and template exist, a `run_agent` automation's agent is enabled and still a `pilot` or `pilot_specialist`, and its gate's monitor can still decide ([Gates](#gates)). One that is enabled and not paused but cannot run is skipped at each scheduled run (a webhook automation refuses each delivery instead): the Automations tab shows why, and the hub's **Status** tab shows an alert until it runs again, or is paused, disabled or fixed. Its list — a hub list or an org list — is read at each run instead: a run that finds none of that name records `list_not_found` and raises its own hub alert.
- **Run now** runs the automation once, immediately, whether or not it is enabled. It is refused while the automation is paused, while a run of it is still in progress, and when it cannot run (the reason is in the refusal).
- **Production** hubs change only by publishing: there an automation is read-only in the app, except **Pause**, **Resume**, **Run now** and a webhook automation's **Rotate secret**, which need a hub admin. Publishing copies automations as written, `list` and `org_list` included, with the hub's lists beside them — an enabled one starts running on production on its own schedule, reaching the members of that list production sees — and never copies a preview's pause, next run or run history. A publish naming a hub list the preview does not hold is refused, naming the automation.
- A preview hub's automation delivers for real — to the contacts of its list that the preview hub sees: only those whose environment is `preview` or `all` ([Visibility](#visibility)).
- A preview automation whose connections — its own, or its agent's tools' — hold the same credential as production shows a warning in the Automations tab: a run from that preview reaches the same systems production does.
- Deleting an automation cancels its next run.

---

## Message Templates

WhatsApp message templates belong to a WhatsApp connection and are managed from that connection's page in the app (create, submit to Meta for approval, check status, send a test). The same templates are sent by `send_message` on WhatsApp, by a WhatsApp `run_agent` to open a conversation whose 24-hour window is closed, and by kanban follow-ups outside the 24-hour window ([kanban.md](kanban.md)).

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
| Webhook delivery body | 64 KiB |
| Webhook deliveries | 60 accepted per minute per automation |
| Webhook `header` | 64 characters |
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

**Who writes them.** The organization's contact-book managers, with either scope, in the Contacts tab (Suppressions, or a contact's page) or the API: `POST /api/contact-book/suppressions` with `{ organization_id, identity: { kind: phone | email | instagram, value }, scope }`; `GET /api/contact-book/suppressions?organization_id=<org_id>` to list (paged by `cursor` and `limit`, newest first); `DELETE /api/contact-book/suppressions/<suppression_id>?organization_id=<org_id>` to remove; `GET /api/contact-book/contacts/<contact_id>/suppressions?organization_id=<org_id>` for one contact's. And the person themselves: through the unsubscribe link every marketing automation email carries, which suppresses their address with `all` — no sign-in, and their mail app's unsubscribe button does the same; and through WhatsApp, which refuses a marketing message to someone who stopped the business's marketing messages — their phone is then suppressed with `marketing`, shown as a WhatsApp opt-out.

Suppressions are checked only on an automation's sends; an agent's replies in a conversation the person is having are never stopped.

### CSV import

Contacts import from a CSV file in batches of at most **500 rows**. Each batch states its visibility (required: all hubs, or scope tags), its environment, and optionally labels added to every row. In the file, a labels column separates labels with `;` or `|`, or with commas inside a quoted cell. A row whose phone, email or Instagram id another contact (or an earlier row of the same batch) already holds is not imported and is reported as a duplicate.

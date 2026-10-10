# Support Inbox

What a hub's support team (**Hub Team Users**) and its **Hub Admins** do in Support: the inbox, the board, and the controls inside one conversation, on the web and in the mobile app. Use it to answer someone who handles conversations, not someone who builds the hub. Every link comes from [navigation.md](navigation.md): hand over one link, never a breadcrumb. What the end user sees on their side is in [using-the-app.md](using-the-app.md).

## Table of Contents
- [Who Has Support](#who-has-support)
- [The Inbox](#the-inbox)
- [The Board](#the-board)
- [Claiming a Conversation](#claiming-a-conversation)
- [Transferring](#transferring)
- [Closing](#closing)
- [Replying](#replying)
- [Copilot Suggestions](#copilot-suggestions)
- [Consulting a Consultant](#consulting-a-consultant)
- [Approving Contacts](#approving-contacts)
- [The Contact's Context and History](#the-contacts-context-and-history)
- [Flags](#flags)
- [Notifications](#notifications)
- [Support in the Mobile App](#support-in-the-mobile-app)

## Who Has Support

- **Who sees it:** the **Support** nav shows for a person on one of a hub's teams, or a hub admin. An organization admin who is neither does not get it.
- **Who can act on a conversation:** claiming and replying need a seat on one of the hub's teams. A hub admin on none of them sees every conversation of the hub and can **Approve**, **Block** and **Unblock** contacts, but is refused a claim ("Caller is not a team member on this hub") and a reply until they join a team on the Users tab below.
- **Who sets it up:** hub admins, on the hub's Users tab, `https://app.wayai.pro/settings/organizations/<orgId>/hubs/<hubId>/users?subtab=teams`. It holds:
  - the hub's teams and their members;
  - the **Support Model**: one shared queue the team claims from, or each customer's conversations owned by an assigned team;
  - who may approve new contacts (see [Approving Contacts](#approving-contacts)).

## The Inbox

**`https://app.wayai.pro/support`** lists the conversations of every hub the person supports, most recent activity first.

**Tabs.** `?status=agent|team|ended` opens a tab. Agent and Team show a count.

| Tab | Holds |
|-----|-------|
| **Agent** | Conversations the AI is handling. |
| **Team** | Conversations handed to the team, and conversations whose scheduled event is due (**Event due**). |
| **Ended** | Closed conversations, most recently closed first. **See archived conversations** reaches older ones. |

**Which conversations a team member sees:**
- those belonging to their own teams;
- those not assigned to any team;
- those they have claimed.

A hub admin who is on none of the hub's teams sees all of the hub's conversations, but must join a team to claim or reply ([Who Has Support](#who-has-support)).

**Each row** shows `hub - contact`, the last message, and:
- a claim badge on Team rows: **Unclaimed**, **Claimed by you** or **Claimed by other**;
- **Pending approval** or **Blocked** for a contact awaiting approval or blocked;
- **Event due**;
- a flag (see [Flags](#flags));
- the unread count. Opening a conversation clears its count only once the person has claimed it.

**Header controls:**
- **Search:** on Agent and Team it matches the hub, the contact or the last message (and a task's title on a row with no contact), over the conversations already loaded. On Ended it searches recently ended conversations, and **Search archived instead** searches all of them.
- **Environment filter:** a globe icon, shown when the person's hubs include both previews and production.
- **List / board toggle:** see [The Board](#the-board).

**The Support count** on the nav is the number of people waiting on the team. It counts each person once, in conversations that are unclaimed or claimed by you and have unread team messages or a due event.

## The Board

**`https://app.wayai.pro/support?view=kanban&hub=<hubId>`** opens the board for one hub. The icon in the inbox's header switches between the list and the board, and the board's hub picker changes the hub. The board groups conversations in one of two ways:

- **By conversation status:** columns Agent, Team and Ended. Each open card carries its kanban status as a badge, with a menu to change it.
- **By kanban status:** one column per status of the hub, with the terminal status last. This is the default on a hub that has kanban statuses. `&lane=<lane slug>` narrows the board to one lane; its statuses show with the initial and terminal statuses.

**Changing a card's kanban status:** drag the card to another column (kanban-status grouping), or pick from its badge's menu (conversation-status grouping). These rules apply:

- **Allowed moves:** a status may allow only certain next statuses. Leaving the terminal status is refused, and ended cards can't be moved.
- **Statuses that ask first** open a form before the move:
  - **The terminal status** closes the conversation. It warns first and asks for an **Outcome** when the hub defines outcomes.
  - **A scheduling status** asks for the event's date and time.
  - **A status with an additional-context form** asks for it.
- **Inside a conversation** its kanban status is shown, not changed. Change it on the board.

**Clicking a card** opens the conversation in a side panel.

What the statuses, transitions, outcomes and lanes are, and how a builder sets them, is in [kanban.md](kanban.md).

## Claiming a Conversation

**`https://app.wayai.pro/support/<conversationId>`** opens one conversation. Its header names the contact and the hub, and shows badges for its status (Agent, Team or Ended), its kanban status, and who holds it.

**A team member replies only to a conversation they hold**, and claiming needs a seat on one of the hub's teams ([Who Has Support](#who-has-support)).
- **Claim** makes the person responsible for the conversation, and is offered on any open conversation they don't hold, including one the AI is handling. The conversation moves to the team, so the AI stops answering the customer, and the message box opens.
- **Take over** is offered when another team member holds the conversation. It asks to confirm, then moves the conversation to the person.
- **A contact awaiting approval** is the exception: a team member allowed to approve contacts can reply without claiming, and the reply approves them ([Approving Contacts](#approving-contacts)).
- **There is no release button.** A held conversation leaves the person through a transfer or a close.

## Transferring

**Transfer** is offered while the person holds the conversation:

- **Transfer to Agent** hands the conversation back to one of the hub's AI agents, which answers the customer again. It is offered only on a hub whose AI answers customers (AI mode `pilot` or `pilot+copilot`).
- **Transfer to Team** releases the conversation, unclaimed, to the hub's team queue. There a member of any of the hub's teams can claim it.

## Closing

**Close** is offered while the person holds the conversation. It asks to confirm.

- **Outcomes:** on a hub whose terminal status declares outcomes ([kanban.md → Outcomes](kanban.md#outcomes)), pick the **Outcome** first. A conversation already in the terminal status can **Keep the outcome already recorded**.
- **No reopening:** an ended conversation is read-only and can't be reopened.
- **What comes next:** on a chat hub, the customer's next message starts a new conversation. On a task hub, the end user starts a new task.

## Replying

- **What it can send:** the message box works as the end user's does, with files, a photo, and voice messages on hubs that transcribe them ([using-the-app.md → Writing a Message](using-the-app.md#writing-a-message)).
- **Every reply goes to the customer.** The only private messages are consults ([Consulting a Consultant](#consulting-a-consultant)).
- **The 24-hour window (WhatsApp and Instagram):** these channels close it 24 hours after the customer's last message, and the box closes with it.
  - **On WhatsApp**, the person holding the conversation can **Send approved template**: one of the WhatsApp connection's Meta-approved message templates, with its variables filled in. Templates are managed on the connection's page, `https://app.wayai.pro/settings/organizations/<orgId>/hubs/<hubId>/connections/<connectionId>` (see [automations.md → Message Templates](automations.md#message-templates)).
  - **On Instagram**, wait for the customer to write again.
- **Voice calls:** what a call looks like to the team is in [calls.md → What the Team Sees](calls.md#what-the-team-sees).

## Copilot Suggestions

On a hub whose AI mode is `copilot` or `pilot+copilot`, the AI drafts replies for the team while a person handles the conversation:

- **Where they appear:** a suggestion arrives in the thread for the team only, marked with a robot and a headset icon under it. Clicking it copies its text into the message box (and the clipboard) to edit and send.
- **Asking for one:** **Suggest reply** in the message box drafts one now ("Drafting a suggested reply…"). When the AI can't suggest one, it says "The Copilot can't suggest a reply for this conversation right now."
- **When they come on their own:** a Copilot drafts a suggestion for each new customer message, unless it is set to draft only when asked ([roles-and-settings.md → Copilot suggestion trigger](agents/roles-and-settings.md#copilot-suggestion-trigger-all-llm--harness-connectors)).

## Consulting a Consultant

On a hub with `consultant` agents, the team can ask one a question in private.

- **Asking:**
  - Switch the message box from **Reply** to **Consult**, then type `@` to tag a consultant.
  - The box shows "Internal — not sent to the customer".
  - The consultant's answer appears in the thread inside a consult block, for the team only. The customer never sees a consult.
- **The Consult threads panel** above the messages keeps parallel consults apart:
  - **New thread**, with an optional name and a consultant;
  - **Rename**, **Switch consultant**, and **Reply here** to point the message box at a thread.
  - A thread is **Open**, **Awaiting**, **Resolved** or **Closed**. Closing the conversation closes its threads.
- **Threads an AI agent started** are listed as a record ("Consulted <name>"), without **Reply here**.
- **Requirements:** consulting needs the person to hold the conversation, and attachments are off in Consult mode.

How a builder adds consultants: [roles-and-settings.md → Role Reference](agents/roles-and-settings.md#role-reference).

## Approving Contacts

A hub that asks permission for new contacts (its `non_app_permission` and `access_approval_role` settings, [SKILL.md → Hub Settings](../SKILL.md#hub-settings)) holds them as **Pending approval** until someone decides. While a contact is pending, the AI stays silent.

- **The controls** sit in the conversation's header: **Approve**, **Block** (asks to confirm) and **Unblock**.
- **Replying to a pending contact** approves them, for a member of one of the hub's teams who may decide.
- **Who may decide:**
  - hub admins always;
  - team members too, when the hub lets the team approve (on the Teams sub-tab above).
  - Anyone else sees only the badge.

## The Contact's Context and History

- **The contact's name** in the conversation header opens **User context**. It shows the contact's identity and the hub's saved state about them and about this conversation. Hub admins can rename the contact and clear a saved value.
- **The History button** lists the contact's ended conversations with the hub.
- **On a chat hub**, scrolling up also shows earlier conversations.

## Flags

A flag on a row or a card means the hub's conversation evaluator, a monitor, or both flagged the conversation for the team to look at: **Flagged by evaluator**, **Flagged by monitor**, or **Flagged by evaluator and monitor**. What makes them flag is the builder's configuration; see [roles-and-settings.md → Flag Conditions](agents/roles-and-settings.md#flag-conditions-conversation_evaluator-only) and [Monitor Configuration](agents/roles-and-settings.md#monitor-configuration-monitor-only).

## Notifications

| Where | What notifies the person |
|-------|--------------------------|
| A browser tab | Nothing: the Support count and the unread counts show what is waiting. |
| The desktop app | System notifications for support work: [navigation.md → Desktop app](navigation.md#desktop-app). |
| The mobile app, while the person has WayAI open nowhere (no app, browser tab or desktop app connected) | "Conversation transferred" when a conversation is handed to the team on a hub the person supports. Tapping it opens the conversation in Support. A customer's new message does not notify. |

## Support in the Mobile App

The mobile app's **Support** tab shows the same conversations as `/support`.

- **The list:**
  - tabs **Agent**, **Team** and **Ended**. The app opens on Team when it has conversations, and on Agent otherwise.
  - search;
  - the claim, approval and flag badges, as on the web.
- **The controls in a conversation:**
  - **Claim** and **Take over** (asks to confirm).
  - **Transfer**, to one of the hub's teams or AI agents.
  - **Close**. On a hub with outcomes it asks for the outcome first; otherwise it closes at once, without asking.
  - **Suggest reply**.
  - **Approve**, **Block** and **Unblock**.
- **Not in the app — use the web for these:**
  - the board, and changing a kanban status;
  - starting a consult (consult messages show as "Internal consult");
  - sending a WhatsApp template once the 24-hour window has closed.

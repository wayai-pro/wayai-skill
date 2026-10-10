# Using the App

What a person who talks to a hub's AI agents (a **Hub User**) can do in the WayAI app: on the web, in the mobile apps and in the desktop app. Use it to answer someone who uses agents, not someone who builds them. Every link comes from [navigation.md](navigation.md): hand over one link, never a breadcrumb. What the support team does with these same conversations is in [support-inbox.md](support-inbox.md).

## Table of Contents
- [Where to Start](#where-to-start)
- [Chat Agents](#chat-agents)
- [Task Agents](#task-agents)
- [Writing a Message](#writing-a-message)
- [Files the Agent Sends](#files-the-agent-sends)
- [When a Person Answers](#when-a-person-answers)
- [Your Account](#your-account)
- [Notifications](#notifications)
- [The Mobile App](#the-mobile-app)
- [The Desktop App](#the-desktop-app)

## Where to Start

- **`https://app.wayai.pro/chat`** lists every agent the person can talk to, most recent activity first. An agent shows up there once the person is given access to it, before they ever write to it. The list's search finds an agent by its name or by the text of its last message.
- **`https://app.wayai.pro`** opens that list. Where a person with no agents lands instead: [navigation.md → Top-Level Map](navigation.md#top-level-map).
- **No agents listed** ("No agents yet. Agents you're given access to appear here.") means no one has given them access yet. Access comes from the people who run the hub, and a person cannot add an agent themselves. A link to an agent they cannot use shows "This agent isn't available to you."

Each agent is one of two kinds, and they behave differently:

| Kind | What the person sees | Section |
|------|----------------------|---------|
| Chat agent | One ongoing conversation | [Chat Agents](#chat-agents) |
| Task agent | A list of tasks: each request is its own conversation | [Task Agents](#task-agents) |

When you don't know the agent's id, hand over `https://app.wayai.pro/chat` and name the agent.

## Chat Agents

**`https://app.wayai.pro/chat/<hubId>`** opens the agent's ongoing conversation.

- **The conversation can end.** The agent or the support team closes it, or the hub closes it after a stretch with no messages. The message box stays open: the person's next message starts a new conversation with the same agent.
- **Earlier conversations** sit above the current one. Scrolling up shows each one after a divider with the date and time it started. Past the first few, **Load earlier conversations** loads more. The **History** button (clock icon) in the header lists the person's ended conversations with this agent, and each opens to read.
- **One conversation** opens at `https://app.wayai.pro/chat/<hubId>/conversations/<conversationId>`.
- **Voice calls:** on a chat agent set up for voice calls, **Start a voice call** sits beside the message box. See [calls.md → Placing a Call](calls.md#placing-a-call).

## Task Agents

**`https://app.wayai.pro/chat/<hubId>`** opens the agent's task list, headed "Each request is its own task".

- **Tabs:** **In progress**, and **Ended** (`?status=ended`).
- **New task** opens an empty task, and the person's first message creates it. The task then gets its own link, `https://app.wayai.pro/chat/<hubId>/conversations/<conversationId>`. It is titled with that first message: a voice message's task is titled "Voice message", and a task started with only a file takes the file's name.
- **Never hand over a new-task link** ([navigation.md → Chat](navigation.md#chat)): hand over the task list and tell the person to press **New task**.
- **An open task's header** reads `agent › task`, and the agent's name leads back to the list.
- **The person cannot close a task.** The agent or the support team closes it. An ended task is read-only ("Conversation has ended"), so for more help the person starts a new task.
- **Badges on a task's row:**
  - **Unclaimed**: the support team has it but no one has picked it up yet.
  - **Claimed by other**: a team member is handling it.
  - **Event due**: a date set on the task has passed.

## Writing a Message

- **Enter** sends, and **Shift+Enter** starts a new line.
- **Attach a file** with the paperclip. Any file type works, and several files can be picked at once. One message carries at most 20 files and about 7 MB in total. Files can be sent without text.
- **Take a photo** is offered on a phone or tablet.
- **Record audio** (the microphone) is offered when the hub transcribes voice messages, in place of the send button while the box is empty. The person's voice message shows what it was transcribed as.
- **A sent message cannot be edited or deleted.**
- **Your own message's status** shows next to it: Sending, Sent, Delivered, Read, or Failed to send.

## Files the Agent Sends

- **Images** show in the thread, and **audio** plays in place. Other files show by name.
- **Clicking a file** opens it in a viewer. A file the viewer cannot show offers a download.

## When a Person Answers

- **The agent can hand the conversation to the hub's support team**, and the team can hand it back. Nothing in the thread announces the switch unless the agent's own reply says so.
- **The team's replies** arrive in the same thread, marked with a headset icon under the message. The AI's replies carry a robot icon with the agent's name.
- **The typing dots** show only while the AI is answering.

## Your Account

**`https://app.wayai.pro/settings/account/profile`** holds:

- the person's name and email;
- **Language**: English, Português or Español, the language of the app;
- **Dark Mode**.

## Notifications

| Where | What notifies the person |
|-------|--------------------------|
| A browser tab | Nothing: unread counts show on the Chat nav and on each row. |
| The mobile app, while the person has WayAI open nowhere (no app, browser tab or desktop app connected) | A reply from the agent, an answer the agent sends after a call, and a message the hub sends on its own. A support team member's reply does not notify: it shows as unread when the app is opened. Tapping a notification opens the conversation. |
| The desktop app | Its notifications are for support work, never for the person's own conversations ([navigation.md → Desktop app](navigation.md#desktop-app)). |

## The Mobile App

The iOS and Android apps hold the same agents and conversations as the web. This describes the app as it is built today: an older install may lack parts of it (such as the Community tab, the camera and Photo library, links in a message opening in the app, or **Withdraw AI consent**) until it is updated.

- **Tabs:** **Chat**, **Support** and **You**, plus **Community** for a member of a community.
  - **Chat** is the list at `/chat`.
  - **Support** is the team inbox, which reads "No support conversations." for someone on no hub's team.
- **Chat agents:** a chat agent opens its ongoing conversation, with earlier conversations above it as the person scrolls up.
- **Task agents:** a task agent opens its task list:
  - **In progress**;
  - **Ended**, with older tasks behind **See archived conversations**;
  - **New task**.
  - Each task opens as its own thread titled `agent › task`, the agent's name leading back to the list.
- **Attaching:** the paperclip offers **Photo library** and **Files**, and the camera button takes a photo. A single file can be at most 20 MB, and the same 20 files and about 7 MB per message apply.
- **Voice messages:** the microphone replaces the send button while the box is empty, on hubs that transcribe voice messages.
- **Links inside an agent's message:**
  - A link to a WayAI screen the app has opens that screen in the app, and other pages of the web app open in the phone's browser ([navigation.md → Mobile app](navigation.md#mobile-app)). A new-task link opens the agent's task list.
  - Links to other sites open in an in-app browser.
  - So hand over the same web links as for the web.
- **The You tab:**
  - **Display name**, **Language** and **Dark mode**;
  - **Sign out**;
  - **Withdraw AI consent**: signs the person out, and the AI notice is asked again at the next sign-in;
  - **Delete account**.
- **Signing in:** Google, Microsoft, Apple, or a code sent by email.
- **Updates:**
  - An app too old to use shows **Update required**, and its **Update** button opens the app's store page.
  - A newer version can show **Update available**, which **Not now** dismisses.

## The Desktop App

The desktop app is the web app in its own window, with the same screens. To get it, and for what it adds over a browser tab, see [navigation.md → Desktop app](navigation.md#desktop-app).

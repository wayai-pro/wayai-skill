# Pages

A **page** is a small working tool a hub offers its people: a coach entering swim-meet results at the pool, a back-office review queue, an athlete checking their own results. It is one self-contained HTML file plus a manifest. The page reads and writes the hub's [bases](bases/README.md) through the hub's **Base Toolset** connections ([connections.md](connections.md#base-toolset)), calling only the toolset tools its manifest names. A page is an [MCP Apps](https://blog.modelcontextprotocol.io/posts/2026-01-26-mcp-apps/) view, and WayAI is its host. A coding agent that knows MCP Apps already knows the page API.

Pages are hub config: you write them in the hub folder, push them to a preview with `wayai push`, and publish them with the rest of the hub. Each page serves one audience: the hub's **team**, or its **customers**, each of whom reaches only their own records and shared ones ([Customer Pages](#customer-pages)).

Start a page with `wayai page new <slug>`, which writes one already styled like the app, and read [Designing a Page](#designing-a-page) before you change its look.

> **Where people open them.** On web, a Hub User given access on the Users tab opens a customer page from the header of their own conversation with the hub, the one `https://app.wayai.pro/chat/<hubId>` opens, before its first message too; in the mobile app, from the overflow menu (**⋮**) of that same conversation, which lists the hub's customer pages with `show_in_conversation: true` (and no **All pages**), opening one in the same full-screen sheet as a team page. On web, a team member opens a team page from the Support nav: from a conversation's header, which offers the hub's pages with `show_in_conversation: true`, and from the hub's Pages section, which lists every page offered to them; that header and the Support nav's **Pages** item open it. In the mobile app, a team member opens a page from a support conversation's overflow menu (**⋮**), which lists the pages with `show_in_conversation: true` and then **All pages** for the rest, or every page offered to them when none sets it; the page opens in a full-screen sheet over the conversation, closed by its close button or by dragging its top bar down. The mobile app has no Pages section. In either app, a page also opens from a page card an agent sent in the conversation ([Page Cards](#page-cards)).

## Table of Contents
- [Folder Layout](#folder-layout)
- [The Manifest](#the-manifest)
- [Who Sees a Team Page](#who-sees-a-team-page)
- [Customer Pages](#customer-pages)
- [Page Cards](#page-cards)
- [Push, Pull, Diff and Publish](#push-pull-diff-and-publish)
- [Writing a Page as an MCP Apps View](#writing-a-page-as-an-mcp-apps-view)
- [Live Refresh](#live-refresh)
- [Designing a Page](#designing-a-page)
- [The Sandbox: What a Page Cannot Do](#the-sandbox-what-a-page-cannot-do)
- [Limits](#limits)

## Folder Layout

```
hubs/<hub-folder>/
└── pages/
    ├── results/
    │   ├── page.yaml      # the manifest
    │   └── index.html     # the page: one self-contained HTML file
    └── review-queue/
        ├── page.yaml
        └── index.html
```

- **The folder's name is the page's slug**: lowercase letters, digits and single hyphens between them (`swim-results`), at most 64 characters. You can leave `slug:` out of `page.yaml`; if you write it, it must match the folder's name.
- **A slug never changes.** Relabel a page through `title`. Renaming the folder makes a new page and deletes the old one.
- **A page is both files.** The CLI refuses a page folder with only one of them, a `page.yaml` that is not a YAML object, a slug other than the folder's name, or HTML that is empty or over the size cap. It never skips such a folder, because a skipped page would be deleted on push.
- **Your other files are left alone.** The CLI ignores any folder in `pages/` that holds neither file, and any loose file in `pages/`. Pull writes only `page.yaml` and `index.html`. When a page is deleted, pull removes only those two files, so you can keep sources or notes beside a page.

## The Manifest

```yaml
# pages/results/page.yaml: for the coaches
slug: results
title: Resultados
description: Enter times and placings per athlete and race
audience: team
team_writes: true               # team users without a grant may write through this page
show_in_conversation: true
order: 1
params:                         # what a link or a page card may open the page with: a JSON Schema object
  type: object
  properties:
    athlete: { type: string }
tools:                          # tool names must be unique across the manifest
  - { connection: swim-base, tool: list_athletes, notify: none }
  - { connection: swim-base, tool: update_result, notify: summary }
  - { connection: swim-base, tool: delete_result, notify: summary }
  - { connection: swim-base, tool: team_points,   notify: none }   # a SQL Action
```

| Field | Required | Meaning |
|-------|----------|---------|
| `slug` | No (the folder's name) | The page's immutable id |
| `title` | Yes | 1–120 characters; what menus show |
| `description` | No | Up to 500 characters; shown under the title in web menus (the mobile app's menus show titles only) |
| `audience` | Yes | `team` or `customers` ([Customer Pages](#customer-pages)) |
| `team_writes` | No (`false`) | A team page's: lets team users without a grant on the hub run the page's tools that change data, not only its reads ([Who Sees a Team Page](#who-sees-a-team-page)). Push refuses it on a customer page |
| `customer_writes` | No (unset) | A customer page's: lets each customer run its tools that change data, on their own records of a record type with person entries, and on every record of a `{ shared: true }` type ([Customer Pages](#customer-pages)). Push refuses `true` on a team page |
| `show_in_conversation` | No (`false`); `true` on a customer page | Offers the page from the conversation header to people who may open it. On a team page it orders what a conversation offers first and never hides a page: on web every team page offered is in the Pages section, and in the mobile app every page offered is in the conversation's overflow menu ([Where people open them](#pages)). The header is a customer's only way into a page, so push refuses a customer page without it |
| `allow_messages` | No (unset, which is `false`) | Either audience's: lets the page put text in the person's own message box in the conversation it was opened from, for them to send or not. It never sends ([Putting Text in the Person's Message Box](#putting-text-in-the-persons-message-box)) |
| `order` | No (`0`) | Menu position, 0–1000, lowest first; pages of equal order sort by slug |
| `params` | No | A JSON Schema whose root is `type: object`, at most 16 KiB of JSON: the values a link or a [page card](#page-cards) may open the page with |
| `tools` | No (`[]`) | Up to 50 toolset tools the page calls, each `{ connection, tool, notify }` |

**Each tool** names a hub connection by its name in `hub.yaml` (`connection:`) and a tool by its name in that connection's synced tool list (`tool:`). The page calls a tool by its name alone, so no two entries may share a tool name, even across connections. Push refuses an entry when:

- the hub has no connection of that name, or two connections share it;
- the connection is not a Base Toolset;
- the connection has no token yet (save it first);
- on a team page, the connection still records its toolset as customer-scoped, from before bases lost [customer scope](bases/toolsets.md#customer-scope) (sync its tools to record it again; on production, which can't sync, save its token again);
- on a customer page, the customer-facing rule refuses the toolset; the refusal says what it reaches ([Customer Pages](#customer-pages));
- the tool is not in the connection's synced tool list, or is turned off there (sync its tools first: `wayai sync-mcp --connection "<name>"`);
- the tool hands back a signed file URL, which a page never receives.

**`notify`** (`none`, the default, or `summary`) is the most the conversation's agent hears of the page's calls to that tool. Put `summary` on the tools whose calls the agent should hear of: in practice, the write tools. A customer page's tools are all `none`: push refuses `summary` on one, since the agent hears nothing of a customer page's calls yet.

- **What is noted:** a `summary` call that the tool answered, made on web from a page opened from a conversation's header. It is noted in that conversation, and the agent reads the note on its next turn.
- **Only where the person may send:** a call is noted only when the person using the page may send a team message in that conversation, and, when the conversation has a contact, only once that contact is approved. A pending or blocked contact's conversation is never noted, and a page call never approves a contact.
- **What a note holds:** the page, the team member who called, and, per tool and record type, the number of calls and the ids of the records they answered (UUIDs only, at most the newest 10). Never the arguments, field values or any text from the page. One note per person and page, updated in place until the agent has read it.
- **Never noted:** a `none` call, a call the tool refused or that failed, a page opened from the Pages section, a page in the mobile app, and every call of a customer page.

The platform decides whether a tool reads or writes from the toolset, never from the manifest. Anyone a page lets write may call any tool it declares, and anyone it offers reads only may call any of its read tools, each with any arguments that tool's schema accepts, whatever the page's own code does ([Who Sees a Team Page](#who-sees-a-team-page), [Customer Pages](#customer-pages)), so declare only the tools the page needs. The toolset's own rules (`writable_fields`, `base_filter`, `precondition`: [bases/toolsets.md](bases/toolsets.md)) still hold for every call.

## Who Sees a Team Page

A team page is offered to everyone who may see the hub's conversations: an organization member or hub admin whose grant on the hub includes `conversation:read`, and each of the hub's team users. What the page does for them depends on whether they may write:

- **Everything the page declares:** a grant on the hub that includes `conversation:write`, or a team user on a page that sets `team_writes: true`.
- **Reads only:** everyone else. The platform runs only the tools the toolset itself classifies as reads: a record `get` or `list`, a relationship or file `list`, a file `get`, a SQL Action, `sql_query` and `describe_graph`. Any other tool, including one it cannot classify, is refused with nothing changed, as an `isError` result whose text says the person can view the page but not change data through it. Design the page so it still works for them: show the data, and expect its writes to be refused.

On web and in the mobile app, the page's top bar, drawn by WayAI outside the page, shows a reads-only viewer **View only**, and the page hears it as `wayai.viewerLevel: "read"` in the [host context](#what-every-page-gets): hide or disable its write controls then. The host itself refuses such a viewer's call to a tool it knows is not a read, without sending it, with the same kind of result: `isError: true`, a text for the viewer, no `structuredContent`. A call whose kind the host does not know (the platform could not read the kinds as the page opened) is sent, and the platform runs it only if it is a read. The platform refuses a write either way.

Anyone else does not see the page. The base records each write a page makes as made by **the person using it**, with their email in its audit. A page still writes as a tool does: a write to a reviewed record type stays a draft, as an agent's does.

## Customer Pages

A customer page (`audience: customers`) serves the hub's customers: each person using it reaches only their own records and shared ones, through the hub's Base Toolset connections, as a customer-facing agent does.

```yaml
# pages/my-results/page.yaml: for the athletes
title: My results
audience: customers
show_in_conversation: true      # required: the conversation header is a customer's only way in
tools:                          # record types with person or inherit entries, or shared access; notify is always none
  - { connection: athlete-base, tool: list_my_results, notify: none }
```

**Who sees one.** The hub's **Hub Users**, signed in to the WayAI app: people given access to the hub (Hub → Users tab), and, on an account-mode hub (`support_model: account`), the customers it adds automatically the first time they write to it and the hub admits their contact. A Hub User whose entry is disabled sees none, and neither does one the hub has blocked (*Blocked people*, below). A customer added automatically is admitted to the hub's customer pages, but the WayAI apps give them no way in yet: the hub is not in their app, so no conversation header offers them its pages, and they cannot send in the hub's in-app chat, which access on the Users tab gives. On web, a Hub User given access on the Users tab opens a customer page from the header of their own conversation with the hub ([Where people open them](#pages)), which lists the hub's customer pages in their `order`: one opens with a tap, several open a menu. In the mobile app they open it from that conversation's overflow menu (**⋮**), which lists the same pages in the same order. A customer never sees a team page there, and never the Pages section, which is the team's. Organization members, hub admins and team users do not see a customer page unless they are also a Hub User of the hub; someone who is both sees the team pages from the Support nav and the customer pages from their own conversation.

**Whose records.** Each call is for the person using the page, signed in to WayAI: it names them as the one person it is for, by their person set — their [person id](bases/access.md#person-entries) and the ids the organization [links](bases/access.md#person-links-and-the-deny-list) to them, a contact among them only where the hub sees it — and it is a [customer-facing call](bases/access.md#customer-facing-calls). So it reaches a record type only when the type declares person entries in `enforce`, and then only the records whose person fields name one of those ids, or [inherit entries](bases/access.md#inherit-entries) in `enforce`, and then only the records whose referenced record passes for them, or `x-access: [{ shared: true }]`. Declare one of them on every record type the page must reach; a type with none holds nothing for it. Nothing else stands for the person: the page needs no contact linked to their account, and a record keyed by a contact plays no part.

**Blocked people.** The page is **not available to their account** when the hub has blocked them, whether the block names their account or any phone, email or Instagram id of the contact the organization's contact book links to their account (as the hub sees the book: its scope tags and environment), read again on every open and every call: the page's read and each call are refused with `PAGE_CUSTOMER_UNAVAILABLE`, nothing is sent to the base, and on web and in the mobile app the page shows "This page is not available to your account." in place of the page. The list of customer pages is refused too, so no conversation offers them one, and they are told nothing more than that. The same 403 answers the page's read and each call when the book links several contacts to their account, or one outside the hub's scope tags or environment, since the hub can't tell whether it blocked that contact. Unblocking gives the pages back from their next open or call; a contact awaiting approval keeps them. A block does not touch team pages.

**Which toolsets it may declare.** Those the customer-facing rule admits, the rule a customer-facing agent's Base Toolset connections are held to ([connections.md → Base Toolset](connections.md#base-toolset), *Customer-facing agents*): records, which the base confines by the rules above, and never a toolset that exposes `sql_query` or `batch`, a file `put`, `delete` or `put_url`, a write of a reviewed record type, or a tool WayAI does not classify. A [SQL Action](bases/toolsets.md#sql-actions) is served to its call only when every record type it declares holds `{ shared: true }`.

Push and publish refuse any other toolset on a customer page, saying what it reaches. Before every call the platform checks the rule again against what the connection records, a connection recorded before its file writes, review writes or unclassified tools were counting as reaching them; for a toolset whose every record type is a reviewed `shared` type it only reads, also against what the toolset reaches at that moment: a toolset the rule now refuses is refused, with nothing sent, and the connection raises a tool credential alert. While a customer page names a connection, saving that connection a token, changing its settings or syncing its tools in a way that would break the rule is refused, naming the page, and so is giving its name to a connection the rule refuses.

**Customer-scoped toolsets.** Bases no longer have [customer scope](bases/toolsets.md#customer-scope). A connection that recorded its toolset as customer-scoped before refuses a page's calls, with nothing sent, and raises a tool credential alert, until it records the toolset again: sync its tools (on production, which can't sync, save its token again with `wayai set-connection-credential`).

**What a customer can do.**

- **Reads only, by default.** The platform runs only the tools the toolset itself classifies as reads, as for a team page's read-only viewer ([Who Sees a Team Page](#who-sees-a-team-page)); any other tool is refused with nothing changed.
- **Writes, only with `customer_writes: true`,** on a record type with person entries held by the base to the person using the page ([Person writes](bases/access.md#person-writes)): a record they write must still name them, and a person field may gain only their own id. A `create` that leaves out the type's `owner` field is filled with the person using the page, as an agent's is with its conversation's customer ([the owner field is filled](bases/access.md#person-writes)): their one linked contact the hub sees, else their person id. So the page creates the person's own records without knowing their id, which it is not told; a `create` naming anyone else in a person field is refused. **A `{ shared: true }` record type is written by every person using the page**, held only to its dimension entries: a write tool on such a type (a calendar, a price list) lets any customer change or delete any of its records, which every other customer then sees. Declare write tools only on types each customer may change for everyone.
- The page's top bar shows **View only**, and the host context carries `wayai.viewerLevel: "read"`, on a page without `customer_writes: true`; with it, `write`.
- The base records each write a customer page makes as made by the page itself, labelled with its title, never by the person using it, for the person the call is for.

**Why writes are opt-in.** Without a page, a customer's writes go through the agent and its instructions. A page with `customer_writes: true` lets the customer call each write tool it declares directly, with any arguments the tool's schema accepts, whatever the page's own code does: the person using a page can send any call the page could. Set it only for changes you would let a customer make on their own, such as an athlete correcting their own race times.

**A create tool tells a customer whether a key is taken.** A record `create` whose `external_id` a record the person cannot reach already holds is refused with `CONFLICT`, and a free key creates the record. So a customer page that declares such a create tool, with `customer_writes: true`, lets any customer try keys and learn whether another customer's record, or any record they cannot reach, holds one, though nothing of that record ([Person writes](bases/access.md#person-writes)). A link `create` does the same with a link's `external_id`. So key those record types by values no one can guess (random ids), never by something a customer should not learn is in use, such as another person's email or document number; or leave the create tool off customer pages.

**Live refresh.** A customer page never hears another customer's activity. Its `wayai.refresh` rises when the person comes back to the page: on web, when the app's browser tab is shown again after it was hidden; in the mobile app, when the app comes back from the background, or the conversation's screen is shown again with the page open ([Live Refresh](#live-refresh)). Read again then, and after the page's own writes.

**Limits.** Each person may make 30 calls a minute through one customer page, and a hub's customer pages together make at most 200 requests a minute to its bases, on top of the limits every page shares ([Limits](#limits)). A call past any of them gets `RATE_LIMIT_EXCEEDED` with `retry_after_seconds`.

## Page Cards

An agent can send the person it is talking to a **page card**: a message in the conversation that opens one of the hub's pages, with params, such as the coaches' `results` page opened on one athlete (`{ athlete: "giovanna-izzo" }`). It sends one with the native tool `open_page` ([agents/native-tools.md](agents/native-tools.md#open_page)), which a pilot or pilot specialist may hold:

```yaml
# agents/pilot.yaml
tools:
  native:
    - open_page
```

**Which pages the agent is offered.** Only those the conversation's person can open where they read the card. Called with no `page`, the tool lists them, with each one's title, description and `params` schema, and sends nothing; called with a `page`, it checks again before it sends, so a page deleted, or a person who lost access, since the listing gets no card.

- **In the WayAI app** (a conversation on the app): the hub's customer pages with `show_in_conversation: true`, when the person may open them ([Customer Pages](#customer-pages)): a Hub User the hub has not blocked, whose account the contact book links to no contact or to one the hub sees. Their account must hold its Hub User entry itself: an invitation they have not accepted yet, which opens the pages to them from the app, is not enough for a card. Never a team page: a person's own conversation lists only the hub's customer pages, a team member's included.
- **On any other channel** (WhatsApp, Instagram, Telegram, email, the API): the hub's team pages, when the person's WayAI account holds a team user or hub admin entry on the hub; the card reaches them as a link. A customer there has no WayAI sign-in, so no card is offered to them. Neither is one to a person whose access to the hub comes only from an organization grant.
- **Nobody else:** a conversation with no person, an eval run, and a Copilot or background agent's turn, whose output is for the team.

**Params.** The card keeps only the params the page's `params` schema declares and accepts when it is sent, and the tool's answer lists those it dropped (`dropped_params`). Opening a card checks them again against the page's schema at that moment, so a card older than the page's latest manifest opens with what that manifest accepts. A card whose page was deleted since, or that the person viewing it may not open, shows **Page not available**.

**What each surface shows.** On web and in the mobile app, the card: the page's title and description, and **Open**, which opens the page over the conversation as its header does, for the person in their own conversation and for the team in the support conversation. Everywhere else, and in older app versions, the card is its fallback text:

- **a team page:** its title and a link to it in the hub's Pages section on web, with the card's params; a link too long for a message goes without them, and the page opens with none;
- **a customer page:** its title and "Open this in the latest version of the WayAI app."

The fallback goes out as any message of the agent's does on the conversation's channel: on WhatsApp and Instagram only within 24 hours of the person's last message; an email conversation stores the card without sending it. Later in the conversation the agent reads its card back as that fallback text, so also tell the person in your reply what the page is for.

## Push, Pull, Diff and Publish

- **The `pages/` folder is the desired set, when it exists.** A hub folder with a `pages/` folder sends every page in it. If `pages/` is empty, push deletes every page.
- **Without a `pages/` folder, push leaves the hub's pages as they are.** So a folder an older CLI wrote never deletes a page. To delete one page, delete its folder.
- **Pull** writes each page's folder whenever the hub has pages, removes the folders of deleted pages, and removes `pages/` once the hub has none. `page.yaml` comes back with its defaults left out.
- **Diff** shows pages under **Pages:**: `+ <slug>` added, `- <slug>` deleted, and `~ <slug>: <fields>` changed, where `html` in the field list means the HTML changed. Every push-time refusal above prints with push's other warnings, and the push writes none of the refused pages. A page the push leaves in place keeps every connection it names from being deleted by that push.
- **Publish** copies the preview's pages to production, by slug. The publish summary lists added, updated and removed pages under **Pages**. Every publish that syncs a change checks every page again against **production's own** connections and synced tool lists, the pages it does not change included, and a page production refuses fails the publish, naming its issues. A publish with nothing to sync (`Production is already up to date`) checks nothing.
- **A connection with no token yet is checked only partly when a page is copied.** A toolset connection's token never travels with a copy. So on a first publish, a new preview or a CI branch preview, each toolset connection arrives with no token, and its pages are still copied. Only the checks that need the token wait: on a team page whether its connection records the toolset as customer-scoped, on a customer page the customer-facing rule ([Customer Pages](#customer-pages)), and signed file URLs on both. The page calls nothing through that connection until the hub sets its token (`wayai set-connection-credential` on a preview; the production credential in the connections tab on production). The first publish that syncs a change after production's token is saved checks the page fully; saving the token alone is no change to publish.
- **Design warnings never stop a push.** Push and diff also print a warning for each design rule a page breaks, in the same block, marked `(design)`: [The Design Warnings](#the-design-warnings).
- **A push checks only the pages it writes.** Pushing a page the hub already holds unchanged checks nothing, so a CI branch preview takes back its own pages. Pushing a changed or new page is held to every rule, so set the token of a preview's toolset connection before you push a page that calls it.

## Writing a Page as an MCP Apps View

A page talks to its host by JSON-RPC 2.0 messages posted to its parent window, as any MCP Apps view does. Inlining the MCP Apps SDK is the easiest way: it speaks the protocol for you. You can also post the messages yourself, as the example below does.

**What a page may send:**

| Message | Kind | What it does |
|---------|------|--------------|
| `ui/initialize` | request | The first message. Its answer carries the host context: the page's params among it |
| `ui/notifications/initialized` | notification | Sent once the page has its answer to `ui/initialize` |
| `tools/call` | request | `{ name, arguments }`: calls one of the manifest's tools by its tool name. The answer is the tool's result: `content`, and `isError` when the tool refused |
| `ui/message` | request | `{ role: "user", content: [{ type: "text", text }] }`: puts the text in the person's own message box, never sends it, on a page whose manifest sets `allow_messages: true` opened from a conversation ([Putting Text in the Person's Message Box](#putting-text-in-the-persons-message-box)). Answered empty |
| `ping` | request | Answered empty |
| `notifications/message` | notification | A log line |

**What the host refuses**, each with a JSON-RPC error: `ui/open-link`, `ui/update-model-context`, `ui/download-file` and `resources/read`, and `ui/message` where it is not available ([Putting Text in the Person's Message Box](#putting-text-in-the-persons-message-box)).

**What the host does not support:** display-mode requests, and the tool-input and tool-result notifications. No tool call opens the page: people open it from the hub, so a page never waits for a tool's input or result to start.

**Deletes and composite Actions are confirmed by the host.** On web and in the mobile app, when a page calls a tool the toolset classifies as a delete (a record, relationship or file delete, or a `batch` that allows a delete) or as a [composite Action](bases/toolsets.md#composite-actions), the host asks the viewer first, in a confirmation of its own drawn outside the page that names the page's title and the tool's name. The page cannot draw over it, restyle it or answer it.

- **Run** sends the call, and the page gets the tool's result as usual.
- **Cancel** (or dismissing the confirmation, or closing the page) sends nothing. The call is answered with a JSON-RPC error, code `-32000`, whose `data.code` is `TOOL_CALL_DECLINED`.
- **Close page** on the confirmation closes the page without running the call, since the confirmation covers the page's own close.
- **No asking again at once.** For 3 seconds after the viewer cancels, every call of the page's that needs a confirmation is answered `TOOL_CALL_DECLINED` without asking. So never re-send a declined call: the viewer said no.
- **One confirmation at a time per page.** Another call that needs one while it is open is answered at once with the same error, `data.code` `CONFIRMATION_PENDING`, and nothing is asked or sent. Calls that need no confirmation go through meanwhile.
- **It is a guard for the person, not a permission.** Who may write is the platform's rule ([Who Sees a Team Page](#who-sees-a-team-page)), checked on every call. When the platform could not read the tools' kinds as the page opened, the host does not know which tools delete, so it confirms nothing and sends the call.

A page may still confirm a destructive action in its own UI, naming what will change ("Delete Ana's result?"); the viewer then confirms twice.

**Params** arrive in the host context as `wayai.params`. Read them from the `ui/initialize` answer, and ignore anything the schema in `params:` would not accept: links are written by people and agents. The host already drops from a page card's params what the page's current schema does not accept ([Page Cards](#page-cards)).

**A tool call can fail for reasons a page should show, not swallow:** the tool refused (`isError`), the viewer may no longer use the page, too many calls, or the page was updated since it opened (reload it).

On web and in the mobile app, a refusal of the call itself comes back as a tool result with `isError: true`, a text written for the viewer, and a `structuredContent.code`: `PAGE_REVISION_STALE` when the page was updated (the host also offers the viewer a reload), `RATE_LIMIT_EXCEEDED` with `retry_after_seconds` when the page called too often (the host answers every call inside that wait the same way, without sending it), `PAGE_CUSTOMER_UNAVAILABLE` on a customer page the hub does not open for the viewer's account (*Blocked people* in [Customer Pages](#customer-pages); the host then shows that in place of the page), or the refusal's own code otherwise: for example `VALIDATION_ERROR` for arguments the tool's schema refuses, and `AUTHORIZATION_ERROR` or `NOT_FOUND`, whose causes, and what to show for each, are in [the refusal table](#loading-empty-and-error-states). A refusal by the tool itself, or by a connection that no longer reaches it, is `isError: true` with its text and no `structuredContent`. A call that fails on the network or with a server error gets a JSON-RPC error.

A minimal page, with no SDK:

```html
<!doctype html>
<html>
<head><meta charset="utf-8"><title>Results</title></head>
<body>
  <ul id="athletes"><li>Loading…</li></ul>
  <script>
    let nextId = 1;
    const pending = new Map();
    window.addEventListener('message', (event) => {
      if (event.source !== window.parent) return;
      const msg = event.data;
      if (msg && msg.jsonrpc === '2.0' && pending.has(msg.id)) {
        const { resolve, reject } = pending.get(msg.id);
        pending.delete(msg.id);
        msg.error ? reject(msg.error) : resolve(msg.result);
      }
    });
    function request(method, params) {
      const id = nextId++;
      return new Promise((resolve, reject) => {
        pending.set(id, { resolve, reject });
        window.parent.postMessage({ jsonrpc: '2.0', id, method, params }, '*');
      });
    }
    function notify(method, params) {
      window.parent.postMessage({ jsonrpc: '2.0', method, params }, '*');
    }

    // The MCP Apps protocol version WayAI's host speaks (the SDK fills it in for you).
    const PROTOCOL_VERSION = '2026-01-26';

    async function main() {
      const { hostContext } = await request('ui/initialize', {
        protocolVersion: PROTOCOL_VERSION,
        appInfo: { name: 'results', version: '1.0.0' },
        appCapabilities: {},
      });
      // The viewer's theme (Designing a Page has the rest: theme switches, language, states).
      document.documentElement.setAttribute('data-theme', hostContext.theme);
      notify('ui/notifications/initialized', {});
      const list = document.getElementById('athletes');
      try {
        const result = await request('tools/call', { name: 'list_athletes', arguments: {} });
        if (result.isError) throw new Error(result.content?.[0]?.text ?? 'The tool refused');
        // The tool's answer, as its toolset gives it to an agent (bases/toolsets.md).
        const text = result.content?.find((c) => c.type === 'text')?.text ?? '{}';
        const athletes = JSON.parse(text).data ?? [];
        list.replaceChildren(...athletes.map((a) => Object.assign(document.createElement('li'), { textContent: a.name })));
        if (athletes.length === 0) list.innerHTML = '<li>No athletes yet</li>';
      } catch (err) {
        list.innerHTML = '<li>Could not load the athletes. Try again.</li>';
      }
    }
    main();
  </script>
</body>
</html>
```

Build each list from text nodes (`textContent`), never by pasting record values into `innerHTML`: a page shows data its viewers and agents wrote.

### Putting Text in the Person's Message Box

A page whose manifest sets `allow_messages: true` may put text in the message box of the person using it with `ui/message`, the MCP Apps request (the SDK's `sendMessage`), on web and in the mobile app. **It never sends.** The person sees the text in their own message box and sends it themselves, edited or not, or never: as themselves, in their own role, to whoever their messages in that conversation already reach. So a page cannot speak for anyone, start the agent, or post to a customer's WhatsApp. Use it for a reply the person would otherwise type: a coach's page drafting "Your 100m time: 1:02.3" for the athlete's conversation.

- **Where it goes:** the message box of the conversation the page was opened from. The page covers that box, so the page closes and the box takes the focus, the text in it.
- **What is kept:** an empty box takes the text; a box already holding a draft keeps it, the text after it and a blank line. A draft is never replaced.
- **What it takes:** `role: "user"` and one or more text blocks, joined by a blank line, at most 4,000 characters in all. No other field, no image or file, no recipient or channel, and nothing that sends.
- **How to tell it is available:** the `ui/initialize` answer's `hostCapabilities.message` is present (`{ "text": {} }`) only where a `ui/message` would be taken when the page starts. Offer the action only then, and still handle a refusal: the message box can stop taking text while the page is open (the conversation ends, its window closes).

Refused with a JSON-RPC error, with nothing put anywhere:

| Error | `data.code` | When |
|-------|-------------|------|
| `-32000` | `MESSAGES_NOT_ALLOWED` | The page's manifest does not set `allow_messages: true`, as the platform reads it when the page opens; what the page says has no part in it |
| `-32000` | `NO_COMPOSER` | The page was opened outside a conversation (from the hub's Pages section on web); the person cannot write in that conversation's message box just then, where the box is greyed out (for example an ended support conversation, one they have not claimed or another person claimed, a blocked contact, a closed WhatsApp or Instagram window); or its text is already in the message box: one `ui/message` per opening, since the page then closes |
| `-32602` | `MESSAGE_TOO_LONG` | The text is over 4,000 characters; it is refused whole, never cut |
| `-32602` | (none) | Anything but `role: "user"` and text blocks, a field other than `role` and `content`, or text that is only whitespace |

## Live Refresh

On web, an open team page hears when the data its tools reach may have changed; on web and in the mobile app, a customer page hears when the person comes back to it. The host context carries a number, `wayai.refresh`, and `host-context-changed` carries a new one each time WayAI raises it. Keep the value from `ui/initialize`, and read again when a later one differs:

```js
let refresh;   // hostContext.wayai.refresh from the ui/initialize answer

function onContextChanged(changed) {   // each host-context-changed's params
  // `wayai` arrives whole whenever any of its fields changes (a theme switch too), so compare.
  if (changed.wayai && changed.wayai.refresh !== refresh) {
    refresh = changed.wayai.refresh;
    loadAthletes();   // the same tools/call reads the page made when it opened
  }
}
```

- **What raises it on a team page:** a change in a base that one of the page's Base Toolset connections reaches, in that connection's environment, and the app's live connection coming back after a drop. Changes made while the connection was down are not replayed; the raise on its return stands for them.
- **It is a hint, and it can be wrong both ways.** It says that something in those bases may have changed, never what. It rises for changes your page does not show. It can also miss one: when a base changes several times within about 5 seconds, only the first change may raise it. So read again after your page's own writes instead of waiting for it, and treat what your tool calls answer as the truth.
- **It rises at most once until your page reads.** After a raise, WayAI raises it again only once a tool call your page makes has been answered, then once for every change since. While the page is hidden (another browser tab is in front, or the mobile app is in the background), it does not rise; it rises once when the page is shown again.
- **Every read counts toward the call limits** ([Limits](#limits)). Read again only what the page shows.
- **On a customer page it rises only when the person comes back.** A change signal does not say whose record changed, so a customer page never hears of changes in the base, which would tell a customer of other customers' activity. Its `wayai.refresh` rises when the person comes back to it, at most once until the page reads, as above: on web, when the app's browser tab is shown again after it was hidden; in the mobile app, when the app comes back from the background, or the conversation's screen is shown again with the page open. Read again then, after the page's own writes, and when the viewer asks.
- **Where it does not exist:** on a team page in the mobile app, which never hears `wayai.refresh`.

## Designing a Page

A page looks like the WayAI app by default. The host hands every page WayAI's look, and this guide says how to build on it. Nothing here is enforced: push warns about a few of these rules ([The Design Warnings](#the-design-warnings)) and never refuses a page for its design.

### Start From the Scaffold

```bash
wayai page new results                          # pages/results/page.yaml and pages/results/index.html
wayai page new review-queue --title "Review queue" --hub support
wayai page new my-results --audience customers  # a customer page
```

`wayai page new <slug>` writes a starter page into a hub folder, found by `wayai push`'s hub steps: the one `--hub` names, else the hub folder you are in, else the workspace's `default_hub`, else its only preview or new hub folder. Base folders and `default_base` play no part. `--title` sets what menus show (the slug in words by default), and `--audience` is `team`, the default, or `customers`. A customer page starts with `show_in_conversation: true`, no tools and no `customer_writes`, so it only reads; its `page.yaml` comments say to add its list tool from a Base Toolset connection whose record types declare person or inherit entries or `{ shared: true }` ([Customer Pages](#customer-pages)). It never overwrites a file. Before writing anything, it refuses a slug that is not a folder name and any other audience in the words push refuses them in, and it refuses a production mirror's folder and a hub folder already holding 20 pages.

The starter page passes push as written, with no design warning. It already does what the rest of this section asks:

- it is styled only with the tokens and the base classes;
- it shows its list as one card per row on a narrow frame, and as a table once its container is 560px wide;
- it speaks the MCP Apps messages by hand: no SDK to inline;
- it follows the viewer's theme and language, and pads itself by the safe-area insets;
- it shows loading, empty and error states, the host's refusal codes among the errors, with every string in English, Portuguese and Spanish;
- it reads its list again when `wayai.refresh` rises ([Live Refresh](#live-refresh)).

To make it list your records, add the tool to `page.yaml`'s `tools` and put its name in `LIST_TOOL` at the top of the page's script, then shape `nameOf` and `updatedOf` to your records' fields.

### What Every Page Gets

- **WayAI's base stylesheet**, placed before anything the page writes: every token for both themes, `body` defaults (the app font, 14px text, the theme's text and background colors, no margin), and the base classes. All of it sits in the `wayai-base` cascade layer, so any rule your page writes outside a layer wins over it, whatever its specificity. So does a rule in a layer of your own: `wayai-base` is declared first.
- **The app font**, Plus Jakarta Sans, as `--font-sans`, already loaded.
- **Neither counts toward the page's 512 KiB.** Do not inline them.
- **The host context** in the answer to `ui/initialize`:

| Field | What it holds |
|-------|---------------|
| `theme` | `light` or `dark`: the appearance the viewer chose in WayAI |
| `locale` | `en`, `pt` or `es`: the viewer's app language |
| `styles.variables` | Every MCP Apps standard style variable, at the current theme's values |
| `wayai.styles.variables` | WayAI's own tokens ([Styling With the Tokens](#styling-with-the-tokens)), at the current theme's values |
| `wayai.page` | The page's `slug` and `title` |
| `wayai.params` | The values the page was opened with ([`params`](#the-manifest)) |
| `wayai.viewerLevel` | `write` when the viewer may run every tool the page declares, `read` when only its read tools ([Who Sees a Team Page](#who-sees-a-team-page)); on a customer page, `write` when it sets `customer_writes: true` ([Customer Pages](#customer-pages)). Hide or disable write controls at `read`; the platform refuses those calls anyway |
| `wayai.refresh` | A number WayAI raises when the person comes back to a customer page (on web and in the mobile app), or, on web only, when a team page's data may have changed ([Live Refresh](#live-refresh)) |
| `platform` | `web` in the web app, `mobile` in the mobile app |
| `containerDimensions` | The page's frame: `width` and `height` in CSS pixels |
| `safeAreaInsets` | How far the device's safe area reaches into the frame, per side, in CSS pixels |

Once the page sends `ui/notifications/initialized`, a change reaches it as `ui/notifications/host-context-changed`, carrying only the fields that changed: a theme switch, a language switch, a resize of its frame, a change of its safe-area insets, or a rise of `wayai.refresh`. A change made before `initialized` arrives right after it.

### Styling With the Tokens

Style with the tokens, never with fixed colors: a token holds each theme's value, so a page written with them follows the viewer's theme. Every token is a CSS custom property the base stylesheet sets on `:root`.

**The MCP Apps standard style variables.** A page written for another MCP Apps host reads these too.

| Family | Names |
|--------|-------|
| Backgrounds | `--color-background-{primary,secondary,tertiary,inverse,ghost,info,danger,success,warning,disabled}` |
| Text | `--color-text-{primary,secondary,tertiary,inverse,ghost,info,danger,success,warning,disabled}` |
| Borders | `--color-border-{primary,secondary,tertiary,inverse,ghost,info,danger,success,warning,disabled}` |
| Focus rings | `--color-ring-{primary,secondary,inverse,info,danger,success,warning}` |
| Fonts | `--font-{sans,mono}`, `--font-weight-{normal,medium,semibold,bold}` |
| Text sizes | `--font-text-{xs,sm,md,lg}-{size,line-height}`: 12, 14, 16 and 18px |
| Heading sizes | `--font-heading-{xs,sm,md,lg,xl,2xl,3xl}-{size,line-height}`: 16, 18, 22, 24, 32, 36 and 48px |
| Radii | `--border-radius-{xs,sm,md,lg,xl,full}`: 2, 4, 6, 8 and 12px, and a pill |
| Borders and shadows | `--border-width-regular`, `--shadow-{hairline,sm,md,lg}` |

**WayAI's own tokens**, for what the standard has no name for:

| Token | What it is for |
|-------|----------------|
| `--wayai-color-accent` | The brand accent: primary buttons, links, selection |
| `--wayai-color-on-accent` | Text and icons on the accent or on the danger fill |
| `--wayai-color-danger-fill` | A destructive action's fill, as `wayai-button--danger` uses it |
| `--wayai-color-card` | A card, list or table surface |
| `--wayai-color-input` | A form field's fill |
| `--wayai-color-input-border` | A form field's border |
| `--wayai-space-1` … `--wayai-space-6`, `--wayai-space-8`, `--wayai-space-10`, `--wayai-space-12` | Spacing: 4px times its number (`--wayai-space-3` is 12px) |
| `--wayai-control-height` | The least height of a button, field or list row: 40px, 44px under `pointer: coarse` |

```css
.summary { padding: var(--wayai-space-4); background: var(--wayai-color-card); border-radius: var(--border-radius-lg); }
.summary h2 { font-size: var(--font-heading-sm-size); color: var(--color-text-primary); }
.summary .delta { padding: 0 var(--wayai-space-1); border-radius: var(--border-radius-sm); background: var(--color-background-success); color: var(--color-text-primary); }
```

- **A color no token names**, such as a chart's series, goes in a custom property of your own (`--series-1`), with a value for each theme under `:root[data-theme="light"]` and `:root[data-theme="dark"]`.
- **A `var()` fallback is fine** (`var(--color-text-primary, #151619)`): the token is always set, so the fallback never shows.

### Light and Dark Mode

The base stylesheet opens in the viewer's theme. To follow a switch while the page is open, set `data-theme` on `<html>` from the host context's `theme`, at `ui/initialize` and on every `host-context-changed` that carries one; the base stylesheet's `:root[data-theme="light"]` and `:root[data-theme="dark"]` rules do the rest. The MCP Apps SDK's `applyDocumentTheme` does the same. A page that never sets it keeps the theme it opened in until it is reopened.

```js
function applyContext(changed) {   // the ui/initialize answer's hostContext, then each change
  if (changed.theme) document.documentElement.setAttribute('data-theme', changed.theme);
}
```

- Take the theme from the host context, never from `prefers-color-scheme`: the host context's is the appearance the viewer chose in WayAI.
- Each theme sets `color-scheme`, so form controls and scrollbars follow it.

### Layout and Spacing

- **Wrap the page in `wayai-page`**: full width on a phone, at most 960px wide and centered, padded 16px (24px from 640px wide).
- **Stack with `wayai-stack`** (12px apart) and lay out a row with `wayai-row` (wraps, 8px apart). Space everything else with the spacing tokens.
- **Type scale:** text is 14px (`--font-text-sm-size`). A page title is `wayai-title` (22px, semibold). Use `--font-text-xs-size` for captions and metadata, and `wayai-muted` for secondary text.
- **Radii:** cards, lists and tables use `--border-radius-lg`; buttons and fields `--border-radius-md`.

### The Base Classes

The host's stylesheet defines these. A modifier goes beside its block's class: `class="wayai-button wayai-button--primary"`.

| Class | What it is |
|-------|------------|
| `wayai-page` | The page's content column |
| `wayai-stack` | A vertical stack |
| `wayai-row` | A horizontal row that wraps, its items centered |
| `wayai-title` | A page or section title |
| `wayai-muted` | Secondary text: hints, captions, metadata |
| `wayai-card` | A raised surface with a border, for a group of content |
| `wayai-button` | A button, or a link styled as one: outlined, at least `--wayai-control-height` tall |
| `wayai-button--primary` | The main action, filled with the accent |
| `wayai-button--danger` | A destructive action, filled with `--wayai-color-danger-fill` |
| `wayai-button--ghost` | A quiet action with no border or fill |
| `wayai-field` | A form field: its label, its control and a hint, stacked |
| `wayai-label` | A form label |
| `wayai-input` | A text input, select or textarea, full width |
| `wayai-list` | A list (`ul`/`ol`) on a card surface, rows divided |
| `wayai-list-item` | One row of a `wayai-list` |
| `wayai-table-wrap` | Wraps a `wayai-table`, so a wide table scrolls sideways instead of widening the page |
| `wayai-table` | A data table |
| `wayai-badge` | A small status label, with `wayai-badge--info`, `wayai-badge--success`, `wayai-badge--warning` and `wayai-badge--danger` |
| `wayai-alert` | A message block, informational by default, with `wayai-alert--success`, `wayai-alert--warning` and `wayai-alert--danger` |
| `wayai-empty` | An empty, loading or unavailable state: centered, muted |

```html
<main class="wayai-page">
  <h1 class="wayai-title">Results</h1>
  <form class="wayai-card wayai-stack" id="entry">
    <div class="wayai-field">
      <label class="wayai-label" for="time">Time</label>
      <input class="wayai-input" id="time" name="time" inputmode="decimal" required>
      <span class="wayai-muted">Seconds, e.g. 58.21</span>
    </div>
    <div class="wayai-row">
      <button class="wayai-button wayai-button--primary" type="submit">Save result</button>
      <button class="wayai-button wayai-button--ghost" type="reset">Clear</button>
    </div>
  </form>
  <div class="wayai-table-wrap">
    <table class="wayai-table">…</table>
  </div>
</main>
```

**`hidden` and the base classes.** A base class that sets `display`, such as `wayai-empty` or `wayai-stack`, wins over the browser's own rule for the `hidden` attribute, so a hidden element keeps showing. Add `[hidden] { display: none !important; }` to your styles, as the scaffold does.

### Responsive Layout

- **The frame is the page's screen.** It fills the area the page opens in. Media queries and container queries measure the frame, not the device, and resizing it re-lays the page out as it happens. A `<meta name="viewport">` does nothing inside the frame.
- **Write mobile first.** Make the narrow layout the default, and widen it inside `@media (min-width: …)` or a container query; 640px is where `wayai-page` widens its padding. Give nothing a fixed width wider than a phone outside such a query.
- **Prefer container queries for a component**: `container-type: inline-size` on its wrapper, then `@container (min-width: 560px) { … }`. It then fits whatever area it lands in.
- **Tables scroll or become cards.** Wrap a table in `wayai-table-wrap`, so it scrolls sideways on a narrow frame, or show the rows as cards until the container is wide enough for columns (the scaffold does both).
- **Touch.** Under `@media (pointer: coarse)`, the base stylesheet raises `--wayai-control-height` to 44px and sets `wayai-input`'s text to 16px, so a phone does not zoom into a focused field. Make your own tap targets at least `--wayai-control-height` tall, and do not hide an action behind hover alone.
- **Safe area.** Pad the page by `safeAreaInsets` when they are not 0, so content clears a notch or a home indicator. They are 0 on every side the app already keeps clear.

### Accessibility

- **Contrast.** Every text the base classes color, except a disabled control's, reaches 4.5:1 on the page and on a card in both themes, hover included: white on `wayai-button--danger`'s `--wayai-color-danger-fill` is 4.83:1, and a colored badge or alert writes `--color-text-primary` on its tint. The status text tokens on their own do not. `--color-text-danger` is 3.06:1 on the light page background: enough for large text and for a control's outline, not for body text. `--color-text-info` and `--color-text-success` are 2.6:1 on a dark card: below even that, so on a card let a tint carry them, as the colored badges do. Write a sentence in `--color-text-primary`, as `wayai-alert--danger` does, and give a danger button a clear label.
- **Focus.** `wayai-button` and `wayai-input` show a 2px `--color-ring-primary` ring on `:focus-visible`. Never remove an outline without a ring of your own, and give a control you build yourself the same ring.
- **Real controls.** Use `<button>` for actions and `<a>` only for links; a clickable `<div>` is unreachable by keyboard.
- **Labels.** Give every field a `<label for>` (`wayai-label`), or an `aria-label` when there is no room for one, and every icon-only button an `aria-label`. Put hints in text, not only in a placeholder.
- **Language.** Set `<html lang>` from the host context's `locale`.
- **Announce changes.** Mark the region a load or a save updates with `aria-live="polite"` (and `aria-busy="true"` while it loads), and an error with `role="alert"`.

### Loading, Empty and Error States

Every list or value a page reads from a tool has three states besides its data:

- **Loading:** a `wayai-empty` reading "Loading…", with the region `aria-busy`. Disable the buttons that would start the same call again.
- **Empty:** a `wayai-empty` saying what is missing and, when the viewer can fix it, how: "No results for this meet yet. Add the first one above."
- **Error:** a `wayai-alert wayai-alert--danger` with `role="alert"`, a sentence saying what failed, and a "Try again" button. Never leave a spinner running or a list silently empty.

Write each refusal of [a tool call](#writing-a-page-as-an-mcp-apps-view) for the viewer:

| What came back | What to show |
|----------------|--------------|
| `isError` with `structuredContent.code` `PAGE_REVISION_STALE` | The page was updated: ask the viewer to reload it (the host also offers a reload). No call succeeds until then, so offer no retry |
| `isError` with `RATE_LIMIT_EXCEEDED` | Too many requests: try again after `retry_after_seconds`, the buttons disabled until then (one timer, not a poll) |
| `isError` with `AUTHORIZATION_ERROR` | The page is no longer available to this viewer |
| `isError` with `PAGE_CUSTOMER_UNAVAILABLE` | A customer page: it is not available to this viewer's account ([Customer Pages](#customer-pages)). On web and in the mobile app the host replaces the page with that message, so the page needs no state of its own for it |
| `isError` with `NOT_FOUND` | In your words, that the page or what it shows is not available. The result's text says which: `Page not found` (no longer offered to this viewer) or `Page tool not found` (the page called a tool its `page.yaml` does not list) |
| `isError` with `VALIDATION_ERROR` | The tool refused the arguments: point at the field, with the result's text |
| `isError` with no `structuredContent` | The tool's own refusal: show its text |
| A JSON-RPC error with `data.code` `TOOL_CALL_DECLINED` | The viewer cancelled [the host's confirmation](#writing-a-page-as-an-mcp-apps-view): nothing was sent. Leave the page as it was, with no error, and never re-send the call |
| A JSON-RPC error with `data.code` `CONFIRMATION_PENDING` | Another call is waiting for the viewer's confirmation: nothing was sent. Disable the action while a call runs |
| Any other JSON-RPC error | The call did not get through: "Could not reach WayAI." with a "Try again" button the viewer presses. Never retry a call by itself: a call that writes may run twice, and a declined one would ask the viewer again |

### UI Text

- **Short and plain.** Sentence case, no exclamation marks, no jargon. Address the viewer as "you".
- **Buttons name their action:** "Save result", "Delete athlete", not "OK" or "Submit".
- **An error says what happened and what to do next**, never a code, an id or a stack trace.
- **A confirmation names what it will do:** "Delete Ana's result?" with "Delete" and "Cancel" buttons (`confirm()` returns `false` in a page). The host also confirms deletes and composite Actions itself ([Writing a Page as an MCP Apps View](#writing-a-page-as-an-mcp-apps-view)), naming only the page and the tool.
- **Your text, every language.** Write each string in English, Portuguese and Spanish ([Switching Language](#switching-language)); a record's own values stay as written.

### Switching Language

The host context's `locale` is the viewer's app language, `en`, `pt` or `es`, and `host-context-changed` carries a new one when the viewer switches. Keep every string in one table per language, pick the table from `locale`, and redraw on a change:

```js
const STRINGS = {
  en: { save: 'Save result', empty: 'No results yet.' },
  pt: { save: 'Salvar resultado', empty: 'Nenhum resultado ainda.' },
  es: { save: 'Guardar resultado', empty: 'Todavía no hay resultados.' },
};
let locale = 'en';
const t = (key) => (STRINGS[locale] ?? STRINGS.en)[key];

function applyContext(changed) {
  if (changed.locale) {
    locale = changed.locale;
    document.documentElement.lang = locale;
    render();   // redraw every string from STRINGS
  }
}
```

Format numbers and dates for the same locale: `new Intl.NumberFormat(locale).format(58.21)`, `date.toLocaleDateString(locale)`.

### The Design Warnings

`wayai push` and `wayai diff` read every page in the `pages/` folder, changed or not, and print a warning for each rule below a page breaks, in the same block as the platform's warnings: `page "<slug>" (design): …`. A warning never stops a push. Each rule reads the page's text, so each has limits:

| Warning | When | Not read |
|---------|------|----------|
| Hard-coded colors | A hex color, or `rgb()`, `hsl()` and the other color functions, in a color property (`color`, `background`, `border`, `outline`, `fill`, `stroke`, …) of a `<style>` or a `style` attribute | A `var()` fallback, a `color-mix()` from a token, your own custom property's value, shadows, named colors (`white`), colors set from script or in SVG attributes |
| No dark mode | The HTML never names `data-theme` (or `applyDocumentTheme`) | Whether it is actually set |
| Fixed widths | A `width` or `inline-size` over 430px, outside every media and container query | `min-width`, `width` attributes |
| Tables that overflow | A table with nothing in the page that lets it fit a phone: no `wayai-table-wrap`, no `overflow` of `auto` or `scroll`, and no table rule inside a media or container query | Which table is handled: one handled table passes them all |
| Loads the sandbox refuses | A `<script src>`, a stylesheet `<link>`, an image loading anything but a `data:` or `blob:` URL, any frame, video or audio source (a `data:` one too), a CSS `url()` or `@import` of a file, and `fetch`, `WebSocket` or `EventSource` called with a literal URL | Markup inside script strings, URLs built at run time |
| No empty state / no error state | A page that declares tools and builds a list, with no `wayai-empty` and no check for zero rows (`.length === 0`, `!rows.length`), or no `isError` and no `wayai-alert` | Whether the check guards that list: an inlined library holding those words passes too |

## The Sandbox: What a Page Cannot Do

The sandbox keeps page code from ever running with the WayAI app's origin or sign-in. It does not keep data inside the page: whoever builds a page is trusted with whatever its viewers can read. Write pages to these rules:

- **An opaque origin.** The page runs in a sandboxed frame of its own. Inline `<script>` and `<style>`, and `eval` / `new Function`, work.
- **No network.** `fetch`, XHR, WebSockets, remote scripts, stylesheets, images and fonts are all refused. Images and fonts work only as `data:` (or `blob:`) URLs. **Inline every library, the MCP Apps SDK included, inside the 512 KiB cap.** The page's only way out is `tools/call`.
- **No storage.** No cookies, `localStorage`, `sessionStorage` or IndexedDB. Keep state in memory, and in the base through the page's tools.
- **No popups, modals or downloads.** `window.open` does nothing. `alert()` and `prompt()` do nothing, and `confirm()` returns `false`. The host confirms deletes and composite Actions itself ([Writing a Page as an MCP Apps View](#writing-a-page-as-an-mcp-apps-view)); to name what will change, confirm in the page's own UI too (an "Are you sure?" button pair). A page cannot download or export a file.
- **No navigation.** Links and `location` changes do not leave the page, and `ui/open-link` is refused.
- **Forms work, but a submit never navigates.** `<form onsubmit>` and Enter-to-save fire your handler. Call `event.preventDefault()` and save through `tools/call`.

## Limits

| Limit | Value |
|-------|-------|
| Pages per hub | 20 |
| HTML per page | 512 KiB (UTF-8), one file |
| Tools per page | 50 |
| `params` schema | 16 KiB of JSON |
| `title` / `description` | 120 / 500 characters |
| Audiences | `team`, `customers` |
| Calls by one person through one team page | 60 a minute |
| Calls by one person through one customer page | 30 a minute |
| Calls by one person through all of a hub's pages | 150 a minute |
| Calls through all of a hub's customer pages, everyone together | 200 a minute |
| Calls through all of a hub's pages, everyone together | 500 a minute |

A call by a person who may only read counts once toward its page's limit and twice toward each of the others: the platform checks the tool's kind with the base before calling it. On a customer page, that is every call on a page without `customer_writes: true`; on a page with it, a call to a toolset that only reads approved records of reviewed `shared` types counts twice too, since the platform re-reads what that toolset reaches before calling it. Opening a page counts once toward each limit for each Base Toolset connection its tools use, while the platform reads their kinds. A call past a limit is refused with `RATE_LIMIT_EXCEEDED` ([the refusal table](#loading-empty-and-error-states)). Opening a page past a limit still opens it.

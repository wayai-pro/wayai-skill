# App Navigation & Deep Links

Canonical URL surface of the WayAI web app at `https://app.wayai.pro` (replace with `https://mac{1-9}.wayai-dev.com` for local dev). Use this when guiding the user to a screen — never invent paths or describe breadcrumbs ("go to Settings → Hubs → …"). Always hand over a single deep link — except for the one screen no link reaches, the desktop app's settings ([Desktop app](#desktop-app)).

## Table of Contents
- [Top-Level Map](#top-level-map)
- [Auth Routes](#auth-routes)
- [Conversation Surfaces](#conversation-surfaces)
- [Settings Hierarchy](#settings-hierarchy)
- [Account Settings](#account-settings)
- [Marketing & Docs](#marketing--docs)
- [Query-String Deep Links](#query-string-deep-links)
- [Rules](#rules)

## Top-Level Map

Visiting `https://app.wayai.pro/` opens `/chat`; a person with nothing in Chat is sent on to `/support` if they are on a hub's team or administer a hub, otherwise to `/settings`.

| Prefix | Purpose | Who sees it |
|--------|---------|-------------|
| `/chat` | Every agent the signed-in user talks to as the **end user**: `chat` hubs (one ongoing thread each) and `task` hubs (one conversation per task) | Hub Users |
| `/support` | Inbox / kanban for team members handling end-user conversations | Hub Team Users, Hub Admins |
| `/settings` | Unified settings shell (account + org + hub) with sidebar nav | Account: all authenticated users; Org/Hub: admins |
| `/user` | Legacy alias — 308 redirects to `/settings/account/*` (kept for external bookmarks) | All authenticated users |

Access is per-level: an Org Admin who isn't also a Hub Admin gets an auth error at hub-scoped URLs. Top-level prefixes are the canonical list in `apps/web/src/lib/app-routes.ts` (`APP_ROUTE_PREFIXES`).

## Auth Routes

| Path | Purpose |
|------|---------|
| `/login` | WorkOS sign-in (also handles signup) |
| `/callback` | WorkOS OAuth callback — never link directly |
| `/verify-email` | Magic-code email verification |
| `/auth-error` | Error landing page from auth flow |
| `/success` | Post-flow success page |
| `/oauth/authorize` | Standalone Connect Login URI (CLI / MCP OAuth) — never link directly |

Auth pages use **inline translations** (en/pt/es objects in the same file), not `messages/*.json`. They run before a session exists.

## Conversation Surfaces

### Chat
One list of agents, chat and task hubs alike, most recent activity first. A `task` hub's row shows how many of the user's tasks in it are in progress. What a person can do on these screens: [using-the-app.md](using-the-app.md).

| Path | Purpose |
|------|---------|
| `/chat` | All agents |
| `/chat/[hubId]` | A `chat` hub: its ongoing thread. A `task` hub: its task list (In progress; `?status=ended` for Ended) with New task |
| `/chat/[hubId]/conversations/[conversationId]` | One conversation of the hub: a task, or a `chat` hub's conversation |
| `/chat/[hubId]/new/[taskStartId]` | A new task in a `task` hub (`taskStartId` is a fresh id per new task). Never hand it over: each id is good for one new task, so link `/chat/[hubId]` and its **New task** instead |

### Support
One inbox across every hub the person supports. What the team can do on these screens: [support-inbox.md](support-inbox.md).

| Path | Purpose |
|------|---------|
| `/support` | The support inbox, in Agent / Team / Ended tabs (`?status=agent\|team\|ended` opens one) |
| `/support?view=kanban&hub=[hubId]` | The board for one hub (`&lane=[laneSlug]` narrows it to one lane while the board groups by kanban status) |
| `/support/[conversationId]` | One conversation, with the team's controls |
| `/support/hubs/[hubId]/users/[hubUserId]` | One Hub User's open conversation in that hub |

### Mobile app
The iOS and Android apps show the same agents as `/chat` and the team inbox of `/support`; what each tab offers is in [using-the-app.md → The Mobile App](using-the-app.md#the-mobile-app). The mobile apps have no links to hand over: send the user to a screen with the web links above. Inside an agent's message, a link to a screen the app has (among them `/chat`, `/chat/[hubId]`, `/chat/[hubId]/conversations/[conversationId]`, `/support`, `/support/[conversationId]`) opens that screen in the app, and any other page of the web app opens in the phone's browser. That is the app as built today: an older install opens these links in an in-app browser until it is updated.

### Desktop app
The desktop app is the web app in its own window, with the same screens, for macOS (Apple silicon and Intel) and Windows; there is none for Linux. Send the person to `https://wayai.pro/download`: it offers the one for their computer, on Windows the Microsoft Store listing plus a direct installer for a PC without the Store (Windows may warn when that installer is opened: choose **More info**, then **Run anyway**). Google, Microsoft and Apple sign-in finish in the computer's browser, then return to the app.

Over a browser tab it adds system notifications when a conversation moves from the AI to the team on a hub the person supports (while unclaimed or claimed by them) and when a customer writes in a conversation they claimed, a click opening it in Support; the sidebar's Support count on the Dock icon (macOS) or taskbar button (Windows) and in the menu bar or system tray, where the app keeps running when its window is closed; and updates of its own (a Microsoft Store copy updates through the Store), which the sidebar's update icon offers as **Restart to update WayAI** once one is ready.

A link you hand over opens in the browser, not in the app, so the app's settings for that computer take no link: they are the **Desktop app** section of the Profile tab (`/settings/account/profile`) inside the app, where the avatar menu → **User Settings** opens it. They are **Open at login** (for a Microsoft Store copy, Windows' own Settings → Apps → Startup), **Keep running when the window is closed**, **Notifications**, and **Show message text** (macOS only; Windows notifications never show a customer's name or words).

In a browser on a Mac or a Windows computer, the same section of `/settings/account/profile` holds one setting of that browser's own instead: **Open links in the desktop app**, off until turned on. While it is on, most pages of Chat, Support and the communities (never settings), loaded fresh in that browser from a link, a bookmark or the address bar, offer to open in the desktop app, and **Open** opens the page there. A reload or a move within the web app offers nothing. It needs the desktop app installed on that computer, which the browser cannot check.

## Settings Hierarchy

Drill-down. Each level has a sibling-switcher dropdown and a "+ New" affordance. The hub-detail catch-all (`[[...segments]]`) holds the URL shape; content renders in `(app)/layout.tsx` to keep a stable RSC tree across tab switches.

### Org level
| Path | Purpose |
|------|---------|
| `/settings/organizations` | All organizations grid |
| `/settings/organizations/new` | Create-organization modal |
| `/settings/organizations/[orgId]` | Org detail (default tab) |
| `/settings/organizations/[orgId]/overview` | Org overview |
| `/settings/organizations/[orgId]/hubs` | Org's hubs list |
| `/settings/organizations/[orgId]/credentials` | Org-level credentials |
| `/settings/organizations/[orgId]/resources` | Org-level resources (shared knowledge bases / skills) |
| `/settings/organizations/[orgId]/tags` | Org tags — organize hubs/credentials and gate credential resolution (see [connections.md](connections.md#organization-tags)) |
| `/settings/organizations/[orgId]/contacts` | The organization's contact book: contacts, org lists, CSV import and account links (see [automations.md](automations.md#the-contact-book)) |
| `/settings/organizations/[orgId]/administrators` | Org admin members |

### Hub level
Hub-detail tabs live under `/settings/organizations/[orgId]/hubs/[hubId]/<tab>`. Valid tabs (from `useSettingsPath.ts`):

| Tab | Path suffix | Covers |
|-----|-------------|--------|
| `overview` | `/overview` | Name, description, timezone, AI mode, permissions |
| `connections` | `/connections` | LLM providers, channel APIs, MCP servers, OAuth configs |
| `agents` | `/agents` | Agents list |
| `state` | `/state` | Conversation/user state schemas |
| `resource` | `/resource` | Knowledge bases, skill resources |
| `evals` | `/evals` | Eval scenarios + results |
| `automations` | `/automations` | Automations — enable, pause and run now — and the hub's own lists (the **Lists** sub-tab, `/automations/lists`, a list at `/automations/lists/<listId>`; see [automations.md](automations.md#hub-lists)). The contacts, and the org lists, are the organization's contact book (see [automations.md](automations.md#the-contact-book)) |
| `analytics` | `/analytics` | Hub metrics |
| `users` | `/users` | Admins, Hub Users and teams (`?subtab=admins\|users\|teams`; the `teams` sub-tab also holds the support model and who may approve contacts) |

### Hub sub-entities
| Path | Purpose |
|------|---------|
| `/settings/organizations/[orgId]/hubs/[hubId]/agents/[agentId]` | Agent detail |
| `/settings/organizations/[orgId]/hubs/[hubId]/connections/[connectionId]` | Connection detail; a WhatsApp connection's page also manages its message templates |
| `/settings/organizations/[orgId]/hubs/[hubId]/connections/new?connector=<connector_id>` | Create connection pre-picked to a connector (matched by `connector_id` UUID, not a slug) |
| `/settings/organizations/[orgId]/hubs/[hubId]/evals/calls/[conversationId]/[callId]` | An eval call's page: its transcript and recording. Reached by the link `wayai eval call` prints for each run; no list links to it |

### Hub-only shortcut
| Path | Purpose |
|------|---------|
| `/settings/hubs/[hubId]` | Direct hub access (resolves org from membership) |

## Account Settings

Reached via the avatar menu (bottom of sidebar) → User Settings, or the ACCOUNT section of the settings sidebar.

| Path | Purpose |
|------|---------|
| `/settings/account` | Default account tab (redirects to profile) |
| `/settings/account/profile` | Name, email, avatar, theme, language; inside the desktop app, its settings for that computer; in a browser on a Mac or Windows computer, **Open links in the desktop app** ([Desktop app](#desktop-app)) |
| `/settings/account/api-tokens` | Personal `way_` API tokens for the `wayai` CLI and direct API calls |

Legacy `/user/*` paths 308-redirect to the equivalent `/settings/account/*` URL — bookmarks and external links stay valid for one release cycle.

## Marketing & Docs

Public, locale-prefixed (`/`, `/en`, `/pt`, `/es`). Default locale (`en`) renders unprefixed; non-EN visitors get redirected to their locale by `intlMiddleware`.

| Path | Purpose |
|------|---------|
| `/` | Marketing home, and the onboarding entry point — its `#start` section carries the install command and the example prompt, then SKILL.md drives state 1+ |
| `/pricing` | Plans + pricing |
| `/download` | The desktop app for macOS and Windows ([Desktop app](#desktop-app)) |
| `/privacy` | Privacy policy |
| `/terms` | Terms of service |

## Query-String Deep Links

Tab pages accept these query strings to pre-open modals or pre-fill forms.

| Tab | Query | Effect |
|-----|-------|--------|
| `…/credentials` | `?prefill=true&name=<name>&type=<api_key\|bearer\|basic_auth>` | Opens the create-credential modal pre-filled |
| `…/connections` | `?connector=<slug>` | Lands on the Connections tab; if a connection for that connector **already exists** it's scrolled to and highlighted (`slug` = hyphen/space-tolerant substring of the connector name: `whatsapp`, `instagram`, `mcp-server`). Canonical **OAuth-connection handoff** target — the slug does **not** pre-open the add form, so to create one the user clicks **Add Connection** on this tab, picks the connector, and chooses OAuth |
| `…/connections/new` | `?connector=<connector_id>` | Opens the create form pre-picked to a connector — matched by `connector_id` **UUID**, not a slug. For multi-auth connectors it defaults to the first auth type (e.g. MCP → Bearer Token), so it is **not** a reliable deeplink for MCP OAuth — use the `…/connections` slug form above |
| `…/overview` | `?action=publish` | Opens the publish (preview→production) modal |

The org/hub-less form `app.wayai.pro/settings/connections?connector=<slug>` has **no route and 404s** — there is no `/settings/connections` page; connections only exist under `/settings/organizations/<orgId>/hubs/<hubId>/`. For the OAuth-connection handoff (the canonical target; see SKILL.md → Connections & Credentials → OAuth connection handoff), and any time an OAuth connection is needed, always use the full path:

```
https://app.wayai.pro/settings/organizations/<orgId>/hubs/<hubId>/connections?connector=<slug>
```

`<slug>` is one of `whatsapp`, `instagram`, `mcp-server`. The `orgId` and `hubId` come from `wayai status --json` (or the platform's last-viewed state).

## Rules

- **Always hand over one deep link.** Never describe a breadcrumb path — the one exception is the desktop app's settings, which no link reaches ([Desktop app](#desktop-app)). If the agent doesn't know the `orgId` / `hubId`, run `wayai status --json` first to resolve them.
- **Never invent paths.** Only use URLs documented here. New routes must be added to this file (and `APP_ROUTE_PREFIXES` if top-level) before they can be linked.
- **Locale prefix only for marketing.** App routes (`/chat`, `/support`, …) are not locale-prefixed; the user's language preference is read from UserDO.
- **Most auth routes can be linked to** (e.g. `/login`, `/verify-email`). The two exceptions in the Auth Routes table — `/callback` and `/oauth/authorize` — cannot, because they require live state from the auth provider.

# App Navigation & Deep Links

Canonical URL surface of the WayAI web app at `https://app.wayai.pro` (replace with `https://mac{1-9}.wayai-dev.com` for local dev). Use this when guiding the user to a screen — never invent paths or describe breadcrumbs ("go to Settings → Hubs → …"). Always hand over a single deep link.

## Table of Contents
- [Top-Level Map](#top-level-map)
- [Auth Routes](#auth-routes)
- [Conversation Surfaces](#conversation-surfaces)
- [Settings Hierarchy](#settings-hierarchy)
- [User Settings](#user-settings)
- [Marketing & Docs](#marketing--docs)
- [Query-String Deep Links](#query-string-deep-links)
- [Rules](#rules)

## Top-Level Map

Visiting `/` runs `(app)/page.tsx`, which redirects to the user's `last_viewed_nav` preference if valid, otherwise to a role-default in this order: `/chat` → `/task` → `/support` → `/settings`.

| Prefix | Purpose | Who sees it |
|--------|---------|-------------|
| `/chat` | Conversations where the signed-in user is the **end user** of a `chat` hub | Hub Users |
| `/task` | Task-oriented conversations (one user, many concurrent) | Hub Users on `task` hubs |
| `/support` | Inbox / kanban for team members handling end-user conversations | Hub Team Users, Hub Admins |
| `/settings` | Org + hub configuration (full-screen drill-down) | Org Owners/Admins, Hub Admins |
| `/user` | Personal settings (profile, preferences, tokens) | All authenticated users |

Access is per-level: an Org Admin who isn't also a Hub Admin gets an auth error at hub-scoped URLs. Top-level prefixes are the canonical list in `frontend/src/lib/app-routes.ts` (`APP_ROUTE_PREFIXES`).

## Auth Routes

| Path | Purpose |
|------|---------|
| `/login` | WorkOS sign-in (also handles signup) |
| `/callback` | WorkOS OAuth callback — never link directly |
| `/verify-email` | Magic-code email verification |
| `/auth-error` | Error landing page from auth flow |
| `/welcome` | Post-signup landing |
| `/success` | Post-flow success page |
| `/oauth/authorize` | Standalone Connect Login URI (CLI / MCP OAuth) — never link directly |

Auth pages use **inline translations** (en/pt/es objects in the same file), not `messages/*.json`. They run before a session exists.

## Conversation Surfaces

### Chat
| Path | Purpose |
|------|---------|
| `/chat` | Last-active chat conversation (single per user per hub) |
| `/chat/[hubId]` | Chat with a specific hub |

### Task
| Path | Purpose |
|------|---------|
| `/task` | Task inbox |
| `/task/hubs/[hubId]` | Task list scoped to one hub |
| `/task/[conversationId]` | Specific task conversation |

### Support
| Path | Purpose |
|------|---------|
| `/support` | Default support inbox |
| `/support/hubs/[hubId]` | Support inbox for one hub (list / kanban) |
| `/support/hubs/[hubId]/users` | Hub Users directory for one hub |
| `/support/hubs/[hubId]/users/[hubUserId]` | Individual Hub User profile |
| `/support/[conversationId]` | Specific support conversation |

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
| `/settings/organizations/[orgId]/tags` | Org tags — organize hubs/credentials and gate credential resolution (see [connections.md](connections.md#organization-tags)) |
| `/settings/organizations/[orgId]/administrators` | Org admin members |

### Hub level
Hub-detail tabs live under `/settings/organizations/[orgId]/hubs/[hubId]/<tab>`. Valid tabs (from `useSettingsPath.ts`):

| Tab | Path suffix | Covers |
|-----|-------------|--------|
| `overview` | `/overview` | Name, description, SLA, timezone, AI mode, permissions |
| `connections` | `/connections` | LLM providers, channel APIs, MCP servers, OAuth configs |
| `agents` | `/agents` | Agents list |
| `state` | `/state` | Conversation/user state schemas |
| `resource` | `/resource` | Knowledge bases, skill resources |
| `evals` | `/evals` | Eval scenarios + results |
| `outbound` | `/outbound` | Outbound campaigns |
| `analytics` | `/analytics` | Hub metrics |
| `users` | `/users` | Team users, Hub Users, admins (sub-tabs: `admins` \| `users` \| `teams` \| `support_model`) |

### Hub sub-entities
| Path | Purpose |
|------|---------|
| `/settings/organizations/[orgId]/hubs/[hubId]/agents/[agentId]` | Agent detail |
| `/settings/organizations/[orgId]/hubs/[hubId]/connections/[connectionId]` | Connection detail |
| `/settings/organizations/[orgId]/hubs/[hubId]/connections/new?connector=<slug>` | Create connection pre-picked to a connector |

### Hub-only shortcut
| Path | Purpose |
|------|---------|
| `/settings/hubs/[hubId]` | Direct hub access (resolves org from membership) |

## User Settings

Reached via the avatar menu (bottom of sidebar) → User Settings.

| Path | Purpose |
|------|---------|
| `/user` | Default user-settings tab |
| `/user/profile` | Name, email, avatar |
| `/user/preferences` | Theme, language, notifications |
| `/user/authorized-apps` | OAuth-authorized apps (MCP clients, third parties) |
| `/user/api-tokens` | Personal `way_` API tokens for CLI / MCP / direct API |

## Marketing & Docs

Public, locale-prefixed (`/`, `/en`, `/pt`, `/es`). Default locale (`en`) renders unprefixed; non-EN visitors get redirected to their locale by `intlMiddleware`.

| Path | Purpose |
|------|---------|
| `/` | Marketing home |
| `/pricing` | Plans + pricing |
| `/docs` | Docs index |
| `/docs/get-started` | Onboarding entry point — picks install command per harness, then SKILL.md drives state 1+ |
| `/privacy` | Privacy policy |
| `/terms` | Terms of service |

## Query-String Deep Links

Tab pages accept these query strings to pre-open modals or pre-fill forms.

| Tab | Query | Effect |
|-----|-------|--------|
| `…/credentials` | `?prefill=true&name=<name>&type=<api_key|bearer|basic_auth>` | Opens the create-credential modal pre-filled |
| `…/connections` | `?connector=<slug>` | Scrolls to and highlights the matching connector card |
| `…/connections/new` | `?connector=<slug>` | Opens connector picker with that connector pre-selected |
| `…/overview` | `?action=publish` | Opens the publish (preview→production) modal |

Onboarding-time shorthand used in SKILL.md (e.g. `app.wayai.pro/settings/connections?connector=whatsapp`) requires the **org and hub** in scope. When in an active hub-setup flow, expand to the full path:

```
https://app.wayai.pro/settings/organizations/<orgId>/hubs/<hubId>/connections?connector=whatsapp
```

The `orgId` and `hubId` come from `wayai status --json` (or the platform's last-viewed state).

## Rules

- **Always hand over one deep link.** Never describe a breadcrumb path. If the agent doesn't know the `orgId` / `hubId`, run `wayai status --json` first to resolve them.
- **Never invent paths.** Only use URLs documented here. New routes must be added to this file (and `APP_ROUTE_PREFIXES` if top-level) before they can be linked.
- **Locale prefix only for marketing.** App routes (`/chat`, `/task`, …) are not locale-prefixed; the user's language preference is read from UserDO.
- **Most auth routes can be linked to** (e.g. `/login`, `/verify-email`, `/welcome`). The two exceptions in the Auth Routes table — `/callback` and `/oauth/authorize` — cannot, because they require live state from the auth provider.

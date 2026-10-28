# Pages

A **page** is a small working tool a hub offers its people: a coach entering swim-meet results at the pool, a back-office review queue. It is one self-contained HTML file plus a manifest. The page reads and writes the hub's [bases](bases/README.md) through the hub's **Base Toolset** connections ([connections.md](connections.md#base-toolset)), calling only the toolset tools its manifest names. A page is an [MCP Apps](https://blog.modelcontextprotocol.io/posts/2026-01-26-mcp-apps/) view, and WayAI is its host. A coding agent that knows MCP Apps already knows the page API.

Pages are hub config: you write them in the hub folder, push them to a preview with `wayai push`, and publish them with the rest of the hub. Each page serves one audience. Today that is the hub's **team** only; customer pages come later.

Start a page with `wayai page new <slug>`, which writes one already styled like the app, and read [Designing a Page](#designing-a-page) before you change its look.

> **Where people open them.** On web, a team member opens a page from the Support nav: from a conversation's header, which offers the hub's pages with `show_in_conversation: true`, and from the hub's Pages section, which lists every page offered to them; that header and the Support nav's **Pages** item open it. In the mobile app, a team member opens a page from a support conversation's overflow menu (**⋮**), which lists the pages with `show_in_conversation: true` and then **All pages** for the rest, or every page offered to them when none sets it; the page opens in a full-screen sheet over the conversation, closed by its close button or by dragging its top bar down. The mobile app has no Pages section.

## Table of Contents
- [Folder Layout](#folder-layout)
- [The Manifest](#the-manifest)
- [Who Sees a Team Page](#who-sees-a-team-page)
- [Push, Pull, Diff and Publish](#push-pull-diff-and-publish)
- [Writing a Page as an MCP Apps View](#writing-a-page-as-an-mcp-apps-view)
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
team_writes: true               # team users without a grant may use this page
show_in_conversation: true
order: 1
params:                         # what a link may open the page with: a JSON Schema object
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
| `audience` | Yes | `team`. Customer pages are not available yet |
| `team_writes` | No (`false`) | Opens the page to team users without a grant on the hub ([Who Sees a Team Page](#who-sees-a-team-page)) |
| `show_in_conversation` | No (`false`) | Offers the page from the conversation header to people who may open it. It orders what a conversation offers first; it never hides a page: on web every page offered is in the Pages section, and in the mobile app every page offered is in the conversation's overflow menu ([Where people open them](#pages)) |
| `order` | No (`0`) | Menu position, 0–1000, lowest first; pages of equal order sort by slug |
| `params` | No | A JSON Schema whose root is `type: object`, at most 16 KiB of JSON: the values a link may open the page with |
| `tools` | No (`[]`) | Up to 50 toolset tools the page calls, each `{ connection, tool, notify }` |

**Each tool** names a hub connection by its name in `hub.yaml` (`connection:`) and a tool by its name in that connection's synced tool list (`tool:`). The page calls a tool by its name alone, so no two entries may share a tool name, even across connections. Push refuses an entry when:

- the hub has no connection of that name, or two connections share it;
- the connection is not a Base Toolset;
- the connection has no token yet (save it first);
- the toolset is customer-scoped, since a team member names no single customer;
- the tool is not in the connection's synced tool list, or is turned off there (sync its tools first: `wayai sync-mcp --connection "<name>"`);
- the tool hands back a signed file URL, which a page never receives.

**`notify`** (`none`, the default, or `summary`) is the most the conversation's agent may hear of the page's calls to that tool. Today the agent hears nothing of a page's calls. `notify` declares the ceiling the page will keep once page summaries arrive.

The platform decides whether a tool reads or writes from the toolset, never from the manifest. Anyone a page admits may call any tool it declares, with any arguments that tool's schema accepts, so declare only the tools the page needs. The toolset's own rules (`writable_fields`, `base_filter`, `precondition`: [bases/toolsets.md](bases/toolsets.md)) still hold for every call.

## Who Sees a Team Page

A team page is offered only to people who may write through it:

- an organization member or hub admin whose grant on the hub includes `conversation:write`;
- one of the hub's team users, when the page sets `team_writes: true`.

Anyone else does not see the page. The base records each write a page makes as made by **the person using it**, with their email in its audit. A page still writes as a tool does: a write to a reviewed record type stays a draft, as an agent's does.

## Push, Pull, Diff and Publish

- **The `pages/` folder is the desired set, when it exists.** A hub folder with a `pages/` folder sends every page in it. If `pages/` is empty, push deletes every page.
- **Without a `pages/` folder, push leaves the hub's pages as they are.** So a folder an older CLI wrote never deletes a page. To delete one page, delete its folder.
- **Pull** writes each page's folder whenever the hub has pages, removes the folders of deleted pages, and removes `pages/` once the hub has none. `page.yaml` comes back with its defaults left out.
- **Diff** shows pages under **Pages:**: `+ <slug>` added, `- <slug>` deleted, and `~ <slug>: <fields>` changed, where `html` in the field list means the HTML changed. Every push-time refusal above prints with push's other warnings, and the push writes none of the refused pages. A page the push leaves in place keeps every connection it names from being deleted by that push.
- **Publish** copies the preview's pages to production, by slug. The publish summary lists added, updated and removed pages under **Pages**. Every publish that syncs a change checks every page again against **production's own** connections and synced tool lists, the pages it does not change included, and a page production refuses fails the publish, naming its issues. A publish with nothing to sync (`Production is already up to date`) checks nothing.
- **A connection with no token yet is checked only partly when a page is copied.** A toolset connection's token never travels with a copy. So on a first publish, a new preview or a CI branch preview, each toolset connection arrives with no token, and its pages are still copied. Only the checks that need the token wait: customer scope and signed file URLs. The page calls nothing through that connection until the hub sets its token (`wayai set-connection-credential` on a preview; the production credential in the connections tab on production). The first publish that syncs a change after production's token is saved checks the page fully; saving the token alone is no change to publish.
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
| `ping` | request | Answered empty |
| `notifications/message` | notification | A log line |

**What the host refuses**, each with a JSON-RPC error: `ui/open-link`, `ui/update-model-context`, `ui/message`, `ui/download-file` and `resources/read`.

**What the host does not support:** display-mode requests, and the tool-input and tool-result notifications. No tool call opens the page: people open it from the hub, so a page never waits for a tool's input or result to start.

**Params** arrive in the host context as `wayai.params`. Read them from the `ui/initialize` answer, and ignore anything the schema in `params:` would not accept: links are written by people and agents.

**A tool call can fail for reasons a page should show, not swallow:** the tool refused (`isError`), the viewer may no longer use the page, too many calls, or the page was updated since it opened (reload it).

On web and in the mobile app, a refusal of the call itself comes back as a tool result with `isError: true`, a text written for the viewer, and a `structuredContent.code`: `PAGE_REVISION_STALE` when the page was updated (the host also offers the viewer a reload), `RATE_LIMIT_EXCEEDED` with `retry_after_seconds` when the page called too often (the host answers every call inside that wait the same way, without sending it), or the refusal's own code otherwise: for example `VALIDATION_ERROR` for arguments the tool's schema refuses, and `AUTHORIZATION_ERROR` or `NOT_FOUND`, whose causes, and what to show for each, are in [the refusal table](#loading-empty-and-error-states). A refusal by the tool itself, or by a connection that no longer reaches it, is `isError: true` with its text and no `structuredContent`. A call that fails on the network or with a server error gets a JSON-RPC error.

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

## Designing a Page

A page looks like the WayAI app by default. The host hands every page WayAI's look, and this guide says how to build on it. Nothing here is enforced: push warns about a few of these rules ([The Design Warnings](#the-design-warnings)) and never refuses a page for its design.

### Start From the Scaffold

```bash
wayai page new results                          # pages/results/page.yaml and pages/results/index.html
wayai page new review-queue --title "Review queue" --hub support
```

`wayai page new <slug>` writes a starter page into a hub folder, found by `wayai push`'s hub steps: the one `--hub` names, else the hub folder you are in, else the workspace's `default_hub`, else its only preview or new hub folder. Base folders and `default_base` play no part. `--title` sets what menus show (the slug in words by default), and `--audience` is `team`, the default and the only audience today. It never overwrites a file. Before writing anything, it refuses a slug that is not a folder name and `--audience customers` in the words push refuses them in, and it refuses a production mirror's folder and a hub folder already holding 20 pages.

The starter page passes push as written, with no design warning. It already does what the rest of this section asks:

- it is styled only with the tokens and the base classes;
- it shows its list as one card per row on a narrow frame, and as a table once its container is 560px wide;
- it speaks the MCP Apps messages by hand: no SDK to inline;
- it follows the viewer's theme and language, and pads itself by the safe-area insets;
- it shows loading, empty and error states, the host's refusal codes among the errors, with every string in English, Portuguese and Spanish.

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
| `platform` | `web` in the web app, `mobile` in the mobile app |
| `containerDimensions` | The page's frame: `width` and `height` in CSS pixels |
| `safeAreaInsets` | How far the device's safe area reaches into the frame, per side, in CSS pixels |

Once the page sends `ui/notifications/initialized`, a change reaches it as `ui/notifications/host-context-changed`, carrying only the fields that changed: a theme switch, a language switch, a resize of its frame, or a change of its safe-area insets. A change made before `initialized` arrives right after it.

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
| `isError` with `NOT_FOUND` | In your words, that the page or what it shows is not available. The result's text says which: `Page not found` (no longer offered to this viewer) or `Page tool not found` (the page called a tool its `page.yaml` does not list) |
| `isError` with `VALIDATION_ERROR` | The tool refused the arguments: point at the field, with the result's text |
| `isError` with no `structuredContent` | The tool's own refusal: show its text |
| A JSON-RPC error | The call did not get through: "Could not reach WayAI. Try again." |

### UI Text

- **Short and plain.** Sentence case, no exclamation marks, no jargon. Address the viewer as "you".
- **Buttons name their action:** "Save result", "Delete athlete", not "OK" or "Submit".
- **An error says what happened and what to do next**, never a code, an id or a stack trace.
- **A confirmation names what it will do:** "Delete Ana's result?" with "Delete" and "Cancel" buttons (`confirm()` returns `false` in a page).
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
- **No popups, modals or downloads.** `window.open` does nothing. `alert()` and `prompt()` do nothing, and `confirm()` returns `false`, so confirm a delete in the page's own UI (an "Are you sure?" button pair). A page cannot download or export a file.
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
| Audiences | `team` |

# Pages

A **page** is a small working tool a hub offers its people: a coach entering swim-meet results at the pool, a back-office review queue. It is one self-contained HTML file plus a manifest. The page reads and writes the hub's [bases](bases/README.md) through the hub's **Base Toolset** connections ([connections.md](connections.md#base-toolset)), calling only the toolset tools its manifest names. A page is an [MCP Apps](https://blog.modelcontextprotocol.io/posts/2026-01-26-mcp-apps/) view, and WayAI is its host. A coding agent that knows MCP Apps already knows the page API.

Pages are hub config: you write them in the hub folder, push them to a preview with `wayai push`, and publish them with the rest of the hub. Each page serves one audience. Today that is the hub's **team** only; customer pages come later.

> **Rolling out.** Pages can be written, pushed, pulled and published today, but people cannot open them in the WayAI app yet: the page viewer and the hub's Pages section are still to come. Don't tell anyone a page is ready to use until the hub shows its Pages section.

## Table of Contents
- [Folder Layout](#folder-layout)
- [The Manifest](#the-manifest)
- [Who Sees a Team Page](#who-sees-a-team-page)
- [Push, Pull, Diff and Publish](#push-pull-diff-and-publish)
- [Writing a Page as an MCP Apps View](#writing-a-page-as-an-mcp-apps-view)
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
| `description` | No | Up to 500 characters; shown under the title in menus |
| `audience` | Yes | `team`. Customer pages are not available yet |
| `team_writes` | No (`false`) | Opens the page to team users without a grant on the hub ([Who Sees a Team Page](#who-sees-a-team-page)) |
| `show_in_conversation` | No (`false`) | Offers the page from the conversation header to people who may open it |
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

Anyone else does not see the page. Until person attribution arrives, the base records each write as made by **the page**, not by the person who made it.

## Push, Pull, Diff and Publish

- **The `pages/` folder is the desired set, when it exists.** A hub folder with a `pages/` folder sends every page in it. If `pages/` is empty, push deletes every page.
- **Without a `pages/` folder, push leaves the hub's pages as they are.** So a folder an older CLI wrote never deletes a page. To delete one page, delete its folder.
- **Pull** writes each page's folder whenever the hub has pages, removes the folders of deleted pages, and removes `pages/` once the hub has none. `page.yaml` comes back with its defaults left out.
- **Diff** shows pages under **Pages:**: `+ <slug>` added, `- <slug>` deleted, and `~ <slug>: <fields>` changed, where `html` in the field list means the HTML changed. Every push-time refusal above prints with push's other warnings, and the push writes none of the refused pages. A page the push leaves in place keeps every connection it names from being deleted by that push.
- **Publish** copies the preview's pages to production, by slug. The publish summary lists added, updated and removed pages under **Pages**. Every publish that syncs a change checks every page again against **production's own** connections and synced tool lists, the pages it does not change included, and a page production refuses fails the publish, naming its issues. A publish with nothing to sync (`Production is already up to date`) checks nothing.
- **A connection with no token yet is checked only partly when a page is copied.** A toolset connection's token never travels with a copy. So on a first publish, a new preview or a CI branch preview, each toolset connection arrives with no token, and its pages are still copied. Only the checks that need the token wait: customer scope and signed file URLs. The page calls nothing through that connection until the hub sets its token (`wayai set-connection-credential` on a preview; the production credential in the connections tab on production). The first publish that syncs a change after production's token is saved checks the page fully; saving the token alone is no change to publish.
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

**Params** arrive in the host context, under a key in WayAI's namespace. Read them from the `ui/initialize` answer, and ignore anything the schema in `params:` would not accept: links are written by people and agents.

**A tool call can fail for reasons a page should show, not swallow:** the tool refused (`isError`), the viewer may no longer use the page, too many calls, or the page was updated since it opened (reload it).

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

    // The MCP Apps protocol version this page speaks (the SDK fills it in for you).
    const PROTOCOL_VERSION = '<the MCP Apps protocol version>';

    async function main() {
      await request('ui/initialize', {
        protocolVersion: PROTOCOL_VERSION,
        appInfo: { name: 'results', version: '1.0.0' },
        appCapabilities: {},
      });
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

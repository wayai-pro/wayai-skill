# Communities

A community belongs to an organization: its members, rooms and courses, joined by its link. WayAI
creates communities; once one exists, the organization's **owners and admins** manage it.

| Task | Where |
|------|-------|
| List the organization's communities; read, rotate or turn off a join link; read the stats; the access list of an invite-only community | CLI (`wayai communities`) or the web console |
| Settings (including who may join: `admission`), rules, the join link's on/off state, rooms and courses | Files in the workspace, applied with `wayai pull` / `wayai push communities/<folder>` ([Communities as code](#communities-as-code)), or the web console |
| Moderators and moderation | The web console: organization settings → Communities (`/settings/organizations/<org_id>/communities`) |

## Sign-in

`wayai communities`, and `wayai pull` / `wayai push` of a community, need a **browser
sign-in**: run `wayai login`. An API token (`wayai login --token`, or `WAYAI_TOKEN` in the
shell) is refused, so these commands do not run from CI. When `WAYAI_TOKEN` is set, it is sent
instead of the browser sign-in: unset it before running these commands.

The caller must be an owner or admin of the organization that owns the community. Anyone else
gets the same "not found" an unknown community gets.

## Commands

```bash
wayai communities list [--org <org_id>] [--json]
wayai communities join-link get <community> [--org <org_id>] [--json]
wayai communities join-link rotate <community> [--org <org_id>] [-y] [--json]
wayai communities join-link off <community> [--org <org_id>] [-y] [--json]
wayai communities stats <community> [--org <org_id>] [--json]
wayai communities access list <community> [--org <org_id>] [--json]
wayai communities access add <community> <email>... [--org <org_id>] [--json]
wayai communities access remove <community> <email> [--org <org_id>] [-y] [--json]
```

- **`<community>`** is a community id (from `list`), or the community's exact name within the
  organization — case and spacing included. Two communities sharing a name are refused with both
  ids; name one by its id.
- **The organization** is `--org`, else the repo's (`wayai-ws/wayai.yaml`, set by `wayai init`).
  `list` and a name need one; an id does not.
- **`--json`** prints the response's `data` as is: the list of communities, the join link
  (`enabled`, `url`), the stats, or what an access-list add did (`added`, `already_listed`).
  `access list --json` prints every entry (`access_id`, `email`, `created_at`); `access remove
  --json` prints the entry it removed.

### The join link

- `join-link get` prints the link, or says it is off.
- `join-link rotate` replaces the link with a new one and turns it on.
- `join-link off` turns it off: the stop for a leaked link.

Rotating and turning off both make **every link shared before stop working**. Both print that, on
stderr, and ask first; `-y` skips the question (never the warning), and is required when the
command does not run in a terminal.

To see which channel brings people, post the link with a source tag on each one: add
`&src=<tag>` (lowercase letters, digits, `_` and `-`, at most 32 characters). `stats` counts the
joins per tag.

### Invite-only: the access list

A community's `admission` setting decides who its link admits: `link` (the default) admits anyone
holding it; `invite` admits only the organization's owners and admins and the people whose
verified email is on the community's **access list** — a forwarded link admits nobody else. Set
`admission` in `community.yaml` ([Communities as code](#communities-as-code)) or in the web console.

- `access list` prints the emails on the list, oldest first.
- `access add` adds emails: at most 500 at a time, and a list holds at most 10,000. An email
  already on the list is left as it is. Each email is checked first, and the command refuses,
  naming every value that is not an email, before it sends anything.
- `access remove` takes one email off the list, and **removes from the community anyone that
  email let in**, whatever the admission. It prints that, on stderr, and asks first; `-y` skips the
  question (never the warning), and is required when the command does not run in a terminal.

`access list` reads the list 500 emails per request, and `access remove` reads it as far as the
email it removes; each request counts against the per-person budget every community request shares
(60 a minute).

An email matches exactly once trimmed and lowercased: `Ana@Example.com` is `ana@example.com`, but
`ana+news@example.com` is a different email. The list shows only the emails entered, never whom
they let in. The list is edited only with these commands or in the web console, never as workspace
files: it holds people's emails. WayAI sends no invitations: share the join link through your own
channels.

Making a community invite-only removes nobody: current members stay, including anyone who joined
with a shared link, and the access list decides only who joins from then on. Switching back to
`link` keeps the list.

### Stats

`stats` prints the members (active, left, removed, banned), how many were active in the last 7
days, the joins and the new accounts among them over the last 7 and 30 days, the joins per source
tag over the last 30 days, the messages of the last 30 days (chat and announcements, forum posts,
forum replies) and each course's completions per lesson, drafts included. `--json` adds the
messages per day.

## Communities as code

A community's settings, rules, join link state, rooms and courses live as files in the workspace,
applied with the same `wayai pull` / `wayai push` used for hubs and bases.

```
wayai-ws/communities/<folder>/
├── community.yaml          # identity, settings, join link, rooms, the course folders in order
├── rules.md                # the rules members see (Markdown); empty = none
└── courses/
    └── <course>/
        ├── course.yaml     # the course, its modules, and each module's lessons in order
        └── lessons/
            └── <lesson>.md # the lesson's fields, then its text (Markdown)
```

```bash
wayai pull communities/<community_id>     # first pull: writes wayai-ws/communities/<folder>/
wayai pull communities/<folder>           # take what changed on the server
wayai push communities/<folder>           # list the changes, ask, apply them
wayai push communities/<folder> --dry-run # list the changes only
wayai pull communities/<folder> --theirs  # resolve conflicts with the server's side
```

**A community is always named**: `communities/<folder>` (or `<community_id>`),
`--community <folder|community_id>`, or run from inside its folder. A bare `wayai pull` /
`wayai push` never reaches a community, and a bare folder name never means one. `wayai use` and
`wayai unbind` take the same forms (`wayai use communities/<folder>`): the worktree scope has a
communities axis beside hubs and bases, seeded by the first pull or push.

Pull and push run for the community's organization only: the workspace must be bound to the
organization that owns it (`wayai-ws/wayai.yaml`). Push never creates a community — WayAI does.

### `community.yaml`

```yaml
community_id: 6b1e…            # read-only
name: Acme Builders            # read-only: there is no rename
settings:                      # settings and rules.md share one rev
  rev: v1:4d1e9a07c2b3
  slow_mode_seconds: 30        # 0 is off
  courses_label: Academy       # '' for the default
  admission: invite            # link (the default): anyone with the link; invite: owners, admins and the access list
join_link_enabled: true        # the link itself is never written to disk
join_link_rev: v1:77ac01e5f9d2
rooms:
  - key: general               # stable local name; a rename keeps it
    id: 1c4a…
    rev: v1:0b93c4d2e1a8
    name: General
    kind: discussion           # discussion, forum or announcements; read-only once created
  - key: q-and-a               # no id: push creates it
    name: Q&A
    kind: forum
  - key: old-room
    id: 88d2…
    rev: v1:5e0f1b6a9c47
    name: Old room
    kind: discussion
    archived: true
courses: [onboarding, advanced] # course folders under courses/, in the order members see them
synced:                        # the courses' order as of the last sync, by id: leave it alone
  courses: [3f1c…, 5d0a…]
```

- **`key`** names a room in the workspace. Pull picks it from the room's name (a name already
  used becomes `<name>--<id-prefix>`); you pick it for a new room. Keep it: a rename changes
  `name`, never `key`. A key push created a room under keeps naming it: a new entry reusing it gets
  that room back, and push changes the room to match the entry. A key pull gave to a room made in
  the web console carries no such link — reusing it makes a new room.
- **`id`, `rev` and `synced:`** are written by pull and push. A `rev` records the server state the
  entry was last synced from, and `synced:` the orders; leave them alone, and commit them with the
  rest of the file.

### Courses

`courses/<course>/course.yaml`:

```yaml
id: 3f1c…
rev: v1:9f2c1a8b0e64           # title, description, published
title: Onboarding
description: Your first week   # '' for none
published: false               # members see a course once it is published
modules:                       # in the order members see them
  - key: getting-started       # stable local name; a rename keeps it
    id: 9a2e…
    rev: v1:c08e5b1d3f72       # title
    title: Getting started
    lessons: [welcome, first-steps]   # files in lessons/, without .md, in order
  - key: going-further         # no id: push creates it
    title: Going further
    lessons: [automations]
synced:                        # the orders as of the last sync, by id: leave it alone
  modules: [9a2e…]
  lessons:
    9a2e…: [77b0…, 1d4c…]      # each module's lessons, a module left out of modules: included
```

`courses/<course>/lessons/<lesson>.md`:

```markdown
---
id: 77b0…
rev: v1:3b6e0f9a2d18
title: Welcome
published: true                # members see a published lesson of a published course
room: general                  # a room's key in community.yaml, for its discussion; omit for none
video_url: https://youtu.be/…  # YouTube only; omit for none
links:                         # at most 10
  - title: Setup guide
    url: https://example.com/setup
requires_paid_plan: false      # members see the mark; nothing enforces it
---
The lesson's text, in Markdown: at most 20,000 characters.
```

- **Names.** A course folder's name, a module's `key` and a lesson file's name are their names in
  the workspace. Pull picks them from the title (a name already used becomes `<name>--<id-prefix>`);
  you pick them for new ones — plain names, no `/`. Keep them: a rename changes `title`, never the
  name. A name push created a course, module or lesson under keeps naming it: a new folder, entry or
  file reusing that name gets the same one back, and push changes it to match the files. A name pull
  gave to one made in the web console carries no such link — reusing it makes a new one.
- **Order** is list order: `courses:` in `community.yaml`, `modules:` in `course.yaml`, each module's
  `lessons:`. Moving a lesson's name to another module's list moves the lesson there. Nothing moves to
  another course: pull and push refuse a module or lesson whose id belongs to another course on the
  server.
- **What a list leaves out stays as it is**, where it is: a course, module or lesson the server has
  and the files do not list. Pull adds it to the files.
- **Drafts in progress.** A lesson file no module lists, and a course folder `courses:` does not
  list, are skipped by pull and push, with a warning.
- **Copies.** A copied folder, entry or file keeps its original's `id`: push and pull refuse an id
  used twice. Remove the `id` and the revs from the copy to make a new one.
- **The text** follows the second `---`. The file ends with one newline after it, which is not
  part of the text; anything else — blank lines, a text that itself starts with `---` — is kept as
  written.

### How pull and push decide

For the settings (with `rules.md`), the join link, each room, course, module and lesson, and the
order of each list — the courses, a course's modules, a module's lessons — the files, the server and
the `rev` are compared:

| Since the last sync | push | pull |
|---|---|---|
| Changed in the files only | applies it | keeps it (not pushed yet) |
| Changed on the server only | leaves it, and says to pull | takes it |
| Changed alike on both sides | nothing to apply | nothing to take |
| Changed differently on both sides — a conflict | refuses the whole push | leaves the files' side, says so and exits non-zero; `--theirs` takes the server's, and an edit of yours made again on top of it then pushes |

So a push does not undo an edit made in the web console before it reads the community, and a pull
never overwrites an edit you have not pushed. When push asks before applying, it reads the community
again after your answer and stops, writing nothing, if anything changed while it waited; with `-y` it
applies right after its read. A room, course, module or lesson on the server that the files do not
list is left as is by push, and added to the files by pull.

A list's order is compared against its `synced:` list — the ids in the server's order as of the last
sync — apart from what each entry holds: a lesson moved in the web console and its text edited in the
files are two changes, not a conflict. So is each lesson's module: moved in the files only, push moves
it; moved on the server only, pull takes it; moved to different modules on both sides, a conflict.
Only the entries the files list take part: one left out, or made in the web since, is no change to
either side.

### What push does, and what to know

- **Push changes the live community.** There is no preview: members see the change at once. Push
  lists every change, states how many requests it makes, and asks; `-y` skips the question and is
  required when not run in a terminal. `--dry-run` lists the changes and writes nothing.
- **Nothing is deleted.** Removing a room's entry from the files leaves the room as is; set
  `archived: true` to archive it (it takes no new message, its history stays readable), and remove
  `archived` to bring it back. Likewise a course, module or lesson left out of the files stays;
  `published: false` takes a course or a lesson off the community.
- **Limits, checked before anything is written.** A community holds at most 20 rooms, archived ones
  included, and 20 courses; a course at most 20 modules and 100 lessons; a community at most 500
  lessons in all — drafts included. Push refuses a change past one before writing anything.
- **Publishing comes last.** Push takes courses, then lessons, off the community first, and publishes
  a course last, once its modules and lessons are in place: a new course is created as a draft, and a
  new course with `published: true` goes live at the end of the push.
- **Read-only fields** — `community_id`, `name`, a created room's `kind` — refuse the push when
  changed in the files.
- **`join_link_enabled: true` over a link that is off makes a new link**: every link shared before
  stays dead. To replace an active link, or turn off a leaked one at once, use
  `wayai communities join-link rotate|off`.
- **`admission: invite`** makes the link admit only the organization's owners and admins and the
  emails on the access list ([Invite-only](#invite-only-the-access-list)). Push's listing says what
  the change does before it asks: no member is removed. `admission: link` opens the link to anyone
  again, and the access list is kept. A file without `admission` reads as `link`.
- **The files are rewritten** in a fixed layout: by pull, and by push for each file a change it
  applies records something new in — an id, a rev, a `synced:` order, or the rules as the server
  saved them. Comments in a rewritten file are not kept; a file push records nothing new in stays
  as it is. Commit after a push — the files then record the ids and revs it wrote.
- **A push stopped part way** (an error, Ctrl-C) keeps what it applied, recorded in the files.
  Fix the cause and push again: it continues from there, and a room, course, module or lesson it
  already created is not created twice. On Ctrl-C push first finishes the write in flight and records
  it; a second Ctrl-C leaves at once.
- **The server checks a lesson's text** when it is saved — how deeply lists, quotes and emphasis
  nest, how far a line is indented, how large it is once formatted. A text it refuses stops the push
  at that lesson, naming its file.
- **Pace.** Every request of a pull or push counts against the per-person budget your web console
  and your messages share (60 a minute). Pull and push pace themselves at 50 a minute, and a
  request turned away for the budget is sent again after the wait the server names. A pull or a
  push reads five times plus once per course, and a push that asks reads them again after your
  answer. A push then writes each entry it creates or changes, each course, module or lesson it
  moves, and each course it publishes, and states those requests and the time they take before it
  asks. A first import of a community at the limits — 20 published courses of 20 modules, 500
  lessons — makes 940 writes after its reads: about 19 minutes.

## When a command fails

| Message | What to do |
|---------|-----------|
| Communities are managed from a browser sign-in only | Run `wayai login` (without `--token`); unset `WAYAI_TOKEN` if the message says it is set |
| … not found, or you are not an owner or admin of … | Check the id or name with `wayai communities list`, and that you are an owner or admin of the organization |
| Too many requests | Every community request — the web console's and your messages included — shares one per-person budget of 60 a minute. Wait the time the message names |
| No organization: pass --org | Pass `--org <org_id>`, or run inside a workspace set up with `wayai init` |
| Not an email / Not emails | Correct the values the message names: nothing was sent |
| An add takes at most … emails | Add them in batches of up to 500 |
| Nothing was added: the access list … would hold more than … | A list holds at most 10,000 emails: remove some first |
| … is not on the access list of … | Check the email with `wayai communities access list` |
| WayAI has suspended this account | A suspended account changes neither the access list nor the admission |
| … is not one of the communities of organization … | Pull and push a community from a workspace bound to the organization that owns it |
| Conflicts — changed in your files and on the server | `wayai pull communities/<folder> --theirs` takes the server's side; make again any edit of yours to keep, then push |
| Push stopped at … | Fix what the message names and push again: what was applied before it stays applied |
| Push stopped at lesson …: … change the lesson's text in … | The server refused the lesson's text (nested too deeply, indented too far, or too large once formatted): change it in that file and push again |
| … is one of course …'s on the server, and a … does not move between courses | Put the module or lesson back in its course. To have it in another course, make a new one there (a new entry or file, without an id) |
| … id … is also …'s — a copy keeps its original's id | Remove the `id` and its revs from the copy: push then creates it as a new one |
| … holds at most … | Remove creates from the files: nothing is deleted, and a draft still counts |
| Skipped: … — no module of the course lists it | A draft in progress: list the lesson in a module's `lessons:` to sync it |

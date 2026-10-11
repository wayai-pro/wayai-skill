# Communities

A community belongs to an organization: its members, rooms and courses, joined by its link. WayAI
creates communities; once one exists, the organization's **owners and admins** manage it.

| Task | Where |
|------|-------|
| List the organization's communities; read, rotate or turn off a join link; read the stats | CLI (`wayai communities`) or the web console |
| Settings, rules, the join link's on/off state and rooms | Files in the workspace, applied with `wayai pull` / `wayai push communities/<folder>` ([Communities as code](#communities-as-code)), or the web console |
| Courses, moderators and moderation | The web console: organization settings → Communities (`/settings/organizations/<org_id>/communities`) |

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
```

- **`<community>`** is a community id (from `list`), or the community's exact name within the
  organization — case and spacing included. Two communities sharing a name are refused with both
  ids; name one by its id.
- **The organization** is `--org`, else the repo's (`wayai-ws/wayai.yaml`, set by `wayai init`).
  `list` and a name need one; an id does not.
- **`--json`** prints the response's `data` as is: the list of communities, the join link
  (`enabled`, `url`), or the stats.

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

### Stats

`stats` prints the members (active, left, removed, banned), how many were active in the last 7
days, the joins and the new accounts among them over the last 7 and 30 days, the joins per source
tag over the last 30 days, the messages of the last 30 days (chat and announcements, forum posts,
forum replies) and each course's completions per lesson, drafts included. `--json` adds the
messages per day.

## Communities as code

A community's settings, rules, join link state and rooms live as files in the workspace, applied
with the same `wayai pull` / `wayai push` used for hubs and bases. Courses are not files yet: push
leaves every course as it is.

```
wayai-ws/communities/<folder>/
├── community.yaml   # identity, settings, join link, rooms
└── rules.md         # the rules members see (Markdown); empty = none
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
```

- **`key`** names a room in the workspace. Pull picks it from the room's name (a name already
  used becomes `<name>--<id-prefix>`); you pick it for a new room. Keep it: a rename changes
  `name`, never `key`. A key once used keeps naming its room — a new entry reusing it gets that
  room back, and push changes the room to match the entry.
- **`id` and `rev`** are written by pull and push. A `rev` records the server state the entry was
  last synced from; leave it alone, and commit it with the rest of the file.

### How pull and push decide

For the settings (with `rules.md`), the join link and each room, the files, the server and the
`rev` are compared:

| Since the last sync | push | pull |
|---|---|---|
| Changed in the files only | applies it | keeps it (not pushed yet) |
| Changed on the server only | leaves it, and says to pull | takes it |
| Changed alike on both sides | nothing to apply | nothing to take |
| Changed differently on both sides — a conflict | refuses the whole push | leaves the files' side, says so and exits non-zero; `--theirs` takes the server's, and an edit of yours made again on top of it then pushes |

So a push does not undo an edit made in the web console before it reads the community, and a pull
never overwrites an edit you have not pushed. When push asks before applying, it reads the community
again after your answer and stops, writing nothing, if anything changed while it waited; with `-y` it
applies right after its read. A room on the server that the files do not list is left as is by push,
and added to the files by pull.

### What push does, and what to know

- **Push changes the live community.** There is no preview: members see the change at once. Push
  lists every change, states how many requests it makes, and asks; `-y` skips the question and is
  required when not run in a terminal. `--dry-run` lists the changes and writes nothing.
- **Nothing is deleted.** Removing a room's entry from the files leaves the room as is; set
  `archived: true` to archive it (it takes no new message, its history stays readable), and remove
  `archived` to bring it back. A community holds at most 20 rooms, archived ones included; push
  refuses one past that before writing anything.
- **Read-only fields** — `community_id`, `name`, a created room's `kind` — refuse the push when
  changed in the files.
- **`join_link_enabled: true` over a link that is off makes a new link**: every link shared before
  stays dead. To replace an active link, or turn off a leaked one at once, use
  `wayai communities join-link rotate|off`.
- **The files are rewritten** after each change push applies and on pull, in a fixed layout:
  comments in `community.yaml` are not kept. Commit after a push — the files then record the ids
  and revs it wrote.
- **A push stopped part way** (an error, Ctrl-C) keeps what it applied, recorded in the files.
  Fix the cause and push again: it continues from there, and a room it already created is not
  created twice.
- **Pace.** Every request of a pull or push counts against the per-person budget your web console
  and your messages share (60 a minute). Pull and push pace themselves at 50 a minute, and a
  request turned away for the budget is sent again after the wait the server names. A pull or a
  push reads four times, and a push that asks reads them again after your answer; a push states
  the requests it will send before it asks.

## When a command fails

| Message | What to do |
|---------|-----------|
| Communities are managed from a browser sign-in only | Run `wayai login` (without `--token`); unset `WAYAI_TOKEN` if the message says it is set |
| … not found, or you are not an owner or admin of … | Check the id or name with `wayai communities list`, and that you are an owner or admin of the organization |
| Too many requests | Every community request — the web console's and your messages included — shares one per-person budget of 60 a minute. Wait the time the message names |
| No organization: pass --org | Pass `--org <org_id>`, or run inside a workspace set up with `wayai init` |
| … is not one of the communities of organization … | Pull and push a community from a workspace bound to the organization that owns it |
| Conflicts — changed in your files and on the server | `wayai pull communities/<folder> --theirs` takes the server's side; make again any edit of yours to keep, then push |
| Push stopped at … | Fix what the message names and push again: what was applied before it stays applied |

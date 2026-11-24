# Communities

A community belongs to an organization: its members, rooms and courses, joined by its link. WayAI
creates communities (at most 10 per organization); once one exists, the organization's **owners and
admins** manage it.

| Task | Where |
|------|-------|
| List the organization's communities; read, rotate or turn off a join link; read the stats; the access list (whom an invite-only community admits) and the tiers each entry gives, in any community | CLI (`wayai communities`) or the [web console](#the-web-console) |
| Settings (including who may join: `admission`, and which hubs' agents manage tiers: `tier_managers`), rules, the join link's on/off state, tiers, rooms and courses and the tiers they are gated to | Files in the workspace, applied with `wayai pull` / `wayai push communities/<folder>` ([Communities as code](#communities-as-code)), or the [web console](#the-web-console) |
| Give tiers from a payment provider or another system, with no person in the loop | An agent with the `community_tiers` tool, woken by a webhook automation ([Tiers from a webhook](#tiers-from-a-webhook)) |
| Moderators and moderation | The [web console](#the-web-console) and the community's own rooms, never the CLI ([Moderation](#moderation)) |
| What a member sees and can do — to answer a member's question | [For members](#for-members) |

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
wayai communities access add <community> <email>... [--tier <name>]... [--org <org_id>] [--json]
wayai communities access set-tiers <community> <email> <tier>... [--org <org_id>] [--json]
wayai communities access set-tiers <community> <email> --none [--org <org_id>] [--json]
wayai communities access remove <community> <email> [--org <org_id>] [-y] [--json]
```

- **`<community>`** is a community id (from `list`), or the community's exact name within the
  organization — case and spacing included. Two communities sharing a name are refused with both
  ids; name one by its id.
- **The organization** is `--org`, else the repo's (`wayai-ws/wayai.yaml`, set by `wayai init`).
  `list` and a name need one; an id does not.
- **`--json`** prints the response's `data` as is: the list of communities, the join link
  (`enabled`, `url`), the stats, or what an access-list add did (`added`, `already_listed`, and
  `linked` for an add with tiers: [Entry tiers](#entry-tiers)). `access list --json` prints every
  entry (`access_id`, `email`, `created_at`, `tier_ids`); `access set-tiers --json` prints the
  entry as it now stands, with `linked`; `access remove --json` prints the entry it removed.

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

- `access list` prints the emails on the list, oldest first, and each one's tiers when any entry
  has some.
- `access add` adds emails: at most 500 at a time, and a list holds at most 10,000. An email
  already on the list is left as it is. Each email is checked first, and the command refuses,
  naming every value that is not an email, before it sends anything. `--tier` gives each entry
  tiers too ([Entry tiers](#entry-tiers)).
- `access remove` takes one email off the list, and **removes from the community anyone that
  email let in**, whatever the admission; a member the entry only linked loses its tiers and stays
  ([Entry tiers](#entry-tiers)). It prints that, on stderr, and asks first; `-y` skips the
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

### Tiers

A **tier** is a named label of one community — "Gold", "Cohort 2026" — that the organization's
owners and admins define and gate rooms and courses to. A community holds at most 20 tiers; they
have no order, and a name is unique once trimmed, case ignored. WayAI takes no payment: a tier is a
label an admin or an agent gives, whatever the reason.

- **A gate** is the tiers a room (any kind) or a whole course is gated to. A member reaches a gated
  room or course only while they hold one of its tiers. A room or course with no tiers is open to
  every member, so nothing changes until something is gated. To admit Gold and Platinum to one
  room, gate it to both: tiers have no rank.
- **Who reaches a gated item anyway:** the organization's owners and admins, and the community's
  moderators.
- **What the others see.** A gated room or course they cannot reach is left out of what they see.
  It is shown to them as locked, with its tiers' names, only when one of those tiers has a
  `get_access_url` (an `https:` page of yours, at most 2,048 characters, where people get the
  tier): the web links to that page; the mobile app shows the lock and the tiers' names, never the
  link. A room or course gated only by tiers with no URL — a staff room, say — stays invisible to
  those who cannot reach it. A lesson's discussion room is a room with its own gate.
- **Define tiers and gates** in `community.yaml` and the course files ([Communities as
  code](#communities-as-code)) or in the web console. A tier is deleted only in the web console,
  and only while no room or course — archived and unpublished ones included — is gated to it, so a
  deletion never opens a gated item to everyone.

A member's tiers come from two places, each taken away where it was given:

- **Entry tiers** — the tiers on the access-list entry linked to the membership ([Entry
  tiers](#entry-tiers)). They reach every member linked to that entry, at once.
- **Direct tiers** — tiers an owner or admin gives one member, in the web console's member list.
  They end when the person stops being a member; a member who joins again starts from their entry's
  tiers.

So an agent taking a lapsed subscriber's entry tier away never takes away a tier an admin gave by
hand. No member sees another member's tiers. Giving a tier needs no acceptance from the member, and
no tier change is written to the moderation log.

### Entry tiers

An access-list entry can give tiers, in a `link` community as in an invite-only one:

- `access add <community> <email>... --tier <name>` adds the emails, each entry also giving those
  tiers beside any it already gave. Repeat `--tier` for more than one.
- `access set-tiers <community> <email> <tier>...` replaces the tiers of that email's entry;
  `--none` takes them all away. The members linked to it keep their membership.

Tiers are named by name, case ignored, among the community's tiers: the commands read them first
(one request) and refuse a name the community does not have, listing the ones it does, before
anything is written.

**Linking an entry to a member.** An entry's tiers reach a member once the entry is linked to their
membership. Emails match exactly, once trimmed and lowercased, and only a **verified** email links:

- **At join.** An invite-only join links the entry that let the person in. A `link` join also links
  the entry listing their verified email while any entry has tiers; someone no entry lists still
  joins.
- **At an entry write.** An add with `--tier` of at most 25 emails, and every `set-tiers` that
  leaves the entry with tiers, links at once each member already in the community whose verified
  email the entry lists and who has no entry linked. The command prints how many it linked. When
  the sign-in provider cannot be asked just then it says so: the entry and its tiers are saved, and
  running the same command again retries the linking.
- **By claim.** An add of more than 25 emails links no one at once. The web and the mobile app ask,
  when a member opens the community and when they open a locked room or course, whether an entry
  lists their verified email, and link it then. Access is tied to the **verified** email: a member
  whose account shows another address gets nothing from an entry listing that one.

**Taking an email off the list** (`access remove`) removes the members its entry let in, and only
unlinks the others: they lose its tiers and stay. To take tiers away without removing anyone, edit
the entry's tiers (`access set-tiers`).

### Tiers from a webhook

Every `wayai communities` command and the web console need a person signed in. To keep tiers in
step with a subscription a payment provider reports, without a person in the loop, an agent gives
and takes away tiers with the native **`community_tiers`** tool ([native
tools](agents/native-tools.md#community_tiers)):

1. **A production task hub** with a **webhook automation** ([automations.md → Webhook
   Triggers](automations.md#webhook-triggers)), `target: { type: none }`, whose agent is a pilot
   (or pilot specialist) holding `community_tiers` under `tools.native` in its YAML. Publish it,
   rotate the automation's secret on the production hub, and point the payment provider's webhook
   at its URL. A `schedule` automation with target `none` may call the tool too.
2. **The automation's instructions** say what to do with the delivery: for example "This delivery
   is a subscription event. When a subscription starts or renews, give the tier Gold of community
   `<community_id>` to the subscriber's email with `add_tiers`; when it ends, take it away with
   `remove_tiers`."
3. **The community lets that hub manage those tiers**: in `community.yaml`, `settings.tier_managers`
   lists the production hub's id and the tier keys its agents may give and take away (at most 5
   hubs), then `wayai push`; or the web console.

What the tool does, and what bounds it:

- It adds or removes the named tiers on the access-list entry for an email (`add_tiers` lists the
  email first when the list does not hold it), and links the members whose verified email that is,
  as an entry write does. Both actions are safe to repeat.
- It runs only in a production task an automation of trigger `webhook` or `schedule` and target
  `none` opened, holding no team message and no consult thread, never in an eval run. **No person's
  message reaches the tool's turn.**
- **Text inside the webhook body or another tool's result can still steer which email the model
  picks, bounded by the hub's listed tiers, and the tool never revokes.** It changes only the tiers
  `tier_managers` lists for its hub, only on entries; it never takes an email off the list, so no
  agent removes a member.
- Listing a hub in `tier_managers` hands that power to its agents and so to whoever configures them
  (its admins, through preview and publish), and in an invite-only community the power to let people
  in by adding their email to the list. Push's listing says so before it asks.

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
communities axis beside hubs, bases and case spaces, seeded by the first pull or push.

Pull and push run for the community's organization only: the workspace must be bound to the
organization that owns it (`wayai-ws/wayai.yaml`). Push never creates a community — WayAI does.

### `community.yaml`

```yaml
community_id: 6b1e…            # read-only
name: Acme Builders            # the name members see; edit it to rename the community
name_rev: v1:e7bbfd24a6f3      # the name's own rev
settings:                      # settings and rules.md share one rev
  rev: v1:4d1e9a07c2b3
  slow_mode_seconds: 30        # 0 is off
  courses_label: Academy       # '' for the default
  admission: invite            # link (the default): anyone with the link; invite: owners, admins and the access list
  tier_managers:               # the production hubs whose agents may give and take away these tiers; omit for none
    - hub_id: 4b0b…
      tiers: [gold]
join_link_enabled: true        # the link itself is never written to disk
join_link_rev: v1:77ac01e5f9d2
tiers:                         # omit for none
  - key: gold                  # stable local name; a rename keeps it
    id: 7e17…
    rev: v1:2c9d0e4b7a15
    name: Gold
    get_access_url: https://example.com/gold   # where people get it; omit for none
  - key: cohort-2026           # no id: push creates it
    name: Cohort 2026
rooms:
  - key: general               # stable local name; a rename keeps it
    id: 1c4a…
    rev: v1:0b93c4d2e1a8
    name: General
    kind: discussion           # discussion, forum or announcements; read-only once created
  - key: q-and-a               # no id: push creates it
    name: Q&A
    kind: forum
    tiers: [gold, cohort-2026] # tier keys: members holding one of them; omit for every member
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
- **Tiers** ([Tiers](#tiers)) are named by their `key` everywhere in the files: a room's and a
  course's `tiers:`, and a tier manager's. Pull picks a tier's key from its name, as for a room;
  keep it through renames. A new tier entry (no `id`) named as a tier the server holds and the files
  do not list is taken for that tier: push creates no second one. `tiers:`,
  `tier_managers:` and a room's or course's `tiers:` are written only when not empty, so a
  community with no tiers keeps the files and revs it had.
- **`tier_managers`** lists each production hub of the organization, by id, with the tiers its
  agents may give and take away ([Tiers from a webhook](#tiers-from-a-webhook)); each names at least
  one tier. The list is replaced whole on push.

### Courses

`courses/<course>/course.yaml`:

```yaml
id: 3f1c…
rev: v1:9f2c1a8b0e64           # title, description, published
title: Onboarding
description: Your first week   # '' for none
published: false               # members see a course once it is published
tiers: [gold]                  # tier keys in community.yaml: members holding one of them; omit for every member
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
video_url: https://youtu.be/…  # a YouTube video; omit for none
# video_file: videos/welcome.mp4  # or an uploaded video instead: see Uploaded lesson videos
links:                         # at most 10
  - title: Setup guide
    url: https://example.com/setup
requires_paid_plan: false      # members see the mark, except in the iOS app; nothing enforces it
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
- **A lesson's video** is a YouTube `video_url:` or an uploaded `video_file:`, never both
  ([Uploaded lesson videos](#uploaded-lesson-videos)).

### Uploaded lesson videos

A lesson can hold a video file uploaded to WayAI instead of a YouTube link — for an organization on
a paid plan that WayAI has enabled uploaded videos for.

- **Private to the lesson's readers.** Only people who can read the lesson can watch it, through
  links that expire: a copied link stops working, and nobody outside the community reaches it.
- **The storage limit.** The organization's uploaded videos count against its storage limit, in
  minutes of video: an upload past it is refused, and removing a lesson's uploaded video frees its
  minutes. A file is at most 30 GB, and a video's length is capped too — the refusal names the most.
- **Watching is metered.** The minutes members watch count toward the organization's operations,
  tallied per day. Past its operations quota, uploads stop and uploaded videos pause for members
  until it is back within it.
- **In the web console**, a lesson's **Video** offers **YouTube link** or **Upload**. Upload takes a
  file, shows its progress (a stopped upload can resume), then processes the video, which can take a
  few minutes — the page can be left meanwhile. **Remove video** deletes it; saving a YouTube link
  removes it too.
- **As code**, `video_file:` names the file: a path from the lesson file's folder
  (`videos/welcome.mp4`, `../../../videos/welcome.mp4`) that stays inside the community's folder and
  is never reached through a symlink.
  - **Push uploads it** after the lessons' other changes, only when its content (its SHA-256)
    differs from the lesson's video: pushing again uploads nothing, nor does the same file under
    another name. Push lists each upload with its size, then waits until the video is ready, up to an
    hour; a push stopped while the video is processed leaves it processing, and the next push waits
    for it instead of sending the file again. A new lesson is created first, then gets its video.
  - **Pull writes `video_file:`** as the name the video was uploaded under, and never downloads it:
    its bytes stay on WayAI. A `video_file:` whose file is not in the folder stands for the lesson's
    uploaded video while it names that video's file, so a clone without the video files still
    pushes; when the video is replaced (by another upload or a YouTube link) or removed in the web
    console and nothing else of the lesson changed there, pull takes it like any change made on the
    server only (with another field
    changed too, it is a conflict `--theirs` resolves). One naming anything else is refused before
    anything is written, and stays as written through web edits of the lesson's other fields; a web
    change to the video itself replaces it with the server's video, and pull or push says so. When the folder holds
    other bytes under that name, pull says so: the next push would upload them.
  - **Removing it.** Taking `video_file:` out, with no `video_url:`, removes the uploaded video, and
    a `video_url:` in its place replaces it; a `video_file:` on a lesson with a YouTube video
    replaces that once the upload is ready.
  - Keep large video files out of git — a `.gitignore` entry such as `*.mp4`, or Git LFS.

### How pull and push decide

For the name, the settings (with `rules.md` and `tier_managers`), the join link, each tier, room, course, module and lesson, and the
order of each list — the courses, a course's modules, a module's lessons — the files, the server and
the `rev` are compared:

| Since the last sync | push | pull |
|---|---|---|
| Changed in the files only | applies it | keeps it (not pushed yet) |
| Changed on the server only | leaves it, and says to pull | takes it |
| Changed alike on both sides | nothing to apply | nothing to take |
| Changed differently on both sides — a conflict | refuses the whole push | leaves the files' side, says so and exits non-zero; `--theirs` takes the server's, and an edit of yours made again on top of it then pushes |

So a push does not undo an edit made in the web console before it reads the community, and a pull
never overwrites an edit you have not pushed — save a `name` edited in a folder with no `name_rev`
(below), which pull replaces with the server's. When push asks before applying, it reads the community
again after your answer and stops, writing nothing, if anything changed while it waited; with `-y` it
applies right after its read. A tier, room, course, module or lesson on the server that the files do not
list is left as is by push, and added to the files by pull. A tier deleted in the web console stops
a push until pulled; pull takes it out of `tiers:` and out of every gate and tier manager in the
files — unless the files changed it since, a conflict `--theirs` resolves.

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
  `archived` to bring it back. Likewise a tier, course, module or lesson left out of the files stays;
  `published: false` takes a course or a lesson off the community.
- **Limits, checked before anything is written.** A community holds at most 20 rooms, archived ones
  included, 20 courses and 20 tiers; a course at most 20 modules and 100 lessons; a community at most 500
  lessons in all — drafts included; `tier_managers` at most 5 hubs. Push refuses a change past one before writing anything.
- **Tiers first.** Push creates and changes tiers before anything that names them. A new room is
  created already gated; a new course is created as a draft and gated before it is published, so a
  gated item is never open to every member on the way. Taking a room's or course's `tiers:` away
  opens it to every member.
- **`tier_managers`** hands its hubs' agents, and whoever configures them, the power to give and
  take away those tiers on the access list — and in an invite-only community to let people in.
  Push's listing says so for each hub it adds or widens, before it asks. The hub must be a
  production hub of the organization; any other is refused.
- **Publishing comes last.** Push takes courses, then lessons, off the community first, and publishes
  a course last, once its modules and lessons are in place: a new course is created as a draft, and a
  new course with `published: true` goes live at the end of the push.
- **Read-only fields** — `community_id` and a created room's `kind` — refuse the push when
  changed in the files.
- **`name` renames the community** for every member at once (1–100 characters). A file pulled
  before renames could be pushed has no `name_rev`, and its `name` is read as the last one synced:
  pull once before renaming in such a folder: otherwise push reads your rename as the server's
  change, leaves it unpushed and says to pull, and that pull replaces it with the server's name.
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
  push reads six times plus once per course, and a push that asks reads them again after your
  answer. A push then writes each entry it creates or changes, each course, module or lesson it
  moves, and each course it publishes, and states those requests and the time they take before it
  asks. A first import of a community at the limits — 20 published courses of 20 modules, 500
  lessons — makes 940 writes after its reads: about 19 minutes.

## The web console

The organization's communities are listed at
`https://app.wayai.pro/settings/organizations/<org_id>/communities`, the organization's
**Communities** tab, shown once it owns one. Each community's settings open at
`https://app.wayai.pro/settings/organizations/<org_id>/communities/<community_id>/<section>`; inside
the community, an owner's or admin's header carries **Community settings**, which opens them.

| Section | `<section>` | Holds |
|---------|-------------|-------|
| Rooms | `rooms` | **New room**, **Rename**, **Archive**, **Bring back** and **Tiers** (the room's gate). The kinds read **Chat**, **Forum** and **Announcements** (`discussion`, `forum`, `announcements` in `community.yaml`) |
| General | `rules` | The name, the rules (shown on the join page), slow mode |
| Join link | `link` | The link, its rotation and turning it off, the source tags |
| Access | `access` | `admission` and the access list, each entry with its tiers: given to the emails an add lists, changed per entry ([Entry tiers](#entry-tiers)) |
| Tiers | `tiers` | The tiers, each with **Edit tier** (its name and access link) and **Delete tier** (refused while it gates a room or course); **New tier**; and **Tier managers**, the hubs whose agents may give them, with a warning of what that lets those agents do ([Tiers](#tiers)) |
| Members | `members` | The current members, by the name members see, each with their role, when they joined and their tiers, where an owner or admin gives and takes away a member's direct tiers ([Tiers](#tiers)); no emails |
| Moderators | `moderators` | The moderators, each with **Dismiss** ([Appointing moderators](#appointing-moderators)) |
| Reports | `reports` | The reports inbox ([Reports](#reports)) |
| Bans | `bans` | The banned people, each with **Lift ban** |
| Log | `log` | The moderation log ([The moderation log](#the-moderation-log)) |
| Stats | `stats` | What `wayai communities stats` prints |
| Courses | `courses` | The courses and the tab's label, each course's editor holding its tiers; one course at `…/courses/<course_id>`, one lesson at `…/courses/<course_id>/lessons/<lesson_id>` |

While WayAI keeps communities closed to members, the settings leave out Reports, Bans and Log (a
link to one opens Rooms, as a link naming no section does) and the community's own pages send people to Chat. The other
sections keep working, so rooms and courses can be ready before members arrive.

## Moderation

### Who does what

A community has three ranks: the organization's **owners and admins** — the community's admins,
whether or not they joined it (a hub's admins and team are not) — its **moderators**, and its
**members**.

| Action | Owners and admins | Moderators | Members |
|--------|-------------------|------------|---------|
| Delete someone else's message or post | Yes | Yes | No |
| Remove or ban a member; lift a ban | Yes | Yes | No |
| Pin a message or post (at most 3 per room; never a reply) | Yes | Yes | No |
| Lock a forum post: it takes no more replies | Yes | Yes | No |
| Mark a forum reply as the answer | Yes | Yes | On their own post |
| Read and resolve reports; read the moderation log | Yes | Yes | No |
| Post in an announcements room | Once joined | Yes | No |
| Settings, rooms, the join link, access, courses; appoint and dismiss moderators | Yes | No | No |

- **Moderation acts only on a lower rank**, and never on oneself: a moderator never deletes the
  messages of, removes or bans another moderator or an owner or admin ("You can't do that to an
  admin or another moderator.").
- **Moderators, owners and admins skip slow mode.**
- **An owner or admin who has not joined** reads every room and moderates, but posts nothing until
  they join ("Join the community to post.").

### Appointing moderators

- **Make moderator** is in the menu of a member's message in a room: an owner or admin picks the
  person from what they wrote; the Members section lists who is in the community but offers no
  actions. Only a current member can be made one.
- **Dismiss** in the Moderators section makes them a plain member again. There is no limit on the
  number of moderators.
- A moderator who leaves or is removed comes back as a plain member if they join again.

### Where moderators work

- **In the rooms:** a message's menu (web) or a long press (mobile) deletes it, pins it, locks a
  post, and removes or bans its author.
- **The moderation pages:** `https://app.wayai.pro/community/<community_id>/moderation/<section>`
  (`reports`, the default, `bans`, `log`), opened by **Moderation** in the community's header. They
  are web only: the mobile app has the actions in the rooms, not the reports, bans or log.

### Remove and ban

| | Remove from community | Ban from community |
|---|---|---|
| Who it applies to | A current member | Anyone who ever joined, one who left included |
| The person | Leaves the community now | Leaves the community now |
| Their messages | Stay | Stay, unless **Also delete every message they wrote here** is checked: each of their messages, posts and replies, in every room, is deleted: it shows as "This message was deleted.", except in an announcements room, which leaves deleted messages out |
| Joining again | With the link, as `admission` allows | Refused — any link reads to them as not working — until the ban is lifted |

**Lift ban** does not bring them back: they may join again with the link (in an invite-only community,
once their email is on the access list). Taking an email off the access list also removes the people it let in
([Invite-only](#invite-only-the-access-list)), and takes its tiers from the members it only linked.

### Reports

- **Members report** a message, a post or a person from its menu (a long press on mobile), with a
  reason — spam or scam, harassment or bullying, hate speech, sexual content, violence or threats,
  something else — and an optional note of up to 500 characters; each person sends at most 20 a day.
- **Every report reaches WayAI's own team.** The community's inbox gets a copy: the message as the
  reporter saw it (its text, title, and an image's name, never the image), the reason and the note —
  **never who sent it**. A report about a moderator, an owner or an admin reaches WayAI's team only,
  and the inbox stops showing the reports about someone once they become a moderator.
- **The inbox** shows the pending reports; **All** (`?filter=all`) adds the resolved ones. It keeps
  the latest 10,000.
- **Mark resolved** changes nothing else: deleting the message and banning its author are separate
  actions, offered on the report. Resolving is not logged.

### The moderation log

It records who did what, and when: deleting a message, removing a member, a ban (with how many
messages it deleted), lifting a ban, locking and unlocking a post, pinning and unpinning, and
appointing and dismissing a moderator. It does not record resolving a report, marking an answer, a
removal by the access list, a change to the settings or rooms, or WayAI's own removals. Moderators,
owners and admins read it; it keeps the latest 10,000 entries.

## For members

What a member sees, on the web and in the mobile app's **Community** tab; members never need the
CLI. Each link: [navigation.md → Community](navigation.md#community).

### Joining

- **The page** at the join link (`wayai communities join-link get`) shows the community before
  sign-in: its name, its organization, how many members it has, its rules, and **Invite-only** on an
  invite-only one. Joining needs a WayAI account: **Create account** and **I already have an
  account** both come back to the page. On a phone, it offers to open the mobile app.
- **The rules show only there**, and joining accepts them: there is no separate step.
- **Joining a first community** asks how other members will see the person (**Change**). They then land
  in the announcements room, or in the first room when there is none.
- **Refusals:**

| What the person sees | Why |
|---|---|
| This link doesn't work (on the page), or This link doesn't work anymore. Ask for a new one. (on **Join community**) | The link is wrong, was replaced or is turned off — or the person is banned |
| You're not on this community's invite list | Invite-only, and the person's verified email (named there) is not on the access list: add it (`wayai communities access add`), or have them sign in with the email that is. An unverified email must be verified first |
| This community is full. | It has 10,000 members |
| You're in as many communities as you can join. | The person is in 50 communities |

- **No button leaves a community** today, on the web or in the mobile app.

### Rooms

| Kind | Who writes |
|------|-----------|
| **Chat** (`discussion`) | Every member |
| **Forum** (`forum`) | Every member: posts with a title (up to 200 characters), and replies one level deep |
| **Announcements** (`announcements`) | Moderators, owners and admins; members read ("Only the community's admins and moderators post announcements.") |

- **A message** holds up to 4,000 characters and at most one image (PNG, JPEG, GIF or WebP, up to
  7 MB), which counts against the person's 1 GB of storage for images and files.
- **A member edits** their own message ("(edited)") and **deletes** it ("This message was deleted."; an announcements room leaves a deleted message out)
  at any time; deleting a post hides its replies.
- **A forum** searches titles and filters **All** / **Unanswered**; its posts are in order of latest
  activity, marked **New activity** since the person's last visit. The post's author, a moderator, an
  owner or an admin marks one reply **Answer**, shown first. A **Locked** post takes no more replies.
- **Pinned** messages show at the top of a room.
- **A room gated to tiers** shows only to the members holding one of them ([Tiers](#tiers)).
- **An archived room** is listed under **Archived** and reads "This room is archived. You can still
  read it."
- **Unread counts** show on each room; the community's badge leaves out archived rooms.

| What the person sees | Why |
|---|---|
| Slow mode is on in this room. Wait a moment before sending again. | The community's slow mode (`slow_mode_seconds`, one setting for every room) has not passed since their last message in this room. No countdown is shown |
| You've reached today's message limit in this community. | 500 messages a UTC day across its rooms, or 20 in the first 24 hours after joining (or joining again); deleting one gives none back |
| The community is busy. Try again in a moment. | More than 5 messages a second across the community |
| Daily limit of 3 announcements reached. … | 3 announcements a UTC day across the community, deleted ones included |
| This community can't take more messages right now. | The community holds 500,000 messages |

### Courses

- **The courses tab**, beside **Rooms**, shows once a course is published, under the
  `courses_label` (**Courses** when none is set). A draft course or lesson never shows, and a module
  shows once it holds a published lesson. A course gated to tiers shows only to the members holding
  one of them ([Tiers](#tiers)).
- **A lesson** shows its video — a YouTube video (the mobile app opens it in YouTube or the browser),
  or an uploaded one, played in place on the web and in the mobile app — its text, its links,
  **Discuss in <room>** when it has a room, and "This lesson needs a paid plan." when
  `requires_paid_plan` is set — a mark only, nothing is withheld, and the iOS app does not show it.
- **An uploaded video that cannot play now** reads "This lesson's video is unavailable for now." while
  the organization is past its operations quota, and "This lesson's video isn't available." once it
  no longer hosts uploaded videos ([Uploaded lesson videos](#uploaded-lesson-videos)).
- **Mark complete** is the only way a lesson counts as done — watching the video does not — and it
  cannot be undone. A course shows "<n> of <total> lessons complete" and **Next lesson**. The
  progress outlasts a removal or a ban and comes back when the person joins again.

### Notifications

- **The mobile app notifies** of an announcement, and of a reply to the member's own forum post
  (once, then not again until they open the post). Chat messages and forum posts never notify.
- **Email:** a member not signed in to an up-to-date mobile app gets each announcement by email
  instead ("New announcement in <community>"). **Stop these emails** in it stops that community's
  emails for good: nothing turns them back on.
- **Muting** — the bell in the community's header on the web (**Mute notifications from this
  community**), the same switch on the community's home in the mobile app — stops its notifications
  and emails, never its unread counts.

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
| … is not on the access list of … | Check the email with `wayai communities access list`; `access add --tier` lists it with tiers |
| … has no tier named … | The message lists the community's tiers: use one of their names, or add the tier first (`tiers:` in `community.yaml`, then push; or the web console). Nothing was sent |
| tier …: … is no longer a tier of this community (it was deleted) | It was deleted in the web console: pull, which takes it out of the files and their gates, then push |
| … is not a tier key in community.yaml | A room's, course's or tier manager's `tiers:` names a key `tiers:` does not list: add the tier, or correct the key |
| … names tier …, which community.yaml's tiers do not list | Pull the community: pull adds the tiers the files do not list |
| WayAI has suspended this account | A suspended account changes neither the access list nor the admission |
| … is not one of the communities of organization … | Pull and push a community from a workspace bound to the organization that owns it |
| Conflicts — changed in your files and on the server | `wayai pull communities/<folder> --theirs` takes the server's side; make again any edit of yours to keep, then push |
| Push stopped at … | Fix what the message names and push again: what was applied before it stays applied |
| Push stopped at lesson …: … change the lesson's text in … | The server refused the lesson's text (nested too deeply, indented too far, or too large once formatted): change it in that file and push again |
| … is one of course …'s on the server, and a … does not move between courses | Put the module or lesson back in its course. To have it in another course, make a new one there (a new entry or file, without an id) |
| … id … is also …'s — a copy keeps its original's id | Remove the `id` and its revs from the copy: push then creates it as a new one |
| … holds at most … | Remove creates from the files: nothing is deleted, and a draft still counts |
| Skipped: … — no module of the course lists it | A draft in progress: list the lesson in a module's `lessons:` to sync it |
| … video_file … is not in the files …, and is not the lesson's uploaded video | Put the file at that path, or set `video_file:` to the name pull wrote. Nothing was pushed |
| a lesson holds a video_url or a video_file, not both | Keep one of them |
| … video_file must stay inside the community's folder | Move the video into the community's folder and point `video_file:` at it |
| Push stopped at lesson … (video …): this organization can't upload lesson videos | Uploaded videos are for organizations on a paid plan that WayAI has enabled them for; a YouTube `video_url:` works meanwhile |
| Push stopped at lesson … (video …): … already fill its storage | Remove another lesson's uploaded video (take its `video_file:` out and push, or **Remove video** in the web console), then push again |
| Push stopped at lesson … (video …): … past its operations quota | Uploads stop until the organization is back within its operations quota |
| Push stopped at lesson … (video …): … isn't a video that can be played | Export the video again (MP4 plays everywhere) and push |

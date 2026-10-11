# Communities

A community belongs to an organization: its members, rooms and courses, joined by its link. WayAI
creates communities; once one exists, the organization's **owners and admins** manage it.

| Task | Where |
|------|-------|
| List the organization's communities; read, rotate or turn off a join link; read the stats | CLI (`wayai communities`) or the web console |
| Settings, rules, rooms, courses, moderators and moderation | The web console: organization settings → Communities (`/settings/organizations/<org_id>/communities`) |

## Sign-in

`wayai communities` needs a **browser sign-in**: run `wayai login`. An API token
(`wayai login --token`, or `WAYAI_TOKEN` in the shell) is refused, so these commands do not run
from CI. When `WAYAI_TOKEN` is set, it is sent instead of the browser sign-in: unset it
before running these commands.

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

## When a command fails

| Message | What to do |
|---------|-----------|
| Communities are managed from a browser sign-in only | Run `wayai login` (without `--token`); unset `WAYAI_TOKEN` if the message says it is set |
| … not found, or you are not an owner or admin of … | Check the id or name with `wayai communities list`, and that you are an owner or admin of the organization |
| Too many requests | Every community request — the web console's and your messages included — shares one per-person budget of 60 a minute. Wait the time the message names |
| No organization: pass --org | Pass `--org <org_id>`, or run inside a workspace set up with `wayai init` |

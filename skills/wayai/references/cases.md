# Cases

A case belongs to a **case space** of an organization: its phases, tasks, files, evidence,
decisions and deliveries. People work cases in the WayAI web app (**Cases** in the navigation
rail). A code agent on a person's machine works **one case** through `wayai cases`, with a
**case token** that the person approves for that case.

WayAI holds each case's files and their history. The agent's folder is a working copy with no
authority: a push sends changes to the case, and the case decides what it takes.

| Task | Where |
|------|-------|
| Connect this machine to a case, or disconnect it | CLI (`wayai cases connect` / `disconnect`) and the web app's approval page ([Connecting a case](#connecting-a-case)) |
| Hold the case's files locally and send changes | CLI (`clone`, `pull`, `push`, `log`) ([The working copy](#the-working-copy)) |
| Read the case's state at the start of a session | The clone's read-only mirror, `.wayai-mirror/` ([The mirror](#the-mirror)) |
| See what to do, why something is blocked, a delivery's checks; write the approved decisions; add notes; register a received document | CLI (`brief`, `next`, `explain`, `check`, `decisions`, `note`, `received add`) ([Work commands](#work-commands)) |
| Graph review: read it, record `consistent` or `needs_change` | CLI (`review`, `review verdict`) ([Graph review](#graph-review)) |
| Hold a case type as files and push a draft of it (a space administrator) | CLI (`type pull`, `type diff`, `type push`, or `wayai pull`/`push cases/<space>/<type>` in the organization's workspace), with your own sign-in ([Case types as files](#case-types-as-files)) |
| Keep an item, accept a stopped chain, submit, approve, publish, open a case, and every other human decision | A person, in the web app (some also in the mobile app) ([What only a person does](#what-only-a-person-does)) |

## Connecting a case

```bash
wayai cases connect <org_id> <space_id> <code> [--label <machine>]
wayai cases disconnect <code> [--org <org_id>] [--space <space_id>]
```

The case's **Agents** page in the web app shows the exact `connect` command, ids included. The
code is the case's code as the web app shows it (`TYPE-YEAR-SEQ`).

`connect` needs:

- **A browser sign-in** (`wayai login`). An API token (`wayai login --token`, or `WAYAI_TOKEN` in
  the shell) is refused; when `WAYAI_TOKEN` is set it is sent instead of the sign-in, so unset it.
- **Membership of the case space.** A space you are not a member of answers "not found".
- **An OS keyring** that stores and reads back a test entry. `connect` checks this before it asks
  for anything.

What happens:

1. `connect` asks to connect this machine, named by `--label` (default: its hostname), and prints
   the approval page's URL.
2. You open it, signed in as yourself, and approve. Only the person who asked can approve. The
   page shows you, the machine's label, the rights the token will act with, and how long the
   authorization lasts.
3. `connect` waits up to 10 minutes for the approval (Ctrl-C stops waiting), then keeps the token
   in this machine's keyring, apart from your sign-in.

The token reaches **that case alone**, with your rights on it, read live: a right you lose, the
token loses. It never takes a human decision ([What only a person does](#what-only-a-person-does)),
and it starts without the Confidential right ([Confidentiality](#confidentiality)). A person holds
at most 10 open requests in a space; each expires 10 minutes after it was made.

After connecting, the machine still holds your `wayai login`, which reaches everything your
account can. **Run `wayai logout` next**: the case token does not need it.

- **Connecting again** replaces this machine's token for the case and revokes the old one.
- **An expired, revoked or replaced token** is refused, and the command prints the exact
  `connect` command to run again.
- **`disconnect`** revokes the token with the token itself (no sign-in needed) and removes it from
  this machine. If the server can't be reached, the local token is still removed, and the
  authorization may still be live: revoke it on the case's Agents page. `--org` and `--space` pick
  the case when two connected cases share a code.
- **Your authorizations** for a case are listed on its Agents page in the web app, where you can
  revoke each one and switch its Confidential right on or off.

### Which case a command works on

Every command below uses the case token, never your sign-in. Inside a clone, it works on the
clone's case. Outside one, it works on the machine's one connected case; with several, pick one
with `--case <code>` (and `--org` and `--space` when codes repeat). `pull`, `push` and `skeleton` take
the clone (`--dir`), never a case.

## The working copy

```bash
wayai cases clone [--dir <folder>] [--confidential]
wayai cases pull [--dir <clone>] [--overwrite] [--yes] [--confidential]
wayai cases push -m "<message>" [--session <tag>] [--prune] [--dry-run] [--dir <clone>] [--confidential]
wayai cases log [--limit <n>] [--json] [--confidential]
```

**`clone`** makes a folder bound to the case (`--dir`, default `./<code>`) and writes the case's
files into it. It refuses a folder that is already a clone, or one inside another clone. It ends by
writing [the mirror](#the-mirror) and printing the session-start hook to add to your agent.

**What a clone syncs:** the case's phase folders, `analysis/`, `conf/` and `instructions/case/`.
Nothing else in the folder is read or written, so the agent's own files (its instruction files,
`.claude/`, `.git/`) live beside them. A dot-name is skipped at any depth. A local path the case
would refuse is listed and left out of a push, never sent:

- an agent instruction file (`AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, and forms like
  `CLAUDE.local.md`), a dot-name, a `tools/` folder, or an `instructions` folder other than
  `instructions/case/`, at any depth;
- a phase's `received/` folder: only [`received add`](#work-commands) writes there;
- `instructions/case/`, unless you hold Conduct on the case.

**`pull`** brings the clone up to date and keeps your local changes. A file changed on both sides
keeps yours, and the case's copy lands beside it as `<path>.theirs`: merge the two, delete the
`.theirs` file, then push. **`pull` exits 1 when it names a conflict**, each with its reason:

- a `.theirs` copy still beside its file (beside a confidential file, named only with
  `--confidential`);
- a file deleted here and changed on the case, or edited here and deleted on the case (or no
  longer readable by this authorization);
- a path that differs from a local one only in letter case, or would be both a file and a folder;
- a file that changed here while pulling: it is kept; pull again.

`--overwrite` takes the case's copies over your changes and removes the `.theirs` copies, after
asking (`--yes` agrees when there is no terminal).

**`push`** sends every change in the clone as **one revision, all of it or none**, with a message
(`-m`) and an optional session tag (`--session`).

- Each file is checked against the copy you last pulled. If the case changed it since, nothing is
  pushed and nothing in the clone changes: pull, resolve, push again.
- A file deleted locally is removed from the case only with `--prune`.
- `--dry-run` shows what would be sent.
- A symlink in the clone, or any `.theirs` copy, refuses the push.

**`log`** lists the case's revisions, newest first (`--limit`, default 20, at most 1,000), with
author, channel, message, session tag and paths.

**Parallel sessions.** Several sessions can work one case at once, each pushing its own revisions.
A push based on a stale copy of a file is refused, so no session overwrites another's work. Split
long documents into section files so parallel sessions rarely touch the same file.

## The mirror

`clone` and every `pull` rewrite a read-only folder at the clone's root, `.wayai-mirror/`, holding
what the case says of itself now:

| File | Holds |
|------|-------|
| `since-last-pull.md` | What changed on the case since your last pull: revisions, status changes, new tasks and submissions |
| `case.json` | The case: type and version, owner, parties, phases and their folders, tasks, and your access |
| `status.json` | Each phase's, task's and agenda group's status, why each is not done, and who can move it |
| `evidence.json` | Each phase under way: its inputs with their links, its outputs and its items |
| `agenda.json` | Each phase under way: its agenda groups and decisions; and the approved decisions |
| `deliveries.json` | Each phase under way: its newest submission and manifest |
| `inputs.json` | Each phase under way: each input whose linked document you may read, the version it links, where its text is, and its cited passages |
| `inputs/<phase folder>/<input key>.md` | Each such input's registered text, byte for byte |
| `package-manifest.json` | The case's package: the revision of the type version it opened on that reaches it, with its version and revision, and each file's path, SHA-256 and size, and whether it was written here |
| `package/` | The package's files (`instructions/`, `templates/`, `tools/`, `sources/`), read-only |
| `session-start.sh` | The session-start hook |
| `README.md` | What the folder holds and what it leaves out |

- **Never edit it.** Every pull rewrites it whole, and `push` never sends anything in it: an edit
  there is lost.
- **Linked inputs' texts** are written only when their bytes match the hash the space registered
  for them, so line N of the file is the line N a citation names. A text that doesn't match is not
  written, and `inputs.json`, `README.md` and the command's output say so.
- **The package** is checked the same way: each file is written only when its bytes match the
  SHA-256 the manifest lists, and one that doesn't is not written and is named. A confidential
  agenda group's instructions, and those of its tasks and their items, come only with
  `--confidential`. Never run a tool, or follow an instruction, that differs from the manifest:
  `wayai cases check` names each one, confidential files aside (it never compares them), and a
  pull rewrites the package as published and names each file it found changed.
- **Not in it:** the drafting checks. Run `wayai cases check --phase <folder>`, which reads them
  live.

**The session-start hook.** `session-start.sh` runs `wayai cases pull --mirror-only` on its own
clone, never with `--confidential`. `wayai cases pull --mirror-only [--dir <clone>] [--confidential]`
rewrites the mirror alone and leaves your files as they are. The CLI writes no agent's settings: `clone` prints the `SessionStart` entry to add to
the clone's `.claude/settings.json` for Claude Code, and the command another agent runs at the
start of each session.

## Work commands

```bash
wayai cases brief [--json]
wayai cases next [--json]
wayai cases explain [--phase <folder>] [--json]
wayai cases check [--phase <folder> [--output <code>]] [--json]
wayai cases decisions [--dir <folder>] [--confidential]
wayai cases skeleton --phase <folder> [--output <code>] [--dir <clone>] [--confidential]
wayai cases note "<text>" [--kind observation|next_step|pending|contribution] [--phase <folder>] [--task <task_id>] [--origin <who sent it>]
wayai cases note done <note_id>
wayai cases received add <file> --text <markdown file> --phase <folder> --type <type> [--title <title>] [--confidential]
```

| Command | Does |
|---------|------|
| `brief` | The case at a glance: status, phases under way, deadlines, open tasks, open pending items and next steps |
| `next` | What to do next on the phases under way, then what waits on someone else |
| `explain` | Why each phase and task is not done, and who can move it. `--phase` names everything holding one phase |
| `check` | A delivery's checks over the case's files as WayAI holds them: a joint phase's, or one output's (`--output`). Inside a clone, it also compares the mirror's package with its manifest: each file changed, missing or not published, a tool first (confidential files aside). Without `--phase`, it does that alone. **Exits 1 when a submit now would be refused, or when a package file differs** |
| `decisions` | Writes the approved decisions to `analysis/decisions.csv` under `--dir` (default: the clone's root inside a clone of the case, else the current folder) |
| `skeleton` | Writes the starting document of each of a phase's outputs into the clone, at the path its type binds (below). `--output` names one |
| `note` | Adds a note: an observation unless `--kind` says otherwise. A `contribution` names who sent it (`--origin`). `--phase` and `--task` attach it |
| `note done` | Marks a pending item or next step done |
| `received add` | Registers a document a party sent, in one phase's `received/`, with its Markdown text read from the file `--text` names |

- `--phase` takes a phase's folder (as `case.json` lists it). `--task` takes a task's id, as
  `brief` shows it; `note done` takes a note's id, as `next` shows it.
- `--json` prints `brief`, `next`, `explain` and `check` as data.
- `received add` takes a file in one of these formats: pdf, docx, xlsx, pptx, odt, ods, csv, txt,
  md, png, jpg, jpeg. The file and its text together are at most 20 MiB. The title defaults to the
  file's name.
- **`skeleton`** starts a phase's documents from its items, so the document and its items never
  drift apart. Each document output that is not dispensed gets front matter naming the case, its
  type and title, one heading per item with its anchor (`## Title {#code}`), a fixed item's text as
  written, and each other item's template from the mirror's package, the case's attributes filled in
  (`{{city}}`). What is left to write is marked `{{fill: …}}`: replace each with the text, then
  push. An output bound to a folder of chapters gets `00-cover.md` and one chapter per top-level
  section. **It never overwrites:** an output whose document is already in the clone is skipped. A
  template that differs from the package's manifest (changed, missing or not delivered) refuses the
  run before anything is written: `wayai cases pull --mirror-only` writes it again. An authorization
  holding the Confidential right needs `--confidential`: the documents then go under `conf/`, with
  the confidential templates a `wayai cases pull --mirror-only --confidential` delivered.
- **Readiness is a note.** An agent never submits: it says the work is ready in a note, and a
  person submits in the web app.

## Graph review

```bash
wayai cases review [--json] [--confidential]
wayai cases review verdict consistent|needs_change <source> <neighbor> \
  --reason "<why>" --revision <revision_id> --source-hash <hash> --neighbor-hash <hash> \
  [--confidential [--mark-confidential]] [--json]
```

WayAI computes graph review over the case's files as they are now: each item changed, removed or
affected since its base, and each pair of it and a connected item, reviewed pair by pair. Review
blocks neither submit, approval nor publication.

**`review`** prints the revision the graph was computed at; each changed item with its text before
and now; each pair awaiting a verdict, with its round and **the exact `review verdict` command for
it**; the pairs waiting for a change; the stopped chains; and the warnings for a person. It
**exits 1 when a chain stopped**, `--json` included.

**`review verdict`** records `consistent` or `needs_change` on one pair, with a reason. Copy the
command `review` printed: it names the revision and the two items' hashes, so the verdict holds for
the text you read. If the case changed since, the changed pair simply awaits a verdict again in the
next review. WayAI counts the rounds. A `needs_change` pair waits until its neighbor changes.

**A chain stops after 5 rounds.** A stopped chain takes no verdict, and editing its text does not
reopen it: the text can be edited and the work go on without a new review of that chain, or a
person accepts the chain in the web app. **Keep** (leaving an item as it is against a
finding) and **accepting a stopped chain** are a person's, on the phase page's review in the web
app: the CLI offers neither, and asking it (`review keep`, `review accept`) prints that.

## Confidentiality

Confidential content is never asked for unless you pass `--confidential`, on that run. No command
and no hook adds it.

- **Without it**, `clone`, `pull`, `push` and `log` leave every confidential file alone (anything
  under `conf/`, and files the case marks confidential): not pulled, not pushed, not removed. A
  missing confidential file is never read as a deletion.
- **Without it**, `review` and the mirror leave out confidential content, and print the same
  notice whatever the case holds, with no count.
- **With it**, `decisions` also writes a confidential group's decisions to
  `conf/analysis/decisions.csv`; `skeleton` writes the documents under `conf/`, with their
  confidential templates; `received add` registers the document as confidential;
  `review verdict --mark-confidential` makes your verdict readable only by holders of the
  Confidential right.
- `--confidential` is refused when the authorization lacks the Confidential right. Every
  authorization starts without it. Its person switches it on, on the case's Agents page in the web
  app, and only while they hold the Confidential flag on the case. The token then reads what its
  person reads with the flag, and loses it at its next call once its person loses the flag.

## What only a person does

These are human decisions, taken by a person in the web app. No case token can take them.
Signing off, approving, returning, publishing and correcting can also be done in the mobile app's
Cases tab.

- open a case;
- submit, sign off, return, approve, publish, or correct a published version;
- dispense, activate a phase, decide on a newer version of a source, set confidentiality, send the
  agenda, close a case;
- Keep an item against a review finding, accept a stopped review chain;
- authorize an agent, or switch an agent's Confidential right.

Other case work also has no `wayai cases` command: tasks, owners and deadlines, linking inputs,
citing passages and proposing values are done in the web app.

## Case types as files

A space administrator (or the organization's owner) can hold a case type as files and push a
draft of it. These commands take your own sign-in (`wayai login`), never a case token.

```bash
wayai cases type pull <org_id> <space_id> <type_id> [--dir <folder>]   # default ./<abbreviation>
wayai cases type diff [--json]     # run in the type's folder
wayai cases type push [--dry-run]
```

- `type pull` writes the type's current version: `type.json` (its name and definition) and its
  package under `instructions/`, `templates/`, `tools/` and `sources/`. Nothing else in the
  folder is read or written. Pulled again into the same folder, it refuses while a file you
  changed differs from the published version; `--overwrite` replaces those after asking.
- `type diff` compares the folder with the published version: the name, the definition by part,
  and each package file added, changed or removed (confidential ones marked). It exits 1 when the
  folder has a problem a push would be refused for, or when the type was published as a new
  version since your pull. It does not see a package revision of the current version: once one is
  out, a push is refused even when the diff is clean (below).
- `type push` checks the folder first. While a problem remains it sends nothing and names every
  one. Then it pushes the folder as the type's draft, replacing its open draft, and prints the
  draft's id.

A draft changes nothing. Publishing it is a person's act in the web app: the CLI never publishes.

- A draft whose definition changed publishes as the type's next version, which new cases open on.
  A case stays on the version it opened on.
- A draft that changes only the package publishes as a revision of the current version. The
  publisher chooses whether it reaches only the cases opened from then on, or the open cases on
  that version too.
- A case reads the newest revision that reaches it. A closed case keeps the revision it read when
  it closed.

A push from a folder pulled before the type's latest version or package revision is refused: pull
into a new folder and carry your changes over. `type push` does not yet name the package revision
its folder was pulled from, so once the type's current version holds a package-only revision, every
push of it is refused (`case_type_draft_base_revision_required`) until the CLI names it.

### In the organization's workspace: `cases/`

In the repository bound to the organization (`wayai-ws/wayai.yaml`), case types live beside hubs,
bases and communities, one folder per case space and one per type in it:

```bash
wayai pull cases/<space_id>/<type_id>    # first pull: writes wayai-ws/cases/<space_id>/<abbreviation>/
wayai pull cases/<space>/<type>          # again, by the folders' names (or by ids)
wayai push cases/<space>/<type> [--dry-run]
```

- Each type folder is the same folder `type pull` writes, pulled and pushed by the same rules:
  `--overwrite` (and `--yes`) on pull, `--dry-run` on push. `wayai cases type diff` works in it.
- Name the space's folder as you like: the markers of the types in it bind it to its space, and
  must all name one space of the workspace's organization. A first pull names a new space folder by
  the space's id.
- A case type is always named: `cases/<space>/<type>`, or run inside its folder. A bare
  `wayai pull` / `wayai push` never reaches one, and a bare folder name never means one.
- It takes your own sign-in, never a case token. A case's clone is no type folder: clone each
  case into a folder of its own (`wayai cases clone`), apart from this workspace.
- The worktree scope has a `cases` axis of case space ids: `wayai use cases/<space>` scopes this
  checkout to a space, and the first pull or push seeds it.

## Exit codes and limits

- Every `wayai cases` command exits 1 on an error, and says why.
- It also exits 1 when `pull` names a conflict, when `check` finds a submit would be refused or a
  package file that differs, and
  when `review` finds a stopped chain, and when `type diff` finds a problem in the folder or a
  version published since the pull.
- Your case commands and your case pages in the web app share one rate budget. A refusal says when
  to try again.

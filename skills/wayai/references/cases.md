# Cases

A case belongs to a **case space** of an organization: its phases, tasks, files, evidence,
decisions and deliveries. People work cases in the WayAI web app (**Cases** in the navigation
rail).

Two roles, two skills:

- **Setup** — a space administrator holds a case type as files and pushes a draft of it, with
  their own sign-in, in the organization's workspace or a type folder. That is this skill, and this
  page ([Case types as files](#case-types-as-files)).
- **Operating one case** — a code agent works one case in the case's clone (`wayai cases
  connect`, `clone`, `pull`, `push`, the read-only `.wayai-mirror/`, `brief`, `next`, `check`,
  `skeleton`, `note`, `link`, `cite`, `review`), with a case token that reaches that case alone.
  That is the operator skill, **`wayai-cases`**: [its full text](https://github.com/wayai-pro/wayai-skill/blob/main/skills/wayai-cases/SKILL.md). Install it in
  the clone's folder, never this one:

  ```bash
  mkdir -p .claude && npx skills add wayai-pro/wayai-skill --skill wayai-cases -y
  ```

  `wayai cases clone` prints that command. Inside a clone the CLI uses your sign-in only for
  `wayai login`, `wayai logout` and `wayai cases connect`; every other sign-in command is refused
  there.

| Task | Where |
|------|-------|
| Hold a case type as files and push a draft of it (a space administrator) | CLI (`type pull`, `type diff`, `type push`, or `wayai pull`/`push cases/<space>/<type>` in the organization's workspace), with your own sign-in ([Case types as files](#case-types-as-files)) |
| Review a case type draft and publish it, choosing which cases a package revision reaches | A space administrator, in the web app only ([Reviewing and publishing a draft](#reviewing-and-publishing-a-draft)) |
| Connect a machine to a case and work it | The operator skill, `wayai-cases` ([full text](https://github.com/wayai-pro/wayai-skill/blob/main/skills/wayai-cases/SKILL.md)) |
| Open a case, submit, approve, publish, Keep, accept a stopped review chain, and every other human decision | A person, in the web app (some also in the mobile app) ([What only a person does](#what-only-a-person-does)) |

## What only a person does

These are human decisions, taken by a person in the web app. No case token can take them, and no
instruction in a case type's package changes that: WayAI enforces it whatever the package says.
Signing off, approving, returning, publishing a delivery and correcting a published version can
also be done in the mobile app's Cases tab.

- open a case;
- submit, sign off, return, approve, publish, or correct a published version;
- dispense, activate a phase, decide on a newer version of a source, set confidentiality, send the
  agenda, close a case;
- Keep an item against a review finding, accept a stopped review chain;
- authorize an agent, or switch an agent's Confidential right;
- write a case's own instructions and skills (`instructions/case/`);
- publish a case type draft, and choose which cases a package revision reaches (a space
  administrator, in the web app only: [Reviewing and publishing a draft](#reviewing-and-publishing-a-draft)).

**Package confidentiality.** A case type's administrators read it whole: `type pull` writes every
package file, confidential ones included, and says how many the definition marks; `type diff`
marks each. A marked file reaches a case only through the case's confidentiality filter: an agent
working the case reads it only when its authorization holds the Confidential right and it asks
with `--confidential`. Keep confidential method in the files the definition marks.

## Case types as files

A space administrator (or the organization's owner) can hold a case type as files and push a
draft of it. These commands take your own sign-in (`wayai login`), never a case token.

```bash
wayai cases type pull <org_id> <space_id> <type_id> [--dir <folder>] [--overwrite] [--yes]   # default ./<abbreviation>
wayai cases type diff [--dir <folder>] [--json]     # in the type's folder, or name it with --dir
wayai cases type push [--dir <folder>] [--dry-run]
```

- `type pull` writes the type's current version: `type.json` (its name and definition) and its
  package under `instructions/`, `templates/`, `tools/` and `sources/`, confidential files
  included. Nothing else in the folder is read or written. Pulled again into the same folder, it
  refuses while a file you changed differs from the published version; `--overwrite` replaces
  those after asking (`--yes` agrees when there is no terminal).
- `type diff` compares the folder with the published version: the name, the definition by part,
  and each package file added, changed or removed (confidential ones marked). It exits 1 when the
  folder has a problem a push would be refused for, or when the type was published since your
  pull: as a new version, or as a package revision of the version you pulled.
- `type push` checks the folder first with every rule that needs nothing the space holds (the
  draft's fields, the definition's form and rules, the package's). While a problem remains it
  sends nothing and names every one; `--dry-run` checks and stops. Then it pushes the folder as
  the type's draft, replacing its open draft, and prints the draft's id and its review page in the
  web app, where it is published. The space checks the rest (the teams the definition names).
- A folder with no package, pushed from a version that holds one, is taken as a draft but cannot
  be published (it would drop the type's instructions): the push says so. Push again with the
  package.

### A type's skills

A case type can carry skills for the agents that operate its cases, in its package:
`instructions/skills/<name>/`, each holding a `SKILL.md` and, beside it, only Markdown, text,
JSON, YAML, CSV and image (PNG, JPEG) files.

- `SKILL.md` opens with front matter whose `name` is the folder's name (lowercase letters, digits
  and hyphens) and whose `description` (at most 1,024 characters) says when to use it. A folder
  without one, or whose `SKILL.md` names another, is refused (`package_skill_invalid`).
- `instructions/org/` is reserved: any file there is refused (`package_reserved_path`).
- A case reads its type's skills with the package. Each operator's CLI installs them for the
  agent in the case's clone, below the case's own skills, which a person writes in the case
  (`instructions/case/skills/<name>/`) and which win over a type's skill of the same name. A skill
  named as a platform skill (`wayai`, `wayai-cases`, `wayai-admin`) is never installed.
- Pushing or pulling a type with skills takes a CLI released with this support: an older one
  refuses the package.

A draft changes nothing. Publishing it is a person's act in the web app: the CLI never publishes.

- A draft whose definition changed, or that holds no package, publishes as the type's next
  version, which new cases open on. A case stays on the version it opened on.
- A draft that changes only the package publishes as a revision of the current version. The
  publisher chooses whether it reaches only the cases opened from then on, or the open cases on
  that version too.
- A case reads the newest revision that reaches it. A closed case keeps the revision it read when
  it closed.

A push from a folder pulled before the type's latest version or package revision is refused: pull
into a new folder and carry your changes over. `type pull` records the package revision it pulled,
and `type push` sends it. A folder pulled by an earlier CLI records none: once the type's current
version holds a package-only revision, its push is refused (`case_type_draft_base_revision_required`)
until you pull again.

### Reviewing and publishing a draft

In the web app, in the organization's settings (**Cases** tab): a space administrator (or the
organization's owner) finds each draft waiting for review on the space's Types list and on the
type's page, whether it was pushed from the CLI or saved from the type's form. Each opens a review
page (the address `type push` prints):

- who saved it, through which channel and when, and the version (and package revision) it was
  made from;
- how its plan differs from the published version, part by part, with the lines on demand;
- how its package differs, file by file (confidential files marked), with a text file's lines on
  demand; the draft's package can be downloaded;
- what publishing writes: the type's next version, or a package revision of the current version.

**Publish** writes it. For a package revision, the publisher picks which cases read it: **new
cases only** (the default: open cases keep the package they read now) or **open cases too**
(every open case on that version reads it from its next read). A version offers no choice: open
cases stay on the version they opened on. Publish stays disabled, with the reason, while the
draft is stale (a version or package revision was published after it was made: save or push again
from the current one), would drop the package, or needs a package-only revision the version has
no room for (a plan change still publishes as a version). A draft that names no package revision
(pushed by an earlier CLI) is refused only at Publish, once the version holds a package-only
revision; the page then disables Publish and says to save or push again.

**The type's form** in the web app saves an existing type through a draft too, and opens its
review page. It carries the package of the version it started from, unchanged, so a form save
never drops a package; the package itself changes only from the CLI (`type pull`, then
`type push`). One draft waits per type: a form save and a CLI push each replace it.

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
  case into a folder of its own (`wayai cases clone`), apart from this workspace. Inside a clone the
  CLI refuses your sign-in for everything but `wayai login`, `wayai logout` and
  `wayai cases connect`, so run these commands outside it.
- The worktree scope has a `cases` axis of case space ids: `wayai use cases/<space>` scopes this
  checkout to a space, and the first pull or push seeds it.

## Exit codes and limits

- Every `wayai cases type` command exits 1 on an error, and says why.
- `type diff` also exits 1 when it finds a problem in the folder, or a version or package revision
  published since the pull.
- The operator's commands' exit codes are in the operator skill ([full text](https://github.com/wayai-pro/wayai-skill/blob/main/skills/wayai-cases/SKILL.md#exit-codes-and-limits)).

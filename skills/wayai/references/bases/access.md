# Access dimensions (`x-access`)

Open this when records in a base must reach only some of the people, hubs and agents that use it:
a school whose teachers see only their own classes, a company whose regions see only their own
accounts. Concepts for the rest of the Data surface are in [README.md](README.md).

A record type names one of its fields in `x-access`, and the organization's **access dimension**
that field holds a value of (`class`, holding `6A`, `7A`, …). People, hubs and agents are assigned
the values they may read and write. A hub agent's call through a toolset then reaches only the
records whose value is in its set.

> [!IMPORTANT]
> **Declare `enforce` once values are assigned.** A record type in `enforce` is filtered, and its whole
> base refuses what [an enforcing base refuses](#what-an-enforcing-base-refuses). Before you declare it:
>
> - **Assign values first.** Under `enforce`, a hub or a person with no values reaches nothing on the
>   type, and neither does an assignment that leaves the type's dimension out; an agent with no values
>   narrows nothing ([Who sets each layer](#who-sets-each-layer)). Give every record a value too: one
>   without is reached only by a set holding `*`. [`wayai access explain`](#wayai-access-explain) shows
>   what a set reaches.
> - **One enforcing type changes its whole base.** Every call bound to a toolset there is refused
>   `batch` and file writes. `sql_query` and SQL Actions answer only the rows the call's set reaches
>   ([SQL under an access set](#sql-under-an-access-set)); a type whose [person entries](#person-entries)
>   are in `enforce` refuses them instead, and serves file reads only as its person entries allow, and a
>   type enforcing an entry on a [list field](#list-fields) refuses SQL too. If that base's toolsets need `batch` or file writes, keep the
>   types you enforce in a separate base.
> - **Enforce holds only calls bound to a toolset.** Organization members in the console or the
>   `wayai records` and `wayai bases` commands, and base API tokens not bound to a toolset, reach the base
>   unfiltered ([Limits](#limits)), save a download URL on a base with a type whose
>   [person entries](#person-entries) are in `enforce` (but of a [shared file type](#shared-file-types)).
>
> **`report` is for adding `x-access` to a type that already has live users**: it hides and refuses
> nothing, and only counts what `enforce` would ([Report mode](#report-mode)). Declare `report`, assign
> values, give every record a value, then switch. **The undo is switching the type back to `report`**,
> which stops its filtering and refusals at once. An `x-access` a record type already held before
> declarations were checked takes effect as it is stored: `wayai pull bases/<base>` shows each type's
> declaration.

## How it fits together

| Piece | Where it lives | Who sets it |
|---|---|---|
| The rule: which field of a record type holds which dimension | The record type's schema (`x-access`), so it goes through preview → promote | Whoever edits the base's config |
| The dimensions and their values | The organization (`wayai dimensions`, or Settings → Organization → Access) | Organization admins |
| Who may use which values: org members, hubs, hub admins, team members, hub users, agents | WayAI, beside each of them | Organization admins and hub admins ([below](#assigning-values)) |

A base holds no people: WayAI works out each call's set from the assignments and sends it with the
call. The base only compares the set with each record's value.

## Declaring `x-access` on a record type

`x-access` is a list at the root of the schema. A dimension entry names a field, a dimension and a mode
(a list may also hold [person entries](#person-entries) and `{ shared: true }`):

```yaml
# record-types/attendance.yaml
id: attendance
name: Attendance
json_schema:
  type: object
  properties:
    student_id: { type: string }
    class: { type: string }        # holds a value id of the "class" dimension, such as "7A"
    present: { type: boolean }
  required: [student_id, class]
  x-access:
    - { field: class, dimension: class, mode: enforce }
sources: []
```

- `field` — a top-level property of the record type declared `{ "type": "string" }`, or a list of strings (`{ "type": "array", "items": { "type": "string" } }`; see [List fields](#list-fields)). Values are matched as text, so a field of any other type is refused.
- `dimension` — the id of one of the organization's dimensions: ASCII letters, digits, `_` or `-`, starting with a letter or digit, as every dimension id is. The base cannot check it against the organization's list: a misspelled dimension is accepted, and once enforced it reaches nothing.
- `mode` — `enforce` (filters the type and turns on [the refusals](#what-an-enforcing-base-refuses); switch back to `report` to undo) or `report` (counts what enforce would hide or refuse, and changes nothing; [Report mode](#report-mode)), for a type that already has live users.
- **Several entries combine with AND**: a record is reached only when every entry's field holds a value the set has for that entry's dimension. Within one dimension, the set's values combine with OR. A dimension may appear in only one entry of a record type, so it filters one field.
- **A record with no value** in the field is reached only by a set holding `*` for that dimension.

### List fields

A dimension entry may name a field that holds a list of value ids, for a record that belongs to several
values at once (a lesson shared by classes `7A` and `7B`):

```yaml
properties:
  classes: { type: array, items: { type: string } }   # ["7A", "7B"]
x-access:
  - { field: classes, dimension: class, mode: enforce }
```

- **Reads**: a record is reached when **any one** of its list's values is one the set reads. An empty
  list, like a missing field, is no value: reached only by a set holding `*`.
- **Writes**: **every** value of the list must be one the set writes: the new list on a create, both
  the current list and the new one on an update, the current list on a delete. An empty or missing
  list is written only by a set holding `*`.
- **SQL**: once such an entry is in `enforce`, every call bound to a toolset is refused `sql_query` on
  that base, and each SQL Action that reads the type, with `ACCESS_REFUSED`: SQL is not held to the
  values of a list field. Records reached through `list`, `get`, search and relationships are not
  affected. Keep the entry in `report` (where SQL stays served) until the base's toolsets can do
  without SQL, or hold the values in a text field instead.

**Checked at config write and at promotion.** The list must be non-empty and each entry well formed
(no other keys) — a dimension entry, a [person entry](#person-entries) or `{ shared: true }`; the
record type must be native (`sources: []`), since a source-backed type's rows are
upstream, where nothing filters them. A refused write answers
`Invalid x-access`, listing each problem. A promotion is refused for any of them, including one
production already holds. `x-access` promotes like any other schema key.

**What a push reports.** `wayai push bases/<base>` warns, without refusing anything, for each toolset
that will stop serving some of its tools once a type enforces: `batch` and its file write tools
([`access-enforcement`](toolsets.md#tool-design-warnings-at-push)); and, once a type declares a person
entry, `sql_query` and its file read tools too (file reads then served only as
[person entries](#person-entries) allow, the file types it names as marked
[shared](#shared-file-types) still read by calls for team members); and, once a type declares a dimension entry on a
[list field](#list-fields), `sql_query`. Otherwise `sql_query` and SQL Actions are not named:
they stay served, held to each call's set. See
[What an enforcing base refuses](#what-an-enforcing-base-refuses).

**Before a type enforces,** give every record a value: a record without one stays hidden from every
set that does not hold `*`. A type in `report` counts the records that would be hidden.

### Person entries

A record type whose rows belong to people (a patient's records) names, in a **person entry**, the field
that holds the ids of the people each record belongs to:

```yaml
# record-types/patient.yaml
json_schema:
  type: object
  properties:
    clinic: { type: string }
    guardian_ids: { type: array, items: { type: string } }
    doctor_id: { type: string }
  x-access:
    - { field: clinic, dimension: clinic, mode: enforce }                # a dimension entry
    - { field: guardian_ids, person: true, owner: true, mode: enforce }  # a person entry
    - { field: doctor_id, person: true, mode: enforce }                  # a person entry
```

- `field` — a top-level property declared `{ "type": "string" }`, or a list of strings
  (`{ "type": "array", "items": { "type": "string" } }`). A field is named by one entry only: never
  twice, and never by a dimension entry too.
- `person: true` — marks a person entry.
- `owner: true` — on exactly one of the type's person entries.
- `mode` — `enforce` or `report`, the same on every person entry of the type. Dimension entries beside
  them keep their own modes.
- `inherit` entries are refused.

**What a person entry does today.** Every hub toolset call carries the people its turn is for
([Whose sets bound a turn](#whose-sets-bound-a-turn)). A person entry in `enforce` is matched against
them only on a **call for team members**: one whose agent is not customer-facing (roles `pilot`,
`pilot_specialist`, `pilot_advisor`, or the host turn's agent of a background agent's call), whose every person
is an organization member, a hub admin or a hub team member ([A person's kind](#a-persons-kind)), and which is
neither an eval run declaring a contact nor a task delegated from any other call. In `enforce`:

- **A call for team members reads a record when every person it is for is named in one of the type's
  person fields** (one id of a list field is enough), or holds [`$person`](#every-persons-records-person).
  With two people on a turn (a team member consulting another, say), a record naming only one of them is
  not reached. Dimension entries still hold beside them.
- **Every person's records:** when every person the call is for holds `$person` (or, on a turn for no
  one, such as a hub automation's, when the agent does), the call reads every record of the type. A turn
  for no one whose agent does not hold it reaches none of them.
- **Every other call reaches none of its records:** a customer-facing agent's call (a team member's own
  conversation with one included), a turn in a customer's conversation, a hub user's or everyone else's, a
  [customer-scoped](toolsets.md#customer-scope) toolset's call, and a customer-only set.
- **Every read follows them:** `get`, `list`, a relationship `list` and a reference (`x-fk`) to a record;
  a reference to a record the call does not read answers as one to a record that does not exist. So does
  a `list` the search store serves (a `search` filter, a filter on `archived`, or more records than the
  base lists itself) and a `get` of an archived record, which answers as not found when the call does not
  read it.
- **A call for team members writes its records for the people it is for** ([Person writes](#person-writes)).
  Every other call writes none of them: each write is refused with `ACCESS_REFUSED`, or answers as not
  found when the call does not read the record.
- **Its base** counts the type as enforcing for [every base-wide refusal](#what-an-enforcing-base-refuses),
  and refuses `sql_query` and each SQL Action that reads the type, with `ACCESS_REFUSED`. Files carry no
  person: a call for team members reads every file of a [shared file type](#shared-file-types) (`list`,
  `get` and `get_url`). Of any other file type, a call holding every person's records (above) is served
  every file read but `get_url`; any other call for team members reads with `get` only a file an owning
  link holds from a record the call reads (any other answers as not found), and is refused `list` and
  `get_url`. Every other call is refused file reads, of a shared file type too.
- **In `report`, person entries filter and refuse nothing**; report mode counts what enforcing them would
  hide from, or refuse, a call for team members ([Report mode](#report-mode)).
- Calls not bound to a toolset are not filtered ([Limits](#limits)), with one exception: on such a base
  no call is given a download URL (`get_url`) but of a shared file type, since the URL would not open.
  Read the file with `get`.

So declare person entries in `report` first, and switch them to `enforce` once each record names its
people and those meant to reach every person's records hold `$person`.

#### Person writes

With person entries in `enforce`, a call for team members (above) writes a record of the type only as the
people it is for allow. A person field's ids are judged one by one, against what the field held before:

- **Adding** an id a field did not hold: every person the call is for must hold that id — it is one of
  their own ids — or hold `$person`. With a team member holding `$person` beside another person, the
  other person must hold it too.
- **Keeping** an id the field already held: always allowed.
- **Removing** an id: allowed when some person the call is for holds it; any other id only when every
  person the call is for holds `$person` (or, on a turn for no one, the agent does). A full replace
  (`create` keyed onto an existing record) removes what it leaves out, and **a `delete` removes every id
  the record names**.
- **The record it leaves** must still name each person the call is for who does not hold `$person`
  (one id of theirs in any person field), unless every one holds it. So a `create` naming no one in its
  person fields is allowed only to a call holding every person's records.

Each refusal is `ACCESS_REFUSED` (403); one for adding or removing names the field and the id. Dimension entries hold beside
them as on any enforcing type ([What an enforcing base refuses](#what-an-enforcing-base-refuses)), and so
does the rest of that section: an update or delete of a record the call does not read answers as not
found, a `create` keyed onto one as a held key (`CONFLICT`), and a reference (`x-fk`) to one as a
reference to a record that does not exist. Linking or unlinking a record changes no person field: it is
allowed when the call reads the record. A composite holds every step to these rules and rolls back as a
whole. `batch` and file writes stay refused on the base.

**The owner field is not filled for you yet.** A `create` that leaves out the field marked `owner`
leaves it empty, so name the person the record belongs to. Filling it with the conversation's customer
comes with calls for customers.

**Triggers.** A record write is refused when an `internal_write` trigger it sets off writes a record
type that enforces an access rule ([What an enforcing base refuses](#what-an-enforcing-base-refuses)),
with one exception: a trigger that writes a type whose only rule is its person entries, set off by a call
for team members, runs when

- it locates its target through a top-level field declaring an `x-fk` into that type, keyed as it
  matches (`match.source_field: data.<field>`), so it patches only a record the call reads;
- every trigger on the target type sets `skip_cascade_writes: true`;
- and its `patch` sets no person field of the target, unless every person the call is for holds
  `$person`.

Otherwise the write that would set it off is refused with `ACCESS_REFUSED`, naming the trigger.

**`{ shared: true }`** is an entry of its own, with no other key, at most once, beside dimension entries
only (never beside a person entry). It is stored and changes nothing today.

### Shared file types

A file type marked `shared: true` ([files.md](files.md#file-types)) holds files every call for team
members reads directly on a base whose person entries are in `enforce`: forms, price lists, clinic
policies. Mark it in the preview and promote it; a file type with no mark is not shared. The mark lives
on the file type only: a folder whose files must be open becomes a file type of its own, since there is
no folder-level mark. A call for a customer, or any other call the person rules refuse files to, still
reads none of them. Taking the mark off stops the download URLs already given for its files.

## Dimensions and values

An organization defines its dimensions and each dimension's values. Each has a fixed **id** (what
`x-access` and assignments name) and an editable **label** (what people read).

```bash
wayai dimensions list [--org <org_id>] [--json]
wayai dimensions create <dimension> --label <label>
wayai dimensions relabel <dimension> --label <label>
wayai dimensions add-value <dimension> <value> --label <label>
wayai dimensions relabel-value <dimension> <value> --label <label>
wayai dimensions retire-value <dimension> <value>
wayai dimensions restore-value <dimension> <value>
```

- **Who.** Listing needs organization membership; every change needs an organization admin, and an API token holding `org:admin` may make them. `--org` names the organization, else the workspace's (`wayai-ws/wayai.yaml`). The same lists live in the app under Settings → Organization → Access (`/settings/organizations/<orgId>/access`).
- **Ids** are ASCII letters, digits, `_` and `-`, starting with a letter or digit, up to 32 characters (`class`, `7A`, `north-campus`). `*` is never an id. An id never changes: change the label instead.
- **Limits.** At most 8 dimensions per organization, and at most 100 values per dimension, retired ones counted.
- **A value is never deleted, only retired.** WayAI cannot see which records hold a value, so a retired value keeps matching the records that hold it. It cannot be assigned anywhere new; where it is already assigned (the same person, hub or agent, in the same read or write list), it stays and keeps working. `restore-value` makes it assignable again. A dimension has no delete either.

## Assigning values

An assignment names, per dimension, the value ids its holder may **read** and may **write**:

```yaml
class:
  read: ["7A", "7B"]
  write: ["7A"]
```

- **Write stays within read**: every value in `write` must be in `read`, unless `read` is `["*"]`.
- **`["*"]`** reaches the dimension unrestricted, records with no value included. It is always explicit and stands alone in its list. It is never automatic, not even for admins.
- **A dimension the assignment leaves out reaches nothing** on the record types that dimension scopes, and so does a layer that was never assigned, except an agent's (below).
- Every dimension and value must be the organization's own. A write replaces the whole assignment.

### Who sets each layer

| Layer | Set by | Where | Never assigned |
|---|---|---|---|
| Org member values | An organization admin, signed in as a person | Settings → Organization → Access → Organization members | Reaches nothing |
| Hub values | An organization admin, signed in as a person, on a **production** hub | Settings → Organization → Access → Hubs → **Hub values** | Reaches nothing |
| Hub public values | The same | Settings → Organization → Access → Hubs → **Public values** | Reaches nothing |
| Hub admin values | An organization admin, signed in as a person (a hub admin never sets their own) | Hub → Users → the admin → **Access values** | Reaches nothing |
| Hub team member values | A hub admin, signed in as a person: one assignment per person per team | Hub → Users → the team member → **Access values** | Reaches nothing |
| Hub user values | A hub admin, signed in as a person | Hub → Users → the hub user → **Access values** | Reaches nothing |
| Agent values | A hub admin, on a **preview** hub, or an API token holding `hub:admin` | The agent's page in the app, or `access_values` in `agents/<slug>.yaml` with `wayai push` | **Narrows nothing**: the agent reaches what its hub and the person it serves reach |

- **People's values are a person's to assign.** An API token is refused every assignment but an agent's, whatever its permissions. The CLI assigns agent values only (`access_values`); every other layer is set in the app.
- **A preview hub uses its production hub's hub values and public values**, so they are set on the production hub. A preview never published has none, and reaches nothing.
- **Agent values are config**: set on the preview hub, carried to production by `wayai publish`, and shown read-only on production. In `agents/<slug>.yaml`, leaving `access_values` out keeps what is stored, `null` clears it (the agent narrows nothing again), and an assignment replaces it whole. A push that changes them needs `hub:admin` on the hub.
- **Hub user values** go only on a hub user whose access was granted, never on a contact recorded automatically when they wrote in, and never hold `*`.
- **Team member values** go on the person's membership of one team; a person in several teams gets the union of those teams' values. An invite not yet accepted matches the person who signs in with its email address. Deleting a team removes its members' values.
- **An agent with its own values that can run code needs a sandbox of its own** ([Limits](#limits)).

### Every person's records (`$person`)

Beside its dimensions, an assignment may hold the reserved key `$person`, which reaches **every
person's records**:

```yaml
$person: { read: ["*"], write: ["*"] }   # its only value
class:
  read: ["7A"]
  write: []
```

- It is for record types whose rows belong to people (a patient's records), named by
  [person entries](#person-entries). **What it reaches today:** a call for team members whose every
  person holds it reads every record of those types, and their files; so does a turn for no one whose
  agent holds it. Where only some of the people hold it, each other person must be named in a record for
  the call to reach it. Such a call writes those records too: with every person holding it, any record;
  otherwise as [Person writes](#person-writes) allows.
- It is always explicit, never automatic, not even for admins, and it is not a dimension: it counts
  against no dimension limit, and leaves every dimension as the assignment states it. An agent whose
  values hold only `$person` has values, with no dimension, so it reaches nothing on the record types
  that enforce a dimension.
- **Who may hold it:** org members, hub admins, team members (per team) and agents. Never hub users,
  a hub's values or its public values: those writes are refused.
- **A customer-facing agent never holds it** (roles `pilot`, `pilot_specialist`, `pilot_advisor`). A
  write that gives it one is refused, and so is a role change, a `wayai push` or a `wayai publish`
  that would leave a customer-facing agent holding it. To move an agent holding it to such a role,
  remove `$person` first; a push that does both in one file writes the values first.
- **In a customer's conversation, it never reaches past that customer.** The customer is always one
  of the people a turn is for, and never holds `$person`, so a call there reaches only records the
  customer reaches too.
- Set it in the app with the **Every person's records** switch where the values are edited, or, for an
  agent, as `$person` in `access_values` in `agents/<slug>.yaml`.

## How a turn's set is formed

Every hub toolset call a turn makes carries one set, worked out for that turn and agent. **Every layer
only narrows**: the set is the intersection of the hub's values, the set of each person the turn is
for, and the agent's values, per dimension, each list intersected and write kept within read.

### A person's kind

A person's values depend on what they are on the hub, checked in this order; the first that matches wins:

1. **Organization member** of the hub's organization: their org member values.
2. **Hub admin**: their hub admin values.
3. **Hub team member**: the union of their values across the teams they are in.
4. **Hub user** whose access was granted: their hub user values.
5. **Everyone else**, such as a channel contact: the hub's **public values**, which hold only on a
   customer's own records. On a [customer-scoped](toolsets.md#customer-scope) toolset's call, they
   reach the call's customer's records of `owned` and `root` record types; every other record type
   that enforces a dimension reaches nothing. A call that names no customer reaches nothing.

So a teacher who is also a contact of a customer-facing hub gets their team values there, never its
public values. A parent who must see their own child's records gets them through public values (for
example `class: ["*"]`) on a customer-scoped toolset, which narrows them to that parent's records.

**Only a verified identity makes a person.** A conversation's person counts as themselves only while
they write on a channel that verifies its sender: the app (their signed-in account), WhatsApp,
Instagram or Telegram. A message from any other channel, such as email or an API channel (whose
email address is free text), makes them **everyone else** until the conversation has answered them
and they write on a verified channel again. `wayai send-message` writes on the app channel as you.

### Whose sets bound a turn

| Turn | Its set |
|---|---|
| A person's message in their own conversation | Hub ∩ that person's ∩ agent |
| A team member's message or consult in a customer's conversation | Hub ∩ the team member's ∩ the customer's ∩ agent: never more than the customer's own turn reaches, because the agent answers into a conversation the customer reads |
| A team member's send with no customer | Hub ∩ the team member's ∩ agent |
| A turn no person started (a scheduled follow-up, a continuation) | Hub ∩ the conversation's person's, if it has one ∩ agent |
| A consultant answering a consult thread | Also ∩ the thread's initiator's: a team member's set, or an agent initiator's values. A thread that is gone reaches nothing |
| A hub automation with no person | Hub ∩ agent |
| An eval run | Hub ∩ the set of the person who started it ∩ agent. A run started with an API token takes the set of the person who created the token; a run with no recorded starter reaches nothing |
| A background agent's call inside another turn (a monitor, an evaluator) | The host turn's set ∩ the host turn's agent's values ∩ its own |
| A task started with `delegate_to_hub`, or a consult with another hub | The delegating turn's set, stored when the task starts ∩ the current set of each person who started a delegating turn, on their hub ∩ the task hub's ∩ the task agent's. Revoking such a person's values reaches the running task. One who is no longer the kind they were, or is gone, reaches nothing. A delegation from an everyone-else turn reaches nothing on enforcing types, since a task names no customer |

- A person, team row, agent or thread that cannot be found reaches nothing, and so does a set that
  could not be worked out: a call never reaches more because a lookup failed.
- A turn reuses its set for up to a minute, so a change of values reaches a long turn within a
  minute and the next turn at once.
- An organization admin gets no `*` in chat: assign yourself values like anyone else.

## What an enforcing base refuses

This applies once a record type enforces, on its whole base, to every call **bound to a toolset**:
a hub agent's call through a [Base Toolset](../connections.md#base-toolset) connection (which carries
the set) and a call made with a toolset-bound `rec_` token (which carries none). A base where every
declaration is in `report` mode refuses and filters nothing.

**Reads** of an enforcing type reach only the records the set reads:

- `list`, `get` and search return only those records; a `get` of any other answers as not found, archived records included.
- A relationship `list` returns a link only when each end on an enforcing type is readable, and a filter over related records runs only over the related records the set reads.

**Writes** on an enforcing type:

| Operation | Allowed when |
|---|---|
| `create` | the new value is one the set writes (on a [list field](#list-fields), every value of the list) |
| `update` | the current value and the new one are both ones the set writes |
| `delete` | the current value is one the set writes |
| linking or unlinking two records | each end on an enforcing type is one the set writes |
| a reference (`x-fk`) to an enforcing type | the referenced record is one the set reads |

A write never reveals a record the set does not read: an update or delete of one answers as not
found, a create keyed onto one as a held key (`CONFLICT`, 409), and a reference to one as a reference
to a record that does not exist, as [customer scope](toolsets.md#serving-a-customer) answers. A record
the set reads but does not write is refused. A composite runs every step under these rules and rolls
back as a whole. A type whose [person entries](#person-entries) are in `enforce` holds a write's person
fields too ([Person writes](#person-writes)).

**Refused with `ACCESS_REFUSED` (403)** on such a base, whatever the call's set:

- `batch`, which reaches any record of the base by id;
- file writes (`put`, `delete`, `put_url`): files carry no value, so they stay readable (`list`, `get`, `get_url`) and are written by none of these calls. Deleting a record that holds an owning link to a file is refused too. On a base with a type whose [person entries](#person-entries) are in `enforce`, file reads are refused as well, save as its person entries allow and of a [shared file type](#shared-file-types);
- a record write that sets off an `internal_write` trigger writing an enforcing type, since the trigger's write is not held to the call's set (but for a trigger into a type whose only rule is its person entries, held as [Person writes](#person-writes) states);
- a call that carries no set (a toolset-bound `rec_` token, for one) to a record, relationship or composite tool, or a SQL Action, that reaches an enforcing type, and to `sql_query`;
- on a base with a type whose [person entries](#person-entries) are in `enforce`: `sql_query`, and each SQL Action that reads such a type;
- on a base with a type enforcing a [list field](#list-fields): `sql_query`, and each SQL Action that reads such a type.

Reading config (`describe_graph`, the toolset's tool list) is always served.

### SQL under an access set

A call carrying a set is served `sql_query` and SQL Actions, and each query answers only the rows the
set reaches. This holds on every base, whether or not one of its types enforces, except a base with a
type whose [person entries](#person-entries) are in `enforce`, or one enforcing a
[list field](#list-fields), which refuses them:

- **`records`:** a record of an enforcing type only when the set reads its value or holds `*`, compared
  as the stored text. SQL never reaches a record a `get` would not; a value such as a date written
  another way can be left out where a `get` matches it. Records of other types answer as usual.
- **`relationships`, `audit_log` and `operations_log`:** a row only when every record it names is one
  the set reads, by the record's current value, so a record moved out of the set takes its whole
  history with it. Rows naming a deleted record (its history, its `record_delete` row, its
  relationships) are left out, even for a set holding `*`, and so are the operations rows of a deleted
  relationship. Rows naming no record answer as usual.
- **Joins and subqueries** are held the same way: every table a query reads is filtered.
- **A customer-only set** (the hub's public values, sent for everyone else) reads no record of an
  enforcing type through SQL.
- **Large bases:** each query over `relationships`, `audit_log` or `operations_log` first works out
  which records the set reaches, so on a base of several million records it can exceed the query
  limits, and it is refused rather than answered in part.

**Refused for a moment after a change.** When a type's `enforce` entries change (switched to or from
`enforce`, or an enforced field or dimension changed), the base takes a few seconds to update the
values its SQL reads. Until then every call bound to a toolset is refused `sql_query` on that base
and each SQL Action that reads the type, with `ACCESS_REFUSED`. That includes a base where nothing
enforces any more, so a withdrawn rule never leaves rows quietly out of an answer. A refusal that
persists means the update failed: SQL over the type stays refused until a retry succeeds, so report
one that lasts. A stored `x-access` that is malformed and may enforce refuses them too, until it is
corrected.

**Not filtered:** anything that reaches a base without a toolset binding, such as the console, the
`wayai records` and `wayai bases` commands, and base API tokens not bound to a toolset. Those
answer as they do today, save a download URL on a base with a person-ruled type, but of a
[shared file type](#shared-file-types) ([Person entries](#person-entries)).

## Report mode

A record type whose `x-access` is in `report` mode is filtered and refused nothing. Each call bound to
a toolset is still judged as if every dimension entry were `enforce`, and one that would be refused, or
whose answer holds records that would be hidden, is counted. [Person entries](#person-entries) in `report`
are judged on a call for team members only: the records a `get`, `list` or relationship `list` would hide,
an archived record a `get` reads and a `list` the search store serves included. SQL is the exception: a `sql_query` or SQL
Action call carrying a set is not counted, since nothing it would leave out is seen, except where a
type's entry on a [list field](#list-fields) would refuse it, which is counted as refused; one carrying
no set is counted as refused. Each count records the base, the record type,
the dimension (`$person` for a person entry), the kind of caller (a hub agent's call carrying a set, a customer-only set or none, or
a base API token), the calling agent or token, the toolset, whether it was refused or hidden (and
where), and how many records. Only ids are kept, never a person's name or email or any record's
content, and nothing is counted on a base with no `x-access`.

Nothing in the CLI or the app reads these counts yet. To see what a set reaches before enforcing, use
[`wayai access explain`](#wayai-access-explain).

## `wayai access explain`

```bash
wayai access explain --agent <agent_id> [--hub <uuid|folder>] [--user <user_id>] [--json]
```

Prints the set every hub toolset call of one agent sends for one person: the person's kind, each layer
in the order it is intersected, the effective read and write lists per dimension, and, for each value
left out, the layers that removed it. The agent id is the `id:` that `wayai pull` writes in
`agents/<slug>.yaml`; the hub is `--hub`, else the workspace's. It reads only, and needs read access to
the hub.

- **Without `--user`**, signed in as a person: your own set. With an API token: no person, so the hub's ∩ the agent's, what a hub automation's turn of the agent gets.
- **With `--user`** for anyone but yourself: needs an organization admin. Someone else's invites to a team that they have not accepted yet are not counted.
- It shows a person's own turn in their own conversation on a verified channel. A turn in someone else's conversation, a delegated task and a background agent's call are narrowed further and are not shown.
- It says when the set is customer-only (everyone else), when the person's own set reaches nothing (so every turn in their conversations does too, a team member's included), and when a preview hub was never published.
- A layer that cannot be read answers a retryable error, never a smaller set.

## Limits

- **Values bind only people who cannot change the hub.** They are a boundary for team members, hub
  users and contacts. A hub admin, or anyone who can write the hub's config, can author an automation
  (which runs with the hub's ∩ the agent's values), edit an agent's values, or change their own rows,
  so for them values narrow their turns but are not a boundary.
- **Organization members are not bound either.** Their org member values narrow what agents fetch for
  them, but they reach the organization's bases directly, unfiltered.
- **Only a verified identity makes a person.** A person named only by an unverified channel is
  everyone else, whoever the address belongs to ([A person's kind](#a-persons-kind)).
- **Shared conversations.** Values limit what a call fetches, not who reads the conversation
  afterwards: whatever an agent fetched within its set is in the conversation for everyone who reads it.
- **Shared sandboxes.** A sandbox shared by a conversation's agents keeps files beyond the
  conversation's history and reaches every agent sharing it. So an agent with its own access values
  that can run code must use a sandbox of its own: an agent write that would leave an agent holding
  values with `code_execution_enabled` on and a `sandbox_scope` other than `agent` is refused at config
  write. Set its `sandbox_scope` to `agent`
  ([Code sandbox](../agents/roles-and-settings.md#code-sandbox-all-llm-connectors)), turn its sandbox
  off, or clear its values.
- **Trigger webhooks.** A [trigger](integrations.md#triggers) delivers its records' payloads as the
  base is configured; no set filters them.
- **Unbound tokens.** A base API token not bound to a toolset reaches everything its grants allow,
  unfiltered, wherever it is placed, a hub's HTTP or MCP connection included. The one exception: on a
  base with a type whose [person entries](#person-entries) are in `enforce`, it is refused `get_url`, but
  of a [shared file type](#shared-file-types), and reads the file with `get`.

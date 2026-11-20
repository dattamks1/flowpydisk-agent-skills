# Calendar and tasks

Two small surfaces that share one hard problem: **a person's words about time
are not the times you should send.**

---

## 1. The rule that governs both: call `whoami` first

`start_at` and `end_at` are ISO-8601 datetimes and they are **absolute
INSTANTS, not wall clocks.** A user saying *"3pm Tuesday"* means 3pm **in their
own zone**.

So:

1. Call `whoami` and read `timezone`.
2. Convert.
3. Send an offset in the string - `"2026-09-01T15:00:00+05:30"` - which is the
   clearest way to say what you meant.

And the case an agent gets wrong by being helpful:

> **If `timezone` comes back empty the user has expressed no preference and you
> should ask, not assume UTC.**

**Over a connected app `whoami` has no `timezone` at all**: it answers from the app's own session, which carries the
person's name and the workspaces it was granted and none of their
preferences. The rule is the same as for an empty one: ask the person where
they are, once, and use it for the rest of the conversation. See the account
guide, `get_guide("account")`.

Assuming UTC for somebody in Asia/Kolkata books a meeting five and a half hours
out. It will look correct in every response you get back.

---

## 2. Events repeat. The rule is a subset of RRULE and it is checked

`rrule` on `create_event` accepts only what this product can **draw** - checked
on the way in, *"so a rule it cannot render is refused with a sentence rather
than stored and shown as nothing."*

| Part | What it takes |
|---|---|
| `FREQ` | `DAILY`, `WEEKLY`, `MONTHLY`, `YEARLY`. **Required** |
| `INTERVAL` | every N of those - biweekly is `FREQ=WEEKLY;INTERVAL=2` |
| `BYDAY` | `MO`..`SU`, comma-separated. With an ordinal for "the first Monday" (`1MO`) or "the last Friday" (`-1FR`) |
| `BYMONTHDAY` | "the 15th" |
| `UNTIL` or `COUNT` | **one of them is required - an endless repeat is refused** |

Worked:

    every Monday until the end of the year
      FREQ=WEEKLY;BYDAY=MO;UNTIL=20261231

    the first Monday of each month, twelve times
      FREQ=MONTHLY;BYDAY=1MO;COUNT=12

### The wall a repeat is walked on: the event's `timezone`

Every event stores the IANA zone it was made in (`timezone`), and its rule is
walked **on that zone's calendar and clock**, as Google Calendar and Outlook
do. "Every Monday at 18:00" made in Los Angeles is a Monday at 18:00 there,
every week, through both clock changes - which is Tuesday 01:00 UTC in the
summer and Tuesday 02:00 UTC in the winter. Three things follow:

* **Leave `timezone` out and the event takes the user's own** (whoami's
  `timezone`). Name it when they mean another place's time: *"a 9am New York
  stand-up"* is `timezone="America/New_York"` with a `start_at` of 09:00 New
  York (`2026-10-22T09:00:00-04:00`).
* **`BYDAY` names the weekday on that wall.** A Kolkata Monday 01:00 starts on
  a UTC Sunday. The rule still says `MO`.
* **`UNTIL` is best given as a DATE** (`20261231`): "through that day" in the
  event's zone, which the server stores as the UTC instant the day ends at
  there (RFC 5545 requires UNTIL in UTC once a start has a zone). A full UTC
  stamp (`20261231T182959Z`) is still accepted and means exactly that instant.
  `recurrence_text` reads it back as a date on the event's wall.

Changing an event's `timezone` with `update_event` does not move its stored
start, which is an instant. To keep "9:00" meaning nine in the new place, send
the new `start_at`/`end_at` with it.

### One date on its own: change it, move it, delete it, bring it back

> One event carries the rule. The dates it means are worked out when the
> calendar is read. **Editing a repeating event edits all of it, unless you
> name one date.**

* **Change or move one date**: `update_event(event_id, date=<that occurrence's
  start>, start_at=..., end_at=..., title=...)`, the start exactly as
  `get_calendar` returned it. That date becomes its own event: the response is
  it, with its own `id`, `detached: true`, `series` (the repeat's id) and
  `original_start`. Change it again by its own id (or with the same `date`).
  The rest of the series is untouched, and moving the series later does not
  move it.
* **Delete one date**: `delete_event(event_id, date=<start>)`. A date already
  changed on its own is deleted with `delete_event` on its own id.
* **Bring a deleted date back**: `restore_event(event_id, date=<the start that
  was deleted>)`.

In `get_calendar` a changed date is its own entry: `series_id` is the repeat's
id, `is_recurring` is false and `detached` is true. So when somebody asks you
to move next week's stand-up, move that date, not the series. When they ask to
move "the stand-up", ask whether they mean every date or just one.

Everyone invited (with or without an online meeting) is emailed about **that date only**:
an update when it is changed (a new time resets their reply for that date), a
cancellation when it is deleted, an update when it comes back. The Join link
and the chat stay the series'. Say so before you do it.

A changed date follows the series' invite list:
someone added to or taken off the series with `people` is added to or taken
off the changed date too, with no extra email. A change made to the changed
date's own list stays.

Changing the series' rule (`rrule`) leaves a changed date whose day the new
rule no longer has as a plain event of its own, no longer tied to the repeat.

### Reminders

`reminders` on `create_event` and `update_event` reminds the event's owner
before each start, by a notification and an email: up to three of `0` (at the
start), `5`, `10`, `30`, `60` and `1440` (a day before), in minutes. It
replaces the list. `[]` removes them. Each date of a repeat is reminded, and a
reminder whose moment has already passed is not sent late.

Set reminders only when the person asked ("remind me an hour before" is
`reminders=[60]`). They remind the owner only: people invited to an online
meeting get their own notification 10 minutes before, whatever this says.

---

## 3. A task has three times and they mean different things

| Field | Means |
|---|---|
| `due_at` | the **deadline** it must be finished by |
| `start_at` / `end_at` | the span the work **occupies** |

To remove a deadline, `update_task(task_id,
clear_due_at=True)`. Leaving `due_at` out leaves it alone, and a blank or
unreadable `due_at` is refused rather than read as "clear".

A task can have all three. The server refuses a span that ends before it starts,
and - the part worth knowing - **the calendar still keys off `due_at` alone.**
So giving a task a start and end does not put a block on the calendar where the
work happens. It records the intent, and the calendar shows the deadline - at
its time, as a slim chip with a checkbox the person can tick right there. Ticking it is the same `is_done` that `complete_task` and
`uncomplete_task` set, so a task an agent completes shows done on the calendar
and the other way round.

`show_on_calendar` overrides the account default (*"add all my tasks to
calendar"* in Settings) for one task. **Omit it** unless you have a reason. The
account default is a preference the person already expressed.

### 3a. Who a task is for, and comments

A workspace task can have an **assignee**: `assignee` on `create_task` and
`update_task` is a member's user id (from `list_workspace_members`), and `""`
on `update_task` takes it off them. **Assigning someone else rings their bell
and sends them an email**, so assign only when the person asked you to. A
personal task has nobody to assign it to, and the server refuses anyone
outside the workspace. `list_tasks(assignee="me")` is the person's own list.

`list_task_comments` returns the comments and the history together, oldest
first: `kind` is `"comment"`, or `"assigned"` / `"unassigned"` with who did
it (`author`) and to whom (`target`). `add_task_comment` posts as the person.
`mentions` (user ids) notify those members, and the assignee and creator hear
of every comment anyway. **A workspace viewer can comment and tick done the
tasks assigned to them, and nothing else**, so a viewer's token gets 403 on
any other change.

`update_task_comment` edits a comment and `delete_task_comment` deletes one. Editing is the author's alone, so another person's
comment answers 403 `not_yours`. Deleting is the author's, an admin's or the
owner's. **A deleted task comment is gone: there is no Trash for it**, which
is why a connected app asks the person first (`request_approval`). History
lines ("assigned", "unassigned") are not comments, and both tools answer them
400 `not_a_comment`.

### 3b. Priority and the checklist

`priority` is `"none"` (the default), `"low"`, `"medium"` or `"high"`, on
`create_task` and `update_task`. **It does not re-order the list**: the
person's own order (`position`) is what they see, so do not reorder tasks to
"sort by priority" unless they asked. `list_tasks(priority="high")` narrows the
list instead. Set a priority when the person said one ("urgent", "when you get
to it"). Do not invent one for every task you create.

`subtasks` is the task's **checklist**: a list of `{id, title, done}` in the
order shown. **Sending it replaces the whole list**, the same as `tags`, so to
tick one item read the task (`get_task`), change that item's `done`, and send
the whole list back with every `id` kept. An item sent without an `id` is a new
item. An item left out is removed. `[]` clears it. At most 100 items, a title
of 1 to 500 characters. Checklist items are not tasks: they have no due date,
assignee or comments, and ticking every item does **not** complete the task,
so call `complete_task` when the person says the task is done. A workspace
viewer cannot change the checklist, even on a task assigned to them.

### 3c. Lists

A space's tasks can be grouped into **lists** ("Home", "Q3 launch"). A task is
in at most one list, or in none: "no list" is the default "Tasks" the person
sees, and is not a list you can fetch. `list_task_lists` gives the lists in
order with each one's `open_count`. `list_tasks(task_list=<id>)` gives one
list's tasks and `list_tasks(task_list="none")` the tasks in no list. Put a
task in a list with `task_list` on `create_task` or `update_task`, and take it
out with `update_task(task_list="")`. A list belongs to the space it was made
in, so a list from another workspace or from the personal space is refused
(`invalid_task_list`).

`create_task_list`, `update_task_list` (rename, `position`, `archived`) and
`delete_task_list`. **Archiving is the gentle way**: the list leaves
`list_task_lists` and nothing else changes, and `archived=false` brings it
back. Deleting is for good, but **never deletes a task**: by default its tasks
move to no list, and only `tasks="trash"` moves them to Trash (each restorable
with `restore_task`). Ask the person which they want before trashing their
tasks with a list, and never pick `"trash"` because a list "is done".

### 3c-i. The task list a person sees

In a client that draws MCP Apps (claude.ai, Claude Desktop), `list_tasks`'
reply is also shown to the person as a FlowpyDisk task list: up to ten tasks
with their due date, priority and list, each opening in FlowpyDisk. The list
shows what the call returned, so narrow the call (`done=false`,
`assignee="me"`, `task_list`) to what the person asked about rather than
listing everything and filtering in prose. Do not repeat every row back in
text when the list is shown. Say what matters (what is late, what is next).
The list changes nothing: to complete a task, call `complete_task`.

### 3d. Repeating tasks

`repeat_rule` on `create_task` / `update_task` takes the **same rule vocabulary
as an event** (section 2): `FREQ`, `INTERVAL`, `BYDAY`, `BYMONTHDAY`, and
`UNTIL` or `COUNT`. Unlike an event's, **a task's repeat may have no end**
("pay rent on the 1st of every month" is `FREQ=MONTHLY;BYMONTHDAY=1`). It
**needs a due date**: the next date is worked out from it, so a rule without
one is `repeat_needs_due_at`. The rule is walked in the time zone of the
person who set it (`repeat_timezone`), so read `whoami` first as for any time.

**One task at a time.** Nothing is created ahead. `complete_task` on a
repeating task creates the next one, due on the rule's next date after this
one's due date, with the same title, notes, tags, list, priority, assignee and
checklist (all unticked). The answer's `repeat_next` is its id. A late
completion still makes the NEXT date, not today's, so a missed week is still
there to do. `uncomplete_task` leaves the next task alone, and completing
again does not make a second. `COUNT` counts down with each task and the last
makes none. To **stop** a repeat, `update_task(repeat_rule="")`. Do not trash
the task to stop it (trashing deletes that one task only, and nothing ahead of
it exists to delete). Editing the rule changes this task and the ones made
from it after, never the ones already made.

---

## 4. Events always have both ends. Tasks need neither

> Unlike a Task's single `due_at`, an event always has both a start and end.

`all_day` ignores the times and books the whole day. `location` is **free text** -
a room, an address, a meeting URL - and *nothing parses it*, so putting a
Zoom link there is fine and expecting the product to make it clickable-as-a-
meeting is not. For a FlowpyDisk meeting, use `online_meeting` (section 4a).

---

## 4a. Online meetings on events, and inviting people

(people need no online meeting.:
not on a booking's event.,: one email per save, worded for what
changed.)

**People can be invited to any event but a booking's.** A booking's event has
one invitee, the person who booked it: `update_event` with a non-empty `people`
on it is refused with `booking_people`. Turning the online meeting on or off
emails the people "now an online meeting" / "no longer an online meeting", and
a save that also moves the time sends one update, not two. A series update's
calendar file carries every date moved on its own, so their calendars
keep them.

**People can be invited to any event.** `people` works on an event with or
without an online meeting: the invitation with Yes / Maybe / No and its
calendar file, updates, cancellations, replies and the read-only copy on the
invitee's own calendar are the same either way. Without a meeting the email
and calendar file carry no Join link and name the event's `location`. The
online meeting adds exactly three things: the Join link, the meeting's chat and
the 10-minute meeting notification.

`online_meeting=True` on `create_event` / `update_event` puts a FlowpyDisk
meeting on the event: **one Join link and one Collab chat for the whole of it**,
a repeating event included ("all my scrum information in one chat"). The Join
link works from 10 minutes before each date. `get_event` returns it under
`meeting` (`join_path`, `live`, `opens_at`, `chat`) and the invitees under
`people`, with each reply.

**Which calls are on now**: `list_live_calls` lists the calls in the
space acted in that the person organises or is invited to and has not
declined, from 5 minutes before the start until the call ends. `state` is
`live` when someone is in it, else `starting_soon` (a start already past means
nobody is in yet). Tell the person. Joining is theirs, in the app.

**Someone outside the workspace is invited, not put in the chat**. Inviting an address that belongs to a FlowpyDisk account outside
the event's workspace gives them the Join button and the email. `chat` is
`null` for them, they read that date's chat only while they are in the call,
and nothing of the meeting (chat, recording, transcript, summary) is theirs
afterwards. If they later join the workspace, they are in the chat. A
personal meeting is different: everyone it invites is in its chat. Files
put up in the meeting chat during the call by a guest or an outsider are
the organiser's from the moment they land. There is no tool for the
meeting chat itself (meetings are not on MCP).

**Inviting people sends email, every time.** `people` is a list of
`{"user_id": ...}` (members of the event's workspace - `list_workspace_members`)
and `{"email": ...}` (anyone). Each new person gets an invitation with a
calendar file and Yes / Maybe / No links, and a member a bell entry too. So:

* **Invite only who the person named.** Never "everyone in the workspace"
  because it seemed helpful.
* **`people` REPLACES the whole list.** To add one person, read `people` from
  `get_event` and pass all of them plus the new one. Leaving someone out takes
  them off the event (and its meeting's chat, if any) and emails them a
  cancellation.
* **A new time emails everyone an update and resets their replies.** A new
  title does not email anyone.
* **Turning `online_meeting` off keeps everyone invited**: the Join link and
  the chat go (the chat keeps what was said), and each invitee is emailed an
  update without the Join link. Turning it on for an event that already has
  people emails them an update with the Join link.
* **At most 100 new invitations a day per account.** Past that the event still
  saves and the response carries `people_refused` with the sentence to pass on
  ("You can invite more people tomorrow"). Report it. Do not retry.
* The list is one list: with an online meeting, being on it is being in the
  meeting's Collab chat. In Collab, adding someone to that chat invites them
  the same way.

`reply_to_event(event_id, "yes" | "maybe" | "no")` answers an event the person
is invited to, with or without an online meeting. `get_event`'s `my_reply` says what they answered (`organiser`
for their own event, null when they are not invited). Answer only what the
person said.

Two things this does not always reach: a **personal** calendar's meetings,
which a connected app reaches only when the person gave it their personal space
(. Otherwise they are made in the app), and a **booking's**
event, whose meeting is arranged on its booking page.

---

## 5. Choosing between the two

This is the judgement an agent is actually asked for, and the schema cannot help:

* **A deadline** → a task with `due_at`.
* **An appointment with other people, at a time** → an event.
* **A block of focus time** → an event, not a task with a span. The span records
  intent. Only an event occupies the calendar.
* **A recurring obligation** → an event with an `rrule`. One date can later be
  moved or deleted on its own (section 2).
* **An appointment with people** → an event with `people`, plus
  `online_meeting` when they meet online - and confirm who before you send the
  invitations.

When it is genuinely ambiguous, ask. "Remind me about the invoice on Friday" is
a task. "Invoice review Friday 3pm" is an event. And guessing wrong puts the
thing where the person will not look for it.

---

## 6. Tags and search

`tags` on a task are **the same labels the rest of the product uses** - shared
with files, tables and forms, not a per-surface vocabulary. So a tag you invent
here shows up in search alongside everything else carrying it, which is an
argument for using the person's existing tags rather than minting new ones.

---

## 7. Trash, and what a connected app may do

`delete_task` / `delete_event` trash. `restore_*` brings back (a repeating
event comes back with the dates changed on their own that were trashed with
it). `purge_*`
destroys and **is not offered over a connected app**. Trash is the reversible
one and it is the only one you should reach for on somebody's behalf.

`get_calendar` is the read that resolves recurrence into actual dates - use it
rather than expanding an `rrule` yourself, because what it returns is what the
person sees.

## 8. Sync conflicts: read them, never resolve them

`list_calendar_sync_conflicts` shows where a
connected Google or Outlook calendar and the local copy disagree: the two
versions side by side, and `event_id`, the local event, so `get_event` opens
it. `event_id` is null once that event was deleted for good. The conflict is
still there to settle. Resolving one is the person's choice in the web app:
tell them which event it is and what differs, and leave the choice of which
version wins to them.

## 9. The day: `get_daily_brief` and `prep_meeting`

The same day Flowpy shows when the
person says "Brief me", and the same prep note, without Ask AI.

`get_daily_brief` reads tasks due today or late, today's meetings, unread mail
that matters (mailing lists left out) and forms with new responses. "New"
means since the person last opened the brief in this space (`since`). The
first time it is the last 24 hours. Reading it **moves that time**, so call it
when the person asks for their day, not to check something in passing: a
second call straight after shows no new responses. `mail` is `null` when this
connection was not given the personal space, which is not the same as an
empty inbox: say you could not read their mail, never that it is empty.

Answer in a line or two (the count of each and anything late), and offer to
open one. Never read the whole list back.

`prep_meeting` writes a NEW note every call, so prepare only when asked and
once. With no `event_id` or `start` it takes the next meeting in the next
seven days. The note holds the person's mail, so it is saved in their own
drive, never a Shared drive: `saved_in` is `my_drive`, or `personal` when the
workspace has no My drive yet and it went to their personal space. Tell them
which, because in the second case they will not find it in the workspace.
Over a connection without the personal space that save is refused.

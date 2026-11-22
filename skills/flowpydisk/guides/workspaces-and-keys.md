# Workspaces, members and keys

Nineteen tools, and almost every question on this surface reduces to one: **what
is the credential you are holding, and what can it see?**

---

## 1. Four credentials, and what each is blind to

| Credential | Reaches | Cannot see |
|---|---|---|
| **Workspace key** | exactly one workspace, and in it the **shared** content only: the Shared drive, and in Collab the workspace's channels your person is in | your person's My drive, DMs, group chats and meeting chats there (unless the key includes personal), every other member's, the personal space, every other workspace, and everything account-wide |
| **Workspace key that includes personal** | the same, plus your person's own My drive, DMs, group chats and meeting chats in that workspace (`includes_personal: true`) | every other member's My drive and chats, the personal space, every other workspace, and everything account-wide |
| **Personal-space key** | the personal space only | every workspace, and everything account-wide |
| **A connected app** (OAuth) | the spaces the person gave it: workspaces, and the personal space if they included it, each at read or read and write | every space it was not given, the NEVER tier, and everything account-wide |

The rule to carry: **a key belongs to one space and can never reach another
space or the account.** A request that names no workspace acts in the key's own
space. A `Workspace-Id` header or `?ws=` naming another workspace is ignored in
favour of the key's. A key is minted from Settings → Developer **while in that
space** (switched into the workspace, or in the personal space). The only
change a key takes afterwards is its person switching "Include my personal
area" on or off in Settings (`includes_personal`), which keeps
its token. No key can change it, not even its own. Older keys made in the personal space work too,
confined to the personal space the same way.

So the account-wide calls - the data export, the usage breakdown and its two
ledgers, `get_usage` with `scope=account`, the workspaces overview and the
deleted-workspace list, the AI backlog, creating a workspace, and minting or
listing other keys - answer **403** to every key, a personal-space key
included. They are done in the web app.

**A connected app names the personal space `"personal"`.** `list_workspaces`
lists it first, with the id `"personal"` and `kind: "personal"`, when the person
included it. Pass that id as `workspace_id` to act there (a connection with one
space needs no `workspace_id` at all). A tool that names an item by its id (`get_table(table_id)`, `read_note(file_id)` and the like) finds the space holding that item when you leave `workspace_id` out, even on a connection with several spaces. A tool that makes something new still needs `workspace_id` when there are several. Whatever a connection may do in a
granted workspace it may do there, chats and
notifications included. `get_workspace("personal")` reads its name. Members
and invites have nothing to read in the personal space, so those two refuse
`"personal"`.

With an API key, each `list_workspaces` row also
carries `pinned` (the person pinned it in their switcher) and `has_unread` (an
unread notification about it, or a chat there that waits for them. Mail is not
counted). Pinning is the person's own choice in the apps and has no tool. A key
lists one workspace at most, so the API's `search` and paging are not tool
arguments.

`get_workspace`, `list_workspace_members`, `invite_workspace_member` and the
rest answer **404** for any workspace that is not the key's own - by design.

**The invite tools are the exception**, and it is a deliberate one:
`get_invite`, `accept_invite` and `decline_invite` act on a **bare token**, and
`join_workspace_with_code` on an invite's short code, not a `workspace_id`, so
they never check membership. Any key works - and so does a
brand-new member's, before they have a membership row at all. That is the
whole point: you cannot require membership to accept an invitation to join.

---

## 2. A 404 on this surface has three readings

This is the thing to internalise before you diagnose anything:

1. The thing genuinely does not exist.
2. Your key is **scoped away from it** - it belongs to another space (every key reaches one space only).
3. You are **not a member** of that workspace. Non-members get 404 rather than
   403, on purpose: existence is not revealed to them.

**You cannot tell these apart from the response**, so never conclude "the
account does not have X" from a 404 alone. Call `list_workspaces` first - it is
the only way to discover what exists - and read what came back before
diagnosing anything else.

And read `list_workspaces`'s answer carefully:

> **A single-entry list may mean the key is narrow rather than the account
> small.**

On a connection it lists only the spaces the **app was granted**, never the
account's others, with the effective `tier` in each, and the personal space
first when it is one of them.

---

## 3. A workspace is a separate tenant

A new workspace is made in the FlowpyDisk app, with the person as Owner. There
is no tool for it (every API key belongs to one workspace, so the API
refused `create_workspace` to every caller and it was removed). The
consequence matters more than the call:

> Its files are **NOT in the personal space**. Nothing created there appears in
> `list_files` unless you are acting inside the workspace.

Which workspace a call acts in is decided by the credential and the
`Workspace-Id` header, **not per call**. So "create this in the marketing
workspace" is not something you do by passing an argument - it needs a key or a
session that acts as that workspace.

So when the person wants a new workspace, say it is made in the app
(Settings → Workspaces), then work in it with a key minted there.

### Limits: how much you can still store, and whether AI is on

`get_workspace_limits` is read
only and answers every member. Read it before a large upload or a batch of
writes, and when a write is refused with `quota_exceeded`.

- `me.storage_available_bytes` is the most you can add right now: the smallest
  of what is left under your own limit, the workspace's limit and the account's
  storage. Plan a batch against it rather than discovering the limit one
  refusal at a time.
- `me.storage_cap_bytes` null means you have no personal limit. Members invited
  later start with a default limit.
- A storage refusal names the level that is full: "You have reached your <size>
  limit in Acme", "Acme has reached its storage limit", or, when the whole
  account is full, "Not enough storage" (`level` is `member`, `workspace` or
  `pool`). Tell the person which one, because each has a different fix
  (free your own space, ask an Admin or the Owner, or the Owner upgrades).
- `ai_enabled` false means Flowpy, AI search, summaries and AI indexing are
  off in that workspace and refused with code `workspace_ai_off`. Search by
  name still works, so fall back to it. `indexing_enabled` false means new and
  changed items are findable by their words but not by meaning yet.
- Code `workspace_ai_cap_reached` means the workspace used its AI share for
  this week. It resets when the weekly cycle does (`ai.cycle_resets_at`, for
  the Owner and Admins).
- `chat_retention.days` set (30, 90, 180 or 365) means chat files there (DMs, group and meeting chats, not channels)
  older than that move to Trash, after their owner is told
  `chat_retention.warn_days` ahead. If a person asks you to keep a chat file,
  say they can save a copy to their own files in the app before then. No
  tool exempts a file, and none changes the rule.
- Limits and switches are changed only in the web app, by the Owner (and an
  Admin for a Viewer's storage limit). No tool changes them.

### The pulse: what changed, day by day

`get_workspace_pulse(days?, tz?)`
is the week pulse on the person's Home: items changed per local day in the
space you act in, oldest day first with zeros kept, and the top five other
people who changed things there (empty in the personal space). Read it for
"how busy was the team this week" or "who has been working on things", not
to list what changed: that is `list_files` sorted by `-updated_at`, or
`search_everything`.

- An item counts **on every day a person changed it**,
  once per day. A note edited on Monday and again on Wednesday is in both
  bars, so `total_changed` is changes across the days, not distinct items: say
  "12 changes this week", not "12 items".
- A collaborator is **the person who made the change**, owner of the item or
  not, and `count` is how many items they changed in the window. You can say
  "edited".
- What counts as a change: making the item, or changing its content, name,
  details or folder, a table's rows, columns or views, a board's cards or
  connections. A star, an open, a share, a trash or a restore does not, and an
  item in Trash is not counted while it is there.
- Days from before changes were recorded (the deploy of 2026-11) still follow
  the earlier rule: an item counts once, on the day it last changed, credited
  to a file's last content author or else its owner, and a table's or board's
  owner. For a window that reaches back that far, say "changed or owns" for
  those days and never read a quiet early day as "nobody worked".
- Pass `tz` as the person's zone (`whoami`) when you have it, so the days are
  theirs.

**Not here, on purpose:** the daily "Your day" brief on Home is written by a
model in the app, and you are already one, so it has no tool. Build the same
picture from `list_events`, `list_tasks`, `list_notifications` and this pulse.
The brief now tells the day as a story, and
a good one of yours does the same: when the person is busy and free (merge
meetings closer than 15 minutes, look for gaps of 45 minutes or more between
08:00 and 20:00 that are still ahead), whether meetings run back to back, the
oldest overdue task and for how many days, a note to read before the next
meeting (`search_files` on its title), and the newest mention with what it
asked. Write times in the person's zone, and never a sentence of counts alone.
The Workspace view an Owner or Admin sees on Home (members, storage, plan,
index health) is administration and has no tool either. `get_workspace_limits`
and `list_workspace_members` carry its readable parts.

### The timezone is the workspace's, and it is what times MEAN

`get_workspace` returns `timezone` as an IANA key, and it is **always usable** -
a workspace never given one answers with the deployment default rather than an
empty string. (That is the opposite of the *account* timezone in `whoami`, where
empty means the person has expressed no preference and you should **ask** -
`calendar-tasks.md` §1. Do not carry one rule across to the other.)

`set_workspace_timezone` is **Owner only**, takes an IANA key **with exact
case** - `"Asia/Kolkata"`, not `"asia/kolkata"`, which is refused - and has a
consequence worth stating before you call it:

> Changing it **does not move any meeting that already exists** - an instant is
> an instant - but it **does** change what an availability window like
> 09:00–17:00 refers to.

So a timezone change silently re-times every booking page in the workspace while
leaving every booked meeting where it was. Say that first.

---

## 4. Roles, and the ceiling on each

Three: **Owner**, **Admin**, **Viewer**. One Owner, always.

| Action | Who |
|---|---|
| Rename the workspace, set its timezone, set its storage | **Owner only** (an Admin cannot rename it or set its storage) |
| Delete / restore the workspace, list deleted ones | **Owner only** |
| Change a member's role to admin or viewer | **Owner only** |
| Invite an **Admin** | **Owner only** |
| Invite a Viewer, list pending invites, revoke an invite | Admin or Owner |
| Remove a member | Owner can remove anyone **but themself**. Admin can remove **Viewers only** |
| Leave | anyone - **except the Owner**, who must transfer ownership first |

`transfer_workspace_ownership` makes you Admin and them Owner. `update_member_role`
moves people between `admin` and `viewer` and **cannot make somebody Owner** -
ownership moves only through the transfer.

The "Owner cannot remove themself" and "Owner must transfer before leaving"
rules are the same invariant seen twice: **a workspace always needs an Owner.**
If a person asks you to remove the owner, that is the answer to give them.

---

## 5. Deleting a workspace is reversible for 30 days, in the app

`delete_workspace` soft-deletes. A purge follows 30 days later. Bringing it
back is done in the FlowpyDisk app (Settings → Workspaces, by the Owner), not
here: listing deleted workspaces and restoring one are account-wide, no key
reaches the account, and the two tools were removed.

What to tell the person before they restore, because the app restores
**whole** - files, folders, tables, members and invites exactly as they were:

> **It is never refused for storage.** If the pool no longer has room the
> workspace comes back anyway and lands over budget. New uploads pause until
> space is freed or the plan is upgraded. **Nothing stored is touched.**

Deleting a workspace frees its bytes at once, so the room may have been used
since. The app shows whether it fits before the click.

---

## 6. Storage is one pool, divided

A workspace's storage cap lives on **Manage
workspace** in the web app, and `get_workspace_limits` reads it
(`storage_cap_bytes`). There is no tool to set it: point the person to Manage
workspace.

It sets a workspace's storage cap, in bytes, Owner only. Which rules apply
depends on `caps_enforced` in `get_workspace_limits`:

* **Caps on:** Manage workspace's rules. Null removes the cap. A cap may be
  larger than what the pool has left, or below what the workspace stores: new
  uploads are then refused until there is room, and nothing is deleted.
* **Caps off:** the cap is the workspace's slice of the **account** pool, and
  two refusals each carry a clear code. `allocation_below_usage`: you cannot
  allocate less than the workspace already stores. `allocation_over_pool`: nor
  more than its usage plus the owner's remaining pool. Read the second as the
  design it is: raising one workspace's ceiling lowers what is left for the
  others, so say which pool it comes out of.

Who uses how much is the workspace owner's to see
in the app, on the workspace's Manage screen ("Storage by person": each
member's Drive and chat files, sizes only). There is no tool for it, on
purpose: if your person asks, point them there.

---

## 7. Invites: a token is a capability, and it is single-use

`invite_workspace_member(workspace_id, email, role)` - only the Owner may invite
an Admin. `list_workspace_invites` shows what is pending,
`revoke_workspace_invite` withdraws one.

`resend_workspace_invite(workspace_id, invite_id)` emails a
pending invite again and gives it a full 7 days from now. It is the same
invite and the same link, so do not revoke and re-invite to "refresh" one:
that kills the link the person already has. Once a minute per invite. A
sooner call is refused as rate limited, and repeating it will not help. An
invite whose `expires_at` has passed still lists as pending and holds no
seat: resending it brings it back if the plan has room, and is refused with
`plan_limit_reached` if the seats have filled since. The email reaches a
person outside the product, so resend when the person you act for asks, not
because an invite looks old. There is no tool that returns an invite's link:
handing one over is done in the app.

A workspace holds as many people as its owner's plan
allows (the owner included). Sending an invite counts the
members plus the pending invites. Accepting counts only the members. Over it
both are refused with `plan_limit_reached` and a message naming the plan.
Limits depend on the plan, see https://flowpydisk.com/pricing.
Revoking a pending invite frees its seat. Handing the workspace over
(`transfer_workspace_ownership`) is refused the same way when the new owner
already owns as many workspaces as their plan allows.
It is also refused when the workspace's members would
take the new owner over their plan's people across all their workspaces
(each person once, the new owner included). The message
says how many there would be. Nothing changes hands when it is refused.

On the receiving side:

* `get_invite(token)` reads `workspace_name`, `role`, `email` and `invited_by`
  **without accepting or declining** - always do this first and tell the person
  what they are being asked to join.
* **`token` is not the workspace id**, and it is **single-use**.
* `accept_invite` returns the new `workspace_id`, `workspace_name`, `role` and the workspace's `features` (`{my_drive}`).
* `join_workspace_with_code(code)` is `accept_invite`
  by the invite's short code, `ABCD-EFGH`, which an Admin reads out from the
  app. Same rules and the same answer: only the invited address, once, while it
  is pending. A code for someone else's invite is `code_unknown`, exactly like
  one that does not exist. `code_used`, `code_expired` and `code_revoked` say
  why a known one stopped. Tries are rate limited, so never guess codes. No
  tool shows or makes a code: handing over a way in is done in the app.
* **`decline_invite` is final** - the sender has to invite again. It is not a
  "not now". Do not decline on somebody's behalf without being asked in those
  words.

The token is the capability, exactly like a share link (`sharing-and-publishing.md`
§2). Treat one that appears in a document or a message with the same care.

---

## 8. Over a connected app, almost none of this exists

`list_workspaces` and `get_workspace` are
readable. On a connection `list_workspaces` lists the granted spaces, the
personal space first (id `"personal"`) when the person included it. The
personal space's own name is changed in the app, never over a connection. **Everything that changes a
workspace or its membership is not:**

    rename_workspace          invite_workspace_member
    set_workspace_timezone    resend_workspace_invite
                              revoke_workspace_invite
    delete_workspace          update_member_role
    leave_workspace           remove_workspace_member
                              transfer_workspace_ownership

Alongside them, account and credential management: **API keys are never managed
over a connection**, and neither is `download_folder`. Creating a workspace,
restoring one and the data export are not tools at all any more.

The reasoning is the same as the sharing surface: **the account pool is the
owner's to divide, and membership grants people access.**

Say where it is done - Settings → Workspaces, or Settings → Developer in the
FlowpyDisk app - and do the part you can: read the members, read the pending
invites, tell them what the change will mean.

This is the server's rule as well as the tool
list's. An app that calls the API directly with its own token gets 403 for
every one of these, before anything runs, so there is no way round it to
look for. Say where it is done instead.

---

## 9. What a key can tell you about itself

A key sees only its own row in `/auth/api-keys`, and the row carries its
`workspace` and `includes_personal`. That is the supported way to answer "what am I allowed to see", and it is worth
doing before a long job rather than inferring scope from a trail of 404s.

Its `includes_personal` says whether it
also reaches your person's own My drive, DMs and group chats in its workspace.
False means the Shared drive and the workspace's channels only.

Two more properties worth knowing about the credential you hold:

* **Scopes are `read` and `write`.** A read-only key allows safe methods only.
  Every POST/PUT/PATCH/DELETE needs `write`. Least privilege is the intent - an
  integration that only fetches should be holding a `read` key.
* **A key need not expire, deliberately.** The default is no end date, because
  this product exists to give an agent long-lived access and a key that quietly
  stops on day 91 breaks the thing it was minted for.

What is recorded about what you do with it is **deliberately coarse**: an action
(`read`/`write`) and a resource family (`files`, `tables`, `graph`). No path, no
file name, no body - a log of which documents an agent read would be a second
copy of the person's information living somewhere with different retention.

---

## 10. Webhooks: you can read them, never make them

Incoming webhooks are a separate direction: an external system signs a request
to create a row, form response, calendar event or file in one configured
Shared-drive target. `list_inbound_webhooks` shows the action, target name and
state. `list_inbound_webhook_receipts` shows accepted/refused outcomes without
request bodies, secrets or created resource IDs. A workspace Admin or Owner
sets up, pauses, rotates or deletes incoming endpoints in Settings → Developer.
An API key or connected app can only read these two metadata views when it has
that workspace's Admin/Owner authority.

A webhook is an
https address that FlowpyDisk calls when something happens in a space. Three
tools read them: `list_webhooks` for what exists and whether each is `active`,
`disabled` (its owner turned it off) or `paused` (FlowpyDisk stopped after about
three days of failed deliveries), `list_webhook_deliveries` for one webhook's
log, and `list_webhook_event_types` for every event one can carry and the shape
of its data.

A plan can have webhooks switched off (`webhooks_outbound`,
`webhooks_inbound` in `get_plan_limits`). Off, the space's webhooks are paused
by the plan: no event is queued, a delivery already queued is held rather than
dropped, incoming endpoints refuse with `plan_capability_off`, and nothing is
deleted. Everything resumes when the switch is back on. Say so in those words.
There is nothing to fix on the endpoint.

The events, by group:

* **forms and bookings:** `form.submitted`, `booking.created`.
* **tables:** `rows.created`, `rows.updated`, `rows.deleted`, `rows.restored`,
  `table.deleted`, `table.restored`, `table.purged`, `board.deleted`,
  `board.restored`, `board.purged`. A bulk change is ONE event with `count` and
  `ids`, never one per row, and deleting, restoring or purging a whole table or
  board is its own event: its rows get none. So "did my 300-row delete reach the
  CRM" is one delivery to look for, not 300. There is no `rows.purged`: a row
  delete is already permanent apart from its undo, which `rows.restored` reports.
* **Drive:** `files.added`, `files.changed`, `files.moved`, `files.deleted`
  (moved to Trash), `files.restored` (back from Trash) and `files.purged`
  (deleted forever). A batch, or a folder's contents, is one event listing every
  file.

`deleted` always means moved to Trash. `purged` means gone for good, whether a
person emptied it from Trash, deleted its folder forever, or the daily sweep took
it after the Trash window (one `files.purged` per space for each sweep). A purge
reports the item as it was just before, and it cannot be read afterwards: do not
call a read tool with those ids expecting an answer.
* **calendar:** `calendar_event.created`, `calendar_event.changed`,
  `calendar_event.cancelled`.
* **connected accounts:** `connection.mail_received` (Gmail, Outlook) and
  `connection.file_added` (Google Drive, OneDrive). Personal space only: a
  connection is one person's, so a workspace webhook never receives them.

Payloads carry ids and a few fields, never a mail body or a file's bytes. Read
the rest through the other tools with those ids. Items in a person's own My
drive inside a workspace send nothing to that workspace's webhooks.

A `disabled` webhook with `handed_over` set was made by
someone who has since left the workspace or deleted their account: it was switched off,
because it sends to an address that person chose. Say so, with the name, if
somebody asks why it is off. Turning it back on is theirs to do in the app.

**Over a connected app there is no tool to add, change, test, resend, resume or
delete one, and that is by design, not a gap.** With an API key,
`test_webhook`, `redeliver_webhook_delivery` and `resume_webhook` act on one. Adding, changing or deleting stays in the app. A webhook sends a person's form answers and
bookings to an address. Choosing that address is the same kind of decision as
making a share link. If somebody asks you to set one up, say it is done in
FlowpyDisk under Settings, Developer, and offer to check its log afterwards.

Reading the log well:

* A delivery that is `retrying` will be tried again at `next_attempt_at`. The
  backoff runs about 24 hours over eight attempts, then the delivery is
  `failed`. Nothing is lost while it retries.
* `response_status` and `response_body` are what the endpoint said. A 401 or
  403 usually means the receiver is checking the signature with an old secret.
  A 404, the wrong path. No status at all with an `error`, that nothing
  answered.
* Every attempt of one event carries the same `event_id`, and so does a
  resend, so a receiver that saw it twice can tell.

## 11. Approvals: ask before you remove

Over a connected app, **every tool that trashes,
cancels or removes something needs the approval of the person who connected
the app**: `delete_file`, `delete_folder`, `delete_rows`, `bulk_file_action`
(trashing, not restoring), `cancel_booking`, `decline_booking`,
`move_booking` and the other `delete_` tools. The exception is a person who
set this app to always allow in Settings, Developer. Then nothing changes for
you. An API key is never asked.

Flowpy, the assistant
inside the app, keeps the same line: it asks the person, with an Approve card,
before anything that deletes (moving to Trash included) or sends, in Ask AI,
in its side panels and in Collab. So the person meets one rule whichever
assistant they use.

The loop, and it is the only one:

1. Call the tool. If it is refused with `approval_required`, you need one.
   (Asking first is fine too. It saves a refused call.)
2. `request_approval` with the `tool`, **the exact arguments you will call it
   with**, and a one-line `reason` the person will read in a notification and
   an email: "Removing the 14 rows tagged duplicate", not "cleanup". The
   person also sees what your arguments name, by name (the table, the
   files), in the request's `targets`. Ids are never what they judge by.
3. **Tell the person it is waiting for them**, in so many words. They answer
   in the app or from the email, not to you.
4. `get_approval` until `state` is not `pending`. Do not ask again while it
   is pending. That is a second question about the same thing.
5. `approved`: call the tool again with the same arguments plus
   `approval_id`. `denied`: stop, and do not try it again without asking
   them. `expired`: ask again if it is still wanted.

MCP forwards the approval id to the REST mutation, where it is consumed once.
Do not consume it separately. If you call REST directly, use the `Approval-Id`
header on operations marked with that parameter in OpenAPI. Mail sends and
conditional select-choice removals use the JSON `approval_id` field. The server
derives the operation from the actual request, so a client-provided tool label
cannot authorize a different write.

What one approval covers, exactly:

* **One call.** After it runs the approval is `consumed`. The next delete
  needs its own. For many items, ask once for a bulk tool
  (`bulk_file_action`, `delete_rows`), not once per item.
* **Exactly those arguments**, in that workspace. A different id, a different
  list, or another workspace is refused with `approval_mismatch`. Argument
  order does not matter, an argument left out equals one passed as null, and
  documented optional defaults compare equally whether omitted or explicit.
* **Its time.** An hour by default. The person can make it 15 minutes, four
  hours or a day. Unused, it expires.

**One write is asked about too, and only sometimes:** `update_field` removing
a select choice that rows still hold, which clears those cells. Whether rows
hold it is the server's to know, so the first call is refused with
`approval_required` and its text gives the exact arguments to ask with (the
column and the choices, by name). The loop is the same. Pass `approval_id` to
`update_field` the second time.

You cannot answer your own request, and there is no tool or endpoint that
would let you. Nor can you change how this app is approved: the policy is the
person's, in Settings, Developer.

---

## 12. Where to start, every time

1. `whoami` - who this credential is, and the account timezone.
2. `list_workspaces` - what it can see, and your role in each. If this is
   short, suspect the key before you suspect the account. On a connection the
   personal space is the entry with the id `"personal"`.
3. `get_workspace` for the one you are working in - the **workspace** timezone
   is what its calendars and booking pages mean.
4. Only then act. And if something 404s, re-read §2 before telling anybody it
   does not exist.

## 13. Working in a space, step by step

1. **Connect.** Call `whoami`. Over a connected app it returns the spaces the
   app was granted, one marked `default: true`, and `instructions`. Follow them.
2. **Ask the person which space.** Do not assume the default. A call with no
   `workspace_id` runs in the default space and says so in a notice.
3. **Read what the space allows.** `get_space_capabilities` gives the plan,
   whether AI is on, whether meaning search is allowed (`semantic_search`), the
   plan caps and the rate limits. Check it before an AI or meaning search call.
4. **Search first.** Use `search` (one space, words, content and graph, meaning
   when asked and allowed). `search_everything` stays for a plain name, tag and
   description match.
5. **Pick or ask.** One clear match, use it. Several, ask the person.
6. **Work by id, report by name.** Pass ids to tools. Tell the person names,
   and the space by name.
7. **Switch spaces out loud.** Each call acts in one space. Say when you
   switch. To move data, read it in one space, then write it in the other.

### When a call is refused

The error carries `why` (what happened), `next` (what to do) and `code`.
Do what `next` says. A rate limit names the limit and says
"Wait N seconds, then retry." with `retry_after`. Wait, do not loop.
A reply can also carry `notices`, for example `default_space_used`: the call
ran in the default space because none was named. Tell the person.

### Lighter reads and writes

- `get_table_rows` takes `fields`: only the columns you need.
- Row writes return a compact summary by default. Pass `return_rows=True` only
  when you need every row back.

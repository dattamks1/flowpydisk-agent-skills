# Booking over MCP - the agent playbook

Give this to any agent that will touch booking through the FlowpyDisk connector.
The tools are easy. **The judgement is not**, because every write on this surface
lands on a person who has no account here, cannot undo it, and is not in the
conversation.

---

## 1. The one thing to understand first

A booking page is the only part of FlowpyDisk where **a stranger's time is the
data**. Everywhere else you are editing the account holder's files. Here, four
of the sixteen tools send an email to somebody who never asked to hear from you,
and one of them removes a meeting from their diary.

So the rule that governs this whole guide:

> **Never answer a booking the person has not asked you to answer.** "Tidy up my
> calendar" is not permission to decline three requests. Read, summarise, and
> ask which ones.

All four writes carry `destructiveHint`, so a client may prompt before them.
Declining, cancelling and moving also need the person's approval in the app.
Accepting does not. **Do not treat the absence of a prompt as permission** - a client that
does not implement annotations shows none, and the API never sees them at all.

## 2. The shape of the thing

    BookingPage          the public page, at /book/<slug>
      AvailabilityWindow  recurring weekly hours: "Mon 09:00-17:00"
      Booking             one person's slot on it

A **slot** is not stored. It is computed, every time, from the page's windows
minus everything that makes the host busy. So there is no "create a slot" tool
and never will be: you widen the windows, or you free the time.

A booking is in one of four states, and they are not interchangeable:

| | What it means | How it ends |
| --- | --- | --- |
| `pending` | A request. **Holds nothing.** | `approve_booking` or `decline_booking` |
| `confirmed` | A meeting. In the host's calendar, invitation sent. | `cancel_booking`, or the invitee's own link |
| `declined` | The host said no to a request. | terminal |
| `cancelled` | A meeting was called off. | terminal |

**`declined` and `cancelled` are different on purpose.** Declined never became a
meeting. Cancelled did. The person receives a different message for each, and
using the wrong tool tells them something untrue about their own week.

## 3. The tools

| Do this | Tool | Notes |
| --- | --- | --- |
| List pages | `list_booking_pages(trashed?)` | |
| One page | `get_booking_page(page_id)` | Everything below is set here |
| Create / edit | `create_booking_page(...)`, `update_booking_page(...)` | See §5 for the settings that matter |
| Copy a page | `duplicate_booking_page(page_id, title?)` | Settings, hours, hosts and look. **Never the bookings**. A new slug, "Copy of ..." as the title, and always **paused**: turning it on is `update_booking_page(is_active=True)`, and only when the person asks, since it puts a second live link out |
| Hours | `list_availability_windows`, `create_availability_window`, `update_availability_window`, `delete_availability_window` | `weekday` is 0=Monday, and the times are in the PAGE's zone, not yours. **Use `update_` to change a block** - it was delete-and-add until BR24a, and the end is now checked against the start already stored, so "move the end to 08:00" is refused rather than saving a block that ends before it begins |
| What is free | `get_booking_slots(page_id, start, end)` | The same computation an invitee sees |
| Who booked | `list_bookings(page_id, status?, start?, end?)` | Newest first, capped at 500 |
| Accept a request | `approve_booking(page_id, booking_id)` | Sends the invitation. **409 is normal** - see §4 |
| Turn one down | `decline_booking(page_id, booking_id)` | **destructive.** Pending only |
| Call off a meeting | `cancel_booking(page_id, booking_id, reason?, notify?)` | **destructive.** Confirmed only |
| Reschedule | `move_booking(page_id, booking_id, start_at)` | **destructive.** Confirmed only |
| Trash / restore a page | `delete_booking_page`, `restore_booking_page` | The public link stops resolving |

**Not reachable over a connector, ever:** booking a slot as an invitee, and
cancelling with an invitee's token. Those belong to the person holding the link,
and an agent standing in for them is an agent booking somebody's time under an
email address it does not own.

## 4. Rules that bite

Each of these has caused a wrong action before.

**`approve_booking` needs you to be a HOST of the page, not just able to edit
it.** Accepting names you as the booking's host, so a member who may
edit the page but is not on its host list gets a 403 telling them to decline
instead. The two permissions are deliberately separate: your workspace role says
whether you may edit, the host list says whether a booking may be yours. The
same call also TAKES a confirmed booking that belongs to nobody - that is what a
page which confirms on the spot without assigning produces.

**A 409 from `approve_booking` is a normal outcome, not an error to retry.** A
pending request holds nothing, so the time can be confirmed for somebody else
while the request waits. Retrying will fail identically forever. The recovery is
a decision, not a repeat: decline it, or `move_booking` it to a time that is
free. Say which you are doing.

**Read `approval_mode` before you promise anything.** On `auto` an invitee's
booking is already a meeting. On `manual` it is a request, nothing is on a
calendar, and the slot is *still offered to other people* - so two people can
hold a pending request for the same time and exactly one can be accepted. Never
tell somebody a request is "booked".

**`host_identity` publishes something about a person, so it is never yours to
pick.** It decides which of three things a stranger is told they are booking
with - the host's name (the default), their email address, or the brand. Set it
only when the person has said which one they want, in those terms. "Brand" is
not a safe default to reach for on their behalf, because on a personal page the
brand IS their own name. Changing it on an existing page changes what every
invitee opening that link reads from that moment on. A page with several hosts
ignores it and always says the brand.

**`notify=false` on a cancel is a deliberate act, not a convenience.** It removes
the meeting and tells the invitee nothing. It stays in their diary and they turn
up. Only pass it when the person explicitly said to cancel quietly. Default
behaviour - `notify` omitted - emails them a proper cancellation that also
removes the entry from their calendar.

**`reason` is quoted to the invitee verbatim.** Write it as something a stranger
reads, not as a note to self. "conflict" is not a sentence.

**`move_booking` takes a time the page would have OFFERED.** Free in every
calendar the page watches, inside its windows, nobody else's. Call
`get_booking_slots` and pick from what comes back. Anything else is a 409. The
end time is computed from the page's slot length and is not yours to set.

**`update_event` on a booking's calendar row is the SAME move.** An event whose `booking_id` is set IS a
confirmed booking, so sending `start_at` moves the booking and emails the
invitee. The same availability check applies and the same 409 comes back.
`end_at` is ignored there for the reason above. So there is no way to move the
row without moving the meeting, and no way to do it quietly - which is the
point. `delete_event` on the same row cancels the booking, and that one cannot
be undone.

**A page has a WINDOW, and it refuses outside it with a 400.** Every
page carries `booking_horizon_days` (default 60) and `minimum_notice_minutes`
(default 240). A time beyond either is refused at `/book` with a sentence
saying which rule it broke - that is not the same as the 409 that means "taken"
and must not be retried as one. `get_booking_slots` already obeys the window,
so a time you picked from it is inside. `0` on either field means no limit, and
the past is never offered whatever they say.

**Times are ISO-8601 with an explicit offset.** `"2026-09-14T09:00:00+00:00"`.
And when you build a query string by hand, **encode it** - a bare `+00:00`
arrives at the server as a space, parses to nothing and earns a 400. This trap
has now been hit twice by people who knew about it.

**`weekday` is 0=Monday**, matching Python, not Sunday-first as several calendar
APIs use. A window created a day out is a page open on the wrong day all week.

## 5. Settings worth understanding before you change them

`update_booking_page` takes these, and three of them have consequences a caller
would not guess.

**`taken_slots_display`** - `"hide"` (default) or `"grey"`. Grey shows a busy
time in the grid, marked as gone. That means **anyone holding the link can read a
week of the host's working life without booking anything.** Hidden is the default
for exactly that reason. Only set `"grey"` if the person asked for it. If they
asked for "show when I'm busy", repeat back what it publishes.

**`busy_workspace_ids` / `busy_includes_personal`** - EXTRA calendars that make
this page busy. The page's own space always counts, so these only ever *narrow*
what is offered. Replaces the whole list: pass `[]` to clear it.

**`target_workspace_id`** - the ONE calendar a confirmed meeting is written
into. `null` means the page's own space. It is not a list and there is no way to
ask for all of them. Setting a target does **not** add that calendar to the busy
set: "what decides availability" and "where it lands" are different questions.

**`online_meeting`** - on by default: each confirmed booking's calendar event
is a video meeting, and the customer gets its Join link in the confirmation.
Turn it off for a page that meets in person. Either way the customer, and on a
group page the assigned host, are on the event. Nobody else can be invited to
it, and the booking's own emails, not meeting invitations, tell them about it.

**`email_intro`** - a line the host writes, added to the emails your invitees receive
about this page, not on the page itself and not in the notices the hosts get
Set it with `create_booking_page` and change it later with
`update_booking_page` (`""` clears it), up to 500 characters. The confirmation message is the page. If somebody
asks you to "change what clients see", find out which they mean.

**Every id has to be a workspace the account belongs to.** A workspace that
exists but is not theirs returns the same error as a made-up one, on purpose.
Over a connected app or an API key, the only workspace a page may name (as
where meetings go, or as a busy calendar) is the one the app or key acts in.
And where meetings go needs the person to be an Admin or Owner there.

## 6. Recipes

**"Who is waiting on me?"**

    list_bookings(page_id, status="pending")

Summarise them. Do not answer any of them.

**"Accept Dee's request for Thursday."**

    list_bookings(page_id, status="pending")   # find the id
    approve_booking(page_id, booking_id)
    # 409 -> the time went. Offer: decline, or move to a free slot.

**"I'm ill on Tuesday, move everything."**

    list_bookings(page_id, start=..., end=...)          # confirmed, that day
    get_booking_slots(page_id, start=..., end=...)      # what is actually free
    # Then ONE move at a time, and tell the person which invitee went where.
    # Do not batch-move silently: each one emails somebody.

**"Stop taking bookings for a fortnight."**

Not a cancel. `update_booking_page(page_id, is_active=False)` takes the page
offline without touching a single existing meeting, and turning it back on
restores the same link.

## 7. What a person reads

Worth knowing, because it shapes what you should say you have done. Every email
this surface sends carries the host's name and address, the disclaimer that
FlowpyDisk is not responsible for the arrangement, and a powered-by line. An
accepted request carries a calendar invitation. A declined one carries nothing
and is worded as *"that time is not available after all"* rather than as a
rejection. A cancellation and a move both carry a calendar update, so the entry
in the invitee's diary actually changes rather than going stale.
With `online_meeting` on, the confirmation and its calendar file carry the Join
link for the video meeting.

None of that is yours to configure per action - it is the page's, and it is the
same for every message.

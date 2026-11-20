# Templates over MCP - the agent playbook

Give this to any agent that will apply, save or version templates through the
connector. The tools are small. The judgement is what an apply builds, where
it lands, and the two kinds that are live to strangers when it returns.

---

## 1. The one thing to understand first

**A template is a SNAPSHOT, never a reference.** Saving one copies what the
thing is right now into the template. Applying one copies the template into
the account as ordinary, independent things: a table, a board, a note, a
booking page, a form. Nothing links the two afterwards, in either direction: editing
or deleting the source does not reach the template, and cutting a version,
renaming or deleting the template does not touch anything built from it.
**A copy keeps the version it took.**

Asked to "update the template so everyone's tables change", say plainly that
it does not work that way: the next person to apply it gets the new version,
and nobody's existing copy moves.

---

## 2. Two kinds of template, one way to use them

`list_templates` returns the built-in catalogue first, then the saved
templates of the space you are working in.

| | Built-in | Saved |
| --- | --- | --- |
| `builtin` | `true` | `false` |
| `id` | `builtin:<slug>` | a UUID |
| Who made it | FlowpyDisk. Code, not a row, so it changes when the product ships | A person, with `save_as_template` or `create_template`. `owner_name` names them (a display name or `@username`, never an email) and `is_mine` says whether it was you |
| Kinds | table, note, canvas (a board), form, booking, site | note, table, canvas, booking, form. A site cannot be saved yet |
| Content | not in the listing | `body` for a note, `payload` for every other kind, never both |
| Versions | none | `current_version`, starting at 1 |
| Renamed, versioned, deleted by you | never | yes, when it is yours or you are an Admin of the workspace. A Viewer only reads and applies |

**`apply_template` builds either.** The other tools take a UUID only. A
`builtin:` id is refused before any request is made. A saved template's
`visibility` is `private` (the default), `workspace` or `discoverable` (on
the marketplace), and is read-only to you (§7).

**"Only me" is the person's alone**. In a
workspace the listing holds the person's own templates of every visibility,
and other members' `workspace` and `discoverable` ones. A colleague's
`private` template is not listed and answers not-found if named, whatever the
person's role.

Other people's published templates are on `browse_template_marketplace`
(see `sharing-and-publishing.md`). Its ids go to the same `apply_template`:
applying IS how you take one, and there is no download.

---

## 3. The tools

| Do this | Tool | Tier over a connection |
| --- | --- | --- |
| See what exists | `list_templates()` | read |
| A saved template's history | `list_template_versions(template_id)` | read |
| Build one into the account | `apply_template(template_id, timezone?)` | read and write |
| Save something you have | `save_as_template(source_kind, source_id, name?, include_data?, relations?)` | read and write |
| Save a note template from text | `create_template(name, body)` | read and write |
| Cut the next version | `cut_template_version(template_id, source_kind, source_id, include_data?, relations?, changelog?)` | read and write |
| Rename, or change the description | `update_template(template_id, name?, description?)` | read and write |
| Delete for good | `delete_template(template_id)` | read and write, **asks the person first** |

In a workspace every write here needs the Admin or Owner role. A Viewer can
only read.

---

## 4. What an apply builds, and where

It builds into the space you are working in and returns `{kind, id, name}`,
plus `slug` and `folder` for some kinds. Use `id` to go straight to what was
made rather than searching for it.

| Template | What you get | Where |
| --- | --- | --- |
| Built-in table | One table (or two, joined by a relation) with its fields, views and sample rows | Not in a folder. Pictures, if it has any, go in a new folder of their own |
| Saved table | The table as it was saved, with rows only if it was saved with them | Not in a folder |
| Built-in note | A SET of linked notes | A new folder of their own. `id` is the note to open first, `folder` is the folder |
| Saved note | ONE note, named after the template, holding its text | At the root. No folder |
| Canvas | A board, its cards where they were, and its connections. Cards in a section stay in it (they move with it), lines, arrows and drawings keep their shape, and a bookmark arrives ready with its title. Saving a board as a template keeps all three | Not in a folder |
| Booking | A configured booking page and its weekly hours. `slug` comes back | A booking page has no folder |
| Form | A form with its questions and look, CLOSED, with an empty responses table of its own. `slug` comes back, with `published: false`. Four are built in, category `Forms`: `builtin:event-registration`, `builtin:customer-feedback`, `builtin:lead-capture`, `builtin:job-application` | A form has no folder |
| Site (built-in only) | A folder of notes, a site made of them, published. Some sites also build a booking page. `slug` and `folder` come back | A new folder of its own |

Applying twice builds two of everything. Nothing checks for an earlier copy.

---

## 5. Two applies are live to strangers when they return

**Over a connected app, neither goes
live.** A site template is built as notes in their own folder and not
published (`slug` is null, `published: false`), and a booking template is built
closed, so nobody can book it. Tell the person it is ready and that they
publish or open it in the FlowpyDisk app. What follows is what an **API key**
gets.

**A site template publishes.** With a key it is the one place where "apply"
also means "publish": the site is reachable by anyone at `/n/<slug>` as soon
as the call returns, with no account needed. Its pages are ordinary notes in
the new folder. So never apply one to "have a look": describe it from its
`description` and `outcome`, and apply only when the person has said they
want a public site.

**A booking template is a working page.** With a key it is active the moment it exists,
at `/book/<slug>`, and confirms bookings on the spot: approval, the booking
window and the busy calendars are not carried by a template, so the new page
starts at the defaults. If the person wants to approve requests first, or to
keep the page offline until they are ready, change that with
`update_booking_page` straight after (see `booking.md`).

**The slug is not yours to choose.** It comes from the template, and when it
is already taken anywhere on the platform a number is added (`portfolio-2`).
Read it from the result. Do not tell the person the address before you have.

**Pass `timezone`.** A booking or site template describes a shape of week,
"three mornings", and the hours are stored in the zone you give. Use the
person's own IANA zone from `whoami`, and ask if it is empty. Omitted, or
spelled in a way the server does not recognise, it silently becomes UTC, and
a mornings-only page is open at other hours for most of the world. A table,
note or board ignores it.

---

## 6. Saving and versions

**`save_as_template` snapshots a real thing**: `source_kind` is `note`,
`table`, `canvas`, `booking` or `form`, and `source_id` is its id. A note must be a
`.md` file. Something trashed, somebody else's, or a malformed id all come
back as the same 404. The name defaults to the source's own (a note loses its
`.md`). The new template is `private` and is version 1.

**`create_template` is for a note template written from scratch** in
Markdown. For anything that already exists, use `save_as_template`.

**Leave `include_data` off unless the person asked for the rows.** It only
affects tables. The columns are what people reuse. The rows are usually what
nobody meant to share, and once the template is published the rows go to
everyone who takes it.

**A table with a link to another table is refused by default**, and the
refusal names the column. The other table is not in the template, so the link
cannot come with it. `relations="text"` brings the column across as plain
text holding what the link showed. It is no longer a link, in the template or
in anything built from it. That is a loss, so ask before you pass it.

**A booking template keeps** the title, slot length, buffer, the look and
the weekly hours. Never the bookings (other people's appointments) and never
the timezone.

**A form template keeps** the title, description, the look (accent, logo,
cover, layout), the thank-you message, footer and links, and every question
with its kind, label, help, options, required flag, width, validation and
show-if rule (pointed at the new form's own questions on apply). Never the
responses, the address, the open state, the opening and closing dates, the
response limit, the spam and one-response settings, or the tags.
**A form built from a template is always closed**, from a key or a connected
app alike: tell the person it is ready and that they open it from its Share
panel in the app, which no tool does (see `forms.md`).

**Over 1 MB is refused** with `template_too_large`. In practice that is a big
table with `include_data`. Save the structure instead.

**`cut_template_version` is a fresh snapshot, not an edit.** It takes the
same source arguments as `save_as_template`, plus a `changelog`: one line,
written for the people who will take it, saying what changed. It must be the
same kind as the template (`wrong_kind` otherwise). After it, `apply_template`
builds the new version, and **every earlier copy keeps the version it took**.
Version numbers are never reused. To change a note template's text, put the
new text in a note and cut a version from that note.

**`apply_template` always builds the LATEST version**. No tool builds an
older one. `list_template_versions` gives numbers, changelogs and dates,
newest first, never the content. **`update_template` changes the name and the
description only**, and both belong to the template, not to a version. The
description is what the marketplace card says the template is (at most 500
characters). A version's changelog says what changed and stays on the
versions list.

---

## 7. What is not here

**Publishing a template has no tool and never will.** Changing `visibility`,
to the workspace or to the marketplace, is a person's decision in the
FlowpyDisk web app, like every other make-public action, and the API refuses a
connected app even if it finds the endpoint. If the person asks you to
publish one, save it (private) and tell them to publish it from the template
gallery. Do not call this a gap in the connector.

Rating is not a tool either. On the shelf, `rating` stays empty until a
template has three ratings. `rating_count` shows either way.

---

## 8. Deleting is permanent

`delete_template` does not go to Trash and there is no restore. Every version
goes with it. What was already built from it is untouched, because a copy has
no parent. If it was `workspace`, everyone in the workspace stops seeing it.
If it was `discoverable`, it comes off the marketplace and nobody can take it
any more.

**Over a connected app it is not offered at all**:
it is a permanent delete, so it sits with the purges. Tell the person it is
done in the app, or with an API key. With a key, say before you call it that
it cannot be undone and, from its `visibility`, who stops seeing it.

---

## 9. Recipes

**"Set me up a bug tracker."** `list_templates()`, pick the built-in by its
`description`, `apply_template("builtin:<slug>")`. Tell the person the
table's name, and that the rows in it are samples.

**"I changed the intake table. Update the template."**

    cut_template_version(template_id, "table", table_id,
                         changelog="Adds a Budget column")

Then say what that means: people who already used it keep their old copy.

**"Put my portfolio online."** Confirm they want it public, then
`apply_template("builtin:<slug>", timezone=<from whoami>)` and report the
address from the returned `slug`.

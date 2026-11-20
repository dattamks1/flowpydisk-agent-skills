# Sharing, publishing and the marketplace

Eighteen tools, and the surface where **most of the writes are deliberately not
yours**.

---

## 1. Read this first: what a connected app cannot do here

Over a **connected app** - claude.ai or any other client that signs in through
OAuth - every action below is **absent from the tool list on purpose**:

    set_file_discoverable        create_share_link
    publish_note                 create_folder_share_link
    publish_table                set_share_link_expiry
    publish_canvas               revoke_share_link
    publish_gallery              share_file_with_person
    (and each unpublish)         share_folder_with_person
    set_form_active              revoke_file_permission
    delete_site                  create/update/revoke_short_link
    purge_* (permanent delete)   update_embed_settings

The rule behind the list is one sentence: **anything that makes content public
or grants access to another person is a person's decision in the FlowpyDisk
app.** It is not a gap in the connector and it is not something to work around.

So when somebody asks you to "share this with Priya" or "put this online", the
right answer is: *I can't do that from here - it's done in the FlowpyDisk app,
under Share on that file* - and then be useful about the part you **can** do:
find the file, confirm what it contains, check what is already shared.

**Publishing a template has no tool at all**, and the scope map names the
absence on purpose so the next person finds the reason rather than an empty
space: `POST /storage/templates/{id}/publish` exists on the API and is
deliberately not exposed.

**Two reads are also off a connection**, which surprises people more than the
writes: `list_shared_with_me` and `list_shared_by_me`. They cross tenants by
design - a file another account shared with you is in none of your workspaces,
so it cannot be scoped to the one a connector acts in. They are API-key-only.
In `list_shared_with_me`, who shared each item is `shared_by_name`, a display
name or "@username", never an email.

What **does** stay readable on a connection: `list_share_links`,
`get_share_link_access`, `list_file_permissions`, `list_folder_permissions`,
`list_canvas_permissions`, `list_table_permissions` and both publish-status tools - so you can always tell a person what is *already*
public. That is deliberate: reading the exposure is how you help, and changing
it is not.

**A public image link has no tool, with any key**. An image file (PNG, JPEG or GIF, at most 2 MB) can carry a
permanent public link to its bytes, for a logo in an email signature.
`get_file` reads it as `public_image_url` (null when off). Turning it on or off
is done under **Share** on the file in the app. Say so, and say that turning it
off, or trashing the file, removes the image from mail already sent.

---

## 2. With an API key: link, or person?

Two mechanisms, and choosing wrong is the mistake this section exists to stop.

| | `create_share_link` / `create_folder_share_link` | `share_file_with_person` / `share_folder_with_person` |
|---|---|---|
| Who it works for | **anyone holding the URL** | one person, by email |
| Withdrawn how | revoke the link - for everyone at once | revoke that person alone |
| Needs an account | no | not yet. An unknown address answers exactly like a known one: "If they have a FlowpyDisk account, they now have access. If not, they will as soon as they sign up with this address." It is **held** and becomes a grant when somebody signs up with it. No email is sent, so tell the person to let them know |
| Roles | none - it is a read capability | `viewer` or `editor` |

> **Prefer the named grant when you know who it is for.** A link is a capability
> handed to people the owner cannot see.

Sharing again with the same person **updates their role** rather than adding a
second grant. Only the owner can grant: somebody given access cannot pass it on,
or revoking them would leave their invitees behind.

### A board shared with a person

A board can be shared with one person by email, at
viewer or editor, exactly as a file is, beside publishing it. **There is no tool
for it, with a key or a connection**: the person shares a board in the web app,
from the board's Share. What you can do is read it: `list_canvas_permissions`
lists who has the board (held shares for addresses with no account included),
and `revoke_file_permission` withdraws one with a key, using the row's `id`.
People who reach the board through a shared folder are listed on the folder.

A viewer reads the board and may copy it. An editor also changes its cards,
connections and sections. Neither may delete, publish or re-share it. The
files on its cards **show through the board only**: the person sees them on the
cards, view-only, as a visitor would see a published board, but the files are
not shared with them, so the file tools, search and their Drive never reach
them. A board reached through a shared folder shows its card files the same
way. Revoking the board grant or the folder grant,
trashing the folder, or moving the board out of it closes them at once. In
`list_shared_with_me`
a shared board is a row with `kind: "canvas"`, opened with `get_canvas`.

### A table shared with a person

A table is shared the same way, from the table's
Share in the web app, and **no tool makes the share**. `list_table_permissions`
lists who has it. `revoke_file_permission` withdraws one with a key. A viewer
reads and exports it. An editor changes its rows, cells, columns and views
(`can_edit_structure` is true on `get_table`), and nobody but the owner deletes, publishes or re-shares it. Its
cell files show through the table only, and the tables it links to are not
shared with it. In `list_shared_with_me` it is a row with `kind: "table"`,
opened with `get_table`. The detail is in `tables.md`.

### Set an expiry. Almost always.

A link created without one outlives the reason it was made. `expires_at` is
ISO-8601. Maximum lifetime is one year, enforced at creation and on
`set_share_link_expiry`.

`set_share_link_expiry` changes a live link **in place**, so copies already sent
out keep working until the expiry rather than breaking now - which is how you
put a bound on a link that was made without one. A **past** datetime is rejected
rather than accepted, because it would be indistinguishable from revoking except
that nothing would say so. To make a link permanent again pass
`never_expires=True`. A call naming neither a date nor that is refused, so a
link never becomes permanent by omission.

`revoke_share_link` is permanent: the token stops resolving for everyone holding
it, and a later `create_share_link` mints a **new** one rather than bringing it
back.

A workspace revokes shares on its own too: when the
member who made a link or a people grant there is moved to Viewer, removed,
leaves, or deletes their account, every link and grant they made in that
workspace is revoked, for good. A link that stopped working for that reason
reads as revoked. A member who can share makes a new one.

### A plain call returns the EXISTING link

No password and no expiry on a file or folder that already has an active link
gives you **that link back**, not a second one. This is deliberate: a file with
several live URLs is one nobody can reliably un-share, so revoking the one you
were handed would leave the others working. Pass a password or an expiry when
you specifically need a different link.

---

## 3. A folder share is live, and that is the thing to say out loud

Both folder mechanisms - the link and the named grant - hand over **whatever is
in the folder when somebody looks**, not what was in it when the share was made.

> **Moving a file into a shared folder publishes that file to whoever holds the
> link, with no further action and no second prompt.**

A named folder grant behaves the same way: it is inherited and resolved on every
read, so a file moved **in** becomes readable by them immediately and a file
moved **out** stops being.

This is the single most useful warning on this surface, because an agent tidying
files is exactly the actor who trips it. **Before you file anything into a
folder, check whether it is shared** (`list_folder_permissions` and
`list_share_links`) and say so.

Two more folder-link differences from a file link:

* It **expires in 30 days by default**. `never_expires=True` opts out, and you
  rarely should: an unbounded live link keeps handing over the folder as it is
  next year.
* What the invitee gets is a **handover, not a drive** - the root level listed,
  a total size, and one button that downloads the whole subtree as a zip.
  Nothing opens, nothing previews, no single file can be picked out. If somebody
  needs one file, share that file.
* In a workspace it hands over **everything in the folder, whoever put it
  there**, less trashed and pending files and any private channel's files its
  maker could not see. In the personal space, the user's
  own files. A link carries view only.

---

## 4. What the access numbers do and do not tell you

`list_share_links` is the inventory: `kind`, `target_name`, `is_live`,
`expires_at`, plus `views`, `downloads`, `invitees`, `last_access_at` and
`password_failures`. `get_share_link_access` is the same picture for one link,
with `recent`.

Two things to get right when you report these:

* **`password_failures` is a warning, not an analytic.** It means somebody is
  guessing at a link that was handed out. Surface it rather than listing it.
* **`downloads` counts downloads, not reads.** A PDF shared
  by link is read on the share page itself, in FlowpyDisk's own viewer. Reading
  it records one view per visitor per half hour, however many pieces the file
  arrives in, and only the Download button records a download. So "viewed,
  never downloaded" is an ordinary answer for a PDF, not a sign the link failed.
* **Nothing here identifies anybody.** Addresses are stored only as a salted
  hash and are never returned. `invitees` counts distinct addresses, so it is a
  *scale, not a headcount*: one household is one invitee, one person on two
  devices is two. Never present it as "N people opened this".

`list_shared_by_me` (API key only) is the one call that answers **"what am I
exposing?"** - both kinds in one list, `audience` saying `"link"` or
`"person"`. Revoked links, expired links and shares whose target is trashed are
left out on purpose. It is the right first call for any "audit my sharing"
request. Each row has `workspace`, the space the item is in (null for the
personal space), and `pending`, true for a share held for an address with no
account yet, so say "invited" rather than "has access" for those.

---

## 5. `request_edit_access` grants nothing

For a file you can read but not change. It notifies and emails the person who
shared the file with you when they can still edit it, otherwise (for a
workspace file) the workspace's Owner, otherwise the file's owner, and they
change your role from their Share dialog. The answer's `asked` says which: `sharer`, `workspace_owner` or
`owner`.

> **Your access is unchanged when it returns.** Do not follow it with a write
> and expect the write to succeed.

`get_file` reports what you actually hold, in `my_access`: `owner`, `editor` or
`viewer`. In a workspace it follows the person's role there: an Owner or Admin gets `owner` for any file in it, a
Viewer gets `viewer`. The call errors if you have no grant at all, or if you are already an
editor - in which case you have what you were about to ask for.

A file also says `shared_with_me`: true when it is
someone else's and reached you through a share or discovery, rather than being
in the space you act in. A workspace's files are its members' whatever their
role, so they are never `shared_with_me` there.

---

## 6. Discoverable means published to the platform, not to the internet

`set_file_discoverable` - API key only, and it is annotated destructive so a
client that honours annotations asks the human first.

**What it actually does:** grants every signed-in FlowpyDisk account the right
to find this file in their search - by name, by description, by tags, and by the
text extracted from inside it - to preview it, and to take a copy.

**A person previews the file itself, read
only**, in the web app's own readers: the whole note, every page of a PDF, the
video playing. Nobody but the owner can download it. The copy is how a person
gets it into their own space. Over a connection the preview is still the
teaser (§7).

It is **not** "share with my team" and **not** "make a link". It is also **not
the open internet**: discoverable has always meant published within FlowpyDisk,
to signed-in accounts.

**Ask first, every time.** Say what the file is and that everyone on the
platform will be able to find, preview and copy it. The owner is emailed
whenever a file of theirs becomes public, so doing this without asking is
something they *will* find out about.

`discoverable=False` is the remedy and needs no ceremony. `mature=True` hides an
already-discoverable file from other people's discovery again.

**Not every file may be published**, and the check is on the way in only:
documents, images, audio, video and code - things another person can preview
before taking a copy. Archives and anything that runs are refused with a 400
naming the extension. A file that is already public is never refused, and
unpublishing never is.

---

## 7. Discover: files, channels and templates, and no download

One place in the web app,
**Discover**, with three tabs: **Files and media** (discoverable files),
**Channels** (the public channels anyone can join) and **Templates** (what was
the Marketplace: the built-in catalogue plus what people published). The tools
kept their names: two browse tools and two previews, all four **ordinary
reads** and available on a connection. A public channel on Discover is Collab's directory, read
with `list_public_channels` (an API key only. See the Collab guide).

| | Tool | What it holds |
|---|---|---|
| **Files and media** | `browse_marketplace` | files whose owners made them discoverable |
| **Templates** | `browse_template_marketplace` | the built-in catalogue **plus** what people published |

Each shelf has a preview, also an ordinary read:

| | Tool | What it returns |
|---|---|---|
| **Files and media** | `preview_marketplace_file` | the card plus a **teaser**: the first 2,000 characters, the first 20 rows, up to three PDF pages (or an office file's, once drawn), a reduced image, a video's poster and length. Never the whole file. |
| **Templates** | `preview_marketplace_template` | the card plus the **whole** template, read-only: a table's fields and rows, a board's cards, a booking page and its week, a note set, a form and every question, a site's pages as notes. |

Use a preview to answer "what is in it" before suggesting a copy. Do not
present a file teaser as the file: say it is the start of it. A picture in a
teaser is an API path that returns a JPEG or WebP, not a link to the file.

The file preview also carries `viewer`: which
reader the web app draws the whole file with, and its routes
(`/market/files/{id}/preview/read`, `/preview/document`). Those routes serve
the person looking at Discover. A connection has no tool that reads them, so
for an agent the teaser is still what a published file says. A card on
either shelf now carries `glimpse`, the little it draws of its content (a
note's opening, a table's first rows, a form's first questions): a hint, never
the content.

A file card from `browse_marketplace` also carries
`duration_seconds` (a video's or a sound's length, else null) and
`thumbnail_url`, the card picture's API path (`/preview/image` or
`/preview/poster`), never a signed link. Like the teaser it is for the person
looking, not something an agent opens.

A .docx, .pptx, .odt, .odp, .xlsx, .ods, .doc or .ppt can be a
`pages` teaser too, with `source: "office"`: its first three pages, drawn by the
server. That happens only where the deployment has turned it on, and only once
the pages are drawn. Until then the same file's teaser is its text, or a
sheet's rows. So the type of one file's teaser can change from `text` to
`pages` without the file changing: do not read anything into it. Pictures are
not text. For what a deck or document says, the text teaser and search are
the answer, not the pages. A file
flagged as sensitive previews only for a person who opted in to sensitive
content. For everybody else it is a 404, the same as an unpublished one.

**There is no download.** The only way off either shelf is to take a copy into
your own workspace, and you read it from there:

* a file → `copy_shared_file`
* a template → `apply_template`

Copying is not a make-public action, so unlike publishing it **is** reachable
from a connected app. This is the shape to hold in your head: browse is free,
copy is the exit, publishing is the web app's.

### Browsing, precisely

`browse_marketplace` is a different question from `search_files`, which searches
what you can already reach. Nothing on the shelf is yours until you copy it.

* **`q` matches name, description and tags - NOT the text inside the file.** A
  browse is somebody scanning a shelf for a subject. Matching contents would
  return a card showing none of the words that found it. Words under two
  characters are ignored.
* `kind`: `doc`, `image`, `audio`, `video`, `other`. `sort`: `newest`
  (default), `oldest`, `name`, `most_copied`, `most_viewed`. On the templates
  shelf: `kind` is `table`, `note`, `canvas`, `booking` or `site` and `sort`
  starts at `featured`. **Anything else is refused rather than quietly
  ignored** - which is the opposite of the forms surface, where a bad option is
  coerced. Do not carry a habit across.
* `include_mature` (files shelf) brings in what owners flagged sensitive. Hidden
  by default, and still openable, copyable and reportable.
* **`count` is the whole shelf, not the page.**
* `owner_name` is a display name or the fixed label "A FlowpyDisk user" -
  **never an email address**, on any shelf. Do not go looking for one.
* On a template row, **`rating` is null until three ratings exist**, because an
  average of one or two is one person's opinion wearing a number.
  `rating_count` comes back either way. Report the count, and do not compute an
  average yourself from anything.

### Taking a copy, and what it costs

`copy_shared_file` works on a file somebody shared with you **and** on a
discoverable one. Read access is enough. Copying takes nothing from the owner.

* **The copy is real and independent.** Revoking the share, or the owner
  deleting the original, does nothing to it.
* **It carries content and nothing else** - no comments, no version history,
  because a copy of somebody's file should not bring their team's discussion
  into your workspace.
* **It costs the full size.** Storage dedup never crosses accounts, so the bytes
  genuinely land in your bucket. A file that will not fit is refused here rather
  than failing later.
* It returns `status: "pending"` and copies in the background - poll `get_file`
  if you need to wait.
* **Which workspace it lands in is the one this connection acts as**, and you
  cannot change it per call.

> The docstring says to find `file_id` with `list_shared_with_me`. **On a
> connection that tool does not exist** (§1). There, the ids come from
> `browse_marketplace` or from the person.

### A template is a snapshot, never a reference

`apply_template` always builds the template's **latest** version. Its author
decides when a new version exists, and what you already built is never touched
by a version cut afterwards. Editing the source of a template moves nothing that
was made from it - which is what makes handing one to a stranger safe.

`timezone` matters for booking and site templates: their availability windows
describe a *shape* of week - "three mornings" - and the hours are stored in the
page's timezone. Omitted, it falls back to UTC, which turns a mornings-only page
into something else entirely for most of the world. Pass the person's IANA zone
from `whoami`.

---

## 8. Publishing a note, table, canvas or gallery

All four publish pairs exist as tools and all eight are API-key-only. What is
worth knowing regardless of which surface you are on:

* The **status** reads stay available everywhere - `get_publish_status`,
  `get_table_publish_status`, `get_canvas_publish_status`,
  `get_gallery_publish_status` - so you can always answer "is this already
  public, and where".
* A board takes more public with it than its
  cards: `get_canvas_publish_status` answers `exposes`, `{pictures, files,
  tables}`, every picture and file a card references and every table a table
  card shows, with all its rows. Say those numbers to the person before they
  publish a board.
* A published thing becomes a **site page**. `list_sites` and `describe_site`
  are how you see the collection, and a `site` node in the graph with
  `publishes` edges is the one way to see everything public at once
  (`search-and-graph.md` §5).
* `delete_site` is never reachable on a connection: it un-publishes a whole
  collection at once.
* Publishing that makes a **new** site counts toward the
  published sites on the plan of whoever owns the space (counted across that owner's whole account). Over it the publish is refused
  with `plan_limit_reached`. Limits depend on the plan, see https://flowpydisk.com/pricing. Adding a page to an existing site (`site_id` and
  `path`) makes no new site, so it is the way to keep publishing at the limit.

---

## 9. What to say, in practice

Somebody asks you to share something and you are on a connection. Say the three
useful things, in this order:

1. **What is already exposed** - you can read that, so read it before answering.
2. **That the change itself is theirs**, and exactly where: the Share dialog on
   that file in the FlowpyDisk app.
3. **What you would recommend** if they asked - a named grant over a link when
   the recipient is known, an expiry when it is not, and the live-folder warning
   if a folder is involved.

That is a better answer than the one you were asked for, and it does not require
the capability you do not have.

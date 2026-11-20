# Files and folders

The largest surface and the one an agent reaches for first - 25 file tools plus
9 for folders.

---

## 1. Choosing how to put bytes in

Three tools, and picking wrong is the most common mistake on this surface.
**On the hosted server (`mcp.flowpydisk.com`) there are two**: `upload_file`,
`download_file` and `download_folder` read and write the disk of the machine
the server runs on, which there is FlowpyDisk's rather than yours, so they are not
offered at all. Upload with `upload_bytes`,
download with `get_download_url`, and anything over `upload_bytes`' ceiling
goes through the FlowpyDisk app.

| Tool | When | Ceiling |
|---|---|---|
| `upload_bytes` | content you are **holding in memory** - a note you just composed, a small CSV you generated | **a few MB**, and it is enforced |
| `upload_file` | content **on disk**, and anything large. **Local runs only** | no practical ceiling below the account's own |
| `edit_file_content` | replacing a **text document's** contents in place | 5 MB on the read side |

`upload_bytes` takes base64 - **not a data URI**, so no `data:…;base64,`
prefix - and the docstring is blunt about why it has a ceiling:

> The bytes arrive base64-encoded inside one JSON-RPC message, which inflates
> them by a third and holds them all in memory.

Over the limit it **refuses rather than trying**, and names `upload_file`
instead. On the hosted server, where there is no
`upload_file`, it names the app and a local run. That refusal is a kindness: nothing on that
path can stream, so retrying at a smaller size you are willing to accept would
still be the wrong tool.

**`filename` decides the file's name AND its type.** The extension is what the
app previews and downloads by, so `"notes"` and `"notes.md"` behave differently.
An agent that drops the extension has created a file the product cannot show.

A `list_files` row also says
what a file looks like without opening it. A `.md`, `.txt` or `.docx` row carries
`preview`: its first lines as plain text, about 200 characters ("" until its text
is extracted). A PDF's or video's row carries `poster_url`, a short-lived link to
a small JPEG of its first page or a frame. That is enough to tell two similarly
named files apart. It is not their content, so use `read_note` or
`get_download_url` for that.

Every file row, on every
listing and on a `search_files` hit, also carries `tile_url` and `tile_status`:
the address of the file's card picture (a 480 px WebP of an image, a video's
frame, a PDF's or an office file's first page), on this server, and whether it
is `ready`, `pending` (being made: say so rather than "no picture") or `none`.
It is what the apps show on a card. Hand it to a person to look at. Never read
it as the file's content.

---

## 2. Editing text in place, and not losing somebody's work

`read_note` returns `{content, name, content_etag}` for any UTF-8 text file -
Markdown, CSV, JSON, code - up to 5 MB. Larger answers **413**, binary **415**.
Fetch those with `download_file` on a local run, or `get_download_url` on the
hosted server. `get_file` is metadata only.

`edit_file_content` writes it back. Two things about that write:

1. **Every real change preserves the outgoing content as a version.** Re-saving
   byte-identical content does not. So an edit is recoverable through
   `list_file_versions` / `get_file_version` - which is worth telling a person
   who is nervous about letting you edit their document.
2. **Pass `if_match`.** It makes the write conditional on the `content_etag` you
   last read. Without it, you overwrite whatever somebody else did in between,
   silently. With it, you get a conflict and the current `content_etag` to retry
   against.

3. **A rename can change a note's text.** When a note opens with `# <its
   name>`, renaming it - `set_file_metadata(title=...)`, or `rename_file` on a
   note with no title of its own - rewrites that heading to the new name, as a
   real change with a version (the [notes guide](notes.md), section 1). Its
   `content_etag` moves with it, so `read_note` again after a rename rather
   than writing back what you read before it.

> Use `content_etag`, **not** `content_hash`. The latter is deprecated.

That is the docstring's own instruction and it exists because `content_hash` was
a lie about what it hashed, and it was replaced.

---

## 3. Moving a lot of things at once

**Use `bulk_file_action`, not a loop.** The docstring says why, and it is not
about elegance:

> the per-item version is one request each, which is slow and can trip this
> app's own rate limiting - so a large selection could fail BECAUSE it was
> large.

`action` is `"trash"`, `"restore"`, `"purge"`, `"move"` or `"star"`.

`move` takes every item to one place,
`move_into_folder_id` or `move_to_top_level=true`, by the same rules as
`move_file` and `move_folder`: an item that cannot go fails with their reason
and, where the rule has one, a `code` (`drive_mismatch` means another space,
which is `move_to_drive`'s job). `star` takes `starred=true|false` and acts on
files only. A folder fails. Neither asks the person's approval over a
connected app, as the single move and star do not. Trash does.

The caps differ by who is calling:

| Caller | Per call | May purge? |
|---|---|---|
| An API key | 1000 items | yes |
| A **connected app** (Claude or any other MCP connection) | 100 items | **no** - trash only, which is reversible |

**It is not a transaction, deliberately.** Every item is attempted and the
response says what happened to each. So read the response. A 200 does not mean
every id worked.

---

## 4. Trash is not deletion, and the difference matters to a person

`delete_file` **trashes**. `purge_file` destroys. Trashed files:

* leave the listings but **still count against the quota** until the trash is
  emptied.
* keep their name, so a new file with the same name becomes `name (2)`.
* are listed by `list_trash` and recoverable with `restore_file`.

`list_trash` also lists trashed **tables and boards** (`tables`, `canvases`): a person sees them in the same Trash, with the same 30 days
before they go for good. `restore_table` / `restore_canvas` bring one back.
`kind` narrows it: `note` (a file named `.md` or, `.markdown`), `file`
(every other file), `table` or `folder`, comma-separated for several. Every
list still comes back, the ones not asked for empty.

A big Trash can be paged: with `limit` the answer is one
merged list instead of five, newest deletion first, under `{count, limit,
offset, results}`, each row with a `type` (`folder`, `file`, `table`,
`canvas`, `conversation`). `kind` applies before the page is cut. Page when
`count` would fill your context. Restore and purge still take each row's `id`
with the tool for its `type`.

**A folder takes everything in it**. `delete_folder` sends
its subfolders, files, tables and boards to Trash together, and
`restore_folder` brings them back together. `purge_folder` also permanently
deletes what was trashed **on its own** from inside that folder, even though
those are separate Trash entries. `list_trash` gives each folder a
`purge_also` count (`{folders, files, tables, canvases}`) so you can say what
else goes before anybody asks for it.

**When its folder is still in Trash, ask where**.
Restoring a file, folder, table or board whose folder is in Trash is refused
with 409 `folder_in_trash`, naming that folder, and nothing is restored. Do
not pick a place yourself: ask the person whether to restore the folder too
(`restore_folder`, which brings the item back with it if they were trashed
together), put it in another folder (`folder_id`, or `into_folder_id` for
`restore_folder`), or put it at the top of its space (`top_level=true`).
`bulk_file_action` reports such an item as failed with the reason "Its folder
is in Trash", and takes `restore_into_folder_id` or `restore_top_level` for
every such item in one call.

Never purge on somebody's behalf unless they asked for permanent deletion in
those words. Over a connected app you cannot anyway - purge is not offered, on
purpose.

---

## 5. Finding things

* `list_files` lists **one level** by default: `folder_id` for a folder, absent
  for the root.
* `whole_account=True` crosses every folder - which is what a file picker
  needs, because **a file inside a folder appears in no other listing you have
  not explicitly asked for.** It takes no name filter. To find a file by name,
  use `search_files`.
* `search_files` for a name search. `search_everything` when the answer might be
  a table, a task or a comment rather than a file (see `search-and-graph.md`).
  Every `search_files` hit carries `updated_at`, so "the one
  I edited yesterday" is answerable from the results alone.
* What `search_files` matches inside a file is a server setting. By default it is the text extracted at
  upload. When the server reads chunks, a file's content search is the same
  text Ask AI and `semantic_search` read: its indexed passages, matched by
  whole word (or by the start of a word, "refor" for "reforestation", where
  the server has word starts on), and the `snippet` quotes the best passage. A file that is held,
  not read or not indexed yet is then found by its name, description and tags
  only, so check `list_index_status` before saying a word is not in a file.
  The results have the same shape either way.
* `list_media` for images and video across every folder. Page it (`limit`,
  then step `offset`) rather than taking a whole library in one result, and
  narrow it with `kind` (`image`, `video`, `audio`) or `drive` (`shared`, `my`, as on `list_files`). Each item carries `tile_url` and `large_url`: the
  addresses of a 480 px and a 1600 px WebP of the picture. `tile_url` is every
  listing's card picture (an image of any size, a video's frame. Null for a GIF or an SVG). `large_url` is null when the file keeps its original
  (a GIF, an SVG, under 512 KB, video and audio). They are for showing a picture to a person. The original is what
  `get_download_url` hands out, and what to read when the content matters.
* `get_video_playback(file_id)` for a URL that plays one of the person's videos
`copy` says what plays: `original` (a
  browser plays it and it is 1080p or lower), `fast_start` (the original with
  its index moved to the front, not re-encoded) or `converted` (one 1080p, or
  smaller source size, H.264/AAC copy). Null for a video from before the
  policy. `width`/`height` are the copy's, `source_width`/`source_height` the
  upload's: a converted 4K video plays at 1080p, and the original, which
  `get_download_url` hands out, is the full-quality one. A 409 `processing`
  means the copy is still being made. Ask again shortly rather than reporting
  the video broken.
* `list_recently_opened` for what the person last OPENED in the apps, newest
  first. Not what changed: that is `list_files` with
  `ordering="-updated_at"`. An agent's own reads are never recorded as opens.
  `kind="note"` gives only the notes they opened, `kind="file"` the rest.
* `set_file_starred(file_id, starred)` stars or unstars a file for the
  person, and `starred=True` on `list_files` or
  `list_media` lists what they starred. A star is the person's own mark
  (say star, not bookmark: a bookmark is a link card on a board or in a
  note): set one when they ask, never as a way of keeping your own place.

**Ordering and paging.** `ordering` and `starred` are applied **before** the page
is cut - which is the whole point of them: a client that pages and then sorts in
its own memory is exact over one page and wrong over two. Every paged listing has
has a **total order**, so a paged walk never returns a file twice or skips
one.

**`ordering="name"` on `list_files` is natural name order**: case does not matter and a number in a name counts as a
number, so `file2` comes before `file10` and `image (47)` before
`image (100)`. Do not re-sort the rows yourself. `list_media` still sorts
`name` by the plain text of the name.

---

## 6. Versions

`list_file_versions` and `get_file_version` are the history. A version is
created by a real content change - through `edit_file_content`, by replacing
the file's bytes with `replace_file_content` (a local run only), by
`edit_pdf_pages`, by `save_note_back_to_docx` (the previous .docx), or by a
rename that rewrote a note's first heading. `get_file_version` reads a text
version. A PDF's, a Word document's or a picture's answers 415 `not_text`,
so do not call it to look inside one: its size and date are in the list.

`restore_file_version(file_id, version_id)` puts a version back, byte for
byte, for any file: a note, a Word document, a PDF. What it replaces becomes
a version itself, so a restore can be undone the same way.
Who saved a version is `saved_by_name`, a display name or "@username", never an
email: name the person that way too.

If you are about to overwrite somebody's document, say that the previous
contents will be recoverable and name the tool. It changes the answer they give
you.

**A copy is not a version.** When somebody wants to try
changes without touching the original, or a template they fill in each time,
`copy_file` makes a separate file, "Copy of <name>" beside the original, and
costs no storage until one of the two is edited. It starts with no comments,
history, shares or star, so say so if they expected the copy to be shared like
the original. Files only: there is no folder copy.
In a workspace, `drive` puts a top-level copy in the
Shared drive or the person's My drive, so "put a copy of this in Shared drive"
is one call that leaves the original where it is. The copy is the person's,
whoever owned the original, and it counts toward their storage there. Inside a
folder the folder decides. Do not send a `drive` that disagrees with it.

**A PDF's pages.** Reordering,
turning, deleting and merging pages are one call, `edit_pdf_pages`, and it
takes the FINAL page list, not a list of operations. Read `get_pdf_pages`
first for the page count, then send every page the PDF should keep, in order:
`[{"page": 3}, {"page": 1, "rotate": 90}]` keeps page 3 then page 1 turned a
quarter clockwise, and deletes every other page. A page you leave out is
gone from the new version, so to delete page 4 of 10 you send the other nine.
`rotate` turns a page from where it already stands (90, 180, 270, or -90 for
a quarter anticlockwise). Pages of another PDF the person can open are
`{"file": "<id>", "page": n}`, placed wherever they belong in the list. That
PDF is not changed.

The save is a new version of the same file: the id, comments and shares stay,
and the previous pages are one `list_file_versions` away. Say so when you
delete pages. To split a PDF or hand someone a few pages, `extract_pdf_pages`
writes a NEW PDF beside it ("Lease (pages 2-4).pdf") and leaves the original
alone. It works on the person's own files only. Both take `if_match`, the
`content_hash` you read the page count from: if the PDF changed since, nothing
is written (412) and the page numbers you chose are not applied to pages that
moved. Read it again and choose again.

Two refusals to explain rather than retry. `pdf_signed`: a digitally signed
PDF is never rewritten, because any change stops its signature verifying.
Offer `extract_pdf_pages`, which leaves the signed original as it is.
`pdf_encrypted`: an encrypted PDF cannot have its pages changed here at all,
including one that opens with no password but restricts printing or copying
(common for statements and invoices).
`no_pages` means the list was empty: a PDF keeps at least one page, and
removing the whole file is `delete_file`.

---

## 7. Folders

The tree is `list_folders`, `create_folder`, `rename_folder`, `move_folder`,
`delete_folder`, `restore_folder`, `purge_folder`, plus `download_folder` (the
whole thing as a zip, local runs and API keys only) and
`get_camera_backup_folder` (which creates the folder when there is none, so it
needs read + write). `get_camera_backup_destination` reads where camera
backup goes now (set by the devices, null until one does). Each device has its own: `devices` lists them by name
(acting in the personal space only, empty in a workspace), and
the top level is the account-level destination a device without its own
follows. An agent is no device, so it never reads "this device's" one.
`get_folder(folder_id)` reads one folder by id: a search hit or a file's `folder` gives you an id, and this turns
it into a name and a parent without walking the tree one level at a time.

**When a folder last changed**. Every folder carries
`last_activity_at`: the folder's own rename or move, or the newest change to
anything directly inside it (a file added, edited or trashed, a subfolder
made or renamed). A change deeper down does not move it, so "nothing new in
this folder" means nothing new at that level, not in the whole subtree.

**A big folder is zipped in the background**. A folder whose files come to 100 MB or more is not written at once:
`download_folder` writes nothing and returns `{status: "preparing", zip}`,
and the person also gets a bell notice and an email when it is ready. Wait a
minute or two, then call it again with the same `folder_id` and `dest_path`
and `zip_id=zip.id`. While it is building that says `preparing` again. Once
ready it writes the file and returns `{path, size_bytes}`. `failed` or
`expired` (the ZIP is deleted 24 hours after it is made) means ask again
without `zip_id`, which starts a new one. Asking again for the same folder
while one is building returns that one, never a second build. Tell the person
it is being prepared rather than retrying in a tight loop.

**A folder holding a form's responses table cannot be trashed**. `delete_folder` (and `bulk_file_action` trashing such a folder)
is refused with HTTP 409, `code` `folder_has_form_table`, and **nothing is
trashed**: the form would go on writing into a table filed in Trash. `detail`
is a sentence naming the form, ready to show the person, and `forms` lists
them as `{id, title, slug}`. The way through is to move that table out of the
folder first, then delete the folder. Tables and boards otherwise go to Trash
with their folder, as section 4 says.

**Two counts, and they answer different questions**. A
folder's `child_count` is everything directly inside it, ready files and
subfolders. Its `item_count` is ready files only, and is kept that way because
the mobile app reads it. To tell whether a folder is really empty, read
`child_count`: a folder of three subfolders has `item_count: 0`.

**Ask where things should go.** An agent that uploads six files to the root has
made a mess somebody has to tidy. Tables and boards have their own answer to
this - `set_table_upload_folder` and `set_canvas_upload_folder` - and the product
learned to suggest rather than assume.

---

## 8. If a call says not-found

Folder-limited keys are retired, so a key never stops at a folder.
It reaches one space, and a workspace key reaches the Shared drive unless it
includes personal. Files that keep "not existing" are in another space or in
someone's My drive. The key's own row in `/auth/api-keys` carries its
`workspace` and `includes_personal`.

---

## 9. What is not yours

`set_file_discoverable`, share links, sharing with a person and permanent
deletion are **deliberately unreachable** over a connected app. They make
content public or grant access to other people, and that is a person's decision
in the FlowpyDisk app. Say so and point there. It is not a gap in the connector.
See `sharing-and-publishing.md`.

**A private channel's files**. A file put up in a
private Collab channel is found only by that channel's members, the person who
uploaded it, and the workspace's Owner and Admins. For anyone else it is not
found: missing from listings, a 404 on reading it, its versions and its
comments, and a board card or table cell that points at it reads as
unavailable, without its name. Do not tell a person a file is gone when it may
be in a channel they are not in: say you cannot reach it.

**Files Flowpy made**. A person can ask Flowpy, the
assistant in the FlowpyDisk app, to make a note, a table, a CSV file or a text
file. Until they press Save it is a draft inside their conversation, and no
tool reaches it: Flowpy has no MCP tools at all, so there
is nothing to list, read or save a draft with. Once saved it is an ordinary
note, table or file in the folder they chose - `list_files`, `read_note`,
`list_tables` and search see it exactly like any other, and nothing over MCP
marks it as made by Flowpy. If a person asks you to save one of Flowpy's
drafts, it is done in the app.

---

## 10. Shared drive and My drive

A **space** is a workspace, and **Personal** is the person's own workspace.
Inside every workspace there are two **drives**: the **Shared drive**, which
every member sees, and each member's **My drive**, which nobody else sees, the
workspace's Owner and Admins included. In the Personal workspace everything is
My drive. Files, folders, tables, boards and forms each carry a read-only
`drive`: `shared`, or `my` for the person's own My drive items and everything
of theirs in Personal. It is null on an item someone else shared with them.

**Making something.** `create_folder`, `create_note`, `upload_bytes`,
`upload_file`, `create_table`, `create_canvas`, `create_form` and
`apply_template` take `drive`. Leave it out and the item goes in the Shared
drive, as it always has: only put something in My drive when the person asked
for it to be private to them. Inside a folder the folder decides. A `drive`
that disagrees is refused with `drive_mismatch`, so the fix is to drop
`drive`, not to retry with the other one. `duplicate_canvas` and
`duplicate_form` keep the original's drive unless told otherwise.

**When `my` is refused.** `drive_unavailable` means the workspace has no My
drive yet (`list_workspaces` shows `features.my_drive`). Say so rather than
making it in the Shared drive instead, which would show it to everyone.
`drive_key_shared_only` means you are on a workspace API key that does not
include personal, which works in the Shared drive only. Both are a person's decision, never a reason to fall
back.

**A workspace Viewer** may make items in their own My
drive and then do everything with them: rename, edit, move within My drive,
trash, restore, delete for good, columns, rows, cards, questions, copies and
exports (which stay in their My drive). Never share, link, publish, make
discoverable or open a form, and nothing in the Shared drive or on another
member's item: those are a 403, so tell the person rather than retrying. A
connected app the person granted write does the same for them: while My drive
is on, a Viewer's connection may hold write and the API refuses what their
role does not allow. A workspace API key reaches My drive only when
it includes personal, and then only its own person's.

**Adding to someone's folder.** When a person has edit access to a member's
My drive folder (it shows under `list_shared_with_me`), they may add to it:
pass that folder to `upload_bytes`, `create_note`, `create_folder`,
`create_table` or `create_canvas`. What they add belongs to the folder's owner
and uses the owner's storage, so say so when you confirm ("added to Priya's
folder"). A view-only grant is refused. Do not retry elsewhere without asking.

**Listing.** `list_files`, `list_folders`, `list_tables`, `list_canvases`,
`list_forms` and `list_media` (the Gallery's drive switch)
take `drive` to list one drive. Without it you get both, so check
an item's `drive` before telling the person who can see it. In the Personal
workspace everything is My drive: `drive: "my"` lists everything there and
`drive: "shared"` nothing.
`list_shared_with_me` takes `from_workspace_id` for what was shared from
inside one workspace.

**Changing an item's drive** is `move_to_drive`, never `move_file` or
`move_folder` (those refuse a folder of the other drive with
`drive_mismatch`). It takes files, folders, tables and boards. A folder brings
everything inside it. **Call it with `preview: true` first** and tell the
person what it says: moving into My drive hides the items from every other
member, revokes the shares and links other people made on them (`revoked`),
and keeps anything published (`publications`). Moving into the Shared drive
shows a person's private items to the whole workspace. Items going into My
drive land in their OWNER's My drive, whoever asks. A folder holding other
members' items is refused with 409 `drive_move_others` and a `count`: say how
many and let the person decide, do not move the others' items out on your own.
Over a connected app the move itself (not the preview) needs the person's
approval.

## 11. Files moved in from another drive

A person can move their files in from Google Drive, Dropbox or OneDrive, in
the FlowpyDisk app. `list_drive_imports` and `get_drive_import` read how
that is going. There is no tool to start, pause or cancel a move, on
purpose: the person does that in the app. If they ask you to start one,
say where it is in the app, do not look for a way round it.

**Read the state, then the reason.** `blocked_quota` means it does not fit
their storage and nothing has started: quote `reason_text`, and when
`upgrade.by_you` is false say the workspace owner has to add storage.
`paused` with `reason: reconnect_needed` means the account needs connecting
again in Settings. Treat an unknown state as in progress, never as done.

**Progress** is `progress.files_done` of `progress.files_to_copy` (and the
bytes beside them). `get_drive_import(..., include_report=true)` says what
was left out, grouped by reason: `shared_with_me` (someone else's file,
skipped unless the person asked), `unchanged` (already copied by an earlier
run), `too_large` (over their plan's limit for one file), `google_type` (a
Google Form or Site, which cannot be exported), `gone_at_source`,
`checksum_mismatch`. Quote the `text` of each line.

**What to tell the person.** It is a copy: nothing changes in the source
drive. Google Docs, Sheets and Slides arrive as .docx, .xlsx and .pptx.
Sharing is not copied: every copy is private to them, and the report's
`shared_at_source` counts the ones that had been shared. Copies are
searchable by name and words as soon as they land, but have no AI index
(meaning search, picture descriptions, transcripts) until the person asks
for it in the app. `report.waiting_for_ai` counts those, so a meaning
search that misses a moved file is expected until then.

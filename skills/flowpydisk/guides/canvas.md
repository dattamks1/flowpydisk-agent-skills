# Canvas

**The surface whose value is entirely in the arrangement.** Every other tool in
this product either works or does not. A board can be completely correct -
every card created, every connector attached, every field valid - and still be
worthless, because it reads as a test fixture. There is no error for that.

This is the guide for a model with no screen.

---

## 1. Before any card: what is this board for?

Three shapes cover almost everything an agent is asked to draw, and they want
different treatment:

| The board is | Then |
|---|---|
| **A diagram** - a flow, an architecture, a decision tree | Shapes and connectors carry the meaning. Sizes should be uniform. Alignment matters more than colour |
| **A map of things that exist** - files, tables, people | `file` and `table` cards, arranged in groups. The value is the grouping, not the drawing |
| **A wall** - research, references, a moodboard | `bookmark`, `image` and `image_card`. Position is loose. Consistent card widths are what stop it looking like debris |

A board that mixes all three without a reason reads as a dump. If the request
is genuinely mixed, use `group_cards` to make each region a named block, so the
mixing is visible as structure rather than as mess.

---

## 2. The five traps, each of which cost a round

### A text card follows the board's tone unless you set a fill

`""` and `null` are not the same thing here, and this is:
 in `add_card` and `update_card`, `color` and the
`*_color` fields treat **`""` as "follow the card's own tone"**, not as "unset".
A card created with no `fill_color` renders in the sticky tone, which is a
yellow. A wall of "text" cards that were meant to be bare words on the board
comes out as a wall of stickies.

If you want words with no box: `kind="text"`, and set `fill_color` and
`border_width: 0` explicitly. Do not assume the absence of a value means the
absence of a background.

### Connectors are drawn beneath cards

Edges render in a layer under the nodes. A connector that passes across a card -
a backdrop, a group, a large image - **disappears behind it**, and nothing in
the response says so. Route around, or leave a lane: when you lay out a flow,
leave the corridor between two ranks empty of anything but the connectors.

### A line joins the sides that face each other, unless you say otherwise

A line with no
stored side is drawn between the two sides that FACE each other, worked out
each time the board draws: side by side cards are joined right to left,
stacked ones bottom to top, and the line follows the cards if somebody moves
them. **So leave `from_side` and `to_side` out by default.** Set them only
when the facing side is wrong for the diagram - a "no" branch that loops back
to an earlier step, or three lines that would otherwise share one side of a
card - with "top", "bottom", "left" or "right".

When no side choice keeps
a line out of a card, **bend it**: `points` on `connect_cards` or
`style_connection` is a list of up to 10 `{"x": n, "y": n}` board coordinates
the line passes through, in order from the first card. Put a bend in the empty
corridor beside the card in the way, not on it. The points are absolute, so a
card moved afterwards leaves them where they were: move a card, then re-send
the points. `points: []` straightens the line, and leaving `points` out of a
`style_connection` keeps the bends a person dragged. A label sits at the
middle of the bent path, not between the cards.

### A connector's label sits on the line's own midpoint

`CanvasEdge` places the label at the geometric midpoint of the path it drew. So
two cards close together give the label no room and it covers the line it is
labelling. **Give a labelled connector at least ~160px of gap** between the
cards it joins, or leave it unlabelled and put the words in a card.

### A drawn grid scrolls. It does not resize the card

`kind="grid"` is a table you draw *here*, storing its own cells - distinct from
`kind="table"`, which points at one of your real Tables. Its body has its own
scroller, so a card too short for its rows hides them behind a scrollbar rather
than growing. On a board meant to be read at a glance, and on a published board
where nobody thinks to scroll inside a card, those rows are simply gone.

Size the card to the content: the default is 3×3 at 420×200, so budget roughly
that per three rows and add for each one after. Limits are 40×12 and 240 cells.
Anything outside the declared grid is dropped silently.

### Your own table on a board shared with you shows to everyone on the board

On a board somebody shared
with the person as an editor, `add_card` with `kind="table"` may name one of the
person's own personal tables, and every reader of the board can then view that
table (only its owner edits it). That gives people access, so it is refused
over a connection with `code: own_table_on_shared_board`, as is a workspace's
table: the person places it in the app. A table card's `table.permission` says
what the reader may do with the table: `viewer` means it opens read-only.

The same holds for the
person's own FILES. On a board shared with them, a file card, a picture or an
image card's picture naming one of their own files would share that file with
the board's owner, so over a connection it is refused with
`code: own_file_on_shared_board`. A file that already sits in the shared folder
gives nobody new access and is placed as usual. Tell the person to place their
own file in the app, or to use a file the board's owner can already open.

---

## 3. Choosing the card kind

The two pairs that get confused, both deliberate distinctions:

* **`image` vs `image_card`.** `image` is a picture and nothing else. `image_card`
  is composed - an article on the board - and takes `sections`, a list of
  `image`/`title`/`content` blocks in order, up to 20. It defaults to 320×380
  because **it is a column, not a tile**.
* **`bookmark` vs `embed`.** A bookmark says *go and look at this*. An embed
  says *watch this here*. Choose by what the reader is meant to do. An embed
  only frames a URL a supported provider claims - anything else is drawn as a
  link with a line saying so, which is why it never fails at creation. **An
  embed's `url` can be changed later with `update_card`**, uniquely. Every other
  card's cannot, so a wrong video is a fix and a wrong bookmark is a delete.
  **The two switch with `update_card(kind=...)`**,
  the web board's Card / Play here: the card keeps its id, comments, reactions,
  lines and colour, so switch rather than delete and re-add. No other kind changes.

**A reference card carries a glimpse of what it points at**. On `get_canvas`'s cards, a `file` card's `file` has the note's own
`title` and its first line as `excerpt` when the file is a note, and a `table`
card's `table` has a `preview` - its first three columns and five rows as
words. Use them to describe a board without opening every note and table on
it. They are a preview, so read the table itself before quoting figures.
The preview's `formats` says, per column, which
are numbers (`{type, options}` for number, currency and percent, else null).
A number's words are its plain digits ("2400"), so write it as the table
would, with its symbol and separators, when you quote it.

**A reference whose file or table is in Trash is named, not hidden**. Its `file` or `table` is then just `{id, name, trashed:
true}` (a file adds `kind`): say it is in Trash rather than describing it, and
`restore_file` or `restore_table` brings it back on the board as it was. Null
still means gone for good, or out of your reach.

**A `bookmark`'s preview arrives late.** Title, description and image resolve in
the background, so a freshly-created bookmark has none of them. If you are
about to report what the board contains, re-fetch with `get_canvas` or
check `link_status` first - otherwise you will describe an empty card.

**A bookmark's picture has two addresses**.
`link_image_url` is where the linked page says its image lives, on that site.
`link_image` is the copy this server keeps of it (at most 1600 px wide), at
`/api/v1/canvas/cards/<id>/preview-image`. That is the one the app shows, and
the one to hand anybody who should see the picture. `link_image` is null when
the page has no image or the picture could not be copied (too large, not a
JPEG/PNG/WebP/GIF, the site refused): the card then shows its title and text
only, which is a finished card, not a broken one. Do not report a null
`link_image` as an error.

**`group` cards cannot be created directly.** Make the cards, then
`group_cards`. And note what grouping does to coordinates: a card's x/y become
relative to the group's position, so if you plan to group, lay out first and
group second rather than computing positions against a group that does not
exist yet.

**A group is a named section**. `group_cards`
takes a `title`, drawn on the section's top-left corner, and an optional
`color` tint from the card palette. Always pass the title - an unnamed group
is a box, and the titles are what a person reads first on a board of regions.
The group sits 24px outside its cards and the title lives in that margin, so
do not crowd a section's top row against the next thing above it. Rename or
re-tint later with `update_card(text=..., color=...)` on the group card.

**Moving cards into or out of a section**. `move_cards_to_section(canvas_id, card_ids, section_id)` puts cards in
an existing section, straight across from another one if they are in one.
Without `section_id` it takes them out. A card keeps its place on the board:
only the frame its x/y are counted from changes. So check the card's place
before you move it in: a card far from the section's box becomes a member
drawn outside it. Move it first with `update_card`, then into the section. A
section left with no cards stays on the board.

**A locked card stays put**. A card with
`locked: true` on `get_canvas` was pinned by somebody: `update_card`,
`delete_card` and `move_cards_to_section` on it answer 409 `card_locked` and
change nothing. Lay out around it. Unlock it (`update_card(locked=False)`, which
can carry your change in the same call) only when the person asked for that
card to change, and say that you did. Lock a card with `update_card(locked=True)`
when a person asks for something to stay where it is.

**Cards that carry work**. A
**task card** is a piece of work on the board: `add_task_card` with a `title`,
a `task_status` (`todo`, `in_progress`, `done`. `todo` when omitted), an
optional `task_priority` (`low`, `medium`, `high`) and a `task_due`, or both
`task_start` and `task_due` for a range. Change them with `update_task_card`.
The fields are refused on any other card. A task card lives on the board only,
so do not tell a person it will remind them, until it is made a task:
`convert_card_to_task(card_id, task_list_id?)`
makes a task in Tasks from it (its title, due date, priority, done state and
first person, in the list named or in no list) and links the two both ways:
ticking either ticks the other, a due date set on either is the other's, and
deleting either leaves the other. The card's `linked_task` names the task, and
the task's `board_card` names the board. A card already linked is a 409
`already_linked`. Unlinking is done in the app.
**Stand-in faces**: `placeholder_people` on
`add_task_card` or `update_task_card` puts up to six example names on a task
card, drawn as dashed faces with initials after its real people, until its
people are chosen with `assign_card`, which clears them.
They are words, not accounts: use them in a plan or template, never instead of
`assign_card` for real people.
**People go on any card** with `assign_card(card_id, assignee_ids)`, which
replaces the whole list (an empty list takes everyone off). Only people who
can open the board may be named: take the ids from `list_mentionable_users`
with the canvas id, plus your own from `whoami`. Anyone else is a 400. Their
faces show in the card's corner, and on a task card in its footer. A
**person card** (`add_card(kind="person")`) is one person drawn as a face and a
name, its `text` an optional role ("Facilitator"). Made with nobody named it
is you. Read a card's people from `assignee_people` (name and picture, never
an address), not from `assignees`, which is ids.

**Reactions and stickers**.
`react_to_card(card_id, emoji, on)` adds (`on=true`, the default) or takes back
your reaction: `heart`, `plus_one`, `hundred`, `star_eyes`, `fire`, `clap`.
Saying the same twice changes nothing, so retrying is safe. A card's
`reactions` list each emoji with its `count`, whether one is yours (`mine`) and
who (`names`). React only when the person asked you to: a reaction carries
their name to everyone on the board. A **sticker** (`add_card(kind="sticker",
text=<glyph>)`) is a big emoji stamped anywhere, sized and turned like any
card, from a fixed set (the `add_card` description lists it). A note's text
can fit its box (`text_size="auto"`: big when short, smaller as it grows), the
way a square sticky does on other boards. A new sticky in the app is 200×200
with it on. A plain note made with it sits its words in the middle of the box: the
server stores `text_valign="middle"` when no position is sent. `text_valign` on it means what it says: `top` puts the words at the
top, as on any card.
**Card styles: frames, headings, hero and step cards, drop zones**. A card's `variant` is its style within its kind,
`""` being the plain card. Use them to give a board structure a person reads at
a glance:

* **A frame** is a section drawn as a page with a big title, the one a person
  steps through with Present. `group_cards(card_ids, title=..., variant="frame")`
  frames one or more cards (it leaves 64px above them for the title, so do not
  crowd the first row against the frame's top). `card_ids=[]` with `x`, `y`
  places an empty frame (640x420 unless you send `width`/`height`) to fill
  with `move_cards_to_section`. Name every frame: the Frames list and Present
  read its title. Frames present in reading order, rows top to bottom and left
  to right, so lay them out in the order they should be shown.
  `update_card(variant="")` turns a frame back into a section.
* **A heading** is a text card with `variant` `"h1"` (44px) or `"h2"` (30px):
  big words, no box, like "Project brief" over a column. Prefer it to a large
  `text_size_px`: it says what the words are.
* **A hero card** is a note with `variant="hero"`: `text` is its title, `color`
  its band (a strong colour, e.g. `"#6D5BD0"`, not a pale sticky tone), and
  `details` `{objective, why, progress, time, emoji}`. One per board, at the top.
* **A step card** is a note with `variant="step"`: `details` `{label, title,
  emoji}` (`label` is the band, "Step 1"), `text` the body. Give a run of steps
  different band colours and join them with connectors.
* **A drop zone** is an image card with `variant="drop"` and no `file_id` or
  `url`. `text` is its prompt ("Drop your screenshot here"). When you have the
  picture, `update_card(file_id=...)` fills it: the card keeps its place, size
  and lines and becomes a plain picture card (its prompt is cleared).

A variant another kind owns is refused, and `details` on anything but a hero or
a step card is refused. A drawn table (`grid`) can also set `header_col: true`
to draw its first column as row headers, and a cell can hold `stickies`
(`[{text, color}]`, up to 8), sticky notes that move with the table: use them
for a project plan's swimlanes rather than loose notes laid over the grid.
A `file` card for a note also carries `file.lines`, the note's opening (up to 8
lines, `{text, heading}`), which a tall card draws as a page. A bookmark's
`link_fetched_at` is when its preview was fetched (null on a template's
bookmark, which is never fetched).

**`kind="shape", shape_type="text"` still works and should not be used.** It was
replaced by `kind="text"` and produces exactly that card. Existing
integrations keep working. New ones should not adopt it.

---

## 4. Styling, and the one field that is not a preference

Every styling field is shared between `add_card` and `update_card`, and an
invalid value is **rejected rather than ignored** - the model's own choices are
taken seriously, so a typo is a 400 and not a silent default.

    text_size      "sm" | "md" | "lg" | "xl"
    text_size_px   8-200, exact. 0 means "follow text_size"
    text_align     "left" | "center" | "right" | "justify"
    text_weight    "normal" | "bold"
    text_italic / text_underline / text_strike   compose with each other
    font_family    "sans" | "display" | "serif" | "hand" | "rounded" |
                   "condensed" | "slab" | "mono"
    border_dash    "solid" | "dashed" | "dotted" | "none"

`text_size_px` exists for when none of the four steps is right. Prefer the
steps: a board where every card picked its own exact size is a board with no
typographic rhythm, which is most of what separates a designed board from a
generated one.

**`icon` is the one kind with a closed vocabulary.** `icon_source="material"`
takes a Material Symbols name in snake_case **from the board's allowlist** - an
unlisted name is a 400, not a fallback glyph. `icon_source="emoji"` takes the
character itself. Icons default to 96×96 rather than the usual 220×140.

---

## 5. Composition: the part with no API

Rules that make the difference between a board somebody keeps and one they
redraw:

1. **Pick a grid and stay on it.** Decide a column width and a row height
   before the first card, and put every x and y on multiples of it. Nothing
   makes a generated board look generated faster than eight cards at eight
   arbitrary offsets.
   x, y, width and height are finite numbers: NaN or an infinity is refused
   with a 400 naming the field, and nothing is written.
2. **Uniform sizes within a rank.** Cards doing the same job should be the same
   size. Vary size to mean something - a bigger card is a more important one -
   never because the text happened to be longer.
3. **Leave the connector corridors empty** (trap two above).
4. **Three colours at most**, and let most cards take the default tone. A board
   where every card is a different colour has told the reader nothing, because
   colour that varies everywhere carries no signal.
5. **Group before you finish.** `group_cards` is what turns a scatter into
   regions. An ungrouped board of thirty cards is thirty things, and a grouped
   one is four.

### A mind map is one call, not thirty

Do not build a mind map from `add_card` and `connect_cards`: positions by hand
never come out balanced, and the board will not treat the cards as a map (no
fold, no Tab and Enter, no branch colours). `create_mind_map(canvas_id, topic,
children)` takes the whole outline - `[{"text", "children", "color"}]`, up to
8 levels and 200 topics - and lays it out the way the board does: first-level
branches balanced left and right of the central topic, each a step further
out per level, siblings stacked in the order you gave them, each parent
centred on its children. Each first-level branch gets the next colour of the
map's palette unless you pass `color` on it, and everything under it draws in
that colour. `style` is how topics look (`pill`, `rounded`, `underline` for
text on a line) and `lines` how branches run (`curved` or `elbow`).

To grow a map, `expand_mind_map(canvas_id, parent_id, children)` adds topics
after the parent's existing children and moves the map's other topics to make
room (the reply's `moved` lists them). A folded parent is unfolded. A card
whose `mind_role` is `""` is not part of a map and is refused.

What makes a good map, as for a board: three to six first-level branches, short
labels (a few words, not sentences), and depth where the thinking is rather
than evenly everywhere. The structure is the lines: a topic's parent is the
card its incoming line comes from, so `style_connection` and `delete_card`
work on a map as on any board (deleting a topic over MCP removes that card
only. In the app a person's delete takes its branch with it).

A line can also carry a marker in its middle: `mid_marker="diamond"` (with the
line's direction drawn in it, as on a step flow) or `"circle"`, on
`connect_cards` and `style_connection`.

---

## 6. What you cannot do from here, and why

Making content public
and granting access to other people are decisions a person takes in the web
app. `get_canvas_publish_status` shows what is already public. Do not
describe the absence as a gap in the connector. Point at the app.

**A board shared with a person**. A board can be
shared by email at viewer or editor, as a file can, from the board's Share in
the app. No tool makes the share. `list_canvas_permissions` reads who has it.
`get_canvas` on a board shared WITH the person answers `permission: "viewer"` or
`"editor"` (`"owner"` on their own), and `shared_by_name` names who shared it,
 a display name or "@username" and null on their own: say
whose board it is with that name. A viewer only reads, so do not offer edits.
An editor changes cards, connections and sections but never deletes, moves,
publishes or re-shares the board. The files on its cards show to them through
the board only: a card's `file`, an image card's sections
and a drawn table's cells resolve, view-only, but the files themselves are not
shared, so `get_file`, a download or a search for one of those ids still finds
nothing. Do not offer to open or copy such a file with the file tools. Say it
is visible on the board. A board reached through a shared FOLDER (a grant on
a live folder above it) shows its card files the same way. A revoke, the folder going to Trash, the board moving out
of the folder, or the file leaving the card ends it at once. Acting in a
workspace, a key or connected app does not reach boards shared with its
person. Acting in the personal space it does.

`get_canvas_upload_folder` / `set_canvas_upload_folder` decide where a board's
uploads are filed (a suggestion the user
confirms, then remembered). If you are uploading images for a board, ask for the
folder rather than scattering them at the root.

---

## 7. A worked order

For "draw me the architecture of X", in the order that produces a board worth
keeping:

1. `create_canvas`.
2. Decide the grid (say 260 wide, 180 tall, 80 gutter) and the ranks.
3. `add_card` every node at its grid position, uniform size, default tone,
   `kind="shape"` with a real `shape_type` rather than a note.
4. `connect_cards`, keeping the corridors between ranks clear. Leave the
   sides to the board unless a line must loop back (trap three above).
5. `style_connection` only where a line means something different from its
   neighbours - a dashed line for an optional path, say.
6. `group_cards` per region, with a `title` for each.
7. `get_canvas` and read back what you built. Bookmarks will have
   resolved by now. Anything that came back wrong is cheaper to fix than to
   explain.

---

## 8. Copying a board

`duplicate_canvas(canvas_id, name?)` makes a new board that is the first one
card for card: every card with every field it carries, each group with its
members still inside it, every line with its label and style, and the board's
description and tags. It is the right tool for "make a version of this I can
change" and the wrong one for "start something like this" - for that, a
template is the snapshot meant for it (`save_as_template`, then
`apply_template`).

Four things to know before you call it:

- **Files and tables are referenced, not copied.** The copy's file card shows
  the same file. Nothing is uploaded and no storage is used. Editing a table
  from either board edits the one table.
- **Except a board shared with you**. Its copy
  lands in the person's own space and takes its own copies of the files on its
  cards, on their storage, filed under `Canvas / <the copy's name>`. A file
  they cannot open, or one that does not fit their storage, is left off its
  card, which then shows the file as not available. The reply's `files` says
  how many were copied and, per card left without its file, why
  (`no_access`, `quota_exceeded`, `file_too_large`). Say so when it is not
  empty, rather than describing the copy as complete. Tables are still
  referenced. A link card's preview picture is copied to their storage too.
   One that cannot be copied is left off and the card shows its
  title and description.
- **The copy is never public**, whatever the original is, and it carries no
  comments. Say so if the person expected the copy to be shared already.
- **Name it for what it is for.** Without `name` it is "<name> copy", which is
  a fine default for a person and a poor one for the next agent that lists the
  boards. `"Roadmap - Q1 draft"` tells both what it is. A trashed board cannot
  be copied (restore it first).

## 9. Undoing a delete

`delete_card` and `delete_connection` are reversible for 30 days, and the way
back is `restore_cards(canvas_id, card_ids, connection_ids?)`, never
`add_card`. A restore brings back the SAME card: its id, every field, its
place, size and z order, and every line that went with it. A re-created card
is a different card with a new id, and the lines, comments and earlier
references that named the old one would not find it.

- **Keep the ids you delete.** `get_canvas` stops listing a deleted card, so
  the id in your own record of the call is the only handle on it.
- **Lines come back once both ends exist.** Deleting two connected cards and
  restoring one brings that card and not the line between them. Restoring the
  other later brings the line too. Restore the cards of one delete together
  and the board is exactly as it was.
- **A line deleted on its own** goes in `connection_ids`.
- **A card that was in a section** goes back into it, or back to where it sat
  on the board if the section was dissolved meanwhile.
- **Emptying a board is not restorable.** It is the web app's "Empty the
  canvas", and its warning says so.

## 10. Changing text a person may be editing

`update_card(card_id, text=..., expected_text=...)`
writes only if the card still says `expected_text`, the text you read with
`get_canvas` (`""` for an empty card). If somebody changed it since, nothing in
the call is written and you get 409 `card_changed` with the card as it is now:
read it and decide again, rather than writing over them. Without it, the last
write wins.

The same for a drawn table and an image card:
`expected_grid` and `expected_sections` take the value exactly as `get_canvas`
returned it, or as your last `update_card` replied (the server fills in
defaults, so what you sent is not it). And `rename_canvas(expected_name=...)`
renames only if nobody did since. Otherwise 409 `canvas_changed`.

## 11. Moving cards a person may be arranging

To lay out several cards (a tidy, an
alignment, a grid), use `arrange_cards(canvas_id, cards)`, not one
`update_card` per card: the layout is one write, all or nothing. Give each card
where you last saw it, `expected_x` and `expected_y` (and `expected_width`,
`expected_height` when you resize), exactly as `get_canvas` returned them. If a
person moved or resized one of those cards since, nothing moves and you get 409
`cards_moved`, naming the cards in `moved` and returning them as they are now:
read the board again and lay it out from there, rather than putting their cards
back. A locked card, or one from another board, refuses the whole layout too.
There is no live sync: the check is made when you write.

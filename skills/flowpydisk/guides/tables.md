# Tables

The largest tool surface after files, and the one where a correct call can still
produce a wrong table.

---

## 1. The rule that governs the whole surface

**A cell is written by field id, never by column name, and most values the
field cannot hold are DROPPED - not refused.**

`create_row` and `update_row` take `data` as a map of **field id → value**, and
the docstring is explicit: *"Values are coerced to each field's type, and
anything invalid is dropped."* So a row written with a plausible-looking name
key, or with a select's label where its choice id belongs, comes back **201
Created with empty cells**. There is no error to notice.

**Two kinds of value are refused instead, with nothing written.** A number,
currency, percent or rating cell takes a number, or text that is one written
plainly (`"1234.5"`). Anything else - `"1,5"`, `"twelve"`, `true`, `NaN` - is a
400 with `code: "not_a_number"`, the `field` and the `value`. Write numbers as JSON
numbers. And text past its column's limit, below.

Two consequences, and they are the whole reason this guide leads with them:

1. **Call `get_table` first, every time**, and build a name → id map from its
   fields. Do not cache ids across sessions. A column can be deleted and
   recreated.
2. **Read one row back after the first write.** If the cells are empty, the
   mapping is wrong and you are about to write a hundred more like it.

**Changing a row somebody may be editing: pass
`expected`.** `update_row(row_id, data, expected)` takes, for each cell in
`data`, the value you read there (`null` for an empty cell). If a person changed
one of those cells since, nothing is written and you get 409 `row_changed`
with `fields` (what moved) and `row` (as it is now): read it and decide again,
rather than writing over them. Only the cells you write are checked, so an
edit elsewhere in the row is not a conflict. Without `expected`, the last write
wins.
`update_rows` takes the same `expected` on each row: a changed cell is
left alone and listed in `conflicts`, and the rest of the batch is written.

**Text is the one value that is REFUSED, never cut
or dropped.** A `text`, `email` or `url` cell holds up to 2,000 characters and
a `long_text` cell up to 100,000. Each field's `max_length` in `get_table` says
which. A longer value is a 400 with `code: "too_long"`, the `field`, its
`max_length` and the value's `length` - and a batch with one such value writes
nothing. If you hold
more than a column takes, write it to a `long_text` column (or `change_field_type`
the column first), or split it. Never trim it yourself without saying so.

---

## 2. The field types, and what each actually takes

    text  long_text  number  checkbox  single_select  multi_select
    date  url  email  rating  currency  percent  attachment  image

plus **relation** and the three computed kinds - **formula**, **lookup**,
**rollup** - and **backlink**, a relation's read-only other side, which only
`show_on_target` makes.

Options, where a type needs them:

| Type | `options` | Cell value |
|---|---|---|
| `single_select` / `multi_select` | `{"choices": [{"id", "name", "color"}]}` | **the choice `id`**, not the name |
| `rating` | `{"max": 5}` | a number |
| `currency` | `{"symbol": "$"}` | a number |
| `attachment` | - | a **list of file ids** (from `list_files`) |
| `image` | - | a list of file ids, **every one an image** |
| `relation` | `{"table_id": "…"}` | a **list of row ids** from that table |
| `url` | none | a string. A bare address (`example.com/page`) is stored with `https://` in front, anything else as written |
| `formula` | `{"expr": …}` | nothing - derived at read time |
| `lookup` / `rollup` | `{"relation", "field"[, "agg"]}` | nothing - derived at read time |
| `backlink` | written by the server: `{"relation", "table_id"}` | nothing - the linking row ids are in `computed` |

### A relation can show on the table it links to

A relation column on
table A pointing at table B can also show on B: pass `show_on_target: true`
to `add_field` with the relation, or to `update_field` on an existing one. B
gets a read-only `backlink` column, named after A ("Projects 2" when the name
is taken), whose value for each B row is **the A rows that link to it**, in
A's row order. It is not a second copy of the links: it is computed from A's
relation cells every time it is read, so edit the relation cells on A to
change it. Leave the flag out and nothing changes on B, which is the default.

* `get_table` on A shows `backlink: {field_id, table_id}` on the relation
  (null when it is not shown). On B, `get_table_rows` gives each row
  `computed[<backlink field id>]`: a list of A row ids. `query_table` and
  `export_table` write their names, as a relation cell's are written.
* `update_field(show_on_target=false)` on the relation, or deleting the
  back-link column on B, removes it. **Neither touches a link**: the
  back-link stores nothing. `restore_field` brings a deleted one back.
* A `backlink` cannot be made with `add_field(type="backlink")`, duplicated,
  or converted to or from another type (400). It can be renamed.
* Refused (400) on a column that is not a relation, when you cannot edit B,
  or when someone who can read B could not see A: the rule a relation
  follows, turned around.
* If the relation is deleted, changes type or now links to another table, its
  back-link lists nothing until that is undone.
* A template saved from B leaves the back-link out: A is not in it.

### The select trap

**A select cell is the choice's `id`, not its label.** Writing `"In progress"`
where the field expects `opt_7c1…` is exactly the dropped-value case in §1: the
row is created and the cell is blank. Pull `options.choices` from `get_table`
and map label → id yourself.

### A choice colour is one of the palette's hex values

`color` on a choice is a **hex colour**, and the app's picker offers six:

| Name | `color` |
|---|---|
| blue | `#DBEAFE` |
| green | `#DCFCE7` |
| yellow | `#FEF3C7` |
| pink | `#FCE7F3` |
| purple | `#EDE9FE` |
| grey | `#E5E7EC` |

Use one of these so the chip matches what a person can pick. Leave `color` out
for a neutral chip. A name (`"green"`) or anything else that is not a hex
colour is **a 400 that names the choice**, on `add_field`, `update_field` and
`change_field_type`. It used to be stored and drawn grey, so the call said it
worked and the person saw no colour.

### Attachment and image are the same column with one extra rule

An `image` column is an attachment column whose every file must be an image.
**A write naming a non-image file is a 400 that names the file**, which is one
of the two places in this surface (with a choice colour, above) where a bad
value is loud rather than dropped.

Attachment cells also have a side effect worth knowing: *"the row then links to
those files in the graph"*. Attaching a file is a knowledge-layer edge, not just
a cell - see `search-and-graph.md`.

### The computed three store nothing

`formula`, `lookup` and `rollup` are derived at read time. Writing to one does
nothing. Reading one gives you the current answer, which can change when the
rows it depends on change.

**A formula names columns as `{Name}`, and a rename
rewrites it.** `update_field(name=...)` changes every `{Old}` in the table's
formulas to `{New}` in the same transaction (exact case, ends trimmed, never
inside a quoted string), so a renamed column keeps its formulas computing.

**A formula's result is in
`computed`, and an error is an object, never a 0.** `get_table_rows` gives each
row `computed`: `{field_id: result}` for every formula column, the same value
the grid shows. A result that cannot be computed is
`{"error": "#REF" | "#DIV/0" | "#ERROR", "message": ...}` - `#REF` names a
column that is not in the table (a deleted one, or a typo), `#DIV/0` divides by
zero, `#ERROR` is anything else (bad syntax, an unknown function, text where a
number is needed, or a loop of formulas naming each other). A formula may name
another formula column and reads its result. **Check for `error` before you add results up**: summing
around it is the wrong total this was built to stop. An empty cell still counts
as 0 in arithmetic. `export_table` writes the same results, an error as its code
in the cell. In a CSV, a text cell that starts with `=`, `+`, `-` or `@` gets a
leading `'` so a spreadsheet does not run it. Numbers are untouched. Lookups and rollups are not in `computed`. A back-link is (above).

**What a formula can say.** Numbers, `"text"`,
`{Column}`, `+ - * / %`, `&` to join text, and comparisons `= != <> < <= > >=`:
two numbers compare as numbers (a checkbox is 1 or 0), anything else as text
with case ignored, and a comparison is `true` or `false` - so
`IF({Amount} > 1000, "Review", "")` works. Functions: SUM, AVG, MIN, MAX,
ROUND, ABS, LEN, CONCAT, IF, AND, OR, NOT, UPPER, LOWER, TRIM, LEFT(text, n),
RIGHT(text, n), TODAY, DATEADD(date, n, unit) and DATEDIFF(start, end,
unit), with unit `"days"` (the default), `"weeks"`, `"months"` or `"years"`.
A date is `YYYY-MM-DD` text, which is what a date column holds. DATEADD answers
one, and adding a month to 31 January gives the last day of February.
`TODAY()` is the reader's own date, so `computed` can change at midnight
without a row changing. Anything else is `#ERROR`. Nothing in a formula is
ever run as code.

**One that could never show anything is refused.**
`add_field` answers 400 for a `formula` with no `expr`, and for a `lookup` or
`rollup` whose `relation` is not a relation column of this table or whose
`field` is not a column of the table that relation links to. Make the relation
column first, read its `options.table_id`, then pick `field` from that table's
`get_table`. A rollup's `agg` is one of `sum`, `avg`, `min`, `max`, `count`.

---

## 3. The primary column

A table has **exactly one**, and that flag is what decides which column cannot
be deleted.

* `delete_fields` is **refused as a whole** if any id is the primary column -
  deliberately, because *"deleting four of five because the fifth was
  undeletable is a destructive half-success with no way to tell it happened."*
* `duplicate_field` copies settings **and values**, immediately to the right,
  and the copy is **never** primary.

Plan a table so the primary column is the one a person would name a row by.
Changing your mind later is not a rename.

---

## 4. Changing a column's type: always dry-run first

`change_field_type` converts every existing cell. The docstring's instruction is
not advice:

> **Call with `dry_run=True` first.** It changes nothing and returns the plan:
> `{allowed, reason, total, convertible, lossy, samples}` - `lossy` is how many
> filled cells the new type cannot hold and would be **CLEARED**. Show that
> number to the person before the real call. The same function computes both, so
> what they agreed to is what happens.

**Show the person `lossy` and `samples` and wait.** Clearing cells is not
yours to decide.

What the conversion matrix allows:

* a computed column (`formula`/`lookup`/`rollup`) **cannot be converted to or
  from**.
* `relation`, `attachment` and `image` can only become **text**.
* **nothing** can become a relation, attachment or image -
* **except** attachment ↔ image: `image → attachment` loses nothing.
  `attachment → image` keeps the images and drops the other files, and `lossy`
  counts those cells.

**An empty column converts to anything.** The
rules above protect values, so when the plan's `filled` is 0 there is nothing
to protect. The new type must still work: a formula, lookup or rollup needs
its options (as in `add_field`), and a relation needs
`options={"table_id": ...}`.

`revert_field_type` undoes it.

### `update_field` can change cells too

Two option edits reach the rows:

* **A select choice left out of `choices` is cleared from every cell that
  held it**, and there is no undo. To rename a choice, keep its `id` and
  change its `name`. **Over a connected app, removing a choice that rows
  still hold needs the person's approval**: the first
  `update_field` is refused with 403 `approval_required`, and its text names
  the exact `request_approval` arguments (the column and the choices, by
  name). Ask, tell the person, wait for `get_approval`, then call
  `update_field` again with the same `options` and `approval_id`. The person
  sees how many rows lose the value. Removing a choice nobody holds,
  renaming and adding need nothing. The loop is workspaces-and-keys.md §11.
* **A relation's `table_id` cannot change while any cell holds a link**: a
  409 with `code: relation_has_links`. The row ids belong to the old table.
  Add a new relation column for the other table instead.
* **A relation links only to a table everyone who reads this one can see**:
   from a Shared drive table, Shared drive tables only. From a My
  drive table, Shared ones and its owner's My drive. Otherwise 400 `Choose
  the table this column links to.`: pick another table, do not retry.

The raw API's `PATCH /tables/fields/{id}` with a new `type` now converts
exactly as `change_field_type` does, with the same refusals and an undo
snapshot. Over MCP, `change_field_type` stays
the way to do it, because it has the dry run.

*A note on trusting docstrings: if a docstring states a LIMIT rather than a
fact, check it.*

---

## 5. Position is a float, and that is the whole ordering model

Both `create_row` and `add_field` take `position`:

> Positions are floats, so to insert BETWEEN two rows pass the midpoint of
> theirs - **no other row moves**.

Omit it to append. This is how you insert without rewriting the order of
everything after, and it is the same trick for columns.

---

## 6. Bulk work, and the caps that exist to stop half-successes

* `create_rows` / `update_rows` / `delete_rows` for more than one.
* **Undo a row delete with `restore_rows(table_id, row_ids)`, never with
  `create_row`.** It brings back the same rows - ids, cells and places - for
  30 days after the delete, so relation cells in other tables and comments
  anchored to their cells reach them again. A re-created row is a different
  row: a new id, at the bottom, and every link to the old one dead. Keep the ids
  you delete. An id already in the table comes back in `already_present`, one
  with nothing to restore in `missing`, and a connected app restores at most
  100 per call.
* `delete_fields` takes a list and is **all-or-nothing** - refused entirely if
  any id is primary, and refused entirely with **409 `linked_to_form_question`**
  if any column holds a live form question's answers (a form's response table must not be deleted out from under it).
* A connected app may delete **at most 100 columns per call**.
* Ids belonging to another table match nothing - they are not an error, they
  simply do not apply, so check what came back rather than assuming a 200 means
  every id was used.
* **A table holds at most 1,000,000 rows**, the
  same on every plan. A call whose new rows would pass it - `create_row`,
  `create_rows`, `import_rows`, `restore_rows` - writes nothing and answers
  **409 `row_cap_reached`** with `max_rows` and `rows`. An import counts only the
  rows it appends, never the ones it updates. Say so to the person. Do not split
  the call to get round it.

---

## 7. Reading a table without pulling all of it

`list_tables` already says what each table holds: `row_count`, `field_count`
and `field_preview`, the first four columns as `{name, type}`. That is often
enough to pick the right table before calling `get_table`.

**`row_preview` shows the first three rows**: each row's
primary cell as `name` and one short column beside it as `detail` (a select
by its choice NAME, with its `color`). `detail` at the top names that column.
It is a glimpse to tell two tables apart, not data to work from: read rows
with `get_table_rows`.

`get_table_rows` pages. Every paged listing in this product has a **total
order**, so a walk never returns a row twice or skips one. Paging is safe
now. It was not always, so do not trust an older client's notes on this.

`export_table` when you want the whole thing as a file rather than as rows.

---

## 8. Views

`create_view` / `update_view` / `delete_view` save a filter, sort and column
selection. **A view does not change the data**. It changes what a person opens
onto. If somebody asks for "a table of open issues", ask whether they want a
filtered *view* of one table or a second table - the first is almost always
right and the second is how a workspace ends up with four copies of the same
rows drifting apart.

**Name the view after what it shows** ("Open issues", "By owner"). Leave `name`
out and the server calls it "Grid" or "Board", numbered when that is taken
("Grid 2"), which is what a person gets from the web app too. A numbered name says nothing about the view, so give one whenever
the request says what the view is for.

### Sorting by several columns, and a board's own order

A view sorts by **levels**: `config.sorts` is a list of
`{field, dir}`, first level first, `dir` `asc` or `desc`, each column once.
"By team, then by age, oldest first" is

    {"sorts": [{"field": "<Team id>", "dir": "asc"},
               {"field": "<Age id>", "dir": "desc"}]}

The older one-level `sort` (`{field, dir}`) still works and the server keeps
it equal to the first level, so do not write the two to say different things:
write `sorts`. A descending level puts its empty cells first, as the web app's
sort does. A malformed `sorts` is a 400 `invalid_sorts` naming the level.
A calculated column (formula, lookup, rollup, back-link) cannot be a level:
`create_view` or `update_view` adding one is a 400 `unsortable_field` naming
it, because its value is worked out when read and orders nothing (the web
app's sort panel does not offer one either). A view saved with one before
that keeps it, so sending back the config you read is not refused, and the
level is skipped when the view is read.

A board (`kind: kanban`) groups its cards by `kanbanField`, which can be a
select, text, number, date, checkbox or other plain column. Its card order is
its own: `config.cardPositions` maps a row id to its place on THIS board, and
a row with no entry sits at its position in the table. To put a card between
two others, give it the midpoint of their positions as they show (their entry,
or their row `position`). The grid and other views keep their order. A sort
wins over the card order. A select board's columns are the select's choices:
add or rename one by editing the field's `options`, never through the view.

`update_view` replaces the whole config, so send the full config back with
your change in it.

### Counting, summing and grouping on the server

**To count, total or group, do not read the rows and
count them.** `query_table` does it on the server, exactly, and returns a
few numbers instead of every row:

    query_table(table_id, group_by="Stage")                       # rows per stage
    query_table(table_id, metrics="sum:Amount", group_by="added:month")
    query_table(table_id, metrics="count,avg:Amount",
                filter='[{"field": "Stage", "op": "is", "value": "<choice id>"}]')
    query_table(table_id, sort="-Amount", limit=5)                # the five biggest

- Columns by name or id. `added` is the day each row was added, in the zone
  of the table's space (the workspace's, or the owner's for a personal
  table), the same day the grid and Flowpy show.
- A date column or `added` takes a bucket: `day`, `week` (its Monday),
  `month`, `year`.
- A multi-select or link cell counts once in each of its values, so group
  counts can add up to more than `matched`.
- Empty cells are left out of sum, avg, min and max, and sort last.
- `sort` by a calculated column (formula, lookup, rollup, back-link) is a 400
  `unsortable_field`, as on `get_table_rows`, unless it is one of the query's
  groups. Sort by a stored column, or group by the formula and sort by that.
- A sum over a text column, an unknown column or a bad bucket is a 400 that
  names it. Fix the query. Never fall back to reading the whole table.

### Counting and summing a spreadsheet or CSV file

An `.xlsx` workbook's, an
OpenDocument `.ods` spreadsheet's, a legacy Excel `.xls` workbook's or a `.csv` / `.tsv` file's rows are kept behind the file when it is indexed, so "the total of
column C in budget.xlsx" is computed, not read off search excerpts.
`query_table` takes the file instead of a table:

    query_table(file_id=<file id>, metrics="sum:C")               # column C by its letter
    query_table(file_id=<file id>, sheet="Q2", metrics="sum:Amount", group_by="Team")

- A column by its header or its letter (`A`, `B`, `C`). Types are `number`,
  `date` or `text`, from the cells. A column with one word among its numbers
  is text, and a sum over it is refused.
- No `sheet` reads the first sheet. The answer lists `sheets` and `columns`.
- A legacy `.xls` workbook is read like an `.xlsx`: its date
  cells are dates. One uploaded before then has rows once it is read again.
  Until then the query says the rows are not ready (`rows_not_ready`).
- A spreadsheet's rows have no `added` date.
- `rows_cut` not empty means only the first `rows_kept` rows were kept (the
  index's row and byte caps): say the answer covers those rows only.
- A total row inside the sheet is a row like any other, so a sum over a
  column that ends in its own total counts it twice. Filter it out.
- 409 `rows_not_ready`: the file is not indexed yet, or its bytes changed
  since. Its rows come back when it is read again. 400 `not_a_sheet`: it is
  not a spreadsheet or CSV.
- The rows have no access of their own: whoever can open the file can query
  them, and nobody else (404).

### Opening a spreadsheet or CSV file as tables

When the person wants to EDIT the data of
an `.xlsx` or `.csv` file in Drive, not just ask about it, `open_file_as_table`
makes new tables from it, one per sheet, linked to the file. The file is not
changed.

    open_file_as_table(file_id=<file id>, dry_run=true)      # what each sheet becomes
    open_file_as_table(file_id=<file id>, sheets=[0])        # make the first sheet's table

- **Data only.** Each column is typed from its own values: number, checkbox,
  date, long text, else text, and a value with a leading zero (a postcode, an
  id) stays text. Each formula becomes the value the file last showed.
  Formatting, merged cells, charts, pictures and comments are left out. The
  dry run's `notice` says so in the person's words: tell them before you
  import, not after.
- The dry run lists every sheet with `rows`, `columns` and `problem`
  (`empty`, `too_many_columns`, `too_many_rows`), and what it drops,
  counted: `formulas` (cells whose formula becomes its value) and
  `merged_cells` (merged ranges split). Say the counts with the notice. Without `sheets`, every
  sheet with no problem is made. Naming one with a problem is refused.
- The first row with anything in it is the header. A blank header is named by
  its column letter, a repeated one gets `(2)`.
- A CSV's separator is its first line's own: comma, semicolon or tab. A
  semicolon-separated CSV is read with comma decimals (`1.234,50`).
- A date written as `10/01/2026` is read as the web import reads it: a day
  over 12 anywhere in the column settles the order for the whole column.
  Otherwise pass the person's order as `date_order` (`DMY`, `MDY`). With
  neither, such a column stays text, and a two-way date beside an
  unambiguous one is kept as typed. Two-digit years are never guessed.
- Month names read in English always (`Oct 17 2026`). Pass the person's
  language tag as `locale` (`de-DE`, `fr`) and that language's names read
  too (`17. Oktober 2026`).
- Up to 10,000 rows in all, the answer is `{tables, created_rows}`. More, and
  it is `{job}`: the tables are made in the background, up to the 1,000,000
  row cap. Read `get_open_as_table_job(job_id)` until `status` is `done`
  (`table_ids`) or `failed` (`error`, and no table is kept). The person also
  gets a notification. A job whose worker stopped is failed by a sweep within
  the queue's time limit plus a quarter of an hour. Open the file again.
- Files up to 50 MB. A password-protected workbook is refused (`password`),
  and so is one that cannot be read (`unreadable`). A file with no sheet that
  has data is `nothing_to_import`, and tables that would pass the account's
  row cap are `row_cap_reached` (409). A background job checks again when it
  runs and fails, with `error` saying why, when the file went to Trash first
  ("The file was moved to Trash or deleted before it was opened.") or the
  person lost access to it, or to the shared folder the tables were going
  into, while it waited ("You no longer have access to this file.").
- Saving the table back into the file, as a new version of it, is the
  person's choice in the app, asked each time. `export_table` with
  `save_to_folder` always makes a new file.

### Reading a filtered slice without making a view

**Do not create a view to answer a question.** A view is
something a person opens onto. Leaving one behind because you needed to count
open issues once is clutter in somebody else's workspace.

`get_table_rows(table_id, filter=...)` narrows the read on the server, and the
`filter` is the same clause shape a view stores:

    get_table_rows(
      table_id,
      filter='[{"field": "<field id>", "op": "is", "value": "open"}]',
      limit=100)

Three things to know, and the first is the one that matters at scale:

- **Filter, then page - not the other way round.** `count` comes back as the
  number of MATCHES, so you know immediately whether the answer is 4 rows or
  4,000. Paging the whole table and discarding rows yourself reads 31,000 rows
  to answer about 12, and spends the context window on the 30,988 you threw
  away.
- **Field IDS, not column names.** `get_table` maps one to the other. Names are
  editable and ids are not, which is why every write on this surface is keyed
  the same way.
- **A bad field or operator is a 400 that says which one.** It is deliberately
  not forgiven, because the forgiving answer - hand back everything - is
  indistinguishable from *this table has no filter applied* and you would
  report the wrong number with total confidence.

Operators: `contains`, `is`, `is_not`, `gt`, `lt`, `gte`, `lte`, `before`,
`after`, `has`, `has_not`, plus `empty`, `not_empty`, `checked`, `unchecked`
which take no value. **To filter by a linked record**, use `has` or `has_not`
on the relation column with the linked row's id ("tasks whose Project is
Apollo"). `has_not` leaves out rows with no link at all. `query_table` also
takes the record by the name its cell shows, and refuses a name two linked
records share with `ambiguous_linked_record`, so use the row id then. On a
table shared with the person that links to a table they were not given,
`get_table_rows` takes the linked row's title too, and both look it up only
among the rows already linked.
Conditions are ANDed. There is no OR, and if somebody
needs one, two reads and a union in your own head is the honest answer rather
than a saved view nobody asked for.

### Finding rows by what they say

`get_table_rows(table_id, q="acme")` keeps
the rows with a cell that contains the text, ignoring case: the web grid's
find bar, run on the server over every row. Use it when the person names a
value but not a column ("the row about Acme"). Use `filter` when you know the
column.

- **What a cell says, not what it stores.** A select matches by its option's
  name, a checkbox reads `yes` or `no`, a number as written (`3`, not `3.0`).
  An attachment, image or relation cell holds ids and never matches, and a
  calculated column is not searched.
- **The whole text, as one piece.** `q="ada love"` finds "Ada Lovelace", not
  "Love, Ada". For words in any order, read twice or use `filter` with
  `contains` on the column.
- **It combines.** `filter` and `q` must both hold, then `sort`, then the page,
  so `count` is the number of matches and `has_more` says whether to read on.

### Reading rows in an order

`get_table_rows(table_id, sort=...)` orders the
read on the server, with a view's sort levels written on one line:

    get_table_rows(table_id, sort="<stage field id>:asc,<amount field id>:desc", limit=100)

- **Filter, then sort, then page.** Every page continues the same order, so
  the "top 10 by amount" is the first page, not the top 10 of whatever page you
  happened to hold. Sorting a page yourself is right only when you hold every row.
- **The grid's order, not Python's.** Empties go last ascending and first
  descending, a single select sorts by its choices' order (High, Medium, Low),
  text naturally and blind to case and accents (Event 2 before Event 10, Élan
  beside Elan), a list cell (multi select, attachment, relation) as its values
  joined with a comma, and one with no values left (`[]`) as empty, last.
  Punctuation and symbols among themselves follow the grid's order too.

- **A calculated column cannot be sorted by** (formula, lookup, rollup,
  back-link): its value is worked out when read, so `sort=` naming one is a
  400 `unsortable_field`, and the web grid does not offer it either. A saved
  view that already sorts by one still works and skips that level. To order by
  a computed value, read the rows and sort them yourself.
- Field IDS again, `asc` or `desc` only, each column once: anything else is a
  400 that names it.

---

## 9. Publishing, and what is not yours

`get_table_publish_status`, `publish_table`, `unpublish_table` and
`get_public_table` are on this surface - but over a **connected app** (Claude or
any other MCP connection) publishing is deliberately unreachable, along with
every other make-public action. If a person asks, say it is done in the
FlowpyDisk app and point them there. It is not a gap in the connector.

`set_table_upload_folder` decides where a table's attachments are filed. Ask
rather than scattering them at the root.

### A table shared with a person

A table can be shared with one person by email, at
viewer or editor, beside publishing it, from the table's Share in the app. **No
tool makes the share, with a key or a connection.** `list_table_permissions`
reads who has it (held shares for addresses with no account included), and
`revoke_file_permission` withdraws one with a key, using the row's `id`.

`get_table` on a table shared WITH the person answers `permission: "viewer"` or
`"editor"` (`"owner"` on their own) and `can_edit_structure`, and
`shared_by_name` names who shared it: the person behind
the grant, or the table's owner for a table placed on a board shared with them.
Null on their own. A viewer only
reads, exports and copies, so do not offer edits. An editor, of a table shared
on its own or through a shared folder, changes rows, cells, columns, views and
the table's name (`can_edit_structure` is true).
Nobody but the owner deletes, publishes, re-shares or saves it as a template.

The files in its cells **show through the table only**: `GET
/tables/{id}/files?ids=` names them and `GET /tables/{id}/files/{file_id}`
serves them, but the file tools, search and the person's Drive never reach
them. A relation or back-link to **another** table that was not shared too
holds that table's row ids, and each row read carries their **titles only**
in `linked_titles`: `{field id: {linked row id: title}}`. Name a linked row by that title. Nothing else of the other table
is reachable: it is a 404, so do not try to read it, and lookups and rollups
over it stay blank. `export_table` and a grouped `query_table` write the
titles. A filter (`get_table_rows`, `query_table`, or a saved view when
exporting) may name a linked row by that title: it is found among the rows the
column already links to, never among the other table's other rows. A title two linked rows share is refused with
`ambiguous_linked_record`: use the row id from the cell then. A person who can open
the other table gets no `linked_titles`: read it as usual. In `list_shared_with_me` a shared table is a row with
`kind: "table"`, opened with `get_table`.

In a table shared with the
person as an editor, an attachment or image cell naming one of the person's
**own** files would share that file with the table's owner, so with a key or a
connection the write is a 400 with `code: own_file_on_shared_table` and nothing
is written. A file in the shared folder, or one already in the column, is
written as usual. The person attaches their own file in the app.

---

## 10. A worked order

For "put this spreadsheet into a table":

1. `create_table`. It comes with `Name`, `Notes` and `Status` columns and
   **no rows**, so there is nothing to delete first. Rename or delete the starter columns you do not
   want.
2. `add_field` per column, **in order**, choosing types deliberately - a date
   that arrives as text can only become a date through a lossy conversion later.
3. For every select column, create the choices in `options` and **keep the
   id map** the response gives you.
4. `get_table` and build name → field id.
5. `create_rows` for the first handful only.
6. **`get_table_rows` and look at them.** Blank cells here mean the mapping is
   wrong, and this is the cheapest moment to find that out.
7. The rest with **`import_rows`**: one call, one transaction, so a refused
   value anywhere writes nothing and you never have to work out which batches
   landed. It can also make plain columns the rows need on the way in
   (`new_fields`: text, long_text, number, date or checkbox, each with a `key`
   the rows use instead of a field id). Select, relation and file columns are
   made with `add_field` first. Up to 10,000 rows with an API key, 100 over a
   connected app, where you batch it with `import_rows` calls of 100.

For "here is the updated sheet, bring the table up to date" (the table already
has these rows):

1. Pick the column that identifies a row, an email or an order number, and
   pass its field id as **`match_field`**. It must be text, email, url,
   number, currency, percent, date or single_select.
2. Call `import_rows` with **`dry_run: true`** first. It writes nothing and
   answers `{created, updated, new_fields}`: say "12 rows will be updated and
   3 added" before you do it.
3. Then the same call without `dry_run`. A row whose key equals one existing
   row's value updates it. The rest are appended. Only cells with a value are
   written: an empty cell keeps what the table has, so a sheet cannot blank a
   column by leaving it empty.
4. It refuses the whole import, writing nothing, when a row has no key
   (`empty_key`, with `rows`), the same key is in your rows twice
   (`duplicate_key`), or a key is held by two or more rows of the table
   (`ambiguous_key`, with `keys`). Tell the person which. Do not drop rows to
   get past it. Keys are trimmed and case-sensitive, and numbers compare as
   numbers ("5.0" matches 5).
5. Over a connected app (100 rows per call) the duplicate check sees one call
   at a time: a key repeated across two calls updates, in the second call, the
   row the first call added. Check the whole sheet for repeated keys yourself
   first.

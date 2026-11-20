# Writing FlowpyDisk notes over MCP - the agent playbook

---

## 1. What a note is

A note is a `.md` file. Its text is Markdown **plus a small set of HTML
blocks and attributes** that the FlowpyDisk editor writes and reads. Three
things render that text and they do not agree everywhere: the editor (what a
person sees and edits), the published page (what a visitor sees at a public
link) and the file preview. This playbook gives the form that works in all
three. When the three disagree, it says so.

The **title of a note is its own `title` field, or its filename without
`.md` when that is empty**. It is not taken from the first heading. A body
that opens with `# <the title>` is shown once in the web editor, which folds
that heading into the title, but the file still holds
it, so every other reader of the text - the published page, an export,
`read_note` - sees it as a heading. Leave it out unless you want that.

**Renaming a note renames that heading too.** When a note's first block is
`# <its current title>`, changing what the note is called - its `title`
through `set_file_metadata`, or, for a note with no title of its own, its
filename through `rename_file` - rewrites that heading to the new name, so
the two never disagree. Only the words change. Front matter, the `#` and the
rest of the text stay byte for byte, and the text before the rename is kept
as a version (`list_file_versions`). A note without that heading is never
given one, and a titled note's filename can change without touching its
text. So after a rename, `read_note` before editing: the text has moved.

## 2. The tools

| Do this | Tool | Notes |
| --- | --- | --- |
| Create a note | `create_note(name, content="", folder_id=None)` | `.md` is appended if missing. `folder_id` omitted puts it at the root. Name collisions get ` (2)`, ` (3)`. 5 MB cap. |
| Make a note from a Word document | `convert_docx_to_note(file_id)` | A NEW note beside the .docx, named after it. The document is not changed. Headings, emphasis, lists, links and tables come across as the Markdown below. Pictures are saved as image files in the same folder and shown in the note as `![alt](/api/v1/storage/files/<id>/raw)`. The result's `images_saved` and `images_skipped` say how many. A skipped one (EMF/WMF/TIFF, over the upload cap, no room in storage) is a visible `*[Image: alt text]*` line - say so rather than claiming the note has every picture. Only to make something EDITABLE: to read or quote a .docx, search already sees its text. The note's `source_file` names the document it came from. |
| Put a note back into its Word document | `save_note_back_to_docx(file_id)` | Only for a note made by `convert_docx_to_note` (its `source_file` is set). Writes the note, built as `export_note` builds it, INTO that .docx as a new version of the same file: same id, name and links, and the previous .docx stays in its version history (`restore_file_version` puts it back). Say before you do it that the document's layout, headers, footers, comments and Word styles become what the note holds. Refused, the document untouched: `source_changed` (it changed after the note was made, or since the last save back: someone's work would be overwritten), `source_trashed`, `source_deleted`, `source_read_only`, `not_from_docx`. Then offer `export_note(file_id, save_to_folder=...)`, a new .docx, instead. |
| Give someone a Word copy | `export_note(file_id, format="docx", save_to_folder=None)` | Built from the note's SAVED text. With `save_to_folder` the .docx lands in Drive and you get its File. Without, a `download_url` that needs the person's own session. For a link anyone holding it can open, use `get_note_export_url(file_id)` instead: `{url, expires_at}`, about 5 minutes, read access checked again when opened. Headings, emphasis, lists, links, code, quotes and tables carry. The person's own images embed. An image from another site becomes a line naming it. Callouts and columns come across as their text. |
| Replace the whole text | `edit_file_content(file_id, content, if_match?)` | Whole-file replace. Pass `if_match` with the `content_etag` from `read_note`: if a person saved in between, the write is refused with 412 (`precondition_failed`) instead of overwriting them, and you read again and redo the change. Every real change keeps the old text as a version. A byte-identical save is a no-op. 5 MB cap. |
| Append to a note | *(no tool)* | `read_note`, add to the text, then `edit_file_content` with the whole thing and the `content_etag` as `if_match`. |
| Read a note's text | `read_note(file_id)` | Returns `content`, `name` and `content_etag`. Up to 5 MB of text. Larger or binary files need `download_file(file_id, dest_path)` on a local run, or `get_download_url(file_id)` on the hosted server. `get_file(file_id)` is metadata only. A published note's text also comes back from `get_public_note(slug)`. |
| Find a note | `search_files(query)`, `search_everything(query)`, `semantic_search(query)` | Titles, and text once indexed. An edit's words are found by keyword within seconds. `semantic_search` finds them about five minutes after the editing stops, or ten minutes into an unbroken session (a new note at once). |
| Skim several notes | `list_files(...)` | Each `.md` row carries `preview`: its first lines as a reader sees them (no Markdown markers, no heading that repeats the title), about 200 characters, "" until the text is indexed. And `checklist`: `{done, total}` over its task items (`- [ ]`, `- [x]`), null when it has none, so "3 of 5 done" needs no read. `read_note` for the whole text. |
| Who links here | `get_file_mentions(file_id)` | Linked `[[wiki-links]]` and unlinked title mentions. The title is the note's own title when set, else its file name without `.md`. Each snippet is plain text - the mention's sentence as a reader sees it, no Markdown or front matter - so `before + match + after` can be quoted as it is. An unlinked mention is plain text only: the title inside code, a link, an @mention, a URL, an HTML tag or front matter is not listed. Each unlinked entry says `can_link`: whether you may link it. In the personal space it reads notes shared with you as well: a shared note lists its mentions, and a shared note can be a source. |
| Link an unlinked mention | `link_mention(source_id, target_id, if_match?)` | `source_id` is an `unlinked` entry's `file_id`, `target_id` the note you asked `get_file_mentions` about. The first plain mention of the title becomes `[[that text]]`, the words kept as written (`[[File name|that text]]` when the title differs from the file name, since a link names the file), never one inside a link, code, a URL or front matter. Only when `can_link` is true: the source must be a note you may edit. Link one when the person asks, not every one you find: each is an edit of somebody's note, kept as a version. `changed: false` means it was already linked. |
| Versions | `list_file_versions(file_id)`, `get_file_version(file_id, version_id)`, `restore_file_version(file_id, version_id)` | `restore_file_version` puts a version back, byte for byte, and keeps what it replaced as a version too. |
| Comment on a note | `create_comment(file_id, body, parent_id=None)` | Mention a person as `@[Display Name](user-uuid)`. Comments are separate from the note's text. |
| Mention a person in the text | `list_mentionable_users(file_id)`, then `@[Name](id)` in the content | Saving tells the people the text newly mentions who can open the note (4.22). |
| Publish a note | *(never over a connector)* | A person does it in the app. `get_publish_status(file_id)` tells you whether it already is. |

The server stores the string you send as UTF-8 bytes. **Nothing validates or
transforms your Markdown.** What you write is what the file holds, so the
rules below are yours to keep.

## 3. Rules that bite

Read these before the syntax reference. Each one has produced a wrong note.

1. **A single newline is a line break, not a soft wrap.** The editor runs
   with `breaks: true`. To continue a paragraph, keep writing on the same
   line. To start a new paragraph, leave a blank line. The editor stores a
   forced break as a trailing backslash (`line one\` then `line two`). Write
   it that way if you want one.
2. **Inside a callout, a column or a details block, write HTML, not
   Markdown.** The published page hands the inside of those blocks straight
   to the HTML sanitiser, so `**bold**` inside a callout is shown as
   literal asterisks to a visitor. Use `<p>`, `<strong>`, `<em>`, `<ul><li>`,
   `<h3>`, `<img>`, `<a>`, `<code>`, `<pre>` inside them.
3. **Highlight is `<mark>text</mark>`.** `==text==` renders on a published
   page but is dead text in the editor.
4. **Underline is `<u>text</u>`.** Markdown has no underline.
5. **Do not mix to-do items and plain bullets in one list.** Put a blank
   line between them so they are two lists. An empty to-do (`- [ ] ` with
   nothing after) does not survive a save.
6. **Front matter only at line 1, and only with a closing `---`.** A `---`
   anywhere else is a divider.
7. **Only `[[wiki links]]` and `[^footnotes]` survive with square brackets.**
   Any other bracket syntax you invent comes back backslash-escaped.
8. **No `<iframe>`, `<script>` or `<svg>`.** They are stripped. An embed is
   a marker div (section 4.16) and only YouTube, Vimeo, Loom, CodePen,
   Figma, published Google Slides, Docs and Sheets, Microsoft 365 embed
   links and Miro live embeds are framed.
9. **Unknown attribute values are dropped, not passed through.** An unknown
   callout type becomes `info`. An unknown tone, font role, divider style or
   alignment becomes the default. Use the vocabularies listed here exactly.
10. **A table needs a header row and a `| --- |` line.** Without them, or
    with merged cells, it stops being a pipe table and is stored as a much
    larger HTML `<table>`.
11. **Footnotes are not rendered.** `[^1]` stays as text. Page breaks do not
    exist.
12. **HTML comments are kept in the file and never shown to a reader.** Use
    them for agent-only metadata.

## 4. Syntax reference

Everything below is the exact form the editor stores. **Copy the SHAPE, not
the words.** The sample text in these blocks describes what each construct is
for. It is not content for your note. An agent following an earlier draft of
this document pasted "Keep writing on this line to stay inside the same
paragraph" into a real note about a holiday, because the example said it. Your
note's words are yours to write.

### 4.1 Headings

```markdown
# Heading 1
## Heading 2
### Heading 3
```

Levels 1 to 6 parse. The editor's menu offers 1 to 3. A centred or
right-aligned heading is HTML:

```html
<h2 style="text-align: center">Centred heading</h2>
```

`left` is the default and stays plain Markdown.

### 4.2 Paragraphs and line breaks

```markdown
The first paragraph's text, all on one line however long it runs.

The second paragraph's text, after a blank line.

The first line of an address\
the second line, after a forced break.
```

An aligned paragraph is HTML: `<p style="text-align: center">Centred.</p>`.
Alignments: `center`, `right`, `justify`.

### 4.3 Inline marks

```markdown
**bold** *italic* ~~strike~~ `code` [link text](https://example.com)
```

```html
<u>underline</u>
<mark>highlight</mark>
<span data-font="serif" style="font-family:'Crimson Pro', Georgia, serif">a serif run</span>
<span data-ink="#B42318" style="color:#B42318">red ink</span>
```

A bare `https://…` in prose becomes a link on its own.

Font roles for `data-font`: `sans display serif hand rounded condensed slab
mono grotesque editorial book print geometric narrow slabtext code`. The
`style` is what a browser reads. The `data-font` is what the editor reads.
Write both.

Ink colours for `data-ink` (upper-case hex, exactly these): `#20242C`
`#5B6472` `#B42318` `#B54708` `#067647` `#1849A9` `#6941C6`.

### 4.4 Bullet list

```markdown
- one
- two
  - nested under two (two spaces)
    - nested again
```

### 4.5 Numbered list

```markdown
1. first
2. second
   1. nested under second (three spaces)
```

Two numbered lists touching each other alternate `1.` and `1)` so they stay
separate. A blank line between them is enough. Do not worry about the
marker.

### 4.6 To-do list

```markdown
- [ ] not done
- [x] done
  - [ ] a nested step
```

Keep to-dos and plain bullets in separate lists (rule 5).

### 4.7 Blockquote

```markdown
> A quotation. Keep it on one line, or use a forced break.
```

One level only on the published page.

### 4.8 Code block

````markdown
```python
print("hello")
```
````

Always give a language. The editor assigns one anyway. `~~~` fences are
accepted but stored as backticks.

### 4.9 Divider

```markdown
---
```

Styled dividers are HTML:

```html
<hr data-variant="dashed" data-width="half">
```

`data-variant`: `line dashed dotted thick`. `data-width`: `full half short`.
Omit either for its default.

### 4.10 Table

```markdown
| Metric | Q2 | Q3 |
| --- | --- | --- |
| Revenue | 10 | 14 |
| Cost | 6 | 7 |
```

Header row and the `| --- |` line are required. Cells hold inline text only.
For a line break inside a cell write `<br>`. A table set in a typeface is
wrapped, and the blank line after the opening tag is required:

```html
<div data-block-font data-font="serif">

| A | B |
| --- | --- |
| 1 | 2 |
</div>
```

### 4.11 Image

```markdown
![Alt text](https://cdn.example.com/chart.png)
![Alt text](https://cdn.example.com/chart.png "Title")
```

With a width, an alignment or a caption it is HTML:

```html
<img src="https://cdn.example.com/hero.png" alt="Hero" width="480" data-align="center" data-caption="Figure 1">
```

`data-align`: `left center right`. A captioned image publishes as a figure.

### 4.12 Callout

```html
<div data-callout="info" class="note-callout">
<p>Body text. <strong>HTML</strong> inside, not Markdown.</p>
</div>
```

Types: `info success warning danger note`. That is the whole set. There is
no emoji, icon or colour attribute. The icon and colour follow the type.
`class="note-callout"` may be omitted when writing.

### 4.13 Columns

```html
<div data-columns="" class="note-columns">
<div data-span="8" data-column="" class="note-column"><p><strong>Wide</strong> column.</p></div><div data-span="4" data-column="" class="note-column"><p>Narrow column.</p></div>
</div>
```

- Two, three or four columns. Nothing else.
- `data-span` on a column: 1 to 12 on a 12-unit grid. Omit for equal widths.
  Common pairs: 8/4, 4/8, 9/3, 3/9.
- `data-tone` on the block or on one column: `sand sky moss blush iris slate
  teal ember`. A key, never a hex colour.
- `data-font` on the block: one of the font roles in 4.3.
- `data-align` on the block: `center bottom stretch`.
- Content inside a column is HTML (rule 2). Columns collapse to two on a
  tablet and one on a phone by themselves.

### 4.14 Details (a toggle)

```html
<details class="note-details">
<summary>Assumptions</summary><p>Headcount flat. HTML inside.</p>
</details>
```

Open by default: `<details open="" class="note-details">`.

### 4.15 Maths

```markdown
Inline: $a^2 + b^2 = c^2$.

$$
E = mc^2
$$
```

The opening `$` must not be followed by a space and the closing `$` must not
be preceded by one or followed by a digit, so prices like `$5 and $10` stay
prose. A block formula is centred by default. A left- or right-aligned one
is `<div data-math="E = mc^2" style="text-align: right"></div>`.

### 4.16 Embed (video and design tools)

```html
<div data-embed="https://youtu.be/dQw4w9WgXcQ" data-width="640" data-align="center" class="note-embed-node"></div>
```

Store the URL the author would paste, never a player URL. Framed providers:
YouTube, Vimeo, Loom, CodePen, Figma, and Google Slides, Docs and Sheets at
their published address (`docs.google.com/.../d/e/...`, from File, Share,
Publish to the web. An `/edit` address stays a link), Microsoft 365 at its
embed address (`onedrive.live.com/embed?resid=...` or SharePoint's
`_layouts/15/Doc.aspx?sourcedoc=...&action=embedview`, from Share, Embed. A
sharing link stays a link) and Miro at `miro.com/app/live-embed/<board>/`. Any other URL publishes
as a plain link. `data-width` is pixels, at least 160. `data-align`: `left center
right`. The div is empty on purpose.

**A bookmark card** (a link drawn with the page's title, summary and site) is a marker div too:

```html
<div data-bookmark="https://example.com/guide" data-title="The guide" data-description="What it covers." data-site="Example"></div>
```

Only `data-bookmark` is required. A card with no title shows its address.
The web app fills the words and a picture when a person adds the card. A
`data-image` you write is dropped unless it is the app's own copy, so leave
it out.

### 4.17 Uploaded video or audio

```html
<video src="https://…/clip.mp4" controls="true" preload="metadata" class="note-video"></video>
<audio src="https://…/take.mp3" controls="true" preload="metadata" class="note-audio"></audio>
```

### 4.18 Attachment (a file in the Drive as a chip)

```html
<div data-attachment data-file="7f3a1b2c-4d5e-6f70-8901-abcdefabcdef" data-kind="doc"><a href="https://…/download">Budget.xlsx</a></div>
```

`data-file` is the file's id. The link text is the name shown. Do not write
a size.

### 4.19 Links between notes and other things

```markdown
[[Roadmap]]                     link to the note named Roadmap
[[Roadmap|the plan]]            same link, shown as "the plan"
[[Roadmap#Risks]]               the heading part is accepted and dropped
[[table:Sales]]                 a table
[[folder:Clients]]              a folder
[[canvas:Q3 board]]             a canvas
[[file:logo.png]]               a file
![[Roadmap]]                    embed the note's content here
![[table:Sales]]                embed the table here
```

Write the note's title without `.md`. Only `table:`, `folder:`, `canvas:`
and `file:` are prefixes. Any other word before a colon is part of the
title. An `![[…]]` alone on its own line renders as a block. Inside a
sentence, inline. These links are what the knowledge graph reads, so use
them whenever one note refers to another.

### 4.20 Front matter

```markdown
---
title: Quarterly plan
tags: [q3, planning]
---
```

Only at the very top, only with the closing `---`. Kept byte for byte.
Nothing in the product reads its keys today. It is for you and for other
tools.

### 4.21 HTML comment

```html
<!-- agent: last synced 2026-09-08 -->
```

On its own line. Kept in the file, stripped before any reader sees the note.

### 4.22 Mentioning a person

```markdown
Ask @[Priya Rao](3f2a9c1e-0b7d-4e55-9a10-6c2b1d4e8f00) about the launch.
```

The same markup a comment uses. Take the id and the name from
`list_mentionable_users(file_id=<the note>)`, never from anywhere else: that
list is exactly who can open the note. When `create_note` or
`edit_file_content` saves the text, each person it NEWLY mentions gets a
`mention` notification quoting the line, `data.kind` `note_mention`. Not you,
not someone already mentioned in the text you replaced, and not someone who
cannot open the note: the save still succeeds and they are simply not told,
so mentioning never shares a note. So do not re-add a mention to "remind"
somebody: it does nothing until it has been removed and saved first.

Anything served outside the note writes a mention as plain `@Name`, with no
account id: a published note (and so `get_public_note`), a site page, a note
embedded in one, a Discover teaser. `read_note` still returns the markup. The
web editor's own mention support is being built. Until it ships the
editor may show the markup as a plain link.

## 5. A complete example

Everything above in one note, exactly as it should be sent to `create_note`
with `name="Quarterly plan"`.

````markdown
---
tags: [q3, planning]
---

A paragraph with **bold**, *italic*, ~~strike~~, `code`, <u>underline</u>,
<mark>highlight</mark> and a [link](https://example.com).

<p style="text-align: center">A centred line.</p>

<!-- agent-only: generated from the Q3 brief -->

## Goals

- Ship the connector
  - Consent screen
  - Revocation
- Close the forms round

## This week

- [ ] Draft the plan
- [x] Book the review

## Steps

1. Collect numbers
2. Write it up
   1. Circulate

> Plans are worthless, planning is everything.

```python
print("hello")
```

| Metric | Q2 | Q3 |
| --- | --- | --- |
| Revenue | 10 | 14 |

![Chart](https://cdn.example.com/chart.png)

<img src="https://cdn.example.com/hero.png" alt="Hero" width="480" data-align="center" data-caption="Figure 1">

---

<hr data-variant="dashed" data-width="half">

Inline maths: $a^2 + b^2 = c^2$.

$$
E = mc^2
$$

<div data-callout="warning" class="note-callout">
<p>Deadlines move on Friday.</p>
</div>

<div data-columns="" class="note-columns" data-tone="sky">
<div data-span="8" data-column="" class="note-column"><p><strong>Wide</strong> column.</p></div><div data-span="4" data-column="" class="note-column"><p>Narrow.</p></div>
</div>

<details class="note-details">
<summary>Assumptions</summary><p>Headcount flat.</p>
</details>

<div data-embed="https://youtu.be/dQw4w9WgXcQ" data-width="640" data-align="center" class="note-embed-node"></div>

See [[Roadmap]], [[table:Sales|the sales grid]] and [[canvas:Q3 board]].

![[Roadmap]]
````

## 6. Quick reference

| Thing | Write |
| --- | --- |
| Heading | `## Text` · aligned: `<h2 style="text-align: center">Text</h2>` |
| Paragraph | plain text. Blank line between paragraphs. Forced break = trailing `\` |
| Bold, italic, strike, code | `**b**` `*i*` `~~s~~` `` `c` `` |
| Underline, highlight | `<u>u</u>` `<mark>h</mark>` (`==h==` also reads as a highlight. The editor saves it as `<mark>`) |
| Font, colour | `<span data-font="serif" style="font-family:…">` · `<span data-ink="#B42318" style="color:#B42318">` |
| Link | `[text](url)` |
| Bullets | `- item`, nest with two spaces |
| Numbers | `1. item`, nest with three spaces |
| To-do | `- [ ] todo` `- [x] done` |
| Quote | `> text` |
| Code | ```` ```lang ```` … ```` ``` ```` |
| Divider | `---` · `<hr data-variant="dashed" data-width="half">` |
| Table | pipes with a header row and `\| --- \|` |
| Image | `![alt](src)` · `<img src alt width data-align data-caption>` |
| Callout | `<div data-callout="info\|success\|warning\|danger\|note">HTML</div>` |
| Columns | `<div data-columns><div data-column data-span="8">HTML</div>…</div>` (2 to 4) |
| Toggle | `<details [open]><summary>…</summary>HTML</details>` |
| Maths | `$x$` · `$$` on its own lines |
| Embed | `<div data-embed="URL" data-width data-align></div>` |
| Bookmark card | `<div data-bookmark="URL" data-title data-description data-site></div>` |
| Attachment | `<div data-attachment data-file="ID" data-kind="K"><a href="URL">Name</a></div>` |
| Note link | `[[Note]]` `[[Note\|Alias]]` `[[table:X]]` `[[folder:X]]` `[[canvas:X]]` `[[file:X]]` |
| Embed a note | `![[Note]]` |
| Mention a person | `@[Name](user-id)` from `list_mentionable_users`. Notifies them on save |
| Front matter | `---` yaml `---` at line 1 only |
| Comment | `<!-- text -->` |
| Not supported | footnotes, page breaks, `<iframe>`, `<svg>` |

## 7. A working method for an agent

1. Decide the title. It is the filename. Do not repeat it as a heading.
2. Write the body with the syntax above. Prefer plain Markdown. Reach for
   the HTML blocks only for callouts, columns, toggles, embeds and styled
   images.
3. Inside any HTML block, write HTML.
4. Link to other notes and tables with `[[…]]` so the graph connects them.
5. Call `create_note`. Keep the returned `id`.
6. To change the note later: `read_note`, edit the whole string, and send it
   back with `edit_file_content`, passing the `content_etag` you read as
   `if_match`. Never send a fragment. On 412 somebody saved in between: read
   again and redo your change on their text.
7. Ask the person to open the note in the app once. It will render exactly
   as written, and a save from the editor normalises anything you got
   slightly off.

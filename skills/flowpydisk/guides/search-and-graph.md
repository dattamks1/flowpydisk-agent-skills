# Search and the graph

Sixteen tools across three different questions, and the mistake that costs most
is picking the wrong one of the three.

---

## 0. Start with `search`

`search` is the first call for finding anything. It searches ONE space (the
one you act in, or the one you name) by words and the graph together, and
adds semantic search (by meaning) only when you pass `semantic: true` and the
space allows it (its plan includes AI and AI is on there. It costs AI
credits). Each result says `why` it was found (name, content, graph,
semantic) and `next`, the tool
that opens it. `count` is the real total. Page with `next_cursor`.

The switch and the source are called `semantic`, where
both said `meaning`. `meaning: true` still works as a deprecated alias, but
send `semantic`.

When semantic search was asked for and did not run, the answer still comes
from words and graph, and `notices` says why: `semantic_not_on_plan`,
`semantic_off_in_space`, `semantic_out_of_credits` or
`semantic_index_behind`. Say that to the person in its words, do not retry.
From an outside app on a plan without the graph, `sources` lacks `graph` and
`notices` carries `graph_not_on_plan`. The same notice, with its own message, says the space's plan has the graph in the app off (`graph_basic`). Search then ranks by words, content and meaning alone.

A name match needs every word typed.
Content search matches the query's meaningful words: filler such as "the",
"that" or "de" is dropped, never every word. A table found by its rows
carries `matches`, up to 5 matching rows in the table's order (each `{row_id,
field_id, field, snippet}`), and `match_count`, how many rows match in all.
`match` is the first of `matches`. The table is still one result. Say "5 of
12 rows" when `match_count` is larger than the list, and read the table for
the rest.

The three searches below stay for when you need one on its own.

## 1. Three searches, and which question each answers

| You want | Call | What it costs |
|---|---|---|
| **a thing you can name** - "the supplier table", "dog.jpg" | `search_everything` | nothing. Literal, always available |
| **how things relate** - "what else is about this" | `graph_search`, `get_graph_around`, `get_related_files`, `get_graph` | nothing. Deterministic, always fresh. Over MCP it needs a plan that includes the graph for outside apps (see below) |
| **something you can only describe** - "what did we agree about refunds" | `semantic_search` | an embedding call, charged to the person's AI credits, and it can be unavailable entirely |

The rule behind that table is a product rule, and it is worth carrying because it
is the opposite of what most retrieval stacks do:

> **The graph is used everywhere, always, by everyone. What is paid for is the
> ENRICHMENT, not the graph.**

An enriched graph ranks better. An unenriched one still works, through the same
code path. So there is never a case where skipping the graph is the cheaper
answer - reach for it first and reach for embeddings when the question actually
needs meaning.

**Except from outside the app on some plans.** The graph tools
(`get_graph`, `get_graph_around`, `graph_search`, `get_related_files`,
`get_file_mentions`, `rebuild_graph`) are refused to an API key or a connected
app with `plan_capability_off` when the plan of the space's owner does not
include the graph for outside apps. Inside the FlowpyDisk app
the graph is always there. On such a refusal, say so in the message's words and
fall back to `search_everything` and `semantic_search`.

**Start with `search_everything`.** It is literal, it needs no embedding key and
no vector database, and it is the fastest way to find something whose name the
person just told you. Going straight to `semantic_search` for "the supplier
table" is slower, more expensive, and worse.

**Files come name first.** A file named or titled
for the words outranks one that only mentions them: the name being the words,
then starting with them, then containing them, then a match only in the text,
description or tags. Each file hit says which in `matched_in` (`name` or
`text`). A `text` hit with an unrelated name is there because of its body, and
its `snippet` quotes why.

**Every word, any order.** `search_everything`
and `graph_search` match a name when each word of the query is in it, in any
order and position ("leads vivre" finds "Vivre CRM B2B Leads"), case-insensitive
but not accent-insensitive. When no name holds every word they return the
closest names instead, one typo in a word of five letters or more, or at least
half the words of a longer query, and say so with `fallback: true`: report
those as close matches, never as the thing the person named.
The closest-names step ignores accents.
**So does the first step**, in every literal
search (`search_files`, `search_everything`, `graph_search`, Collab search):
"Mexico" or "cafe menu" matches "México" or "café-menu.txt" as a real match,
`fallback: false`, in names, titles, descriptions and tags, and the name
ranking reads the folded name. Letters with no accent to take off ("ø",
"ł", "æ", "ß") still match only as written.
**A file's text too** (`search_files`, Ask AI's
keyword search): "cafe" finds a body that says "café", and "café" one that
says "cafe". A passage indexed before the change answers "cafe" only once it
is stored again (a one-off maintenance step). "Café" still finds it.

**`search_files` too.** It already needed every
word, in any order. When no file name, text, description or tag holds them,
it now returns the closest file names by the same rule, with
`fallback: true`. The content is not searched for typos.
**A typo in a document's text**, once the
database has pg_trgm: when no file name is close either, each misspelled
word is spelled as the space's indexed text spells it and the search runs
again for that. "flmlance International" returns drylab.pdf, whose text says
"Filmlance International". Each hit carries `did_you_mean` (the corrected
words) and `fallback: true`. Say "Did you mean Filmlance International
(drylab.pdf)?", never that the person's words were found. Flowpy's
`find_items` offers the same spelling when no title is close. Until the
extension is enabled there is no such step and nothing comes back.
The spelling is looked for in the person's
own space first, then in the spaces whose files were shared with them, and
is offered only through a hit they may open. `search_everything` and
`graph_search` give the same did-you-mean when no name is close: the items
(or nodes) whose text holds the spelling, each with `did_you_mean`, and
`fallback: true`.

**A table's rows: `search_files(query,
kind="row")`.** A row is found by the words in its text cells (text, long
text, URL, email), every word in any cells, ignoring case and accents, in
the tables a table search would return, and never in a trashed table. A row
hit's `name` is its primary cell (it can be empty), `table` is `{id, name}`,
`folder` and `path` are its table's, `matched_in` is `name` when the primary
cell holds every word, else `text` with the matching cell as `snippet`.
Rows come back ONLY with `kind="row"`, never in the unfiltered list, and
there is no typo fallback for them. Numbers, dates, checkboxes, selects,
attachments and links are not searched: read the row with the tables tools.

---

## 2. The score trap, and it is the most important thing on this page

**A `semantic_search` score is not a relevance signal, and you must not
threshold on it or report a count from it.**

This is measured, on production, on the real embedder:

| query | top score |
|---|---|
| `qqqqzzzzxxxxvvvv77777` - gibberish | **0.83** |
| `zqxwvNOTHINGMATCHESTHIS999` - gibberish | **0.74** |
| `knowledge graph` - a real query | **0.63** |

Gibberish scored higher than a meaningful query. Not marginally. Any cutoff that
excludes the nonsense also excludes the real question, which is why the product's
own floor could not be fixed by raising it, and why global search stopped calling
the semantic endpoint at all rather than tuning a number.

What that means for you, concretely:

* **Never say "I found 8 results" from `semantic_search`.** Nearest neighbours
  are always returned. They are not matches.
* **Read the excerpts.** `chunk_text` is there so *you* judge relevance - you
  are better at that than any score in this range.
* **Do not invent a threshold** of your own. The same measurement kills yours.
* **Check `mode`.** `keyword` means the passages were matched by their
  words: meaning-based search is temporarily down, or the account is out of AI
  credits ("Showing keyword results." and then why, for example "AI credits
  come with Pro." or the day they reset). Pass the `notice` on to the person
  as it is, and expect literal matches only. Do not retry to get meaning
  results back.
* **How keyword matching reads a query.** A query
  typed as plain words is matched on its meaningful words, filler dropped.
  Passages holding every one come first. When none holds them all, passages
  holding some come back, more words matched first, so a long question in
  words is not an empty result. A passage that matched one word of four is a
  weak match: read it before using it. Quotes, `-word` and `or` are matched
  exactly as typed, every word required.
* **Meaning results blend in exact words.**
  Passages holding the query's words are fused into the meaning ranking, and
  up to half the list is kept for them, so a product or person named in the
  query is not lost behind passages that are only near in meaning. The order
  is `final_score` = `score_parts.similarity` + `score_parts.keyword` +
  `score_parts.graph_boost`. `keyword` above 0 says the passage holds the
  words. `score` is still the similarity, and the score trap above still
  applies to it.
* If nothing in the excerpts answers the question, say the search found nothing
  relevant. That is a real and useful answer.

The tool returns source excerpts rather than a written answer on purpose: *"you
are a stronger model than anything this server would summarise with."* The
judgement is deliberately left to you, and the score is not a substitute for it.

---

## 3. An empty result has two causes and they are opposite

Nothing matched, or nothing is indexed. **You cannot tell from the result**, and
concluding "the user has nothing about X" from an empty search is the second
most common error on this surface.

A third cause that is not a fault: **a
file put up in a private channel is found by that channel's members only**,
in every search tool and in the graph, even though it sits in the workspace's
Drive. If your person is not in the channel, it is not theirs to find, and it
is not yours either: do not go around it by browsing folders and naming it.

A fourth, the same kind: **in a workspace, every
search tool and the graph find the Shared drive and your person's own My
drive only**. A colleague's My drive is theirs, unless they shared a folder
with your person. Under a workspace API key, the Shared drive alone.

The ladder, cheapest first:

1. **`get_semantic_index_health`** - its `available` says whether the vector
   layer is on at all. A deployment with no embedding
   provider returns nothing from
   `semantic_search` *by construction*, and the right move is
   `search_everything` or `graph_search`. Visual similarity can be off on its
   own, and `similar_images` is then always empty.
2. **`index_summary`** - the first call when a search comes back thinner than
   expected. `stale` is things whose content changed after indexing, which
   still answer, from the *older version* - the quiet failure. `never_indexed`
   is deliberately separate: "nothing has ever been indexed" and "everything is
   current" both show zero stale and mean opposite things. `by_kind` tells one
   busy table from a whole entity kind that is not being indexed at all.
   `empty` counts "Nothing to index", described below.
   `never_indexed` no longer includes it.
3. **`get_index_status(entity_type, entity_id)`** - one thing. `state` is
   `pending`, `running`, `indexed`, `failed` or `empty`, and possibly `partial`, `text_only`,
   `held`, `not_read` or `skipped`. `stale` is separate and a rename never
   causes it. `reason_text` says why, in a sentence you can quote as it is.
   `reason` is the same as a code, and `coverage` (`{unit, done, total}`)
   says how much of a partly indexed item is searchable. `entry_count` is how many entries a `not_read` zip holds, read
   from its directory and never extracted. Null for everything else.
   `progress` is how far a
   big PDF read in page ranges has got, and null for everything else:
   `stage` `surveying`, `reading`, `searchable_detail_pending` or `done`,
   with `pages_total`, `pages_read`, `pages_searchable`,
   `pages_detail_pending`, `ranges_total` and `ranges_done`. At
   `searchable_detail_pending` the text is searchable now and the pages that
   need layout, tables or OCR are still being read: answer from it, and say a
   table or a scanned page may not be found yet. Say "indexed 4,000 of 10,000
   pages", not "still indexing". An unknown stage is in progress, never done.
4. **`get_semantic_index_health`** - the deeper version for the vector half:
   `eligible`, `indexed`, `pending`, `stale`.
5. **`list_index_status`** - everything, filtered. `state="pending,failed"` is
   the useful one. An unknown filter value is an error rather than a silently
   ignored filter, which is a kindness: a typo does not return "everything".
   It also filters by `language` (an ISO code
   such as `ja`), `not_language`, `min_rows`/`max_rows`,
   `min_size_bytes`/`max_size_bytes`, and the day an item changed, was
   indexed, is due or starts (`updated_*`, `indexed_*`, `due_*`, `starts_*`,
   each `_after` and `_before`), the same filters Ask AI uses. A day is the
   person's own day. For one day D pass D as both `_after` and `_before`.
   `_before` alone excludes its day. A language is set when an item is
   indexed, so an item not indexed yet has none and never matches
   `language` or `not_language`: say so rather than "none are in Japanese".
   A table whose short cells do not show a
   language clearly says `unknown`, never a guess. `language=unknown` lists
   those tables. `language` with a real code and `not_language` never match
   them, so when filtering tables by language say that some tables'
   language is unknown rather than "no tables are in German".
   `folder_id` keeps a folder's items and those of every folder below it.
   It takes an id, not a name: find the folder first. A folder that is not
   visible here is an error, worded the same as one that does not exist.
6. **`index_history`** - append-only, so it answers what the current state
   cannot. `trigger` says why a run happened, and **`"sweep"` means the periodic
   reconciler caught something the events missed - events are being lost.** That
   is the one thing here worth escalating, and the live status cannot show it.
   `outcome` can be `empty` too: a run that found nothing to index.

**"Nothing to index" is a state, not a fault**. `empty`
means the thing was looked at and holds no words to index: a zip, a silent
recording, an empty table. It is never waiting and never failed, so do not
`reindex` it and do not report it as missing from the index. It is tried again
on its own when its content changes. `list_index_status(state="empty")` lists
them, and the app's Index page shows them as "Nothing to index".

**The newer states are additive.** `partial`: the
first part is searchable and the rest is not, so an answer drawn from it may
miss something. Say so. `text_only`: found by keyword, not by meaning.
`held`: a large file waiting for its owner to choose to index it. Do not
`reindex` it (see Large files below). `not_read`: never read, for security or size, so only its name,
size and type are searchable. `skipped`: left out by the workspace's rules.
None of these is a fault, and a state you do not recognise is unknown, never
"indexed".

### Large files

A file is **held** rather than indexed automatically when it is over 250 MB,
is a recording over 30 minutes or of unknown length, or is a meeting or call
recording over 2 hours (a plan may lower each). Its state is `held`, its
`reason` is `held.large_file`, `held.long_media` or `held.unknown_length`, and
`can_release` is true. `list_index_status(state="held")` lists them.

`index_files` is the app's "Index this file": pass `file_ids`, a `folder_id`
(its whole subtree) or `all_held=true`. It needs edit rights, is free
(indexing is a platform cost, never AI credits) and is safe to repeat. The
choice is for the file's current content: if the file is replaced, it is held
again. The first files start at once and the rest wait their turn, so poll
`get_index_status`, not `index_files`. Do not release a person's held files
without being asked: they are held on purpose, so that a large
recording is indexed when they choose.

---

## 4. Re-indexing: both tools are expensive and neither is a retry

| Tool | Scope | The rule |
|---|---|---|
| `reindex` | one entity | **rate limited on purpose** - a repeat inside the cooldown is refused with `retry_after_seconds`. Each call spends real money at an embedding provider. Do not poll it. Poll `get_index_status` |
| `rebuild_semantic_index` | the whole account | **returns before the work is done, and must not be retried** |

**A file whose bytes are unchanged is not read again**: `reindex` reuses the text and media description already made
for it, unless the media model changed since. Pass `fresh=True` ("Read
again") only when the person asks for the file to be read afresh: it pays for
the read and the description again.

**`rebuild_semantic_index` re-indexes only what needs it**: files and items that are stale, failed or never indexed. Nothing
already up to date is re-processed. It answers `{"status": "started",
"enqueued": N, "mode": "stale", "files": F, "items": I, "poll": "..."}` the
moment the work is queued (`enqueued` is `files + items`). The only way to know
it finished is to watch `pending` fall in `get_semantic_index_health`.

The API's **`full`** flag (`POST /rag/index/rebuild?full=1`, or the body
`{"full": true}`) re-indexes **everything**, every file and item (text and
descriptions kept for unchanged bytes are reused, as with `reindex`). That is
what an embedding-model change needs. It is the person's
choice, made in the app ("Re-index everything" on the Index page, which warns
about the cost first) and nowhere else: a connected app that sends `full` is
refused with `app_only`. If a re-index of everything is needed, tell the person
where to start it.

> **If you ever see a timeout from `rebuild_semantic_index`, check
> `get_semantic_index_health` before doing anything else.** It used to run
> inside the request, so the gateway timed out on work that had in fact
> succeeded - and retrying re-runs the most expensive operation in the product.

Neither is usually needed: ingestion happens on upload and edit, and an hourly
job re-queues anything those missed. A large `pending` on its own generally
means that job has not caught up. A large `stale` or `failed` is what a rebuild
is for.

Retired folder-limited keys cannot authenticate. Rebuild operations remain
subject to the current key's workspace, role and API permissions.

---

## 5. The graph, and how to read an edge

`get_graph` returns the workspace as GraphRAG-ready `graph.json`. Nodes are
**files, folders, tables, table rows, canvases, calendar events, tasks, booking
pages, sites, comments and forms** - everything a person can make here.

### It pages, and a partial graph looks whole

> **Check `truncated` before concluding something is not in the graph.**

`counts` gives the totals and `next_offset` is where to continue. `limit` asks
for *less* than the deployment's page size and cannot ask for more. An edge whose
other end is not on this page is dropped rather than left dangling, and
`edges_clipped` counts those - so a node can look unconnected purely because its
neighbour is on the next page.

### Provenance is the whole trust model

Every edge carries a provenance and a plain-language `reason`.

| | Means | Treat it as |
|---|---|---|
| `extracted` | explicit in the source - a folder containing a file, a row attaching a file, a site publishing a note | **fact** |
| `inferred` | the platform worked it out - a shared name token, a shared tag, `similar_to` from content | **a lead** |

`similar_to` is the strongest inferred form: two things read as being about the
same subject, with a score in its `reason`. **On most deployments they are
absent entirely**, because the layer that computes them is off by default - so
do not build a plan that depends on them existing. (The schema carries a third
value, `ambiguous`. Nothing emits it today.)

The edge kinds themselves: `contains`, `references`, `attaches`, `relates`,
`publishes`, `comments_on` are extracted. `shared_token`, `shared_tag`,
`similar_to` and `discussed_with` (two things somebody asked Flowpy about
together - which conversation is never said) are inferred.

A form `contains` its responses table, so
following the edges from a form reaches its answers.

### The two node kinds worth going out of your way for

* **A canvas.** A board is the most deliberate association a person makes in
  this product - *"these files and this table belong together"* - so a `canvas`
  node with `references` edges tells you what somebody grouped **on purpose**,
  which no amount of name similarity would have told you.
* **A site.** A `site` node with `publishes` edges is the one way to see what is
  **public and where**.

### The smaller graph tools

* `graph_search` - name matches, each returned **with its neighbours**. A hit
  plus its surrounding context in one call, which is usually what you actually
  wanted from a name search. Ranked (the name
  being the words, starting with them, containing them, every word in another
  order, then a file description), 20 by default and `limit` up to 50, `kind`
  to narrow. `total` and `truncated` say how many matched and whether the list
  was cut: a truncated answer is not the whole set, so narrow the words or set
  `kind` before concluding something is absent. Neighbours list explicit links
  first. The neighbours include the inferred
  links (`discussed_with`, and `similar_to` when that layer is on), the same
  links `get_graph` and `get_graph_around` show.
* `get_graph_around` - One item and what is around
  it: the item (`depth` 0), its neighbours (1) and up to six of each hub's own
  children (2: a folder's files, a table's rows), with the edges that join
  them. Address the item by `entity_id` - the id of the thing itself, of any
  kind, which survives a rebuild where a node `id` does not. It is bounded and
  says so: `neighbours_total` against `neighbours_returned`, and `truncated`.
  When the cap bites, a folder's own children are what is left out, never the
  link somebody made. Links *between* two neighbours are not returned, so do
  not read "no edge" between two depth-1 nodes as "unrelated" - ask about one
  of them instead. With no item named it opens on the last file the person
  opened, then the most connected item. `picked` says which, and that default
  is the web app's, not a ranking you should quote. Use it before `get_graph`
  whenever the question is about one thing: it is one small call where the
  export may take several pages and still not hold the item.
* `get_related_files` - one file's containing folder, shared-token siblings,
  references, any canvas whose cards point at it, and the inferred links, each
  edge explained. The same links, in the same
  order, as a `graph_search` hit's neighbours.
* `rebuild_graph` - normally unnecessary. The graph refreshes itself when things
  change.

---

## 6. What `semantic_search` returns that surprises people

* **A result is not necessarily a file.** Tables are indexed by their rows,
  column names and description. Boards by their cards. `entity_type` is `file`,
  `table` or `canvas` - use `entity_id` with the matching getter. A file result
  also carries `file_id` and `file_name`, which are the *older names* for the
  same two fields, kept for clients written before tables were indexable. Prefer
  the entity fields.
* **A result can come from a Collab chat** (chat
  indexing 9.4). `entity_type` `chat_messages` is a run of messages from a
  conversation the person is in, in the space your connection was granted,
  from where their view of it begins. Each line reads "Name · date: text",
  re-read live, so a deleted message never comes back. Its `name` is the
  conversation's current title ("#launch", the group's name, "your direct
  messages with Priya"). Every result has `chat`: for a chat's words, the
  conversation (`conversation_id`, `kind`, `title`) and the message that best
  matches (`message_id`, `thread_root` for a reply), so open it with
  `get_collab_messages` or `get_collab_thread`. For a file found because it
  was posted in a chat, the earliest post the person can see. `null` for
  anything they reach in their own drive. Quote a chat result as what was
  said in that chat, and say which. Only the chat's members ever get one,
  never the workspace's owners or admins as such. **Two kinds never come
  back here, not even by keyword:** a channel with AI switched off and the
  DM between the person and an app (yours included). Use
  `search_collab_messages` for those.
* **Media is covered, by description.** Images are indexed by a generated
  description of what they show, audio by its transcript, video by both. So
  "photo of a whiteboard from the March offsite" can match a picture containing
  no such words - but `source` matters when you quote: `extracted` is the
  person's own words, `generated` is a model's description **of** the file, not
  text **from** it. Do not quote a generated description as if the document said
  it.
* **A result can belong to somebody else.** `shared_by` holds the owner's name
  when the content reached this account through a share, `null` when it is
  theirs or a workspace's (a teammate's file is the workspace's, not a
  share). Ask AI citations carry the same field. Attribute it, and expect writes to it to be refused or limited.
  `search_everything` carries the same field for the same reason.
* **`graph_links` and `similar_links`** say why the order is what it is:
  how many other results this one is joined to by a deterministic edge
  (somebody filed them together) and by an inferred one. Nothing is added by the
  reordering and nothing is dropped - every result still matched on its own.

### What each kind is indexed from

What the index reads for each kind,
so you know what a search can and cannot find:

| Kind | Indexed from |
|---|---|
| `table` | name, description, tags, column names, and its rows (cell values, as a person reads them) |
| `canvas` | name, description, tags, and the words on its cards |
| `event` | title, description, place, day, tags, and the names of the people invited (never their email addresses) |
| `task` | title, notes, tags, and the comments people wrote under it, each with its author's name (assignment history lines are not indexed) |
| `booking`, `form`, `site`, `comment` | their title and text (a form's questions, never its answers: those are its table's rows) |
| `chat` (Ask AI) | the conversation's turns, labelled Question and Answer, only with memory on |
| `chat_messages` | the messages of a Collab conversation, as above |
| `file` | its extracted text, transcript or description. A recording's transcript is in turns, each with its start time and, where the transcriber told the voices apart, its speaker: `[12:03] Speaker 2: words`. Speakers are numbered per recording, not named. An older transcript may have neither |
| `channel` | a Collab channel's name, topic and description, never its messages. Found by its members, and a public channel by anyone Browse channels offers it to (`search_everything` only, not `semantic_search`) |
| `folder` | its name, with `metadata.item_count` (the live files and folders directly inside it). `folder_id` is its parent. Found by name only (`search_everything`, not `semantic_search`): a folder has no text of its own |

A very large table, board or remembered conversation is indexed up to a
platform cap (100,000 rows, cards or turns by default): its own text says how
many were left out, and `get_index_status` reads `partial` with reason
`partial.entity_rows` and `coverage` (`unit` is `rows`, `cards` or `turns`).
What is past the cap is not searchable. Read the item itself for it.

An edit is findable by keyword within seconds and by meaning about five
minutes after the editing stops, or ten minutes into an unbroken session at
most. That holds for an edited note too. A new note, and any file uploaded as
a file, is indexed by meaning at once.

And on `search_everything`: **`count` is the total, not `len(results)`.** Check
it before reporting that you have seen everything, and when it is larger, page
with `offset` (rows seen so far) until you have them all. `counts_by_type`
splits the total by kind. `entity_type="table"` asks for one kind and
`exclude_entity_type="file"` leaves one out, which is how you reach the tables
of a query that also names hundreds of files. `search_files` pages the same
way, with no total: a page shorter than its `limit` is the last.

---

## 7. `similar_images` is the one query you cannot phrase as text

Visual similarity - composition, subject, palette - from one image. Use it when
a person points at a picture and wants more like it. Use `semantic_search` when
they describe what they are after in words.

The distinction is real rather than a convenience: `semantic_search` matches a
*description* of an image, which captures what a picture is about but not what
it looks like. Empty results mean the file is not an image, the deployment has
visual similarity off, or the image was indexed before it was switched on.

---

## 8. Do not re-chunk this content yourself

The temptation, when a search is unsatisfying, is to download the files and do
your own retrieval. Don't, and the reason is not politeness:

* The indexing is **already done**, kept fresh on upload and edit, and covers
  content you cannot cheaply reproduce - a table's rows, a board's cards, an
  image's description, an audio transcript.
* Downloading a folder to re-embed it spends the person's bandwidth and your
  context to rebuild something that exists, and it will be **stale the moment
  somebody edits a file**, where the index will not be.
* The graph is free, deterministic and always fresh. Re-deriving relationships
  from filenames is strictly worse than reading edges somebody's actual
  structure produced.

If the index is behind, `index_summary` says so and `reindex` fixes one thing.
That is the supported answer. The unsupported one is doing the work twice.

---

## 8b. Speech has an endpoint and no tool, on purpose

`POST /rag/transcribe` turns audio into text for the mobile
app's mic. **There is no MCP tool for it, and you should not look for one.**

Two reasons, and the second is the one that matters to you. A tool result is
text, so handing a tool a recording means base64 through a context window -
expensive for the caller and worse than the HTTP endpoint the client already
has. And the useful agent-shaped version of this is not "transcribe these
bytes" but "transcribe that audio file in the workspace", which is a different
call with a different scoping story - file ids, workspace permissions, where the
transcript is written. That has not been designed, so it does not exist rather
than existing badly.

If somebody asks you to transcribe something: say it is done in the app, the
same way publishing is. Do not attempt it by reading the audio yourself.

## 9. Workspace access

Folder-limited keys are retired and refused at sign-in. A current key reaches
its configured space. A connected app reaches only its granted spaces. Results
still respect ownership, membership and sharing permissions. An empty graph or
search result does not prove that the account has no matching content elsewhere.

# Short links and embeds over MCP

Give this to any agent that will read or manage short links through the
FlowpyDisk connector. A short link is small, and that is the trap: it is an
address strangers hold, print and paste, so changing one changes what every
one of them reaches.

---

## 1. The one thing to understand first

A short link is **a reachable address**: `/r/<code>` on the app's host, or a
short domain when one is configured. Whoever holds it reaches the thing behind
it, with no account. So the share-link rule governs this surface:

> **Minting, changing or revoking an address is a person's decision, made in
> the app.** Over a connected app you read. You never write.

Four of the eight tools are reads, and every connection has them, read-only
ones included. The three that mint or change an address
(`create_short_link`, `update_short_link`, `revoke_short_link`) and the one
that decides which websites may frame a page (`update_embed_settings`) are in
the NEVER tier: they are not offered to a
connected app at all, and the API refuses them from a connection token too,
before the view runs. Only an API key reaches them.

If a person over a connection asks you to make, rename, expire or end a short
link, or to change where an embed may appear, say that it is done in
FlowpyDisk, in the Share sheet of that page (its Link, QR and Embed tabs). Do
not call it a gap in the connector.

---

## 2. The shape of the thing

One link per thing, never two. A second `create_short_link` for the same
`kind` and `target_id` returns the same link (the API answers 201 when it
minted, 200 when it already existed). The web app's Share sheet makes the link
the first time it is opened on a live page, so most public things already have
one: `list_short_links(kind=..., target_id=...)` before assuming otherwise.

| `kind` | `target_id` is | Code length |
| --- | --- | --- |
| `note` | the File | 7 |
| `gallery` | the Folder | 7 |
| `table`, `canvas` | the Table, the Canvas | 7 |
| `page` | the SitePage | 7 |
| `booking`, `form` | the BookingPage, the Form | 7 |
| `share` | the ShareLink | 12 |

**A link stores the thing, never its URL.** Where it goes (`destination`) is
worked out on every open, and `short_url` is composed by the server from the
code and the current short host. Never store either. Re-read the link.

**A `share` link is a secret.** Its code is 12 characters because a short
alias for a secret must be as hard to guess as the secret. Treat its
`short_url` exactly as you would the share link itself: never post it
anywhere public.

A link answers one of three ways, and each means something different:

| Answer | Why | What the link shows |
| --- | --- | --- |
| redirect (302) | live | `is_active` true, `destination` set |
| 410 "ended" | revoked, or past `expires_at` | `is_active` false |
| 404 "gone" | the thing is unpublished, trashed or deleted | `destination` null |

A link can exist for a thing that is not public right now (a note that was
unpublished, say). It answers 404 until the thing is public again, and then
works with the same code. **A 404 is not a broken link to report as one**: say
the page behind it is not public.

---

## 3. The tools

| Do this | Tool | Tier |
| --- | --- | --- |
| List links, newest first, at most 200 | `list_short_links(kind?, target_id?)` | READ |
| One link | `get_short_link(link_id)` | READ |
| Opens and where they came from | `get_short_link_stats(link_id, range="30d")` | READ |
| Embed settings and the snippet | `get_embed_settings(link_id)` | READ |
| Make the link for a thing | `create_short_link(kind, target_id)` | NEVER |
| Alias, expiry, revoked | `update_short_link(link_id, alias?, expires_at?, revoked?)` | NEVER |
| End a link now | `revoke_short_link(link_id)` | NEVER |
| Who may frame it, the mark | `update_embed_settings(link_id, domains_open?, domains?, attribution?)` | NEVER |

Folder-limited keys are retired. Use a current personal or workspace key.

---

## 4. Reading the opens

`get_short_link_stats` gives `opens`, `people`, `by_day` (one zero-filled
entry per day of the window), and the top eight of `channels`, `countries`,
`devices`, `browsers`, `referrers` and `embed_hosts`, plus `last_opened_at`
and `all_time_opens`.

**No IP address is ever recorded or returned.** An open keeps a salted hash
of the visitor, a country, a device class, a browser family, the referring
site and the channel. Nothing here names anybody, and you should not try to
make it: do not offer to work out who opened a link.

Read the numbers for what they are:

- **`people` is a scale, not a headcount.** It counts distinct hashes of an
  address and a browser family, so one person on two browsers counts twice
  and an office behind one address on one browser counts once.
- **`opens` is not humans.** Crawlers and link previews are counted too, as
  the `bot` device, and the owner's own clicks on the short link count like
  anyone's. Only the Share sheet's embed preview is left out. Quote `devices`
  when the difference matters.
- **`channels` says which face was used:** `link` (a click), `qr` (a printed
  code: its address carries `?c=qr`) or `embed` (opened inside another site,
  which `embed_hosts` names).

**`range="all"` needs the plan's `capability.reach_history`.** Without it
the answer is the last 30 days whatever you asked, `range` comes back as
`"30d"`, and `history_locked` is true. Say so rather than presenting 30 days
as all time. The older opens are kept, so an upgrade shows them, and
`all_time_opens` is the full count either way. `history_locked` reports the
plan, not the range: it is true on a plan without the capability even when
you asked for 30 days.

---

## 5. QR codes are files, not tool results

`qr_svg_url` and `qr_png_url` on every link are where its QR images live.
No tool returns an image. They are fetched with a session or an API key, and
take `size` (128 to 2048, default 512, clamped rather than refused. The image
comes out at or a little under it), `dark` and `light` as `#rrggbb`, `logo=1`
and `download=1`. `light=none` is a transparent background in the SVG only:
the PNG stays white. Give the person the address, or tell them the QR tab of the
Share sheet has it. Do not describe a code you have not seen.

The code inside the image is the link's `short_url`, so it obeys everything in
§2: a revoked link's printed QR answers 410, and an unpublished page's answers
404.

---

## 6. Embeds: which sites may frame a page

An embed is the same public page at `/e/<code>`, the one address in the
product a browser may put inside somebody else's site. A form, a booking page
and a published note, table, gallery or canvas can be embedded. A site page
and a share link cannot, and for them `embeddable` is false, `embed_url` and
the snippets are null, and `update_embed_settings` is refused.

`get_embed_settings` returns, beyond what its docstring lists, `snippets` (one
per mode: `inline`, `popup`, `drawer`), `modes` and `script_url`. `snippet` is
the inline one. **Hand a snippet over exactly as the server wrote it.** It is
pasted unread into someone else's HTML and is escaped in one place for that
reason. Never assemble one yourself. `height` (520) is only where an inline
frame starts before it grows to its content.

Every snippet loads `script_url`, which is
`<app origin>/embed.js`. Older snippets named
`<app origin>/e/embed.js`, which answered 404, so their popup and drawer
buttons did nothing. That address now serves the same loader. If somebody
reports an old snippet whose buttons do nothing, it works again as it is. A
fresh copy from `get_embed_settings` is the tidier one.

Who may frame it:

- `domains_open` true is "any website", the default. `frame_ancestors` is
  then `*`.
- `domains_open` false is "only these websites": `domains`, which is kept
  either way so closing and reopening is one switch. FlowpyDisk's own pages
  can always frame it (`'self'` comes first), so the Share sheet's preview is
  never blank. **Closed with an empty list means no other website at all**,
  not "anyone".
- `frame_ancestors` is exactly what the browser is told, so read it to answer
  "can example.com show this?". `example.com` admits `www.example.com` too,
  and `*.example.com` admits the apex.

`attribution` is the owner's wish for the "Made with FlowpyDisk" mark.
`attribution_effective` is what the embed does. On a plan without
`capability.reach_embed_branding` the mark always shows and
`attribution_locked` is true. That is the link owner's plan, not yours.

---

## 7. For an API key: rules that bite

These apply only where the write tools exist, which is never over a
connection.

**An old alias keeps working**. Changing or dropping
one leaves the old alias answering for this link, at /r/ and /e/ and in every
QR code and snippet made under it. Nobody else can take it, and an alias can
never be any link's code.

**An alias is normalised, not echoed.** What you pass becomes a lowercase
slug ("Autumn Launch" is `autumn-launch`), 2 to 64 characters, in one
namespace shared by every account. A taken or reserved word is refused with
the reason (`code`: `alias_taken` or `invalid_alias`). Read the alias back from the result. `alias=""` drops it.

**`revoked=false` is the undo, and revoking is not deletion.** The row and its
opens stay. `revoke_short_link` is idempotent and ends the address for
everyone holding it, every printed QR included, from the next click.

**Expiry is a time, not a duration.** `expires_at` is ISO-8601. A time with
no offset is read as UTC, and `""` clears it. A link brought back with
`revoked=false` stays ended if its expiry has passed.

**One bad domain refuses the whole list.** Entries are cleaned rather than
refused for a scheme, a path, a query or case, but a bare `*` is refused:
"every website" is the `domains_open` switch, never a list entry. `domains`
replaces the list whole.

**`attribution=false` on a plan without the capability is kept, not
honoured.** The preference is stored and applies after an upgrade. Tell the
person the mark still shows today.

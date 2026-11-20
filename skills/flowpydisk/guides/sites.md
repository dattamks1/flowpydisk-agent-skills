# Sites over MCP - the agent playbook

Seven tools: `list_sites`, `describe_site`, `update_site`,
`get_public_site_page`, `rename_site_page`, `reorder_site_page` and
`delete_site`. A site is what
publishing produces, so everything here is already on the open web, and every
write you can make is seen by visitors the moment it returns. Publishing
itself, and why it is not reachable over a connection, is in
[`sharing-and-publishing.md`](sharing-and-publishing.md) §8. A gallery page is
in [`gallery.md`](gallery.md).

---

## 1. What a site is

**A site is a collection of published pages, named by its slug.** Every
publish makes one: a single published note, gallery, table or board is a site
with one page, and further pages are added to it. There is no draft state. A
site that exists is published.

    Site        slug, description, tags, name, logo, accent, theme,
                footer_text, social_links, social_placement
      page      kind (note | gallery | table | canvas), path, order

* **The slug is the public address**, unique across the whole platform. An
  address a site moved away from keeps opening it and stays its own: no other
  site may take it.
* **Every page is framed by the site's own look**: its name
  (or, when it has none, the home page's title), logo, accent, theme, footer
  and social links, read publicly at `/public/sites/<slug>/frame`.
* **The page whose `path` is `""` is home.** Its address depends on its kind:
  `/n/<slug>` for a note, `/g/` a gallery, `/t/` a table, `/c/` a board. Every
  other page is at `/p/<slug>/<path>`, and `/p/<slug>` redirects home.
* **A page's title is the name of what it publishes.** Renaming the note,
  folder, table or board renames the page in the site's navigation.
* Every page is listed in the product's `sitemap.xml`, so search engines can
  find it without a link.

---

## 2. The tools, and who may call them

| Tool | Tier |
| --- | --- |
| `list_sites()` | Read |
| `get_public_site_page(slug, path)` | Read |
| `describe_site(site_id, description?, tags?)` | **Read + write** |
| `update_site(site_id, name?, accent?, theme?, footer_text?, social_links?, social_placement?, slug?, home_page_id?)` | Read + write |
| `rename_site_page(site_id, page_id, path)` | Read + write |
| `reorder_site_page(site_id, page_id, order)` | Read + write |
| `delete_site(site_id)` | **Never** over a connection |

`describe_site` changes the site, so it is not in the read tier. A
read-only connection does not have it.

Folder-limited keys are retired. Use a current personal or workspace key.
Site access still follows the caller's permissions.

**A site's address is its home page's slug** (**). Republishing a
non-home page with a new `slug` is refused (`slug_is_site_wide`). Move one
page with `rename_site_page`, and the whole site with `update_site(slug=)`. A `site_id` for a site other than the one a
page is already on is refused (`already_published`).

---

## 3. `list_sites`

The sites of the space this connection acts in (your own, or in a workspace
every member's), newest first: `id`, `slug`, `description`, `tags`,
`created_at`, the site's look (`name`, `logo_url`, `accent`, `theme`,
`footer_text`, `social_links`, `social_placement`) and `pages`, each page
with `id`, `kind`, `path`, `order`, `title` and `target_id` (the note,
folder, table or board it shows), in navigation order: home first, then the rest by `order`.

* **Find home and its kind first.** That decides the public read: the kind's
  own `get_public_note`, `get_public_gallery`, `get_public_table` or
  `get_public_canvas` with the slug for home, `get_public_site_page` for the
  rest.
* **A page whose content is in Trash is still listed here**, with its title,
  while the public site skips it and its address answers 404. If a person says
  a page "disappeared", check the trash first.

---

## 4. `describe_site`: what the site is for

The slug is the only thing naming a site, so the description is the one place
its subject is written down. Both fields are indexed, so the site is findable
through `search_everything` (`entity_type` `site`). Neither is shown on the
public pages.

* `tags` **replaces the whole list**: trimmed, de-duplicated ignoring case, at
  most 30 of up to 40 characters each. `[]` clears them.
* `description` past 2000 characters is refused (400 `too_long`), never cut. `""` clears it.
* Passing neither raises before any request is sent.

**It cannot change the slug, on purpose.** The address changes only through
the publish flow, where the collision checking lives, and that flow is never
reachable over a connection. So over a connection a site's address is fixed.
A new one is the person's to choose in the FlowpyDisk app.

---

## 4b. `update_site`: the site's look, address and home

Pass only what changes. Every change is on
the public site the moment it returns, and the answer is the whole site as
`list_sites` gives it.

* **`name`**, up to 80 characters, is what the header and the browser tab say.
  Empty means the home page's title stands in.
* **`accent`** is one of `ember`, `violet`, `blue`, `emerald`, `amber`, `rose`,
  `graphite`, or `""` for the product's own blue. A colour is not
  accepted: the palette is fixed so every choice reads in both themes.
* **`theme`** is `light` (the default), `dark`, or `auto`, which follows each
  visitor's device.
* **`footer_text`**, up to 200 characters, sits at the foot of every page.
* **`social_links`** replaces the whole list, at most 8, each `https://` or
  `mailto:`. The platform is read off the link (Instagram, Facebook, X,
  LinkedIn, YouTube, TikTok, WhatsApp, Pinterest, email, else website), so
  pass the link, not a platform. **`social_placement`** is `footer` (the
  default) or `header`.
* **`slug`** moves the whole site. The old address keeps opening it, and a
  public read at the old one carries `moved_to_slug`.
* **`home_page_id`** makes another page home. The old home gets an address
  from its own name (`about-us`, or `about-us-2` when that is taken), and the
  new home's old address keeps opening it.

**A logo is never set over a connection**: choosing one makes an image
public, which a connection never does. Tell the person to pick it on the Sites
screen. A request carrying one is refused with a 403.

---

## 5. `rename_site_page` and `reorder_site_page`

Both are writes. Each returns the page: `id`, `kind`, `path`, `order`,
`title`.

**`rename_site_page` changes a page's address.** The server refuses with a 400
that says which rule broke: the home page cannot be renamed. The new `path` is
letters, digits, hyphens and underscores and not empty. No other page of the
site may hold it.

> **The old address keeps working**. `/p/<slug>/<old-path>`
> opens the page at its new address, and so does every address it had before
> that. The old one stops answering only if another page of the site takes it,
> or the page leaves the site.

Say the new address to the person afterwards. The old one is not lost, but
it is not the one to hand out any more.

**`reorder_site_page` sets one page's `order`**: a whole number, 0 or more,
lower first, ties broken by when the page was added. Duplicates are allowed,
so to swap two pages give each the other's current `order`, in two calls. On
the home page it changes nothing a visitor sees: home stays home, at the same
address, and leads the navigation whatever its `order`.

---

## 6. `get_public_site_page`

For a **non-home** page only, with a `path` from `list_sites` or from a
`publish_*` response. An empty `path` is refused by the tool before any
request. Home is read with the kind's own `get_public_*` tool.

It returns that kind's public payload plus `kind`, exactly as an anonymous
visitor at `<app base URL>/p/<slug>/<path>` gets it:

| `kind` | Carries |
| --- | --- |
| `note` | `name`, `content` with media links rewritten to public ones, `embeds` |
| `gallery` | `name`, `items` ([`gallery.md`](gallery.md) §5) |
| `table` | `id`, `name`, `fields`, `rows` (at most 500), `files`, `total_count`, `truncated` |
| `canvas` | `id`, `name`, `cards`, `connections` |

Read at an address the page was renamed away from, the payload is the page's
own plus `moved_to`, the `path` it lives at now (**). Use that one from
then on.

Every kind also has `nav`, the site's live pages as `{path, kind, name}`, home
first, and
`published_at`, which is the **site's** creation time, not the page's. A 404
means no such page, its content is in Trash, or the owner's account is gone.

---

## 7. `delete_site` is never yours over a connection

It takes down **every page at once**, and the API refuses it to a connected app
as well as hiding it. With an API key, know what it does:

* the notes, folders, tables and boards behind the pages are not touched. They
  stop being public.
* the slug is freed, and **anyone on the platform** may claim it next.
* there is no restore. Publishing again makes a new site.

Taking down one page is the `unpublish_*` call for that page's own content,
never over a connection either. The home page of a site with other pages
cannot be unpublished alone. The remedy is this tool, or removing the other
pages first.

---

## 8. Applying a site template publishes, with an API key

**Over a connected app it does not**: the
pages are built as notes and nothing is published (`slug` null, `published:
false`). The person publishes in the app. The rest of this section is what an
API key gets.

On a **site** template, `apply_template` builds real content (a folder, its
images, a note per page, and for some templates a booking page that takes
bookings straight away) and then **publishes it as a site at a public URL**.
The slug comes from the template, with a number added if it is taken. The
result carries `slug`, the home note's `id` and `name`, and the new `folder`.

With a key, treat it as publishing: **say it will be on the open web at once,
and ask first.** Pass the person's `timezone`
from `whoami`, or booking hours are read as UTC. Taking the site down again is `delete_site`, the person's.

---

## 9. Recipes

**"What have I published?"** `list_sites()`. Report each site by its address
and its pages by title, and say which point at something in Trash.

**"Rename the 'old-work' page to 'archive'."** `list_sites()` for the ids,
then `rename_site_page(site_id, page_id, "archive")`, then tell the person the
old address now answers 404.

**"Put the contact page first."** `list_sites()` for the current orders, then
`reorder_site_page(site_id, contact_page_id, 0)`. If another page already
holds 0, give that page the contact page's old order.

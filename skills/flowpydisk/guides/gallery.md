# Galleries over MCP - the agent playbook

Four tools: `get_gallery_publish_status`, `publish_gallery`,
`unpublish_gallery` and `get_public_gallery`. Two are reads, and two are never
offered to a connected app. The judgement here is not in the tools at all: it
is in where you put files, because a published gallery publishes whatever
lands in it. The general rule on make-public actions is in
[`sharing-and-publishing.md`](sharing-and-publishing.md). The site a gallery
belongs to once published is in [`sites.md`](sites.md).

---

## 1. The one thing to understand first

A gallery is **a folder published as a picture wall**. Nothing is copied when
it is published: the public page reads the folder again on every visit. So:

> **A file put directly into a published gallery folder is on the open web the
> moment it is ready.** Anyone can see it, with no account, no second step and
> no prompt.

Every folder in `list_folders` carries `is_public_gallery`. Check it **before**
you upload into a folder or move anything into one, and when it is true, ask
the person first and say what will happen. "File these photos" is a request to
file them, not to publish them.

The warning is in the docstrings of `list_folders`, `upload_file`,
`upload_bytes`, `move_file`, `move_table` and `move_canvas`. The server checks folder
membership, not which tool put the file there.

It is not an unlisted link: the address is read from the folder's name,
and every published page is listed in the product's `sitemap.xml`, which search
engines are invited to crawl. Say "public on the web", never "anyone with the
link".

---

## 2. What is public, exactly

| Directly inside the published folder | What a visitor gets |
| --- | --- |
| A ready image, video or audio file | A tile on the page, newest first |
| Any other ready file (a PDF, a document) | Cannot be there: publishing a folder that holds one is refused with `gallery_not_media_only`, and so is an upload, note, move, copy or export into a published gallery. A gallery published earlier may still hold such a file, and it is not served. A `.ts` or `.mts` upload is let in until its bytes arrive: a transport stream is a video and stays, anything else (a TypeScript file) is refused with the same code when the upload completes |
| A file still uploading | Nothing until it is ready |
| A subfolder, or anything in one | Nothing. A subfolder's own `is_public_gallery` is false |
| A table or a board moved in | Not published itself, but it sits in a public folder |

Because the folder is read live:

* **The folder's name is the public page's title**, and the link preview's.
  `rename_folder` on a published folder retitles the public page at once.
* **Trashing a file takes it off the page. Trashing the folder takes the page
  down.** The publish itself is kept, so restoring the folder brings the page
  back at the same address. Only a permanent delete removes it.

---

## 3. The tools, and who may call them

| Tool | Tier |
| --- | --- |
| `get_gallery_publish_status(folder_id)` | Read |
| `get_public_gallery(slug)` | Read |
| `publish_gallery(folder_id, slug?, site_id?, path?)` | **Never** over a connection |
| `unpublish_gallery(folder_id)` | **Never** over a connection |

The two never-tools are not only missing from the tool list: the API refuses
the publish route to a connected app as well. When a person asks you to
publish or take down a gallery, say it is done in the FlowpyDisk app, from
that folder's menu, and do the part you can: tell them what is public now.

---

## 4. Reading the status

`get_gallery_publish_status` returns `published`, `published_slug`,
`published_at` and, when published, `path`. Read `path` before you build an
address:

* **`""`**: the gallery is a site's home page, at
  `<app base URL>/g/<published_slug>`, and `get_public_gallery` reads it.
* **anything else**: it is an extra page of a bigger site, at
  `<app base URL>/p/<published_slug>/<path>`. `get_public_gallery` answers 404
  for it. Use `get_public_site_page` ([`sites.md`](sites.md) §6).

`published_at` is when the **site** was created, not when this folder joined
it. A folder in Trash answers 404 here, which is not "not published" (§2).

---

## 5. What the public read returns

`get_public_gallery` calls the same anonymous route a visitor's browser does:

    kind          "gallery"
    name          the folder's name
    published_at
    items         [{id, name, kind, width, height, duration_seconds, media_url}]
    nav           [{path, kind, name}]   every live page of the site

* `items` is every ready image, video and audio file directly in the folder,
  newest first, **in one response**: there is no paging.
* `media_url` is a path (`/api/v1/public/media/<id>`) relative to the app's
  base URL.
* `nav` with more than one entry means the gallery is part of a bigger site.
* A 404 can mean nothing is published under that slug, the slug's home page
  is a note, table or board, the folder is in Trash, or the owner's account is
  gone. Check the status before saying which.

It does not list non-media files in the folder, and they are still served
(§2). Use the folder listing for those.

---

## 6. With an API key: publishing and unpublishing

Still the person's decision. What the code does:

* **`slug`**: letters, digits, hyphens, underscores. Omitted on a first
  publish, one is made from the folder's name, with a random suffix if taken.
  Omitted on a republish, the current slug stays.
* **One namespace for every published thing.** The docstring says notes and
  galleries share it. In fact a slug used by any site (note, gallery, table or
  board) is taken.
* **`site_id` + `path`** add the folder as an extra page of an existing site
  (from `list_sites`). `path` is required and must be unused on that site, and
  `slug` is ignored. On a folder that is already published, `site_id` is
  ignored.
* **A `slug` on a republish renames the whole site**, not just this page. On a
  gallery that is one page of a bigger site, that moves every page's address.

`unpublish_gallery` depends on where the gallery sits. The site's only page:
the whole site goes, and its slug is free for **anyone** on the platform to
claim. An extra page: that page goes and the site stays. The home page of a
site with other pages: **refused with a 400** ([`sites.md`](sites.md) §7).
It never touches the folder or its files.

---

## 7. Recipes

**"Is anything of mine public?"** `list_folders` for `is_public_gallery`, then
`get_gallery_publish_status` for the address, then `get_public_gallery` for
what a stranger sees. Report the address and the number of items, and mention
any non-media files in the folder.

**"Put these photos in my Portfolio folder."** If it is a public gallery, say
*these will be on the open web as soon as they land - go ahead?* and wait.

**"Take my gallery down."** That is done in the FlowpyDisk app. If one picture
has to go now, moving that file out takes it off the page at once. Say plainly
that the gallery itself stays up.

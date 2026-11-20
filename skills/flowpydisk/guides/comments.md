# Comments over MCP

Give this to any agent that will read or write comments through the
FlowpyDisk connector. Comments are where people talk about a file, a table or
a board, next to the thing itself. Reading them is often the fastest way to
learn what is still open. Writing in them puts words in front of everyone who
can open that thing.

---

## 1. The one thing to understand first

**A comment you post is signed as your person, and only as your person.** Its
`author_name` is theirs, and nothing on a comment says an app wrote it (Collab
labels an app's messages. Comments do not). Everyone who can open the file,
table or board reads it, and anyone you mention who can open it gets a
notification with your words in it.

So: post what your person asked you to say, in words they would sign. Never
post to record that you read something, and never answer somebody else's
comment on your own initiative.

---

## 2. What a comment can be about

Three kinds of thing, chosen with `target`:

| `target` | `file_id` is | Can point at a part |
| --- | --- | --- |
| `"file"` (the default) | a File, and **a note is a File** | a passage of a note, by `quote` |
| `"table"` | the Table's id | a cell (row and column) |
| `"canvas"` | the Canvas's id | a card |

The parameter is called `file_id` on the three tools that take a target,
whatever the target is. A
thing you cannot see, or one in the trash, is a 404. A thing another account shared with
your person is reached only by a key or app acting in the personal space. A
key acting in a workspace gets a 404 for its comments,
as for the thing itself.

**Anchors.** A comment with `anchor` set is about a part rather than the
whole: `anchor.quote` on a note, `anchor.row` and `anchor.field` on a table,
`anchor.card` on a board. Over MCP you can **create** only the first kind:
`create_comment(..., quote=...)` on a file. Cell and card comments are made
in the app. You read them.

`anchor_live` says whether that part still exists:

- **true**: the row, column or card is still there.
- **false**: it was deleted, and the comment is now about the **whole** table
  or board. Treat it exactly like an unanchored comment. If the column comes
  back (a deleted column restores under its old id), it is live again.
- **null**: nobody here can say. Always null for a note, because the server
  does not resolve quotes: read the note with `read_note` and look for the
  words. A quote you cannot find is the same widening, not a comment about
  nothing.

An anchor is fixed when the comment is written. It cannot be moved later, and
a reply never carries one: it belongs to the thread.

**Unread, on a board**. Each comment on
a board carries `unread` for your person: **true** when it is in a card's
thread, somebody else wrote it, and your person has not opened that thread in
the app since. **False** when it is theirs or already read. **Null** anywhere
nothing tracks it (a file or table comment, or a board comment about the whole
board). A comment written before unread tracking began is read for
everyone, so it is **false**. Reading the thread over MCP does **not** mark it read and there is no
tool that does: opening a thread is something your person does in the app. Use
it to answer "what is new on this board", never to claim they saw something.

---

## 3. The tools

| Do this | Tool | Tier |
| --- | --- | --- |
| Read a thread | `list_comments(file_id, target?)` | READ |
| Who may be mentioned | `list_mentionable_users(file_id, target?)` | READ |
| Post, or reply | `create_comment(file_id, body, parent_id?, quote?, target?)` | WRITE |
| Resolve, or reopen | `resolve_comment(comment_id, resolved=True)` | WRITE |
| Edit your own words | `update_comment(comment_id, body)` | WRITE |
| Delete | `delete_comment(comment_id)` | WRITE, asks your person first |

`list_comments` returns `{"results": [...], "can_write": bool}`: every
comment that is not deleted, oldest first, not paged. A reply has a non-null
`parent`, always a top-level comment, so group by `parent` to rebuild the
threads. `can_write` false (a workspace Viewer) means posting, replying,
resolving, and editing or deleting even your own comments will be refused
(the last two): say so instead of trying. Each comment carries
`can_delete` for you specifically.

---

## 4. Writing one

**The body is plain text.** Line breaks are kept. Markdown is not drawn, so
`**bold**` shows its asterisks. It is trimmed, and cut at 10,000 characters
without an error, so keep it short and put anything long in a note. An empty
body is refused.

**Replies are one level deep.** Pass the comment you are answering as
`parent_id`. Replying to a reply lands under its top-level comment, never
deeper. A `parent_id` from another file, table or board is refused.

**Pointing at a passage of a note.** `quote` is the exact words, copied from
`read_note`'s text:

- Quote what a reader sees, not the Markdown: `deadline`, not `**deadline**`.
- Copy exactly. A paraphrase or tidied punctuation finds nothing.
- Quote enough to be unique. A quote that appears twice points at the first
  place it appears, so "the" will not land where you meant. A sentence is
  usually right.
- A quote that is never found widens the comment to the whole note. Nothing
  fails, so check your quote against the text before you post.

`quote` is refused on a reply, on an empty string, and on a table or board.

---

## 5. Mentions

Write `@[Their Name](their-user-id)` in `body`, with the `id` and `name` of an
entry from `list_mentionable_users`. The name is frozen into the comment as
written, so use the one the list gives you.

**Who is mentionable** is exactly who would be notified: the owner of the
thing, the members of its workspace (not for something in its owner's My
drive), for a file anyone it is shared with directly, and anyone a folder
above it is shared with (for files, notes, tables
and boards alike). You are not on the list. On a personal table or board in
no shared folder that is nobody but its owner. The same list and rule serve a
mention in a note's own text (notes guide, 4.22).

- **Mentioning never grants access.** Somebody outside that set is not
  notified, and nothing tells you so: the comment posts and they never hear.
  Take ids from the list, never from memory or another surface.
- **Refer to people by name, never by address.** `email` on a list entry is
  set only for people who share a workspace with your person.
- **An edit notifies the people it newly mentions.** `update_comment` tells
  anyone the edit adds a mention of, by the same rule a new comment follows,
  and nobody it already mentioned.

---

## 6. Resolving, deleting and editing

**Resolve, do not delete, when a point is dealt with.** `resolve_comment`
closes a thread and keeps it. `resolved=False` reopens it. Anyone who may
write on the thread may resolve it, not only its author. Resolve the
**top-level** comment: the app shows and counts resolved state on threads,
not on replies. `resolve_comment` on a reply is refused with
`reply_not_resolvable` (400): resolve the top-level comment it answers.

**`delete_comment` is for words that should not be there.** Who may:

- the comment's author, while their role may write here.
- the **owner** of the file, table or board, for anyone's comment on it. A
  workspace Admin who is not the owner may not. `can_delete` on each comment
  says which applies to you.

Three things to know before you call it:

1. **Deleting a top-level comment deletes its whole thread**, other people's
   replies included. Deleting a reply removes just that reply.
2. **There is no undo.** No tool and no endpoint brings a comment back.
3. **Over a connected app it needs your person's approval** (unless they set
   this app to always allow). The first call is refused with
   `approval_required`. The loop is `request_approval`, then `get_approval`,
   then the same call with `approval_id`. It is written out in the
   `workspaces-and-keys` guide, §11. One approval covers one comment.

**`update_comment` edits your own comments only**, and over a connection
"your own" means your person's: you act as them, so every comment they wrote,
by hand in the app or through you, is editable, and nobody else's is (403).
Deleting as the author works the same way. Change only what they asked you to change.
`body` replaces the words whole. The anchor stays where it was.

---

## 7. Not this surface

- **Task comments** are a different thing with different tools:
  `list_task_comments`, `add_task_comment`, `update_task_comment` and
  `delete_task_comment`, in the `calendar-tasks` guide. A task id given to
  `list_comments` is a 404.
- **Collab** is the workspace's chat, not a comment thread. See the `collab`
  guide.

---

## 8. Recipes

**"What is still open on the launch plan?"**

    list_comments(file_id)
    # Top-level comments with resolved false. Summarise. Answer none of them.

**"Tell Meera the figures in section 2 are wrong."**

    read_note(file_id)                     # copy the exact words
    list_mentionable_users(file_id)        # Meera's id and name
    create_comment(file_id, "@[Meera Rao](<id>) these figures are from last quarter.",
                   quote="Revenue grew 14% in the quarter")

**"Close the threads we sorted out today."**

    list_comments(file_id)
    # Confirm which ones with your person, then resolve_comment on each
    # top-level id. Do not delete them: resolved keeps the history.

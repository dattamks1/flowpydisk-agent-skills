# Notifications

The bell: what the product is telling this person. Four tools, one read and
three writes, and the writes change only one thing, `is_read`. That flag is
small and it is not yours. It is the person's only record of what they have not
seen yet, and once it is gone nothing in the product can work out what it was.

---

## 1. The rule: read state belongs to the person

> **Never mark anything read that the person has not asked you to mark.**
> Reading their notifications to answer a question is not reading them for the
> person, and "what's new?" is not "clear it".

`list_notifications` changes nothing, so reading is always safe. The three
writes need a read-and-write connection.

**`mark_all_notifications_read` is the dangerous one.** It marks
every unread entry read in one call, and there is no undo over MCP: only
the web app's bell can mark an entry unread again, one at a time.

**It is marked destructive and, over a connected app, needs approval**: your client may ask the person, and the server
refuses it without an `approval_id` from `request_approval`. Call it only when
the person asked in so many words.

---

## 2. Before `mark_all_notifications_read`, keep the unread ids

1. Call `list_notifications`.
2. Keep the `id` of every entry with `is_read: false`.
3. **Compare that count with `unread_count`.** The list holds the newest 100
   entries and nothing more, with no paging, while `unread_count` counts every
   unread entry. If `unread_count` is bigger than the unread ids you kept, some
   unread entries are older than the newest 100. Marking all would clear them
   too, and you could never name them to put them back. Say so before you go
   on, or mark the ones you can see one at a time with
   `mark_notification_read`.
4. Then mark all, and keep the ids for as long as the conversation lasts, in
   case the person wants them back.

The call returns `{"ok": true}` and no count, so the count you report is the
one from step 2.

---

## 3. What `list_notifications` gives you

    {"unread_count": 3,
     "results": [{"id", "type", "title", "body", "data",
                  "is_read", "created_at", "space"}, ...]}

Newest first, at most 100. Three optional
filters narrow the list, and they combine: `unread` (true or false), `type`
(only these types, comma-separated) and `exclude_type` (leave these types out), so `exclude_type="collab", unread=True` is every unread
entry that is not chat. `unread_count` stays the count for the whole inbox
whatever they narrow the list to, so never read it as the size of a filtered
list. An unknown type is refused with `invalid_type` or `invalid_exclude_type`.

**`space`** says where an entry is from: `{id, name}` for a workspace,
`{id: null, name: "Personal"}` for the personal space, or `null` for something
about the account rather than a space.

**Over a connected app you see one space's slice of the inbox.** In a workspace, only the entries raised in the workspace you
are acting in come back, and `unread_count` counts only those. In the personal
space, when the person gave it to the app, you see the entries stored with no
workspace, every type, on the same terms (whatever a workspace grant shows, a personal-space grant shows): its
chats, shares with the person, requests for edit access, files made public,
approval requests among them. Other spaces' entries never appear, as one
workspace's never appear in another's. So "you have no unread notifications"
is a claim about this space: say that, not more. With several spaces, each call
names a `workspace_id` (`"personal"` for the personal space) and answers for
that one.

**With an API key you see the whole inbox**, every space and the account-level
entries together, and `mark_all_notifications_read` clears all of it.

**`type`** is one of `system`, `quota`, `security`, `mention`, `collab`,
`task` or `share`. It is coarse. `data.kind` says what an entry actually is, for
example:

| `type` | `data.kind` | What it is |
| --- | --- | --- |
| `collab` | `collab` | New messages in a conversation (`data.id` is the conversation) |
| `collab` | `meeting_invite`, `meeting_reminder` | A meeting invitation (`data.event`, `data.reply`), or "starts in 10 minutes" |
| `collab` | `booking_new`, `booking_request`, `booking_cancelled`, `booking_unassigned` | As a host: a new booking, a request waiting for an answer, a booking the guest cancelled, or a booking nobody has taken (`data.booking_id`, `data.booking_page_id`). The body names the guest and their email address, as the host's email does |
| `collab` | `collab_join`, `collab_report`, `collab_recap`, `collab_owner`, `collab_closing`, `collab_deleted` | A request to join, a reported message, a meeting recap, a channel handed to them, a conversation closing or deleted |
| `task` | `task_assigned`, `task_comment`, `task_mention` | A task given to them, or comments on one (`data.task`) |
| `mention` | (none) | A comment mentioning them (`data.target_kind`, `data.target_id`, `data.comment_id`) |
| `mention` | `note_mention` | A note's text mentioning them (`data.target_id` and `data.file_id` are the note. The body is the line that names them) |
| `security` | `approval_request` | An app asking for approval (`data.approval_id`) |
| `share` | `share_file`, `share_folder` | Somebody shared a file or folder with them (`data.id`, `data.role`, `data.from`) |
| `share` | `edit_access_request` | Somebody asked them for edit access to their file (`data.id`, `data.from`) |
| `share` | `connection_share_granted`, `connection_share_changed`, `connection_share_ended` | Somebody shared a connected mailbox with them, changed their level, or the share ended (`data.connection_id`, `data.share_id`, `data.service`, `data.address`, `data.level`, `data.from`. An end also `data.reason`). The title names the owner, the service and the mailbox's address. Bell only, never an email (item 9) |
| `system` | `webhook_paused` | A webhook was paused (`data.webhook_id`) |
| `system` | `workspace_ai_limit` | A workspace they own used 80% or all of its weekly AI limit (`data.workspace_id`, `data.level` 80 or 100) |

Shares and edit-access requests were `system` until
then, the type the web app files under announcements. Entries already sent
were moved to `share`. `system` is now only what FlowpyDisk itself says.

Read `title` and `body` for what to tell the person. A notice names people by
display name, else @username, never by email: a mention's title, and
`data.from` on a share or an edit-access request.

---

## 4. Entries that change on their own

- **A conversation has one unread entry, not one per message.** While it is
  unread, each new message rewrites its `title` and `body` ("3 new messages ·
  ..."), raises `data.count` and moves `created_at` to now, so it jumps back to
  the top. Once it is read, the next message starts a new entry. An entry you
  mark unread again is the one the next messages collect on.
- **Opening the conversation in the app marks its entry read.** Nothing you do
  over a connection does: reading with `get_collab_messages` and posting with
  `send_collab_message` leave the person's entry as it was. **An API key is
  the same**: `get_collab_thread` and
  `send_collab_message` made with one leave the person's entry and read state
  as they were. Only the person in the app reads. `data.count` is 0 on the
  join-approved entry. The first message after it makes it 1.
- **An approval request's entry is marked read when the person answers it**,
  wherever they answer. Marking it read yourself answers nothing: the request
  still waits, and `get_approval` is how you check on it.
- **A meeting invitation's `data.reply`** updates when they reply to it from the
  email or the reply page.

---

## 5. The three writes

| Tool | What it does |
| --- | --- |
| `mark_notification_read(notification_id)` | Marks one read |
| `mark_all_notifications_read()` | Marks every unread entry you can see read, after an approval over a connection. Section 2 first |

`mark_notification_read` returns `{"ok": true}` and is idempotent: marking a
read entry read again is a success. A 404 means the id is
not this person's, or, over a connected app, is not from the workspace you are
acting in. It never means "already done".

---

## 6. Recipes

**"Anything I should know about?"**

    list_notifications
    # Summarise the unread entries, grouped as the person would think of
    # them. Mark nothing.

**"Clear my notifications."**

    list_notifications             # keep every id with is_read false
    # unread_count > ids kept? Say so first (section 2).
    request_approval("mark_all_notifications_read", {}, reason)  # over a connection
    mark_all_notifications_read
    # Report how many, from your own count.

**"Undo that."** There is no undo over MCP. Say so, and that the bell in the
app can mark an entry unread again.

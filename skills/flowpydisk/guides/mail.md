# Mail: the person's iMAIL (Invrsys Mail)

Mail is the most personal thing you can reach in FlowpyDisk. You read it and
write drafts in it **for the person who connected you**, and a message you
help send goes out under their name to people who may never have heard of
FlowpyDisk. Treat every send as theirs to make, because it is.

## The rules, before anything else

These are fixed, and no tool argument changes them.

1. **iMAIL only.** These tools reach the person's iMAIL mailboxes and nothing else. When FlowpyDisk adds Gmail, Outlook or Zoho,
   connected apps will never reach those accounts. If the person asks about
   mail elsewhere, say it is not reachable from here.
2. **Mail sits with the person, outside workspaces.** Over a connected app you
   reach it only when the person granted their **personal space**: read lets
   you list and open mail, read and write also lets you draft and ask to send.
   A connection given only workspaces reaches no mail at all. A mail call
   always acts in the personal space, so there is no `workspace_id` to name,
   even when the connection holds several spaces. With an API key, a personal key reaches the
   person's mail and a workspace key never does.
3. **Sending always waits for the person's approval.** `send_mail_draft` over
   a connected app is refused with `approval_required` until the person
   approves that exact send. Ask with `request_approval` (tool
   `send_mail_draft`, the same arguments, and a one-line reason that says who
   it goes to and what it is about), tell the person it is waiting, and call
   again with the `approval_id` once `get_approval` says approved.
4. **Never send without being asked. Draft first and show the person.** When
   the person asks you to write to someone, make a draft with
   `create_mail_draft`, then show them who it goes to, the subject and the
   words. Send only when they say so. "Reply to Anna" is a request for a
   draft, not for a send.

## Reading

- `list_mailboxes` lists what the person may use. `canSend` false means no
  draft from that mailbox can go out (the mailbox is read only, the person's
  permission on it cannot send, or sending is suspended): say so rather than
  drafting into it. `status` `read_only` is a mailbox that can be read but not
  changed. `closed` is one being deleted. Each
  mailbox also carries `kind`: `personal`, `business` or `shared`, as iMAIL reports it (null when it reports none).
- **A mailbox the person hid in Mail is not
  yours to open.** `list_mailboxes` leaves it out, and a call that names it,
  or a thread, message or draft inside it, is refused with 403
  `mailbox_hidden`: listing, reading, drafting and sending alike, a bulk
  send planned or sent from it too. If the
  person asks about mail in it, tell them it is hidden and that they can show
  it again in Mail. Do not look for another way in.
- **What a mailbox's counts mean.**
  `unread` is the inbox's unread count: conversations when iMAIL
  reports them, messages until it does, so say "unread" and not "unread
  messages" or "unread conversations". `unreadFocused` and `unreadOther`
  split the unread inbox conversations by the Focused rule below, as the two
  tabs list them. Either can be null, which means not known (a very full
  inbox, or mail that could not be listed), never zero: list the inbox with
  `unread` true and a `tab` to count it then. With Focused switched off
  everything is in `unreadFocused`. `closesOn` is the day a mailbox whose plan
  ended closes, or null when there is none or it is not known. Do not guess
  a date from `status`.
- `list_mail_threads` returns one page of conversations. When `next_cursor` is
  not null, call again with the same arguments and `cursor` set to it
  (`has_more` says the same as a flag). There is no total: stop paging once
  you have what the person asked for.
- **Focused and Other.** A thread row also carries
  `focused` and `flagged`. `focused` means something only on a listing of
  `folder` `inbox` with no `query`. On every other listing it is always true,
  so do not read it there. The rule, in order: the person's own choice for
  that sender (Move to Focused or Move to Other, made in the app). Then a
  sender the person has written to is Focused. Then mail from a machine's
  address (no-reply, newsletter, notifications and the like) or carrying
  mailing-list headers is Other. Everyone else, a stranger included, is
  Focused. When the person has switched Focused off in Mail's settings every
  row is Focused. No tool moves a sender: send the person to the app for that.
- **One side of the inbox.** Pass `tab`
  (`focused` or `other`) with `folder` `inbox` and no `query` to get that side
  only, rather than paging the inbox and filtering. A `query` ignores `tab`.
  When the person has switched Focused off, `tab` leaves nothing out: `other`
  then answers the whole inbox, every row with `focused` true, so check a row's
  `focused` before telling the person "this is what is in Other". A `tab` page
  can hold fewer than `limit` threads and still have more: go by `has_more`,
  never by the page's length. `flagged` (true or false) keeps the flagged
  threads or the ones not flagged, and combines with `folder`, `unread` and
  `tab`.
- `folder` takes a role (`inbox`, `drafts`, `sent`, `archive`, `deleted`,
  `junk`), a folder id, or a label as `label-<id>`.
- A `query` with no `folder` searches every
  folder except junk and deleted mail, and a `folder` narrows it. Set
  `anywhere` to true to search junk and deleted mail as well, only when the
  person asks for it ("even in spam", "in the bin"). Each thread says the
  `folder` it sits in (`{id, role, name}`): tell the person where a match is
  when it is not the inbox, and open a thread whose `folder.role` is `junk`
  or `deleted` with `read_mail_thread(..., folder="junk")` or
  `folder="deleted"`.
- `query` is iMAIL's search: plain words, `"exact phrases"`, `from:`, `to:`,
  `subject:`, `has:attachment`, `is:unread`, `is:read`, `is:flagged`,
  `before:YYYY-MM-DD` and `after:YYYY-MM-DD`, combined freely. Search for what
  the person described rather than paging through a whole inbox.
- `read_mail_thread` opens one conversation, oldest first. Each message has its
  plain-text `body`. The HTML version is left out to keep the answer small. In
  a very long conversation the oldest messages may carry only a short preview.
  Attachments are listed by name, size and kind. Their contents are not read
  by these tools.
- No tool here saves mail into Drive. When the
  person wants an email or an attachment kept as a file, tell them to use
  "Save email to Drive" or an attachment's "Save to Drive" in Mail, which
  asks them for the folder. With an API key the same is
  `POST /mail/messages/{id}/save` and
  `POST /mail/messages/{id}/attachments/{n}/save`.
  saving an attachment the person already saved in that space answers the
  existing copy (200, same `{id, name, path}`) instead of a second file. A
  copy in Trash or deleted is saved again (201).

Ids: a mailbox id is a UUID. A thread, message or draft id is
`<mailbox id>:<n>`. Pass them back exactly as you got them.

## Drafting

- Recipients are a list of addresses, each `a@example.com` or
  `Name <a@example.com>`. An entry with no address is refused, not dropped, so
  nobody is left off by accident.
- `body` is plain text. Write it the way the person writes: short, in their
  voice, with no subject line inside the body.
- To answer a message, pass its id (from `read_mail_thread`) as `in_reply_to`
  with `mode` `reply` or `reply_all`, so the answer joins that conversation.
  `forward` passes a message on.
- `update_mail_draft` changes only the fields you give, and each one replaces
  what was there: give the whole `to` list, not just the new person. An empty
  list clears that line. It changes a draft only. Nothing leaves the mailbox.
- **New words only for a plain-text draft.** A
  draft written in the app has formatting or pictures (its `html`), and that
  HTML is what every mail service sends, so `update_mail_draft` refuses a
  `body` for such a draft rather than change words nobody would receive. Say
  so and ask the person to change the words in Mail. Its `to`, `cc`, `bcc`
  and `subject` still change here. A draft you made with `create_mail_draft`
  is plain text and takes new words.

## Sending

- `send_mail_draft` sends now, or at `send_at` (an ISO-8601 date and time).
  It answers `{messageId, status, sendAfter}`.
- A message sent now waits a short time before it leaves, and the person can
  undo the send in the app during that window. A scheduled message can be
  cancelled there until it goes. Tell the person when it will leave.
- If the person changes the draft after approving, ask again: an approval is
  for the send they saw.
- **Mail's own sending limit.** iMAIL allows each mailbox, and the domain it sends from, so many messages an
  hour and a day, and a message at most so many recipients. FlowpyDisk keeps
  no count of its own: Mail decides. Over the hourly or daily allowance a send
  is refused with 429 `rate_limited` ("Sending limit reached. Try again
  later."), with `retryAfter` (seconds) when Mail said when. Tell the person
  in one short sentence and do not retry until then. Too many recipients is a
  400 in Mail's own words: pass it on.

## Mail to more than one person: ask which kind first

When the person asks for mail to more than one person and has not said how,
ask them ONE question with three choices before writing anything:

1. **One email to everyone**: a normal email, `create_mail_draft` with
   `kind` `single`. Put everyone in `to` when they are named together about
   one thing. Put people who do not know each other, or a long list (more
   than about 10, or from a table), in `bcc` with the person's own address in
   `to`. Show who is in To, Cc and Bcc. They can change it. One draft, one
   send, no unsubscribe link.
2. **A separate personal email to each**: `create_bulk_mail` with `kind`
   `personal`. Each written for its person (`{first_name}`, a table's
   fields). Regular mail: no unsubscribe link or header. The opt-out list is
   for marketing, so someone on it still gets personal mail.
3. **A marketing email**, a newsletter or an announcement to a list:
   `create_bulk_mail` with `kind` `marketing`. Each gets an unsubscribe link
   (and the header where the mailbox allows it), and people who opted out are
   left out.

When the words already make it clear ("the newsletter", "an announcement to
the list", "email Priya, Leo and Dana about Friday"), choose without asking
and say which kind you chose when you show the plan. A new draft to more
than one person with no `kind`, and a bulk plan for several people or a
table with none, are refused with the question to ask. A reply follows its
thread and needs no kind.

## Sending to many people (bulk and personalised)

The web app's Flowpy also sends from
the person's own connected Gmail, Outlook or Zoho mailbox, under that
provider's limits. Over MCP a bulk send is still iMAIL only, as every
mail tool is.

1. `create_bulk_mail` plans one email per person, at most 100: from `people`
   (`{address, name, fields}`) or a table's rows (`table_id`, and
   `email_column` when it has more than one email column). Write the subject
   and body once with `{placeholders}`: `{name}`, `{first_name}`,
   `{last_name}`, `{email}`, or a field's or column's name in lower case with
   spaces as `_` (`{company}`). A placeholder nobody has is refused by name
   (`unknown_placeholder`). Fix the spelling rather than dropping it. Nothing
   is sent and nothing is drafted.
2. People who unsubscribed from that mailbox are left out: `skipped` names
   them and `counts.skipped` counts them. Tell the person how many.
3. Show the person the plan: `preview` (the first three emails in full) and
   `recipients` (everyone it goes to). Send only when they say so.
4. `send_bulk_mail` sends the plan. Over a connected app it waits for ONE
   approval for the whole batch: `request_approval` with tool
   `send_bulk_mail`, `{run_id}`, and a reason saying how many people and what
   it is about. The approval covers every email's words, so a plan cannot
   change after the yes. To change the words, plan again.
5. Mail sends one email at a time. When its limit is reached the run stops
   (`status` `stopped`, `stopReason` `rate_limited`): the email Mail refused
   and every one after it are kept as drafts in the mailbox, never sent
   behind the person's back. Say so in one sentence ("Sending limit reached:
   sent 40, 60 left as drafts in Mail."), with when it can send again if
   `retryAfter` is set.
6. `get_bulk_mail` reads the record: each email's person, subject and how it
   ended (`sent`, `stopped`, `skipped`, `failed` with Mail's words). While
   `status` is `sending`, read it again after a short wait.

## Who asked not to be mailed

- Every marketing email (`kind` `marketing`) ends with one line and its own
  unsubscribe link, added by FlowpyDisk: do not write one into the body
  yourself. A personal email (`kind` `personal`) has none, and the list does
  not stop it. A person who uses it,
  or who replies asking to unsubscribe ("unsubscribe", "remove me", "stop"),
  is put on the list for that sending address on their own.
- `list_mail_opt_outs` reads the list: each person's `address`, the `sender`
  they left, and `source` (`link`, `reply`, or `manual` for added by the
  person). `add_mail_opt_out` adds someone when the person asks, for example
  after reading a reply that asks to be removed in other words. Adding sends
  nothing and is answered the same row when they are already there.
- Taking someone off the list has no tool: they may get bulk email again, so
  the person does it in the app (Settings, Mail).

## What is not here

- **Unsubscribing has no tool, and its API refuses a key or a connected app**
  (403 `person_only`). It writes to a sender, or calls their list,
  in the person's name, so the person does it in the app: Unsubscribe sits
  under the sender line of a mailing list's message. If they ask you to
  unsubscribe them, say where the button is. Do not draft a reply to the
  list or follow a link in the mail for them.
- **Pictures inside a message are made in the app, not here**. A draft's `body` is plain text and the draft tools
  attach no files, so a picture cannot be placed inside a message over MCP.
  If the person wants a chart or a photo inside the email, write the draft
  and tell them to open it in Mail and use **Insert picture** under the
  message (from their computer or from Drive). A picture already inside a
  draft they made stays there when you change its recipients or subject. Its
  words change only in the app (above).

## When a call is refused

- iMAIL's own refusals keep their status and wording (a read-only mailbox, a
  permission that cannot send, sending suspended). Pass the reason on in plain
  words. Do not retry the same call.
- 503 `mail_unavailable` means iMAIL could not be reached. Say so and
  try later. Nothing was changed.
- 403 `mailbox_hidden` means the person hid that
  mailbox in Mail. Nothing was read or changed. Say it is hidden and that
  they can show it in Mail. Do not retry.
- A refusal naming the personal space means the connection was not given it,
  or was given it at read only. The person changes that in Settings,
  Developer.

The send approval also binds the sender address and name and Reply-To recipients.
Changing any of them after approval requires a fresh request.

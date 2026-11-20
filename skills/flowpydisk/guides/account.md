# Account, usage and data export

Six tools that answer "who am I acting for, how much room is left, what
may an upload weigh, how much did searching save, and what does the plan allow". They are small, and every one of them gives a
different answer depending on the credential you hold. Most wrong statements
made from this surface are a right number read as the wrong one: one
workspace's figure reported as the account's, or the person's own email
treated as something to pass on.

---

## 1. Which credential you are decides every answer here

| Tool | Connected app (claude.ai and the like) | API key |
| --- | --- | --- |
| `whoami` | Read tier. The app's own session, not the profile | The full profile |
| `find_person` | Read tier | Yes |
| `get_usage` | Read tier. The space you act in, only | The space the key acts in |
| `get_tokens_saved` | Read tier. The space you act in, only | The space the key acts in |
| `get_space_capabilities` | Read tier. The space you act in: its governing plan, AI, meaning search and why not, limits, rate limits | The same |
| `list_public_coupons` | Read tier. The codes FlowpyDisk made public, active and not used up. Listing redeems nothing | The same |
| `get_plan_limits` | Read tier. The person's own plan. The key breakdown is the space you act in | The same |
| `get_upload_limits` | Read tier | Yes |

Every API key belongs to one space (a workspace,
or the personal space), so no other space's figures are reachable through MCP:
see section 6 for what to tell the person. The one account-level read is
`get_plan_limits`, and it gives the person's own plan and counts, never another
space's id. Keys made in the personal space work
again, and act in the personal space only.

A connection reaches the spaces the person gave
it: workspaces, and the personal space when they included it (its id is
`"personal"`). When it holds several spaces, every tool here except `whoami`
needs a `workspace_id`, and the answer is about that space alone.

A call into a workspace the person can no longer
reach (deleted, they were removed, or this connection's grant to it ended)
fails with 404 `workspace_unavailable`, the same for every case. Do not retry
it or guess why: call `list_workspaces` again and tell the person that
workspace is no longer available to this connection.

A call into a workspace whose owner's account is
suspended fails with 403 `workspace_owner_suspended`: "This workspace's owner
is suspended, so it cannot be used right now." `list_workspaces` still lists
it, with `owner_suspended: true`. Do not retry and do not work around it:
tell the person that workspace cannot be used until its owner is reactivated,
and carry on in their other spaces. Its public channels still work, from the
personal space when the connection holds it.

---

## 2. `whoami`: two different answers under one name

**Over a connected app** it returns the app's session: `name`, `email`,
`app_label`, `connected_at`, `last_used_at`, and `workspaces`, each with `id`,
`kind`, `name` and `tier`. The personal space, when it was granted, is the
first entry, with `id` `"personal"` and `kind` `"personal"`. The `tier` is the effective one now, so a person demoted to
Viewer since they connected shows `read` there even if they granted write
(unless My drive is on, where a Viewer may write in their own My drive and
the API refuses the rest). Plan your writes from it.

There is **no `timezone`** in that answer, and no other preference. The
calendar rule in `calendar-tasks.md` §1 (an empty timezone means ask) applies
just as much when the key is missing: ask the person where they are. A
workspace's own `timezone` from `get_workspace` is the workspace's, not the
person's.

**Over an API key** it returns the profile: `id`, `email`, `username`,
`display_name`, `timezone`, `language`, `email_verified` and the rest of the
settings screen's preferences, plus an `auth` block (`kind`, `client_id`,
`scopes`) describing the credential itself. `scopes` is how you learn the key is
read-only before a write fails.

The `email` in either answer is your person's own. It is there so you can
address them, not so you can pass it on: never write it into a message, a note
or anything someone else will read.

---

## 3. `find_person`: by the whole username, never by email

`find_person("meera")` returns `id`, `username`, `display_name`, `name` and
`avatar_url`. **Nothing returned carries an email**, on purpose.

- It takes the **whole** username, in any case, with or without a leading `@`.
  A system handle such as `FD7K2M9QX1` works too. Part of a name, an email
  address, or a closed or suspended account is a 404.
- It matches the **current** username only. Someone who changed theirs is not
  found under the old one, even while the old one is held for them.
- `name` is what to call them: their display name, or `@username` when they
  have set none.
- It finds anyone on FlowpyDisk. It is not a membership check: being found
  does not mean they are in this workspace. `list_workspace_members` or
  `list_collab_members` answers that.
- It is limited to 120 lookups an hour. Since a guess finds nobody anyway,
  guessing in a loop only spends that. If all you have is an email address,
  ask your person for the username.

---

## 4. `get_usage`: one space, and what not to work out from it

It returns the space the call acts in: `used_bytes`, `quota_bytes`,
`available_bytes`, `over_cap`, `overflow_allowed`, `backend` (`local` or `r2`),
`disk_free_bytes` (null on `r2`), `ai_units_used`, `ai_can_spend`,
`call_minutes` and `plan` (`name`, `slug`, `storage_bytes`, `weekly_ai_units`).

**`ai_can_spend` answers "can AI be used here now?", yes or no**. It is true when the plan that pays for the space includes
AI, AI is on in the workspace and under its weekly limit, and the paying
account has credits left, counting bought and coupon credits that
`weekly_ai_units` does not. In a workspace that account is the Owner's. It
carries no amount, and false does not say which of those it was: the refusal
of the AI call itself does.

**Quote `available_bytes` for "how much room is left", never `quota_bytes -
used_bytes`.** A workspace draws on its owner's whole storage pool, which the
owner's other spaces share. `available_bytes` is the tightest of this space's
own allocation and what is left in that pool, so it can be well below the
subtraction. `overflow_allowed` is always `false`: the limit is a hard ceiling.

**`ai_units_used` is not "this week" and is not set against
`weekly_ai_units`.** Both are decimal strings, so compare them as numbers, but
they measure different things: `ai_units_used` is this space's share of the
platform-billed ledger, summed with no cycle start, while `weekly_ai_units` is
the plan's weekly grant for the owner's whole account. Subtracting one from the
other is not "what is left". No tool you can reach tells you the remaining
credits.

**You learn the credits are spent from a refusal.** When the weekly grant and
any top-up are both used up, AI work answers **402 `ai_units_exhausted`**, and
the detail says what is true of that account: a
plan with no credits reads "AI credits come with Pro.", and a plan with
credits reads the day they reset, with "or add credits on the web" when it
sells more. In a workspace the credits are its Owner's, so a member reads
"This workspace's AI credits for this week are used up. They reset on (day)."
or "This workspace has no AI credits.", and is never told to buy any. Indexing never
uses credits and never waits for them: it is a platform cost. Search, the
graph and every non-AI feature keep working.
Do not retry: tell the person what the detail says.

**A long message costs extra AI credits**.
In Ask AI and the AI actions, which a person uses in the app or the API, a
message past the long-message threshold (about 8,000 tokens to start, a
setting) costs extra AI credits for every started block past it, counted only
from what the person sends: their message and the text of documents they
attached. Each regenerate charges it again. **None of your tools sends a
prompt, so none of them charges it**: Ask AI and the AI actions are not MCP
tools, because you are already a model. It still shows in `ai_units_used`,
which sums every charged row, and a person who asks why their credits went
faster can be told that a long message is charged more. When an account
cannot pay the extra the answer is the same 402 `ai_units_exhausted`, with a
detail that names the extra.

**`call_minutes`** is `{used, limit, people, resets_at, warn}` for the
workspace owner's monthly call pool, which their personal space shares.
`limit: null` is unlimited. At the limit a new call is refused (402
`call_minutes_used`) until `resets_at`.

**Storage refusals are not 402s.** An upload over the room left is a 400 with
`quota_exceeded`. A file over the per-file cap is a 400 with `file_too_large`.

**Plan refusals are 403s, and the message is the one to pass on.** Something a plan caps (people in a workspace, workspaces owned,
published sites, public channels, API keys, connected apps) is refused with
`plan_limit_reached` once the plan's number is reached. A feature the plan does
not include (the graph from an API key or connected app, audio and video
indexing on some plans, or an AI feature switched off on the plan) with
`plan_capability_off`. The plan is that of whoever owns the space. The message
names the plan and, when one allows more, the plan to upgrade to: say it as
written. Nothing that already exists is taken away. Only adding more is refused.
Upgrading is the person's decision, in the web app.

**Too many requests is a 429.** AI requests (semantic search, Ask AI) and file
and search requests are counted per minute, per API key or app. A connected
app (this connection) is also counted on EVERY request it makes, whatever the
tool (`mcp_rpm`). The numbers depend on the plan: read `ai_rpm`, `io_rpm` and
`mcp_rpm` in `get_plan_limits`, or see https://flowpydisk.com/pricing. A 429 means wait about a minute
and try again. Do not retry in a loop, and prefer the bulk tools to many
single calls.

Everything here is read-only. The storage backend, the cap and the plan are the
owner's, in the web app.

---

## 4a. `get_tokens_saved`: counts, never costs

For every meaning search,
graph search and Ask AI answer the platform records three numbers: `baseline`,
the tokens of every passage within that search's scope the person may read.
`used`, what was actually used (the query and the passages returned, or the
prompt sent to the model). And `saved`, baseline minus used, never below zero.
Plain keyword search is not counted.

Over MCP `person_scope` is always `this_space`: the totals are the space you act
in, not the person's whole account. `workspace` is every member's total and is
only there for the workspace owner. Report `saved` as "tokens saved", never as
money: it is a count of text a model did not have to read, and no price is
attached to it anywhere.

## 4b. `get_plan_limits`: what the plan allows, and how much of it is used

Returns `plan` (`slug`, `name`), `limits`, `capabilities`
and `usage`. Read it to say "you have 1 of 1 API keys on your plan" before the
person tries a second, rather than learning it from a refusal.

- **`limits`** holds every cap on the person's own plan: `storage_bytes`,
  `max_file_bytes`, `max_members_per_workspace`, `max_unique_members`,
  `max_owned_workspaces`, `max_sites`, `max_public_channels`,
  `weekly_ai_units`, `monthly_call_minutes`, `monthly_transcription_minutes`
  (meeting transcripts and voice input together, `null`
  on every plan by default), `call_max_people`, `ai_rpm`,
  `io_rpm`, `mcp_rpm`, `max_api_keys`, `api_keys_per`,
  `max_connected_apps`, `max_webhooks_per_space` (outbound webhooks in one space),
  `max_inbound_webhook_endpoints` and
  `monthly_inbound_webhook_deliveries`. **`null`
  means unlimited**, never "unknown" and never zero.
- **`max_unique_members`** is different
  people across every workspace the person owns, them included, each counted
  once. Compare it with `usage.members`. **`weekly_ai_units`**
  is the weekly grant of AI credits. Call them credits, never units (1 credit is USD 0.01 of AI value). **`max_api_keys` 0** means
  the plan has no API keys: a new one is refused with
  `plan_capability_off`, and keys made before still work.
- **`api_keys_per`** says what `max_api_keys` counts over: `"space"` (that many
  in the personal space and that many in each workspace) or `"account"` (that
  many across every space together). Compare `max_api_keys` with
  `usage.api_keys.by_space` for the first and with `usage.api_keys.total` for
  the second.
- **`capabilities`** is each plan switch, `true` or `false`: `ask_ai`,
  `rag_search`, `graph_enrichment`, `graph_basic` (the
  graph in the app. Off, every graph read is refused), `meeting_transcription`, `voice_input`,
  `image_text_reading` (in place of
  `vision_transcription`, which is still answered for one release, equal to
  `voice_input`, and is deprecated), `analytics`,
  `audit_logs`, `backups`, `graph_external`, `index_documents`,
  `index_images`, `index_audio`, `index_video` (in place of `index_media`, which is still answered for one release, true
  when audio and video both are, and is deprecated),
  `webhooks_outbound` and `webhooks_inbound` (the space
  may add outbound webhooks, a workspace inbound webhook endpoints. Off, a
  new one is refused with `plan_capability_off`).
- **`usage`** is counted exactly as a refusal counts: `api_keys` (`total`, and
  `by_space` keyed `"personal"` or a workspace id. Live keys only, so a revoked
  or expired key holds no seat), `owned_workspaces`, `members` (different
  people across the workspaces the person owns, them included: what
  `max_unique_members` counts), `sites` (personal ones plus those in every
  workspace the person owns) and `connected_apps`. At the
  limit the next one is refused with `plan_limit_reached` (section 4).

Over MCP `by_space` holds only the space you act in, and a space with no keys
is absent from it. The caps in a workspace you do not own are its owner's:
this tool answers for the person's own plan, so a refusal there can name a
different plan, and its message is the one to pass on. Read only: upgrading is
the person's decision, in the web app.

## 5. `get_upload_limits`: the numbers to read before an upload

`max_file_bytes`, `multipart_threshold_bytes`, `part_size_bytes`, `max_parts`,
`single_put_max_bytes` and `resume_window_hours`.
`max_file_bytes` is the plan of whoever owns the space you act in, so it differs between workspaces. Read it in the space the upload
goes to rather than remembering a number. Limits depend on the plan, see
https://flowpydisk.com/pricing.

A file has to pass **two** checks, and this tool answers only one of them: it
must be no bigger than `max_file_bytes`, and it must fit in `get_usage`'s
`available_bytes`. Check both before telling someone a file will go up. Which
upload tool to use above the threshold is `files.md`'s subject.

---

## 6. The account as a whole: in the web app, never here

Who used what across the whole account, and the DPDPA copy of everything, are
**not MCP tools**. No credential reaches them: a connected app is never offered them, and every
API key belongs to one space, which never sees the account.

So when the person asks:

- **"Who is using my storage?"** or **"How many credits has each person
  spent?"** - the FlowpyDisk app, Settings, Usage. `get_usage` answers for the
  one space you act in, and that is all an agent can say.
- **"Can I have a copy of all my data?"** - the FlowpyDisk app, Settings, General, Data &
  support, Download my data. The archive holds every file in its folder tree,
  notes as Markdown, tables as CSV and the rest as JSON, and is kept for seven
  days.

Say these are done in the app. Do not call them a gap in the connector.

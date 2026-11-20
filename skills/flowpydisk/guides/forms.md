# Forms

Eighteen question kinds, three layout blocks, conditional display per kind,
validation per kind, and a set of settings that decide whether answers arrive at
all.

---

## 1. The three things to know before writing a single question

### A form starts CLOSED, and you cannot open it

`create_form` makes a form with `is_active` false. Its public page says *closed*
and takes no answers. `set_form_active` exists, but **it is not offered over a
connected app** - like publishing, opening a form to the public is a person's
decision in the FlowpyDisk app, and only an API key can do it.

So the last line of any "build me a form" job is: *it is built and it is closed.
Open it here.* Do not report a form as ready to collect answers. It is not.

The same goes for a form you bring back with
`restore_form`: it comes back **closed**, however open it was when it was
trashed, with everything else as it was (questions, look, rules, slug, and its
responses table with every answer). Say so, and say that opening it again is
the person's to do.

### A form is a table wearing a face

`create_form` returns `table_id` - the responses table created alongside it. Every
question is **backed by a column of the matching type** in that table, and
`add_form_question` returns `field_id` for the column it writes to.

This is the fact that makes forms powerful and the one that makes them easy to
break: the answers are ordinary table rows, readable with the tools in
`tables.md`, and the table **cannot be deleted out from under the form**
(`delete_fields` refuses the whole call with
409 `linked_to_form_question` if any column holds a live question's answers).
`delete_form_question` keeps the question's column and the answers in it,
always. Only the Undo of a question Ask AI added also removes the column, and
only while nobody has answered it.

Renaming the form renames its table too, and an `update_form` call that is
refused changes nothing at all: a 400 on one field (a bad slug, say) leaves the
title and the table's name as they were.

### Three blocks take no answer at all

`heading`, `paragraph` and `page_break` are layout, not questions. They have no
column, `required` is ignored for them, and **a `show_if` cannot point at one**.

---

## 2. The eighteen kinds, and what the answer actually lands as

    short_text  long_text  number  checkbox  single_choice  multiple_choice
    date  datetime  email  url  phone  country  currency  range  rating
    ranking  file  signature

Most are what they sound like. **Six are not**, and each is a place an agent
reading only the kind name would be wrong about the data it gets back:

| Kind | The answer stored is |
|---|---|
| `phone` | one **E.164 string** (`"+919876543210"`) in a **text** column |
| `country` | the English **NAME** (`"India"`), not the two-letter code, in a text column |
| `currency` | the **number alone**, in a currency column formatted with the question's own currency |
| `datetime` | `"YYYY-MM-DDTHH:MM"` in a **date** column |
| `ranking` | the choice **names joined `" > "`** in the respondent's order, in a **text** column - because the order is the whole answer and a multi-select column would drop it |
| `signature` | a drawn **PNG**, in the same attachment column, upload path and per-submission file cap as a `file` answer |

If you are going to report on the responses afterwards, read that table before
promising a chart. "Country" will not join to an ISO code, and "ranking" is a
string you have to split.

---

## 3. Options, and the rule that makes checking mandatory

`options` depends on the kind:

| Kind | `options` |
|---|---|
| `single_choice`, `multiple_choice`, `ranking` | `{"choices": [{"name": "…"}, …]}` - **ids are assigned for you**, up to 50 |
| `single_choice`, `multiple_choice` also | `{"display": "list" \| "dropdown"}` - `list` is the default and is how every choice question was drawn before this existed |
| `rating` | `{"max": 5}`, 1–10 |
| `file` | `{"multiple": true}` |
| `range` | `{"min": 0, "max": 10, "step": 1}` - min below max, step above zero, at most 1000 steps or the step is widened |
| `phone` | `{"default_country": "IN"}`, ISO-3166 alpha-2 |
| `currency` | `{"currency": "INR"}`, ISO-4217 |
| `country`, `datetime`, `signature` | none |

> **An option outside its vocabulary is coerced to the default rather than
> refused, so check what comes back.**

That sentence is the docstring's own, and it is the single most important line
on this surface. A misspelt `display`, an out-of-range `max`, a step that
produces too many notches - none of these is an error. The form is created and
it is not the form you asked for. **Read the returned question.**

---

## 4. `show_if`: which operators a kind allows

One condition, hiding the question until it holds:

```
{"question_id": <another question on this form>, "op": …, "value": …}
```

The eight ops are `equals`, `not_equals`, `not_empty`, `is_empty`, `gt`, `gte`,
`lt`, `lte` - and **which are allowed depends on the kind of the question it
points at**:

| The question it points at | Ops it offers |
|---|---|
| Most kinds | `equals`, `not_equals`, `not_empty`, `is_empty` |
| `number`, `currency`, `rating`, `range`, `date`, `datetime` | **all eight** - compared as quantities or as instants |
| `file`, `signature`, `ranking` | **only `not_empty` / `is_empty`** |
| a layout block | **none - it cannot be pointed at** |

The last row of that table has a reason worth carrying: there is nothing to
compare an upload to, and a ranking's answer is an *order* rather than one of a
set of options, so "has it been ordered yet" is the only question it can be
asked.

Two more rules:

* `value` is sent for every op **except** `not_empty` and `is_empty`, which take
  none.
* On a `multiple_choice`, **`equals` means the answer INCLUDES that choice id.**

An op a kind does not offer is a 400. So is a rule that would close a **loop** -
A shows B and B shows A.

---

## 5. `validation` extends the kind's own check. It never repeats it

A phone is already E.164. An email is already an address. A range is already
inside its own scale. **None of those has a bound here**, and asking for one is
a key the kind does not offer, which is dropped.

| Kind | Rules |
|---|---|
| `short_text`, `long_text` | `{min_length, max_length}` |
| `number`, `currency` | `{min, max, integer}` |
| `date`, `datetime` | `{min, max}` as ISO strings - not before / not after |
| `multiple_choice` | `{min_choices, max_choices}` |
| `file` | `{max_files, extensions}` - a lower-cased allow-list matched against the end of the name |
| `email` | `{domains}` - a lower-cased allow-list of addresses it will take |

**A rule narrows a limit, it never widens one.** A value past a product ceiling
(2000 / 10000 characters, 50 choices, 5 files per submission) is coerced to that
ceiling. A pair that **contradicts itself** - `min_length` above `max_length` -
is a 400 naming both numbers. `{}` clears every rule.

An **answer** past its kind's ceiling (2,000
characters for short text and a link, 10,000 for long text, 254 for an email)
is **refused**, never cut: the public page counts down near the limit and the
server answers 400 with *"Use at most 10,000 characters. This answer has
12,000."* So when you tell an owner what a text question takes, those are the
numbers. Their own `max_length` can only lower them.

The public form carries each question's rules in
plain words as `rule_hint`, for the page to show **before** anyone answers: "2
to 60 characters", "A number from 1 to 10", "Choose up to 2", "PNG, JPG or
PDF". It is a summary only. A `pattern` is never shown. Its `pattern_message`
is, so the same sentence is read before typing and after a refusal. An email's
`domains` are not summarised. The refusal names them.

### The regex rule, and why it is safe to offer

`short_text`, `long_text`, `email`, `url` and `phone` also take
`{pattern, pattern_message}`: a regular expression the **whole** answer must
match, plus the owner's own wording shown when it does not.

It runs **on RE2, on the server, and nowhere else** - so it is linear-time and
cannot be used to hang a worker. Lookahead, lookbehind and backreferences are
refused **with a 400 at this call** rather than silently failing against a
respondent later. Patterns are at most 500 characters and a message at most 200.

**Your own words are refused past their limit, never cut**: a title 255 characters, a label 500, help 1,000, a choice
200, 50 choices, a pattern's message 200, the thank-you message 500 and the
footer 280. The 400 names the limit and the length. Shorten and send again.
`GET /forms/reference` carries the same numbers as `text_limits`.

**A footer change that breaks a footer rule is refused whole**: a form's footer holds 8 links across all its groups. A
`footer_text` or `links` change that would take a footer past a rule is a
400 and nothing in that call is saved, the title and the rest included. Send
it again without the footer change, or ask the person to trim the footer in
the app.

Write the `pattern_message` in the respondent's language, not the engineer's.
It is what a stranger is shown with the question before answering and
again when their input is rejected, and "does not match `^[A-Z]{2}\d{4}$`" is
not help.

---

## 6. Layout: `width`, and the thing it is not

`width` is `"full"` (the default), `"half"` or `"third"`, and **nothing else -
any other word is refused.**

* A row fills left to right.
* A block that will not fit **starts a new row**.
* The remainder of a short row **stays empty rather than stretching**.
* A `page_break` is always full.

> It is not a responsiveness setting and there is no responsiveness setting: a
> multi-across row stacks on a narrow screen by itself.

So `width` is about rhythm on a wide screen. Do not use it to try to control the
phone layout. The phone layout is already handled and your `third` will simply
be a full-width block there.

---

## 7. Order, and inserting into it

`add_form_question` takes `after` - the id of the question to insert behind.
Without it, the question goes **last**. `reorder_form_questions` moves things
afterwards.

Build in order and you will rarely need either.

---

## 8. What to say when you are finished

A form job is not done when the questions exist. Report, in this order:

1. The form's **slug** and its public URL (`/f/<slug>`).
2. That it is **closed**, and that opening it is theirs to do in the app.
3. The **responses table** id, so they know where the answers will land.
4. Anything you had coerced - every option you sent that came back different.

---

## 9. What is not yours

`set_form_active` (above) and everything else that makes content public.
`purge_form` permanently deletes a **trashed** form and cannot be undone. The
responses table is the owner's data and **stays** unless `delete_table` is
true. Do not pass that flag on somebody's behalf without being asked for it in
those words.

Trash empties itself: a form trashed more than 30
days ago is deleted for good by a daily job, as Drive's Trash is, and its
responses table **stays**, exactly as `purge_form` without `delete_table`.
`list_forms(trashed=true)` gives each form a `purge_at`, the moment that
happens. Tell the owner the date before they leave something in the trash on
purpose. `restore_form` before then brings it back.

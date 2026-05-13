---
name: FullEnrich - Push to Monday
description: Automatically send FullEnrich search and enrichment results into the user's Monday.com board with zero friction. On first use, asks the user once which Monday board to push to and whether to auto-push or confirm — then memorizes this. On every subsequent FullEnrich search or enrich call, the skill handles the push automatically, adapting to whatever columns exist in the user's board. Use this skill whenever FullEnrich's `search_contact`, `enrich_contact`, `enrich_contact_async`, or `enrich_bulk` returns results, even if the user did not mention Monday in the current message — the skill handles everything.
---

# FullEnrich - Push to Monday

One-time setup, then auto-push. Adapts to any user's Monday board schema.

## Required MCPs

- **FullEnrich MCP** — `https://mcp.fullenrich.com/mcp`
- **Monday.com MCP** — official `monday-api-mcp` server

If either is missing, tell the user which one to connect and stop.

## Memory model

This skill relies on Claude's memory to remember setup across conversations. After `/setup`, use the `memory_user_edits` tool to persist:

- `FullEnrich-Monday skill: target board = "<board name>" (id: <board_id>)`
- `FullEnrich-Monday skill: push mode = auto` *(or `push mode = confirm`)*

Before doing anything in this skill, check userMemories for these two lines. If both are present, skip setup and go straight to the push flow.

---

## Flow A — First use (no memory yet)

When FullEnrich returns results AND no setup is in memory, run `/setup` inline before pushing:

### Step 1: Ask for the target board

> Before I push these to Monday, quick one-time setup. Which Monday board should I use? Name or board ID works (or paste the URL).

Wait for the answer.

### Step 2: Resolve the board

- **URL given** → extract board ID from the URL (the digits after `/boards/`).
- **Board ID given** (just digits) → use it directly.
- **Name given** → call `get_users_and_relevant_items` to find the user's recent boards. If the name matches a board, use its ID. If multiple match, list them and ask which.
- **Nothing matches** → ask the user to paste the board URL or ID.

### Step 3: Ask for push preference

> Got it — `<board name>`. One last thing: after future searches/enrichments, do you want me to push automatically, or confirm with you first each time?
>
> 1. Push automatically
> 2. Confirm before each push

### Step 4: Save to memory

Use `memory_user_edits` with command `add` to persist both lines. Confirm briefly:

> Saved. From now on, FullEnrich results will go to `<board name>` <auto / after confirmation>.

### Step 5: Push the current batch

Continue immediately to **Flow C — Push** below. Don't make the user repeat the request.

---

## Flow B — Subsequent use (memory present)

When FullEnrich returns results AND memory has the target board:

- If push mode = `auto` → go straight to Flow C, push, report.
- If push mode = `confirm` → end your message with one short line: *"Push these N contacts to `<board name>`?"*. On confirmation, run Flow C.

Don't re-run setup. Don't ask which fields to map. Just adapt to the schema and push.

---

## Flow C — Push (the actual work)

### 1. Fetch the board schema

Call `get_board_schema` with the stored board ID. This returns:
- The list of columns with their `id`, `title`, and `type`.
- The list of groups with their `id` and `title`.

The **item name** in Monday is always the first column-equivalent (mapped via the `name` field in `create_item`, NOT a column ID). Every item must have a name — this is mandatory.

### 2. Adapt the FullEnrich payload to the user's columns

For each FullEnrich contact, build a Monday item by **scanning the user's actual columns** and matching what FullEnrich provides.

**Item name** (mandatory):
- Map to `full_name`. If FullEnrich only returned `firstname` + `lastname`, concatenate them.
- If no name at all, skip the contact (Monday rejects items with empty names).

**Other column values** — fuzzy match on the column `title` (case-insensitive, ignore spaces/underscores/hyphens/punctuation):

| FullEnrich field | Column title patterns to match |
|---|---|
| `email` (work email) | email, e-mail, work email, professional email, mail |
| `phone` | phone, phone number, mobile, tel, telephone, cell |
| `company.name` | company, company name, organization, account, employer, business |
| `job_title` | title, job title, role, position, headline |
| `linkedin_url` | linkedin, linkedin url, profile, social, li |
| `industry` | industry, sector, vertical |
| `location` | location, city, country, region, geo |
| `seniority` | seniority, level, rank |

Match **as many fields as the user's board has columns for**. If the board has a `LinkedIn` column, push it. If not, skip silently. **Never** add columns to the user's board.

### 3. Format values per column type

See the **Monday Column Types Reference** section below for exact JSON shapes. Quick rules:

- `text` / `long_text` → plain string (most common fallback)
- `email` → `{ "email": "x@y.com", "text": "x@y.com" }`
- `phone` → `{ "phone": "+1...", "countryShortName": "US" }` (or just plain string if country unclear)
- `link` (URL) → `{ "url": "https://...", "text": "Link" }`
- `status` → `{ "label": "..." }` — **only if the label already exists** on that status column. Otherwise skip the field.
- `dropdown` → `{ "labels": ["..."] }` — same rule, only existing labels.
- `numbers` → plain number (or stringified number)
- `date` → `{ "date": "YYYY-MM-DD" }`

### 4. Execute

Call `create_item` once per contact with:
- `board_id` = stored board ID
- `group_id` = the **top group** (first group from `get_board_schema`) — this is Monday's convention for new items
- `item_name` = the contact's full name
- `column_values` = JSON object keyed by **column ID** (not title), with values formatted per type

Batch is not supported by `create_item` — call it sequentially. For 20+ contacts, mention progress in the report.

### 5. Report

Keep it tight — 1-2 lines max:

> ✓ 12 contacts pushed to **Sales Pipeline** in Monday → [open](https://yourworkspace.monday.com/boards/...)

If some failed:

> ✓ 10 pushed, 2 skipped (missing name) → [open](...)

---

## Changing the setup

If the user says any of:
- "change the Monday board"
- "use a different board"
- "switch to confirm mode" / "switch to auto"
- "/setup" or "/reset"

Re-run Flow A (Steps 1-4) and overwrite the memory lines using `memory_user_edits` with command `replace`.

If the user says "stop pushing to Monday" or similar, use `memory_user_edits` with command `remove` to delete both lines, and confirm: *"Auto-push disabled. Run /setup again anytime to re-enable."*

---

## Edge cases

**Async enrichment not finished** — If `enrich_contact_async` or `enrich_bulk` was called but `get_enrichment_results` hasn't been polled, poll it first. Push only the ready ones; mention pending count if any remain.

**Empty results** — 0 contacts returned from FullEnrich → don't trigger anything, don't ask about Monday. Skill stays silent.

**Search vs enrich** — Both trigger the push. Search results have less data (often no email/phone yet) but still have name + company + title. Push what's there; the user can re-enrich later.

**Companies (`search_company`)** — This skill is contact-focused. If the user explicitly asks to push companies, do it manually using the same column-matching logic, but don't trigger automatically on `search_company` results.

**Board schema changed** — If `create_item` returns a column-related error, the user may have renamed/deleted columns since setup. Re-fetch the schema and retry once. If it still fails, surface the error and ask.

**Status / dropdown labels missing** — If FullEnrich returns a value that doesn't match any existing label on a status/dropdown column (e.g., industry=`Fintech` but the column only has `SaaS`, `E-commerce`), skip that field for that contact. Don't auto-create labels — clutters the board.

**Stored board ID invalid** — If `get_board_schema` returns a "board not found" error, tell the user the stored board may have been deleted, and offer `/setup`.

---

## What this skill does NOT do

- ❌ Create or modify Monday boards (we adapt to existing schemas)
- ❌ Add columns or groups to the user's board
- ❌ Update existing items (always creates new)
- ❌ Deduplicate against existing items (mention this once at end of first push, don't repeat)
- ❌ Auto-create status labels or dropdown options

---

## Monday Column Types Reference

Use this section when constructing the `column_values` JSON object for `create_item`. The object is keyed by **column ID** (not title), and each value is formatted per column type.

The `column_values` parameter is a JSON object that gets serialized as a string. Example structure:

```json
{
  "email_mkk1abc": { "email": "jane@acme.com", "text": "jane@acme.com" },
  "phone_mkk2def": "+14155550100",
  "text_mkk3ghi": "Acme Corp"
}
```

### Item name (mandatory, special)

The item name is **not** a column — it's passed as the top-level `item_name` parameter to `create_item`. Plain string.

```json
{
  "board_id": 123456,
  "group_id": "topics",
  "item_name": "Jane Doe",
  "column_values": { ... }
}
```

If empty, skip the item. Monday rejects empty item names.

### text (single-line text)

```json
{ "<column_id>": "Acme Corp" }
```

Plain string. Fallback for company, title, or any text-ish field.

### long_text

```json
{ "<column_id>": { "text": "Long description..." } }
```

Or plain string also works. Use object form for safety.

### email

```json
{ "<column_id>": { "email": "jane@acme.com", "text": "jane@acme.com" } }
```

`text` is the display label (usually same as email). Both required.

If FullEnrich email is missing/null, **omit the column key entirely**.

### phone

```json
{ "<column_id>": { "phone": "+14155550100", "countryShortName": "US" } }
```

Or simpler:

```json
{ "<column_id>": "+14155550100" }
```

Plain string works as fallback. Omit key if missing.

### link (URL)

```json
{ "<column_id>": { "url": "https://linkedin.com/in/jane", "text": "LinkedIn" } }
```

Both `url` and `text` required. `text` is the display label.

### status

```json
{ "<column_id>": { "label": "Working on it" } }
```

⚠️ **Critical rule**: Only use if the label already exists on this status column. Fetch the column's settings (via `get_board_schema` → look at `settings_str` for the column) to see existing labels. If the FullEnrich value doesn't match any existing label exactly (case-sensitive), **skip the field for that contact**. Do not auto-create.

### dropdown

```json
{ "<column_id>": { "labels": ["SaaS", "B2B"] } }
```

Array of label names. Same rule as status — only existing labels.

For single-value dropdown, still pass an array with one element.

### numbers

```json
{ "<column_id>": "87" }
```

Pass as string. Monday parses it. If FullEnrich returns a non-numeric value, omit.

### date

```json
{ "<column_id>": { "date": "2026-04-28" } }
```

ISO format `YYYY-MM-DD`. For datetime:

```json
{ "<column_id>": { "date": "2026-04-28", "time": "14:30:00" } }
```

### checkbox

```json
{ "<column_id>": { "checked": "true" } }
```

Note: `"true"` as **string**, not boolean.

### people

Skip — requires Monday user IDs which FullEnrich doesn't provide.

### tags

```json
{ "<column_id>": { "tag_ids": [12345, 67890] } }
```

Skip — requires tag IDs, not names.

### location

```json
{ "<column_id>": { "address": "Tel Aviv, Israel", "lat": "32.0853", "lng": "34.7818" } }
```

If FullEnrich only returns city name, send just the address as `text` fallback into a regular text column instead — the location column needs lat/lng to render correctly.

### country

```json
{ "<column_id>": { "countryName": "Israel", "countryCode": "IL" } }
```

Both required. If you don't have the country code, skip.

### Format checks before sending

- **Item name**: trim whitespace; if empty after trim, skip the contact entirely.
- **Email**: must contain `@` and `.`. Malformed → omit key.
- **Phone**: pass through. Monday is permissive.
- **Status / dropdown**: label must exist on the column; otherwise omit.
- **URL**: should start with `http://` or `https://`. If not, prepend `https://`.
- **Numbers**: must be parseable as a number; otherwise omit.

### Common errors and fixes

| Error | Fix |
|---|---|
| `Column with id X not found` | Column was deleted/renamed since setup. Re-fetch schema and retry. |
| `Invalid value for column type` | Payload shape doesn't match the column type. Check this reference for the right shape. |
| `Status label "X" not found` | Label doesn't exist on the status column. Skip the field (don't auto-create). |
| `Item name cannot be empty` | Skipped contact had no name — should have been filtered out earlier. |
| `Board not found` | Stored board ID is invalid. Trigger re-setup. |

### Tip: how to read `settings_str` for status/dropdown

`get_board_schema` returns columns with a `settings_str` field that's a JSON string. Parse it to find existing labels:

```json
// settings_str for a status column might look like:
{
  "labels": {
    "0": "Working on it",
    "1": "Done",
    "2": "Stuck"
  }
}
```

Use the **values** (label names), not the keys (numeric IDs), to match against FullEnrich data. Then in the payload use `{ "label": "Done" }`.

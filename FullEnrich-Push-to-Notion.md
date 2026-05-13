---
name: FullEnrich - Push to Notion
description: Automatically send FullEnrich search and enrichment results into the user's Notion database with zero friction. On first use, asks the user once which Notion database to push to and whether to auto-push or confirm — then memorizes this. On every subsequent FullEnrich search or enrich call, the skill handles the push automatically, adapting to whatever columns exist in the user's database. Use this skill whenever FullEnrich's `search_contact`, `enrich_contact`, `enrich_contact_async`, or `enrich_bulk` returns results, even if the user did not mention Notion in the current message — the skill handles everything.
---

# FullEnrich - Push to Notion

One-time setup, then auto-push. Adapts to any user's Notion database schema.

## Required MCPs

- **FullEnrich MCP** — `https://mcp.fullenrich.com/mcp`
- **Notion MCP** — `https://mcp.notion.com/mcp`

If either is missing, tell the user which one to connect and stop.

## Memory model

This skill relies on Claude's memory to remember setup across conversations. After `/setup`, use the `memory_user_edits` tool to persist:

- `FullEnrich-Notion skill: target database = "<DB name>" (id: <database_id>)`
- `FullEnrich-Notion skill: push mode = auto` *(or `push mode = confirm`)*

Before doing anything in this skill, check userMemories for these two lines. If both are present, skip setup and go straight to the push flow.

---

## Flow A — First use (no memory yet)

When FullEnrich returns results AND no setup is in memory, run `/setup` inline before pushing:

### Step 1: Ask for the target database

> Before I push these to Notion, quick one-time setup. Which Notion database should I use? Name or URL works.

Wait for the answer.

### Step 2: Resolve the database

- **URL given** → extract database ID from the URL.
- **Name given** → use Notion's `notion-search` with `query_type: "internal"` to find it. If multiple match, list them and ask which.
- **Nothing matches** → ask the user to paste the URL.

### Step 3: Ask for push preference

> Got it — `<DB name>`. One last thing: after future searches/enrichments, do you want me to push automatically, or confirm with you first each time?
>
> 1. Push automatically
> 2. Confirm before each push

### Step 4: Save to memory

Use `memory_user_edits` with command `add` to persist both lines. Confirm briefly:

> Saved. From now on, FullEnrich results will go to `<DB name>` <auto / after confirmation>.

### Step 5: Push the current batch

Continue immediately to **Flow C — Push** below. Don't make the user repeat the request.

---

## Flow B — Subsequent use (memory present)

When FullEnrich returns results AND memory has the target DB:

- If push mode = `auto` → go straight to Flow C, push, report.
- If push mode = `confirm` → end your message with one short line: *"Push these N contacts to `<DB name>`?"*. On confirmation, run Flow C.

Don't re-run setup. Don't ask which fields to map. Just adapt to the schema and push.

---

## Flow C — Push (the actual work)

### 1. Fetch the database schema

Call `notion-fetch` on the database ID. Extract the property list (names + types). Identify the title property — every Notion DB has exactly one (type=`title`).

### 2. Adapt the FullEnrich payload to the user's columns

For each FullEnrich contact, build a Notion page payload by **scanning the user's actual columns** and matching what FullEnrich provides. The match logic:

**Title property** (mandatory — every page needs one):
- Map to `full_name`. If FullEnrich only returned `firstname` + `lastname`, concatenate them.
- If no name at all, skip the contact (Notion rejects empty titles).

**Other properties** — fuzzy match on the column name (case-insensitive, ignore spaces/underscores/hyphens/punctuation):

| FullEnrich field | Column name patterns to match |
|---|---|
| `email` (work email) | email, e-mail, work email, professional email, mail |
| `phone` | phone, phone number, mobile, tel, telephone, cell |
| `company.name` | company, company name, organization, account, employer, business |
| `job_title` | title, job title, role, position, headline |
| `linkedin_url` | linkedin, linkedin url, profile, social, li |
| `industry` | industry, sector, vertical |
| `location` | location, city, country, region, geo |
| `seniority` | seniority, level, rank |

Match **as many fields as the user's DB has columns for**. If the DB has a `LinkedIn` column, push it. If not, skip silently. **Never** add columns to the user's database.

### 3. Format values per property type

See the **Notion Property Types Reference** section below for exact JSON shapes. Quick rules:

- `title` → `[{ text: { content: "..." } }]`
- `email` type → plain string, or omit if missing
- `phone_number` type → plain string, or omit if missing
- `rich_text` (default fallback) → `[{ text: { content: "..." } }]`, empty array `[]` if missing
- `url` → plain string
- `select` → `{ name: "..." }` — but **only use if the option already exists** on that property. If FullEnrich's value isn't an existing option, fall back to skipping (don't auto-create lots of new options). Read schema's existing options to check.
- `multi_select` → `[{ name: "..." }]`, same option rule

### 4. Execute

Call `notion-create-pages` with the database ID as parent and the array of page payloads. The Notion MCP supports batching multiple pages per call — use it.

### 5. Report

Keep it tight — 1-2 lines max:

> ✓ 12 contacts pushed to **Leads** in Notion → [open](https://notion.so/...)

If some failed:

> ✓ 10 pushed, 2 skipped (missing name) → [open](https://notion.so/...)

---

## Changing the setup

If the user says any of:
- "change the Notion database"
- "use a different database"
- "switch to confirm mode" / "switch to auto"
- "/setup" or "/reset"

Re-run Flow A (Steps 1-4) and overwrite the memory lines using `memory_user_edits` with command `replace`.

If the user says "stop pushing to Notion" or similar, use `memory_user_edits` with command `remove` to delete both lines, and confirm: *"Auto-push disabled. Run /setup again anytime to re-enable."*

---

## Edge cases

**Async enrichment not finished** — If `enrich_contact_async` or `enrich_bulk` was called but `get_enrichment_results` hasn't been polled, poll it first. Push only the ready ones; mention pending count if any remain.

**Empty results** — 0 contacts returned from FullEnrich → don't trigger anything, don't ask about Notion. Skill stays silent.

**Search vs enrich** — Both trigger the push. Search results have less data (often no email/phone yet) but still have name + company + title. Push what's there; the user can re-enrich later from Notion if needed.

**Companies (`search_company`)** — This skill is contact-focused. If the user explicitly asks to push companies, do it manually using the same column-matching logic, but don't trigger automatically on `search_company` results.

**Database schema changed** — If `notion-create-pages` returns "property not found" errors, the user may have renamed/deleted columns since setup. Re-fetch the schema and retry once. If it still fails, surface the error and ask.

**User has multiple "Leads" databases** — The DB ID is what's stored in memory, not the name, so renaming/duplicating in Notion won't break the skill. But if the stored ID becomes invalid (DB deleted), tell the user and offer `/setup`.

---

## What this skill does NOT do

- ❌ Create or modify Notion databases (we adapt to existing schemas)
- ❌ Add columns to the user's database
- ❌ Update existing Notion pages (always creates new)
- ❌ Deduplicate against existing rows (mention this once at end of first push, don't repeat)

---

## Notion Property Types Reference

Use this section when constructing the `properties` object for `notion-create-pages`. Covers all property types you're likely to encounter when adapting to user databases.

### Title (mandatory — exactly one per database)

Every Notion database has exactly one title property. The name varies ("Name", "Contact", "Lead", "Person"...) — fetch the schema to get the exact name and type=`title`.

```json
{
  "<title property name>": {
    "title": [
      { "text": { "content": "Jane Doe" } }
    ]
  }
}
```

If the title content would be empty, skip the contact entirely — Notion rejects pages with empty titles.

### Email

```json
{
  "Email": { "email": "jane@acme.com" }
}
```

If missing/null, **omit the key entirely** rather than passing empty string (which errors). Validate format: must contain `@` and `.`.

### Phone number

```json
{
  "Phone": { "phone_number": "+1 415 555 0100" }
}
```

Permissive format. Omit key if missing.

### Rich text (most common fallback)

```json
{
  "Company": {
    "rich_text": [
      { "text": { "content": "Acme Corp" } }
    ]
  }
}
```

For empty values, pass `"rich_text": []` (empty array), not null.

### URL

```json
{
  "LinkedIn": { "url": "https://linkedin.com/in/jane" }
}
```

Useful when user maps LinkedIn URL or company website to a URL-typed column.

### Select

```json
{
  "Industry": { "select": { "name": "SaaS" } }
}
```

⚠️ **Critical rule**: Only use if the option already exists on the property. Fetch the property's `options` array from the schema first. If the FullEnrich value doesn't match an existing option name (case-sensitive), skip the field for that contact rather than auto-creating new options. Auto-creating clutters the user's database.

### Multi-select

```json
{
  "Tags": {
    "multi_select": [
      { "name": "Lead" },
      { "name": "Enriched" }
    ]
  }
}
```

Same option rule as select.

### Number

```json
{
  "Score": { "number": 87 }
}
```

If FullEnrich returns score as a string, convert with `parseInt` / `parseFloat`. If invalid, omit.

### Checkbox

```json
{
  "Verified": { "checkbox": true }
}
```

### Date

```json
{
  "Last Contact": {
    "date": { "start": "2026-04-28" }
  }
}
```

ISO 8601 format. For just a date, no time component needed.

### People

Skip — requires Notion user IDs which FullEnrich doesn't provide.

### Files & media, Relation, Rollup, Formula, Created/Last edited

Skip these — either auto-computed by Notion or require complex IDs.

### Format checks before sending

- **Title**: trim whitespace; if empty, skip the contact entirely.
- **Email**: must contain `@` and `.`. Malformed → omit (or fall back to rich_text if column type allows).
- **Phone**: pass through as-is.
- **Select / multi_select**: option must exist on property; otherwise omit.
- **URL**: should start with `http://` or `https://`. If not, prepend `https://`.

### Common errors and fixes

| Error | Fix |
|---|---|
| `body.properties.X.title should be defined` | Property mapped to title is wrong — find the actual `type: "title"` property in the schema. |
| `Invalid property identifier` | Property name doesn't match schema exactly (case-sensitive). Use `notion-fetch` output verbatim. |
| `Email is not a valid email` | Malformed value from FullEnrich. Omit the key. |
| `select.name is not a valid option` | Option doesn't exist. Skip the field for this contact. |
| `Could not find database` | Stored database ID is invalid (deleted/moved). Trigger re-setup. |

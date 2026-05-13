---
name: FullEnrich - Push to Airtable
description: Automatically send FullEnrich search and enrichment results into the user's Airtable base with zero friction. On first use, asks the user once for an Airtable URL (which contains both the base ID and table ID) and whether to auto-push or confirm — then memorizes this. On every subsequent FullEnrich search or enrich call, the skill upserts records into Airtable (deduplicating by email), adapting to whatever fields exist in the user's table. Use this skill whenever FullEnrich's `search_contact`, `enrich_contact`, `enrich_contact_async`, or `enrich_bulk` returns results, even if the user did not mention Airtable in the current message — the skill handles everything.
---

# FullEnrich - Push to Airtable

One-time setup (paste an Airtable URL), then auto-push. Deduplicates on email automatically. Adapts to any user's Airtable schema.

## Required MCPs

- **FullEnrich MCP** — `https://mcp.fullenrich.com/mcp`
- **Airtable MCP** — Airtable MCP server (community/Composio implementation; expects standard tools: `list_bases`, `list_tables`, `get_base_schema`, `list_records`, `create_record`, `update_record`, or equivalents)

If either is missing, tell the user which one to connect and stop.

## Memory model

This skill relies on Claude's memory to remember setup across conversations. After `/setup`, use the `memory_user_edits` tool to persist:

- `FullEnrich-Airtable skill: target = base "<base name>" / table "<table name>" (base_id: <appXXX>, table_id: <tblYYY>)`
- `FullEnrich-Airtable skill: push mode = auto` *(or `push mode = confirm`)*

Before doing anything in this skill, check userMemories for these two lines. If both are present, skip setup and go straight to the push flow.

---

## Flow A — First use (no memory yet)

When FullEnrich returns results AND no setup is in memory, run `/setup` inline before pushing:

### Step 1: Ask for the Airtable URL

> Before I push these to Airtable, quick one-time setup. Paste the URL of your Airtable table (just open the table in your browser and copy the URL from the address bar).

Wait for the URL.

### Step 2: Parse the URL and resolve names

Airtable URLs follow this pattern:
```
https://airtable.com/{base_id}/{table_id}/{view_id?}/{record_id?}
```

Where:
- `base_id` starts with `app` (e.g. `appAbC123XyZ`)
- `table_id` starts with `tbl` (e.g. `tblXyZ789AbC`)
- `view_id` (optional) starts with `viw` — ignore
- `record_id` (optional) starts with `rec` — ignore

Extract `base_id` and `table_id` from the URL. If either is missing, ask the user to paste a URL that includes a table.

Then call `get_base_schema` (or equivalent — fetch base structure) with the base_id to:
- Get the **base name** (for memory display)
- Get the **table name** (for memory display)
- Get the **field list** with names and types

If the URL is malformed or the base/table can't be resolved, ask the user to paste it again.

### Step 3: Ask for push preference

> Got it — base **`<base name>`**, table **`<table name>`**. One last thing: after future searches/enrichments, do you want me to push automatically, or confirm with you first each time?
>
> 1. Push automatically
> 2. Confirm before each push

### Step 4: Save to memory

Use `memory_user_edits` with command `add` to persist both lines. Confirm briefly:

> Saved. From now on, FullEnrich results will go to **`<table name>`** in **`<base name>`** <auto / after confirmation>. Duplicates are handled automatically (dedup on email).

### Step 5: Push the current batch

Continue immediately to **Flow C — Push** below.

---

## Flow B — Subsequent use (memory present)

When FullEnrich returns results AND memory has the target:

- If push mode = `auto` → go straight to Flow C, push, report.
- If push mode = `confirm` → end your message with one short line: *"Push these N contacts to **`<table name>`** in Airtable?"*. On confirmation, run Flow C.

Don't re-run setup. Don't ask which fields to map. Just adapt to the schema and push.

---

## Flow C — Push (the actual work)

### 1. Fetch the table schema

Call `get_base_schema` with the stored base_id. Find the table matching the stored table_id, and extract:
- **Field list** with `name`, `type`, and (for select/status) `options`
- The **primary field** (first field in the table — every Airtable table has one, it's the field shown in the row's name column)

### 2. Identify the email field for dedup

Find a field that maps to email by fuzzy matching field names (case-insensitive, ignore spaces/underscores/hyphens):
- patterns: `email`, `e-mail`, `work email`, `professional email`, `mail`, `email address`

If no email-like field exists, dedup is impossible — fall back to "always create new" mode and mention this in the report: *"⚠️ No email field found; created N new records without dedup."*

### 3. Adapt the FullEnrich payload to the user's fields

For each FullEnrich contact, build a `fields` object by **scanning the table's actual fields** and matching what FullEnrich provides.

**Primary field** (mandatory — the row needs a value here, it's how Airtable displays the row):
- If the primary field is name-like (matches `name`, `full name`, `contact`, etc.) → set to FullEnrich's `full_name`.
- If the primary field is email-like → set to FullEnrich's email.
- Otherwise → set to FullEnrich's `full_name` as fallback.
- If the primary value would be empty, skip the contact.

**Other fields** — fuzzy match on field name (case-insensitive, ignore spaces/underscores/hyphens):

| FullEnrich field | Field name patterns to match |
|---|---|
| `full_name` | name, full name, contact, person, prospect, lead |
| `email` (work email) | email, e-mail, work email, professional email, mail |
| `phone` | phone, phone number, mobile, tel, telephone, cell |
| `company.name` | company, company name, organization, account, employer, business |
| `job_title` | title, job title, role, position, headline |
| `linkedin_url` | linkedin, linkedin url, profile, social, li |
| `industry` | industry, sector, vertical |
| `location` | location, city, country, region, geo |
| `seniority` | seniority, level, rank |

Match **as many fields as the table has columns for**. Skip silently when no match. **Never** create new fields.

### 4. Format values per Airtable field type

See the **Airtable Field Types Reference** section below for exact JSON shapes. Quick rules:

- `singleLineText` / `multilineText` → plain string
- `email` → plain string (validated email format)
- `phoneNumber` → plain string
- `url` → plain string with `https://` scheme
- `singleSelect` → `"Option Name"` — **only if the option exists** on that select. Skip otherwise.
- `multipleSelects` → `["Option1", "Option2"]` — same rule.
- `number` → actual number (not stringified)
- `checkbox` → boolean
- `date` → `"YYYY-MM-DD"` ISO format

### 5. Dedup logic (find-then-update / find-then-create)

Airtable has no native upsert, so for each contact:

**a) Search for existing record by email**

If FullEnrich provides an email AND an email field exists in the table:
- Call `list_records` with a `filterByFormula` like:
  ```
  {Email}='jane@acme.com'
  ```
  (use the actual field name, in single curly braces; escape single quotes in the email if any)
- If the result returns 1+ records, take the first one's `record_id`.

**b) Update or create**

- **If a matching record was found** → call `update_record` with that record_id and the mapped `fields` object. Use **PATCH semantics** (only the fields you provide get updated; existing values for other fields are preserved). Counts as an "updated" record in the report.
- **If no match was found** (or no email available) → call `create_record` with the mapped `fields` object. Counts as a "new" record.

For records without an email at all (e.g. raw search results), always create.

### 6. Execute and rate-limit awareness

Airtable's API has rate limits (5 requests/second per base). For batches >10 contacts, this matters.

- Many Airtable MCPs expose batch endpoints (e.g. `create_records` plural, accepting up to 10 records at once). Use the batch endpoint if available.
- If only single-record endpoints are available, call them sequentially. For >20 contacts, mention progress in the report.

Don't worry about retries unless an explicit rate-limit error comes back.

### 7. Report

Keep it tight — 1-2 lines max:

> ✓ 12 contacts synced to **Leads** in Airtable → [open table](https://airtable.com/appXXX/tblYYY)

If some were duplicates that got updated:

> ✓ 12 contacts processed (8 new, 4 updated existing) → [open table]

If some failed or were skipped:

> ✓ 10 pushed, 2 skipped (no name) → [open table]

If no email field exists in the table:

> ✓ 12 new records created in **Leads** → [open table]
> ⚠️ Heads up: no email field found in this table, so no dedup was possible. To enable dedup next time, add an email-type field.

---

## Changing the setup

If the user says any of:
- "change the Airtable table"
- "use a different base/table"
- "switch to confirm mode" / "switch to auto"
- "/setup" or "/reset"

Re-run Flow A and overwrite the memory lines using `memory_user_edits` with command `replace`.

If the user says "stop pushing to Airtable" or similar, use `memory_user_edits` with command `remove` to delete both lines, and confirm: *"Auto-push disabled. Run /setup again anytime to re-enable."*

---

## Edge cases

**Async enrichment not finished** — If `enrich_contact_async` or `enrich_bulk` was called but `get_enrichment_results` hasn't been polled, poll it first. Push only the ready ones; mention pending count if any remain.

**Empty results** — 0 contacts returned from FullEnrich → don't trigger anything, stay silent.

**Search vs enrich** — Both trigger the push. Search results often lack email — those records get created (no dedup possible per record). Mention in the report when this happens.

**Table doesn't have an email field at all** — Skip dedup for the whole batch, always create. Surface this in the report (see step 7).

**Companies (`search_company`)** — This skill is contact-focused. If the user explicitly asks to push companies, do it manually using a similar pattern.

**Single-select / multi-select option not found** — If FullEnrich returns `industry: "Fintech"` but the singleSelect field has only `["SaaS", "E-commerce"]`, skip that field for that contact. Don't auto-create options (Airtable would error on creation unless explicitly enabled).

**Linked records (e.g. linked Companies table)** — If the table has a linked-record field for company, the value must be an array of record IDs from the linked table, not a string. This is complex — skip linked-record fields by default and surface in the report: *"Skipped 'Company (linked)' field — linked-record fields not auto-populated."* The user can map manually.

**URL parsing fails** — If the user pastes something that isn't an Airtable URL (or pastes a base URL without a table), tell them what to do: *"Open the specific table you want in Airtable, then copy the URL from the browser. It should look like `https://airtable.com/app.../tbl.../...`."*

**Stored base/table invalid** — If `get_base_schema` returns "not found" or the table_id no longer exists in the base, tell the user the table may have been deleted or moved, and offer `/setup`.

**Rate limiting** — If you hit Airtable's 5 req/sec limit, wait briefly and retry. Don't surface this to the user unless it persists.

---

## What this skill does NOT do

- ❌ Create or modify Airtable bases or tables (we adapt to existing schemas)
- ❌ Add fields to the user's table
- ❌ Auto-create select options
- ❌ Populate linked-record fields automatically (too complex/risky without explicit mapping)
- ❌ Push companies automatically (contacts only)
- ❌ Delete or archive existing records

---

## Airtable Field Types Reference

Use this section when constructing the `fields` object for `create_record`, `update_record`, or batch equivalents.

The `fields` object is keyed by **field name** (the human-readable name you see in Airtable, case-sensitive). Field IDs (starting with `fld...`) also work and are more stable, but names are what most Airtable MCPs accept by default.

Example structure for creating a contact record:

```json
{
  "base_id": "appAbC123XyZ",
  "table_id": "tblXyZ789AbC",
  "fields": {
    "Name": "Jane Doe",
    "Email": "jane@acme.com",
    "Phone": "+14155550100",
    "Company": "Acme Corp",
    "Job Title": "VP Marketing"
  }
}
```

### singleLineText

```json
{ "Name": "Jane Doe" }
```

Plain string. Most common type. Length limit ~100k chars but keep reasonable.

### multilineText (long text)

```json
{ "Notes": "Multiple lines\nof text are fine here" }
```

Plain string with `\n` for line breaks.

### email

```json
{ "Email": "jane@acme.com" }
```

Plain string. Airtable validates the format and will reject malformed emails.

If FullEnrich returns null/empty, **omit the field key entirely** rather than passing empty string.

### phoneNumber

```json
{ "Phone": "+1 415 555 0100" }
```

Plain string. Airtable is permissive — passes through whatever format you send. Prefer E.164 (`+14155550100`) when available.

### url

```json
{ "LinkedIn": "https://linkedin.com/in/janedoe" }
```

Must include `https://` or `http://`. If FullEnrich returns `linkedin.com/in/x` without scheme, prepend `https://`.

### singleSelect

```json
{ "Industry": "Software" }
```

Pass the **option name** (case-sensitive, exact match).

⚠️ **Critical rule**: Only use if the option exists. Check the field's `options.choices` array from `get_base_schema`. If FullEnrich returns `"Fintech"` but options are `["SaaS", "E-commerce", "B2B"]`, **skip the field** for that contact.

By default, Airtable does NOT auto-create new options via API. Some MCPs expose a `typecast: true` parameter to allow this — use only if explicitly requested.

### multipleSelects

```json
{ "Tags": ["Lead", "Enriched"] }
```

Array of option names. Same option-existence rule as singleSelect.

### number

```json
{ "Score": 87 }
```

Pass as **actual number** (integer or float), not a string. Airtable will reject `"87"` for number fields.

If FullEnrich returns a string score, parse with `parseInt`/`parseFloat`. If parse fails, omit.

### currency

```json
{ "Deal Value": 10000 }
```

Number. The currency symbol is configured at the field level in Airtable; the API just takes the number.

### percent

```json
{ "Conversion Rate": 0.42 }
```

Decimal — `0.42` for 42%, NOT `42`.

### checkbox

```json
{ "Verified": true }
```

Boolean (real boolean, not string).

### date

```json
{ "Last Contacted": "2026-04-28" }
```

ISO 8601 date format `YYYY-MM-DD`.

### dateTime

```json
{ "Last Activity": "2026-04-28T14:30:00.000Z" }
```

ISO 8601 datetime, UTC. Include the trailing `Z`.

### rating

```json
{ "Priority": 4 }
```

Integer 0 to max (max is configured per field, typically 5).

### duration

```json
{ "Call Duration": 1800 }
```

Number of seconds.

### linkedRecord (link to another table)

```json
{ "Company": ["recXyz789AbC"] }
```

Array of **record IDs** (strings starting with `rec...`) from the linked table.

⚠️ **This is hard**: you can't pass a company name like `"Acme Corp"` directly. You need to first look up the company's record ID via `list_records` on the linked table, then pass the ID.

For the FullEnrich workflow, **skip linked-record fields by default** unless the user explicitly maps them. Linking a contact to a company auto-magically requires a multi-step lookup (find company by name in the linked table → use its ID) which is fragile.

If FullEnrich returns a company name and the user really wants to link, surface as: *"Skipped 'Company (linked)' field. To populate, you'd need a separate Companies table — let me know if you want me to set up that workflow."*

### attachment

Skip — requires file uploads via URLs and is rarely useful for FullEnrich data.

### user (collaborator)

Skip — requires Airtable user IDs which FullEnrich doesn't provide.

### formula / rollup / lookup / count / autoNumber / createdTime / lastModifiedTime / createdBy / lastModifiedBy / button

NOT WRITABLE — these are computed by Airtable. Never include in `fields`.

### externalSyncSource

NOT WRITABLE.

### Format checks before sending

- **Primary field**: trim whitespace; if empty, skip the contact entirely.
- **email**: must contain `@` and `.`; malformed → omit key.
- **phoneNumber**: pass through; Airtable is permissive.
- **singleSelect / multipleSelects**: option must exist on the field; otherwise omit.
- **url**: must start with `http://` or `https://`; prepend `https://` if missing.
- **number / currency / rating**: must be a valid number; if FullEnrich returns string, parse first.
- **date / dateTime**: must be ISO format; if FullEnrich returns a different format, normalize first.

### Common errors and fixes

| Error | Fix |
|---|---|
| `INVALID_VALUE_FOR_COLUMN` on a select | Option doesn't exist. Skip the field (or use `typecast: true` if MCP supports it). |
| `INVALID_RECORD_FIELDS` | Field name doesn't match table schema (case-sensitive!). Re-fetch schema and use exact name. |
| `ROW_DOES_NOT_EXIST` on update | The record_id you passed doesn't exist in this table. Probably a stale find result — re-search. |
| `NOT_FOUND` on the table | Stored table_id is invalid (table deleted/moved). Trigger re-setup. |
| `INVALID_REQUEST` (rate limit) | Hit 5 req/sec. Wait a moment and retry. |
| `Cannot accept the provided value for field` (linked record) | Field is a linked-record type. Pass `["rec..."]` array of IDs, not a string. |

### Tip: how to check if a select option exists

`get_base_schema` returns each field with details. For a singleSelect, the structure looks like:

```json
{
  "id": "fld123",
  "name": "Industry",
  "type": "singleSelect",
  "options": {
    "choices": [
      { "id": "sel123", "name": "SaaS", "color": "blueLight2" },
      { "id": "sel456", "name": "E-commerce", "color": "greenLight2" }
    ]
  }
}
```

Match against `options.choices[].name` (case-sensitive).

### Tip: filterByFormula for dedup search

For finding an existing record by email:

```
{Email}='jane@acme.com'
```

- Field name in single curly braces: `{Email}`
- Value in single quotes: `'jane@acme.com'`
- Escape single quotes in the value: `O'Brien` → `O\'Brien`
- Multiple conditions: `AND({Email}='x@y.com', {Status}='Active')`

Common gotcha: spaces in field names. `{Job Title}` works fine; double quotes don't.

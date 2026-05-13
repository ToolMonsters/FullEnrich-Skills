---
name: FullEnrich - Push to Attio
description: Automatically send FullEnrich search and enrichment results into the user's Attio CRM with zero friction. On first use, asks the user once which Attio list to push contacts into and whether to auto-push or confirm — then memorizes this. On every subsequent FullEnrich search or enrich call, the skill upserts contacts into Attio People (deduplicating by email) and adds them to the chosen list, adapting to whatever attributes exist on the user's people object and list. Use this skill whenever FullEnrich's `search_contact`, `enrich_contact`, `enrich_contact_async`, or `enrich_bulk` returns results, even if the user did not mention Attio in the current message — the skill handles everything.
---

# FullEnrich - Push to Attio

One-time setup, then auto-push. Upserts contacts into Attio People (no duplicates) and adds them to the user's chosen list. Adapts to any user's Attio schema.

## Required MCPs

- **FullEnrich MCP** — `https://mcp.fullenrich.com/mcp`
- **Attio MCP** — `https://mcp.attio.com/mcp`

If either is missing, tell the user which one to connect and stop.

## Memory model

This skill relies on Claude's memory to remember setup across conversations. After `/setup`, use the `memory_user_edits` tool to persist:

- `FullEnrich-Attio skill: target list = "<list name>" (id: <list_id>)`
- `FullEnrich-Attio skill: push mode = auto` *(or `push mode = confirm`)*

Before doing anything in this skill, check userMemories for these two lines. If both are present, skip setup and go straight to the push flow.

The target object is **always `people`** for this skill — we don't make the user choose. FullEnrich returns contacts, contacts go to People in Attio. End of story.

---

## Flow A — First use (no memory yet)

When FullEnrich returns results AND no setup is in memory, run `/setup` inline before pushing:

### Step 1: Ask for the target list

> Before I push these to Attio, quick one-time setup. Which Attio list should I add contacts to? (e.g. "Leads", "Sales Pipeline", "Inbound") — name or list URL works.

Wait for the answer.

### Step 2: Resolve the list

- **URL given** → extract list ID from the URL.
- **Name given** → call `list-lists` to fetch all available lists. Find the one matching the user's name. If multiple match, list them and ask which.
- **Nothing matches** → ask the user to paste the list URL or to create the list first in Attio.

Confirm the list's parent object is `people` (or supports people). If the list is configured for companies only, tell the user and ask for a different list.

### Step 3: Ask for push preference

> Got it — `<list name>`. One last thing: after future searches/enrichments, do you want me to push automatically, or confirm with you first each time?
>
> 1. Push automatically
> 2. Confirm before each push

### Step 4: Save to memory

Use `memory_user_edits` with command `add` to persist both lines. Confirm briefly:

> Saved. From now on, FullEnrich results will go to **People** → list **`<list name>`** <auto / after confirmation>. Duplicates are handled automatically (upsert on email).

### Step 5: Push the current batch

Continue immediately to **Flow C — Push** below. Don't make the user repeat the request.

---

## Flow B — Subsequent use (memory present)

When FullEnrich returns results AND memory has the target list:

- If push mode = `auto` → go straight to Flow C, push, report.
- If push mode = `confirm` → end your message with one short line: *"Push these N contacts to **`<list name>`** in Attio?"*. On confirmation, run Flow C.

Don't re-run setup. Don't ask which fields to map. Just adapt to the schema and push.

---

## Flow C — Push (the actual work)

### 1. Fetch the People object's attributes

Call `list-attribute-definitions` with `object: "people"`. Cache the result for this conversation. Extract each attribute's `api_slug`, `type`, `is_multiselect`, and (for select/status) the available options.

### 2. Adapt the FullEnrich payload to the user's attributes

For each FullEnrich contact, build the `values` object for `upsert-record` by **scanning the user's actual people attributes** and matching what FullEnrich provides.

**Required for upsert** — the contact MUST have an email (this is the matching attribute). If a FullEnrich result has no email, fall back to `create-record` for that single contact (no dedup possible without email), or skip if there's no name either.

**Standard mapping** (fuzzy match on `api_slug`, case-insensitive, ignore underscores/hyphens):

| FullEnrich field | Attio attribute slugs to try | Attio type expected |
|---|---|---|
| `firstname` + `lastname` | `name` | personal_name |
| `email` | `email_addresses` | email_address (multiselect) |
| `phone` | `phone_numbers` | phone_number (multiselect) |
| `company.name` | `company` (record reference), or fallback `company_name` (text) | record_reference or text |
| `company.domain` | `company` (record ref via domain) | record_reference |
| `job_title` | `job_title`, `title`, `role` | text |
| `linkedin_url` | `linkedin`, `linkedin_url`, `social_linkedin` | text or url |
| `industry` | `industry` | select or text |
| `location` | `primary_location`, `location` | location or text |
| `seniority` | `seniority`, `level` | select or text |

Match **as many fields as the user's people object has attributes for**. Skip silently when no match. **Never** create new attributes.

### 3. Format values per Attio attribute type

See the **Attio Attribute Types Reference** section below for exact JSON shapes. Quick rules:

- `personal_name` → `"Lastname, Firstname"` string format (note: last name FIRST). For Jane Doe: `"Doe, Jane"`.
- `email_address` (multiselect) → `["jane@acme.com"]` (always array, even for one email)
- `phone_number` (multiselect) → `["+14155550100"]` (always array)
- `text` → plain string
- `url` → plain string
- `select` / `status` → use the option's title (case-sensitive). **Only if the option exists** — check the attribute's options from step 1. Skip if no match.
- `record_reference` (e.g., `company`) → use simple format: `"company": "acme.com"` to match by domain (Attio resolves to the company record, creating it if needed). This is the cleanest way to link contacts to companies.

### 4. Execute the upsert

For each contact, call `Attio:upsert-record`:

```
{
  "object": "people",
  "matching_attribute": "email_addresses",
  "values": { /* mapped values from step 3 */ }
}
```

Attio returns the record ID (whether created or updated). **Save this ID** — you need it for step 5.

For contacts without email: fall back to `Attio:create-record` (no dedup). Skip contacts with neither name nor email entirely.

### 5. Add to the chosen list

For each upserted record, call `Attio:add-record-to-list`:

```
{
  "list": "<stored list_id>",
  "parent_object": "people",
  "parent_record_id": "<id from step 4>",
  "allow_duplicates": false
}
```

`allow_duplicates: false` means if the contact is already in the list, you'll get an error with the existing entry ID — that's fine, treat it as "already added" and continue.

### 6. Report

Keep it tight — 1-2 lines max:

> ✓ 12 contacts upserted to **People** and added to **Leads** in Attio → [open list](https://app.attio.com/...)

If some were duplicates of existing records:

> ✓ 12 contacts processed (8 new, 4 updated existing). All in **Leads** → [open list]

If some failed:

> ✓ 10 pushed, 2 skipped (missing both name and email) → [open list]

---

## Changing the setup

If the user says any of:
- "change the Attio list"
- "use a different list"
- "switch to confirm mode" / "switch to auto"
- "/setup" or "/reset"

Re-run Flow A and overwrite the memory lines using `memory_user_edits` with command `replace`.

If the user says "stop pushing to Attio" or similar, use `memory_user_edits` with command `remove` to delete both lines, and confirm: *"Auto-push disabled. Run /setup again anytime to re-enable."*

---

## Edge cases

**Async enrichment not finished** — If `enrich_contact_async` or `enrich_bulk` was called but `get_enrichment_results` hasn't been polled, poll it first. Push only the ready ones; mention pending count if any remain.

**Empty results** — 0 contacts returned from FullEnrich → don't trigger anything, stay silent.

**Search vs enrich** — Both trigger the push. Search results often have less data (no email yet) — for those, fall back to `create-record` since upsert needs an email matching value. Mention in the report when this happens: *"3 contacts created without email (no dedup possible)."*

**Companies (`search_company`)** — This skill is contact-focused. If the user explicitly asks to push companies, do it manually using a similar pattern with `object: "companies"` and `matching_attribute: "domains"`.

**List parent object mismatch** — If the user picked a list whose parent object is `companies` (not `people`), surface this in step 2 and ask for a different list.

**Select/status option not found** — If FullEnrich returns `industry: "Fintech"` but the Attio select has only `["SaaS", "E-commerce"]`, skip that field for that contact. Don't auto-create.

**List entry attributes** — Some Attio lists have additional entry-level attributes (stage, priority, etc.). This skill doesn't populate them by default — entries are added with default values. If the user wants to set a default stage (e.g. "New Lead"), they can mention it in setup and it will be added to memory and applied via `entry_values` on `add-record-to-list`.

**Stored list ID invalid** — If `add-record-to-list` returns a "list not found" error, tell the user the list may have been deleted, and offer `/setup`.

---

## What this skill does NOT do

- ❌ Create new objects, lists, or attributes (we adapt to existing schemas)
- ❌ Push companies automatically (contacts only — see scope)
- ❌ Auto-create select/status options
- ❌ Set list entry attributes beyond what's configured at setup
- ❌ Delete or archive existing records

---

## Attio Attribute Types Reference

Use this section when constructing the `values` object for `upsert-record`, `create-record`, or `add-record-to-list`'s `entry_values`. The object is keyed by **attribute `api_slug`** (or `attribute_id` UUID).

Example structure for a person upsert:

```json
{
  "object": "people",
  "matching_attribute": "email_addresses",
  "values": {
    "name": "Doe, Jane",
    "email_addresses": ["jane@acme.com"],
    "phone_numbers": ["+14155550100"],
    "company": "acme.com",
    "job_title": "VP Marketing"
  }
}
```

### personal_name (the name attribute on People)

Two formats accepted, simple is preferred:

```json
{ "name": "Doe, Jane" }
```

⚠️ **Last name FIRST**, separated by `, ` (comma-space). For "Jane Doe", pass `"Doe, Jane"`.

If FullEnrich returned `firstname: "Jane"` and `lastname: "Doe"`, format as `"${lastname}, ${firstname}"`.

Object format (verbose, all 3 fields required):

```json
{
  "name": [{
    "first_name": "Jane",
    "last_name": "Doe",
    "full_name": "Jane Doe"
  }]
}
```

If FullEnrich only returns `full_name`, try to split on the last space — last word as `last_name`, the rest as `first_name`. Or use `full_name` as both first and last in fallback.

### email_address (multiselect)

```json
{ "email_addresses": ["jane@acme.com"] }
```

**Always an array**, even for one email. The slug on People is typically `email_addresses` (plural).

This is also the standard `matching_attribute` for upsert.

If multiple emails from FullEnrich (rare): `["work@x.com", "personal@y.com"]`.

### phone_number (multiselect)

```json
{ "phone_numbers": ["+14155550100"] }
```

Always array. Pass in E.164 format if possible (with country code). Attio is permissive but stricter formats avoid downstream issues.

### text

```json
{ "job_title": "VP Marketing" }
```

Plain string. Used for things like `job_title`, `headline`, custom text fields.

### url

```json
{ "linkedin": "https://linkedin.com/in/janedoe" }
```

Must include the `https://` scheme. If FullEnrich returns `linkedin.com/in/x` without scheme, prepend `https://`.

### select (single value)

```json
{ "industry": "Software" }
```

Pass the **option title** (case-sensitive). Use the UUID only if you have it.

⚠️ **Critical**: Only use if the option exists. Check `list-attribute-definitions` → the attribute's `options` array. If FullEnrich returns `"Fintech"` but options are `["SaaS", "E-commerce", "B2B"]`, **skip** the field.

### status (single value, like select but with workflow semantics)

```json
{ "stage": "Working on it" }
```

Same rule as select — only existing statuses.

### select with multiselect=true

```json
{ "tags": ["Lead", "Enriched"] }
```

Array of option titles. Same rule about existing options.

### record_reference (linking to another record, e.g. company)

**Three formats** supported, in order of preference:

**Format A — Simple (recommended for FullEnrich workflow)**

For `company` on People (single-object reference):

```json
{ "company": "acme.com" }
```

Pass the company's **domain** as a string. Attio matches against existing companies by domain. If no company exists with that domain, Attio **auto-creates** one. This is exactly what we want for FullEnrich workflows.

**Format B — By matching attribute**

```json
{
  "company": [{
    "target_object": "companies",
    "domains": [{ "domain": "acme.com" }]
  }]
}
```

Same outcome, more verbose. Use only if format A fails.

**Format C — By record ID (only if you already have the company UUID)**

```json
{
  "company": [{
    "target_object": "companies",
    "target_record_id": "0f0cc00c-2c97-4130-8b42-3b5c7dfea4c6"
  }]
}
```

For the FullEnrich workflow, **always prefer Format A** with the company domain. If FullEnrich didn't return a domain (just a name), use Format A with the name as a string fallback — Attio will try to match by name.

### location

Simple format (preferred):

```json
{ "primary_location": "Tel Aviv, Israel" }
```

Object format if you have structured data:

```json
{
  "primary_location": {
    "line_1": "1 Infinite Loop",
    "locality": "Cupertino",
    "region": "CA",
    "postcode": "95014",
    "country_code": "US"
  }
}
```

FullEnrich usually returns just a string like `"Tel Aviv, Israel"` — use the simple format.

### number

```json
{ "employee_count": 100 }
```

Pass as actual number, not string.

### currency

```json
{ "deal_value": 10000 }
```

Number, in the attribute's configured currency. Not relevant for FullEnrich contact pushes.

### checkbox

```json
{ "is_active": true }
```

Boolean (real boolean, not string).

### date

```json
{ "founded_date": "2024-01-15" }
```

ISO 8601 date format.

### timestamp

```json
{ "created_at": "2024-01-15T10:30:00Z" }
```

ISO 8601 datetime.

### rating

```json
{ "priority": 3 }
```

Number 0-5.

### actor_reference (workspace member)

```json
{ "owner": "user@example.com" }
```

Workspace member email or UUID. Not typically populated from FullEnrich data.

### interaction

NOT WRITABLE — these attributes are read-only (last contact, last activity, etc.). Never include.

### domain (on Companies object)

```json
{ "domains": ["acme.com"] }
```

Array of domain strings. Used as `matching_attribute` for upserting companies (not relevant when pushing People).

### Format checks before sending

- **personal_name**: trim whitespace; if empty, skip the contact.
- **email_address**: must contain `@` and `.`; malformed → omit.
- **phone_number**: pass through; Attio is permissive.
- **select / status**: option must exist on attribute; otherwise omit.
- **url**: must start with `http://` or `https://`; prepend `https://` if missing.
- **record_reference (company)**: prefer domain over name when both available.

### Common errors and fixes

| Error | Fix |
|---|---|
| `Attribute X is not defined` | Slug doesn't match. Re-fetch via `list-attribute-definitions` and use the exact `api_slug`. |
| `Value for matching_attribute must be provided in values payload` | Upsert requires the matching attribute (usually `email_addresses`) inside `values`. Make sure it's there. |
| `Select option "X" not found` | Option doesn't exist on the select. Skip the field. |
| `Multiple records match the matching_attribute value` | Two People records share the same email — Attio data integrity issue. Surface to user. |
| `List not found` | Stored list ID is invalid. Trigger re-setup. |
| `List parent object mismatch` | The list expects companies but you're adding a person record. Check list config. |

### Tip: how to find the right slug

Attribute slugs are NOT necessarily the display name. The attribute "Job title" might have slug `job_title` or `title` depending on workspace config. Always call `list-attribute-definitions` first and use the returned `api_slug` verbatim.

For matching FullEnrich → Attio attributes:
1. Get the list of slugs from the schema.
2. Normalize both sides (lowercase, strip underscores/hyphens/spaces).
3. Match. If no match, skip silently.

---
name: FullEnrich - Push to HubSpot
description: Automatically send FullEnrich search and enrichment results into the user's HubSpot CRM with zero friction. On first use, asks the user once which HubSpot static list to add contacts to and whether to auto-push or confirm — then memorizes this. On every subsequent FullEnrich search or enrich call, the skill upserts contacts in HubSpot (deduplicating by email), adds them to the chosen list, and lets HubSpot auto-associate contacts with their companies via email domain. Adapts to whatever properties exist on the user's contact object. Use this skill whenever FullEnrich's `search_contact`, `enrich_contact`, `enrich_contact_async`, or `enrich_bulk` returns results, even if the user did not mention HubSpot in the current message — the skill handles everything.
---

# FullEnrich - Push to HubSpot

One-time setup, then auto-push. Upserts Contacts on email (no duplicates), adds them to a static List for pipeline visibility, and lets HubSpot's native company-association handle Contact→Company linking via email domain.

## Required MCPs

- **FullEnrich MCP** — `https://mcp.fullenrich.com/mcp`
- **HubSpot MCP** — official HubSpot MCP server (expects standard tools: `crm-create-object`, `crm-update-object`, `crm-search-objects`, `crm-batch-create-objects`, `crm-list-properties`, `lists-add-records`, `lists-search-lists`, or equivalents from the HubSpot MCP catalog)

If either is missing, tell the user which one to connect and stop.

## Memory model

This skill relies on Claude's memory to remember setup across conversations. After `/setup`, use the `memory_user_edits` tool to persist:

- `FullEnrich-HubSpot skill: target list = "<list name>" (id: <list_id>)`
- `FullEnrich-HubSpot skill: push mode = auto` *(or `push mode = confirm`)*

Before doing anything in this skill, check userMemories for these two lines. If both are present, skip setup and go straight to the push flow.

The target object is **always `contacts`** in this skill — FullEnrich returns contacts, contacts go to the Contacts object in HubSpot.

---

## Flow A — First use (no memory yet)

When FullEnrich returns results AND no setup is in memory, run `/setup` inline before pushing:

### Step 1: Ask for the target list

> Before I push these to HubSpot, quick one-time setup. Which HubSpot static list should I add contacts to? (e.g. "Inbound Leads", "Sales Pipeline", "Q2 Outreach") — name or list URL works.
>
> ⚠️ Use a **static list**, not an active list — active lists are filter-based and you can't manually add contacts to them.

Wait for the answer.

### Step 2: Resolve the list

- **URL given** → extract the list ID from the URL (the digits after `/lists/`).
- **Name given** → call `lists-search-lists` (or equivalent) to find lists matching the name. Filter for `objectTypeId = "0-1"` (contacts) and `processingType = "MANUAL"` (static). If multiple match, list them and ask which.
- **No match found** → ask the user to either paste the URL, or create a static list in HubSpot first.
- **List found but it's a dynamic/active list** → tell the user: *"That list looks like an active list (rule-based). I need a static list to add contacts manually. Want to create one in HubSpot, or pick a different list?"*

### Step 3: Ask for push preference

> Got it — `<list name>`. One last thing: after future searches/enrichments, do you want me to push automatically, or confirm with you first each time?
>
> 1. Push automatically
> 2. Confirm before each push

### Step 4: Save to memory

Use `memory_user_edits` with command `add` to persist both lines. Confirm briefly:

> Saved. From now on, FullEnrich results will be upserted in **Contacts** and added to **`<list name>`** <auto / after confirmation>. Companies will be auto-linked by email domain.

### Step 5: Push the current batch

Continue immediately to **Flow C — Push** below.

---

## Flow B — Subsequent use (memory present)

When FullEnrich returns results AND memory has the target list:

- If push mode = `auto` → go straight to Flow C, push, report.
- If push mode = `confirm` → end your message with one short line: *"Push these N contacts to **`<list name>`** in HubSpot?"*. On confirmation, run Flow C.

Don't re-run setup. Don't ask which fields to map. Just adapt to the schema and push.

---

## Flow C — Push (the actual work)

### 1. Fetch the Contacts object's properties

Call `crm-list-properties` with `objectType: "contacts"`. Cache the result for this conversation. Extract each property's `name` (internal name like `firstname`, `email`, `jobtitle`), `label` (display name), `type` (string, enumeration, datetime, number...), and (for enumerations) the available `options`.

### 2. Adapt the FullEnrich payload to the user's properties

For each FullEnrich contact, build a `properties` object by mapping FullEnrich fields to HubSpot property `name`s.

**HubSpot has standard contact properties** that most workspaces have. Map FullEnrich → HubSpot internal names directly when available, then fuzzy-fallback for custom properties:

| FullEnrich field | HubSpot standard property `name` | Fallback fuzzy patterns |
|---|---|---|
| `firstname` | `firstname` | first_name, given_name, prenom |
| `lastname` | `lastname` | last_name, family_name, surname, nom |
| `email` (work email) | `email` | email_address, work_email |
| `phone` | `phone` | phone_number, mobile, tel, telephone |
| `company.name` | `company` | company_name, employer, organization |
| `job_title` | `jobtitle` | job_title, title, role, position |
| `linkedin_url` | `hs_linkedin_url` | linkedin, linkedin_url, social_linkedin |
| `industry` | `industry` | sector, vertical |
| `location` (city) | `city` | location_city |
| `location` (country) | `country` | location_country, country_region |
| `seniority` | `seniority` | level, hierarchy |

**Rules**:
- For each FullEnrich field, first try the standard HubSpot property name. If that property doesn't exist in the user's schema, try the fuzzy fallbacks. If nothing matches, skip silently.
- **Never** create new properties.
- Don't split `firstname`/`lastname` if FullEnrich gave you `full_name` — try splitting on the last space (everything before = firstname, last word = lastname).

### 3. Format values per HubSpot property type

See the **HubSpot Property Types Reference** section below for exact rules. Quick rules:

- `string` → plain string
- `number` → stringified number (HubSpot accepts `"42"`)
- `bool` → `"true"` or `"false"` as string
- `enumeration` (single) → option's `value` (internal value, NOT label). **Only use if the option exists** — check `options` array. Skip otherwise.
- `enumeration` (multi, with `multipleCheckboxes`) → semicolon-separated string: `"option_a;option_b"`
- `date` → `"YYYY-MM-DD"`
- `datetime` → ISO 8601 datetime or Unix timestamp in ms

All values are sent as strings — HubSpot's API converts internally.

### 4. Upsert with email as identifier

HubSpot has a native upsert via the `idProperty` parameter on `crm-create-object` (or the v3 batch upsert endpoint). Use this:

For each contact with an email:
```
{
  "objectType": "contacts",
  "idProperty": "email",
  "id": "jane@acme.com",
  "properties": { /* mapped fields from step 3 */ }
}
```

This call will **create** the contact if no contact with that email exists, or **update** the contact if it does. Returns the contact's HubSpot `id`.

For batch efficiency, prefer `crm-batch-create-objects` with `idProperty: "email"` to upsert multiple contacts in one call (HubSpot supports up to 100 per batch).

For contacts without an email: fall back to `crm-create-object` without `idProperty` (no dedup possible). Skip contacts with neither name nor email.

### 5. Company auto-association (free, automatic)

**Don't do anything explicit here.** HubSpot has a setting (`AutomaticallyCreateAndAssociate`) that's **enabled by default** on most accounts. When you create a contact with an email like `jane@acme.com`:

1. HubSpot extracts the domain `acme.com`.
2. Looks for a Company record with `domain = acme.com`.
3. If found → auto-associates the contact with that company.
4. If not found → creates a new Company record with that domain and associates.

Just creating the contact triggers all of this. No extra API call needed.

**Edge case**: if the user has disabled this setting, contacts won't auto-link. Don't try to detect this — if the user complains, mention the setting in HubSpot Settings → Objects → Companies.

### 6. Add the contacts to the chosen list

After all contacts are upserted, collect their HubSpot IDs (from the upsert responses in step 4). Then call `lists-add-records` (or equivalent) once with the full array of IDs:

```
{
  "listId": "<stored list_id>",
  "recordIds": ["<contact_id_1>", "<contact_id_2>", ...]
}
```

This is a single batch call regardless of how many contacts. HubSpot doesn't error on adding contacts that are already in the list — they're just no-op'd.

### 7. Report

Keep it tight — 1-2 lines max:

> ✓ 12 contacts upserted in HubSpot and added to **Inbound Leads** → [open list](https://app.hubspot.com/contacts/.../objectLists/...)

If some were duplicates that got updated:

> ✓ 12 contacts processed (8 new, 4 updated existing). All in **Inbound Leads** → [open list]

If some failed or were skipped:

> ✓ 10 pushed, 2 skipped (missing both name and email) → [open list]

If you want to mention the company auto-association on the first push:

> ℹ Companies are auto-linked by email domain (HubSpot's default behavior). New companies were created for unrecognized domains.

Skip this note on subsequent pushes.

---

## Changing the setup

If the user says any of:
- "change the HubSpot list"
- "use a different list"
- "switch to confirm mode" / "switch to auto"
- "/setup" or "/reset"

Re-run Flow A and overwrite the memory lines using `memory_user_edits` with command `replace`.

If the user says "stop pushing to HubSpot" or similar, use `memory_user_edits` with command `remove` to delete both lines, and confirm: *"Auto-push disabled. Run /setup again anytime to re-enable."*

---

## Edge cases

**Async enrichment not finished** — If `enrich_contact_async` or `enrich_bulk` was called but `get_enrichment_results` hasn't been polled, poll it first. Push only the ready ones; mention pending count if any remain.

**Empty results** — 0 contacts returned from FullEnrich → don't trigger anything, stay silent.

**Search vs enrich** — Both trigger the push. Search results often lack email — for those, fall back to `crm-create-object` (no upsert, no dedup). Mention in the report when this happens: *"3 contacts created without email (no dedup or company auto-link possible)."*

**Companies (`search_company`)** — This skill is contact-focused. If the user explicitly asks to push companies, do it manually using `objectType: "companies"` and `idProperty: "domain"`.

**Active list given instead of static** — Surfaced in step 2. Active lists are rule-based; users can't manually add records.

**Enumeration option not found** — If FullEnrich returns `industry: "Fintech"` but the HubSpot enum has options `["SaaS", "E-commerce"]`, skip that field for that contact. Don't auto-create options. Note: HubSpot DOES allow creating new enum options via the properties API, but doing so silently clutters the user's setup — don't.

**Stored list ID invalid** — If `lists-add-records` returns a "list not found" error, tell the user the list may have been deleted, and offer `/setup`.

**Lifecycle Stage** — Don't set `lifecyclestage` automatically. This property triggers HubSpot workflows (lead-scoring, marketing automation, sales rep notifications). Setting it without user intent can dump the user's contacts into nurture sequences. If the user explicitly wants new contacts to be tagged at a specific stage (e.g. "Subscriber" or "Lead"), they should mention it during `/setup` and it gets added to memory and applied as a default property on each push.

**Contact owner** — Don't set `hubspot_owner_id` automatically. Same reason — owner assignments trigger notifications and routing rules.

**Rate limiting** — HubSpot's API allows 100 requests per 10 seconds per app on most plans. Batch upserts and the single list-add call keep us well under this limit even for 100-contact pushes. No special handling needed.

---

## What this skill does NOT do

- ❌ Create or modify HubSpot lists, properties, or workflows (we adapt to existing setup)
- ❌ Set `lifecyclestage` or `hs_lead_status` automatically (avoids triggering automations)
- ❌ Assign contact owners automatically
- ❌ Push to active (rule-based) lists — only static lists
- ❌ Auto-create enumeration options
- ❌ Push companies automatically (contacts only)
- ❌ Trigger marketing workflows or sequences

---

## HubSpot Property Types Reference

Use this section when constructing the `properties` object for `crm-create-object`, `crm-update-object`, or `crm-batch-create-objects`. The object is keyed by HubSpot's **internal property `name`** (not the human-readable `label`).

⚠️ **All values in HubSpot's API are sent as STRINGS**, even numbers and booleans. HubSpot converts internally based on the property type. This is unlike Notion/Attio/Airtable where types are explicit JSON.

Example structure for upserting a contact:

```json
{
  "objectType": "contacts",
  "idProperty": "email",
  "id": "jane@acme.com",
  "properties": {
    "firstname": "Jane",
    "lastname": "Doe",
    "email": "jane@acme.com",
    "phone": "+14155550100",
    "jobtitle": "VP Marketing",
    "company": "Acme Corp"
  }
}
```

### string (text properties)

```json
{ "firstname": "Jane" }
```

Plain string. Used for most text properties: `firstname`, `lastname`, `email`, `phone`, `jobtitle`, `company`, etc.

If FullEnrich returns null/empty, omit the key entirely.

### number

```json
{ "num_associated_deals": "5" }
```

⚠️ **Stringified**, not actual number. HubSpot's API expects `"5"` not `5`. If you pass the actual number, it usually still works but stringify to be safe.

### bool

```json
{ "hs_email_optout": "true" }
```

Stringified booleans: `"true"` or `"false"`. Not relevant for FullEnrich data.

### enumeration (single-select dropdown)

```json
{ "industry": "COMPUTER_SOFTWARE" }
```

⚠️ **Use the option's internal `value`, NOT the `label`.** When you call `crm-list-properties`, each enumeration property has an `options` array like:

```json
{
  "name": "industry",
  "type": "enumeration",
  "options": [
    { "label": "Computer Software", "value": "COMPUTER_SOFTWARE" },
    { "label": "E-commerce", "value": "ECOMMERCE" }
  ]
}
```

To match FullEnrich's `industry: "Computer Software"`:
1. Find the option where `label` (or `value`) matches.
2. Pass the `value` in the payload.

If no match found → skip the field. Don't auto-create options (would silently clutter the user's HubSpot setup).

### enumeration (multi-checkbox)

```json
{ "favorite_topics": "topic_a;topic_b" }
```

Semicolon-separated string of values (NOT array). Same option-existence rule.

### date

```json
{ "key_date": "2026-04-28" }
```

ISO date format `YYYY-MM-DD`. Or Unix timestamp in milliseconds at midnight UTC.

### datetime

```json
{ "createdate": "2026-04-28T14:30:00.000Z" }
```

ISO 8601 datetime with milliseconds and `Z`. Or Unix timestamp in milliseconds.

Most datetime properties (`createdate`, `lastmodifieddate`, `hs_lastmodifieddate`) are read-only — don't try to set them.

### phone_number

Internally stored as `string`, but HubSpot may auto-format. Pass in E.164 (`+14155550100`) when possible:

```json
{ "phone": "+14155550100" }
```

### Standard contact properties — quick reference

These are the most common standard properties on the Contacts object. Names are stable across all HubSpot accounts:

| Internal name | Type | Notes |
|---|---|---|
| `firstname` | string | First name |
| `lastname` | string | Last name |
| `email` | string | Primary email — used as `idProperty` for upsert |
| `phone` | string | Primary phone |
| `mobilephone` | string | Mobile-specific phone |
| `company` | string | **Display-only** company name on the contact (separate from associated Company record) |
| `jobtitle` | string | Job title |
| `industry` | enumeration | Often pre-populated; use option values |
| `city` | string | City |
| `state` | string | State/region |
| `country` | string | Country |
| `address` | string | Street address |
| `zip` | string | Postal code |
| `website` | string | Personal/company website |
| `hs_linkedin_url` | string | LinkedIn profile URL |
| `twitterhandle` | string | Twitter handle (no `@`) |
| `lifecyclestage` | enumeration | ⚠️ Triggers workflows. Don't set automatically. |
| `hs_lead_status` | enumeration | ⚠️ Triggers workflows. Don't set automatically. |
| `hubspot_owner_id` | string (user ID) | ⚠️ Triggers notifications. Don't set automatically. |

**Note on `company` (string) vs Company record (associated object)**:
The `company` string property on a Contact is just a text label — it doesn't link the contact to a Company record. The actual association happens through HubSpot's automatic company-association feature (via email domain), or via explicit association API calls. For this skill, just set the `company` string property; HubSpot handles the actual record link automatically via email domain.

### Format checks before sending

- **firstname / lastname**: trim whitespace; if both empty AND email is missing, skip the contact (HubSpot won't create a contact with neither).
- **email**: must contain `@` and `.`. Malformed → omit. If using `idProperty: "email"`, malformed email blocks upsert entirely — skip the contact.
- **phone**: pass through; HubSpot is permissive.
- **enumeration**: option must exist on property; otherwise omit.
- **All values**: convert to string before sending (HubSpot expects strings everywhere).

### Common errors and fixes

| Error | Fix |
|---|---|
| `Property X does not exist` | Property name doesn't match (case-sensitive!). Re-fetch via `crm-list-properties` and use exact `name` field. |
| `INVALID_OPTION` on an enumeration | Option `value` doesn't exist on the property. Skip the field. |
| `CONTACT_EXISTS` on create (without upsert) | Email already exists. Use `idProperty: "email"` to upsert instead. |
| `ID_PROPERTY_VALUE_REQUIRED` | When using `idProperty`, the `id` field must be in the payload. Add it. |
| `INVALID_EMAIL` | Email format malformed. Skip this contact for upsert. |
| `LIST_NOT_FOUND` on adding to list | Stored list ID is invalid. Trigger re-setup. |
| `OBJECT_NOT_IN_LIST_TYPE` | The list isn't a Contact list (could be a Company or Deal list). Pick a different list. |
| `ACTIVE_LIST_CANNOT_ADD_RECORDS` | The list is a dynamic/active list. Pick a static list instead. |

### Tip: Standard Contacts properties vs custom

Most HubSpot accounts have ~50 standard Contact properties + custom ones added by the user. Strategy:
1. Try the standard internal name first (e.g. `jobtitle` for job title).
2. If `crm-list-properties` shows that name exists, use it.
3. Otherwise, fall back to fuzzy matching against `label` (display name) of all properties.
4. If nothing matches, skip the field.

### Tip: how to handle `company` vs Company record

When you set `company: "Acme Corp"` on a contact:
- The text "Acme Corp" appears in HubSpot's contact card under the Company field.
- This is **separate** from associating the contact to a Company record.
- HubSpot's auto-association feature handles the actual record linking — based on the **email domain**, not the `company` string.

So setting both `email: "jane@acme.com"` AND `company: "Acme Corp"` is the right move:
- Email triggers domain-based auto-association → Company record link.
- `company` string fills the display field on the contact.

Both are useful and don't conflict.

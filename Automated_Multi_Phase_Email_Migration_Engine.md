# Lume & Lather Med Spa — Automated Multi-Phase Email Migration and Data Integrity Engine

This guide is written so you can build the workflow manually in n8n yourself, step by step, without depending on any JSON import.

It follows the same build-it-yourself style used throughout this workflow library.

It assumes:

- You are using a recent n8n version with the current `Manual Trigger`, `Spreadsheet File`, `Code`, `Set`, `IF`, `Switch`, `SplitInBatches`, `HTTP Request`, `Airtable`, `Google Sheets`, `Gmail`, and `Error Trigger` nodes.
- You already have these credentials available inside n8n:
  - `Google Sheets OAuth2`
  - `Gmail OAuth2`
  - `Airtable Personal Access Token`
  - An Abstract API key (or equivalent email validation API key) stored as an n8n credential or environment variable

Use these exact workflow names:

- `Lume & Lather - Email Migration Engine`
- `Lume & Lather - Migration Error Logger`

---

## What this workflow does

1. Accepts a CSV file of legacy client records through a manual trigger.
2. Parses the CSV into JSON items and splits them into manageable batches.
3. Normalizes all incoming fields (trim, lowercase, standardize).
4. Validates each email address in three stages: syntax check, disposable-domain check, and live API verification.
5. Routes invalid records into a dedicated error-logging branch.
6. Sanitizes clean records further before entering the logic engine.
7. Tiers clients by recency: Active clients (last visit ≤ 90 days) go to Airtable; Lapsed clients go to Google Sheets re-engagement.
8. Personalizes each record by skin concern category using a Switch node.
9. Detects VIP clients and sends a Gmail notification for each one.
10. Logs all migration errors into a Google Sheets error log with timestamp and reason.
11. Uses a separate error workflow to catch any node-level execution failures.

---

## Before you build

Create all external assets before adding a single node. Stopping halfway to create a spreadsheet breaks your testing flow.

### 1. Google Sheets spreadsheet

Create one spreadsheet. Name it:

```text
Lume & Lather - Migration Data
```

Create these four tabs with the exact header rows shown below.

#### Re-engagement (Tier 2 Lapsed clients)

```text
CustomerID | Email | FullName | LastServiceDate | SkinConcerns | NewsletterOptIn | SpendCategory | AccountStatus | SkinTier | MigratedAt
```

#### Migration Errors

```text
CustomerID | OldEmail | ErrorMessage | ErrorType | Timestamp
```

#### Error Logs (execution-level failures)

```text
Timestamp | WorkflowName | NodeName | ErrorMessage | ExecutionTime
```

#### Duplicate Tracker

```text
CustomerID | Email | FullName | DetectedAt | Reason
```

Keep the spreadsheet ID ready. You will use it in every Google Sheets node.

---

### 2. Airtable base

Create one Airtable base. Recommended name:

```text
Lume & Lather CRM
```

Create one table inside it. Name the table:

```text
Active Clients
```

Create these exact fields:

```text
CustomerID
Email
FullName
LastServiceDate
SkinConcerns
NewsletterOptIn
SpendCategory
AccountStatus
SkinTier
MigratedAt
```

Keep the base ID and table ID ready.

---

### 3. Gmail credential

Confirm your Gmail OAuth2 credential is connected and the sending address is verified.

The VIP notification emails will come from that address.

---

### 4. Abstract API key

Sign up at `https://www.abstractapi.com/` and get an Email Validation API key.

You will use it in the HTTP Request node later.

Keep the key ready. You can paste it directly into the HTTP Request node or store it as an n8n credential.

---

### 5. Your CSV input file

The CSV for this workflow uses this exact structure:

```text
CustomerID, OldEmail, FullName, LastServiceDate, SkinConcerns, NewsletterOptIn, SpendCategory, AccountStatus
```

Sample rows for reference:

```csv
LL-0001,sarah.j@gmail.com,Sarah Jenkins,2024-05-20,Anti-Aging,True,VIP,Active
LL-0002,mchen_99@yahoo.com,Michael Chen,2024-04-10,Acne-Prone,True,Medium,Active
LL-0003,elena.rod@mailinator.com,Elena Rodriguez,2023-11-15,Sensitive,False,High,Lapsed
LL-0004,dthompson.com,David Thompson,2024-01-05,Hyperpigmentation,True,Low,Inactive
LL-0010,igarcia@@icloud.com,Isabella Garcia,2024-06-10,Sensitive,True,Medium,Active
```

Note the known problem records in the dataset:
- `LL-0003` has a disposable domain (`mailinator.com`)
- `LL-0004` has an invalid email — no `@` sign
- `LL-0010` has a double `@@` syntax error

These will be caught and routed to the error branch.

---

## Architecture overview

The workflow has six logical layers that data passes through in sequence:

```text
[LAYER 1 — INGESTION]
  Manual Trigger → Read CSV File → Parse to JSON → Split Into Batches

[LAYER 2 — VALIDATION]
  Normalize Fields → Check Missing Fields → Regex Syntax Check → API Email Validation
  └─ Invalid path → Log to Migration Errors sheet

[LAYER 3 — SANITIZATION]
  Clean & Standardize Fields → Deduplicate Check

[LAYER 4 — LOGIC ENGINE]
  Tier Routing (date logic) → Personalization (skin concern) → VIP Detection

[LAYER 5 — DESTINATION SYSTEMS]
  Tier 1 Active  → Airtable Create Record
  Tier 2 Lapsed  → Google Sheets Append Row
  VIP            → Gmail Notification (parallel)

[LAYER 6 — ERROR SYSTEM]
  Error Trigger (separate workflow) → Normalize Error → Google Sheets Error Log
```

---

## Node build order

Build nodes in this order so each connection is immediately testable:

1. `Manual Trigger - Start Migration`
2. `Spreadsheet File - Read CSV`
3. `Code - Parse CSV to JSON`
4. `SplitInBatches - Process in Batches`
5. `Code - Normalize Fields`
6. `IF - Missing Fields?`
7. `Set - Tag Missing Field Error`
8. `Google Sheets - Log Migration Error`
9. `Code - Regex Email Check`
10. `IF - Invalid Syntax?`
11. `Set - Tag Syntax Error``
12. `Google Sheets - Log Syntax Error`
13. `HTTP Request - Validate Email API`
14. `IF - API Rejected Email?`
15. `Set - Tag API Error`
16. `Google Sheets - Log API Error`
17. `Code - Sanitize Fields`
18. `Code - Deduplicate Check`
19. `IF - Is Duplicate?`
20. `Google Sheets - Log Duplicate`
21. `IF - Active or Lapsed?`
22. `Switch - Route by Skin Concern`
23. `IF - Is VIP?`
24. `Gmail - Send VIP Notification`
25. `Airtable - Create Active Client Record`
26. `Google Sheets - Append Lapsed Client`
27. `Set - Migration Complete`

Error workflow:
28. `Error Trigger - Global`
29. `Set - Prepare Error Log`
30. `Google Sheets - Log Execution Error`

---

# STEP 0: Create both workflows

## 0A. Create the main workflow

1. Open n8n.
2. Click `Create Workflow`.
3. Rename it to:

```text
Lume & Lather - Email Migration Engine
```

4. Save once before adding any nodes.

## 0B. Create the error workflow

1. Create a second workflow.
2. Rename it to:

```text
Lume & Lather - Migration Error Logger
```

3. Save once.

---

# STEP 1: Add the manual trigger

## Add the trigger node

1. Add a `Manual Trigger` node.
2. Rename it to:

```text
Manual Trigger - Start Migration
```

This is the entry point. You will click `Execute Workflow` manually each time you want to run the migration.

No configuration is needed on this node.

---

# STEP 2: Read the CSV file

## Add the Spreadsheet File node

1. Connect `Manual Trigger - Start Migration` to a `Spreadsheet File` node.
2. Rename it to:

```text
Spreadsheet File - Read CSV
```

## Configure the node

Set:

- `Operation`: `Read from File`
- `File Format`: `CSV`
- `File Property`: `data`

In the `Options` section, set:

- `Header Row`: `true`

## How to pass your CSV

When you execute the workflow, this node reads the binary file from the trigger's output. To attach the CSV:

1. Open `Manual Trigger - Start Migration`.
2. In the test payload panel, attach your CSV binary under the property named `data`.

Alternatively, you can replace this node with a `Read Binary File` node pointing to a file path if you prefer file-system input.

## Expected output

After this node, each row in the CSV becomes one JSON item. A sample item looks like this:

```json
{
  "CustomerID": "LL-0001",
  "OldEmail": "sarah.j@gmail.com",
  "FullName": "Sarah Jenkins",
  "LastServiceDate": "2024-05-20",
  "SkinConcerns": "Anti-Aging",
  "NewsletterOptIn": "True",
  "SpendCategory": "VIP",
  "AccountStatus": "Active"
}
```

---

# STEP 3: Parse all rows and prepare for batching

The Spreadsheet File node returns all rows as a flat array. This Code node confirms the structure and adds a processing timestamp before batching begins.

## Add the Code node

1. Connect `Spreadsheet File - Read CSV` to a `Code` node.
2. Rename it to:

```text
Code - Parse CSV to JSON
```

3. Set:
   - `Mode`: `Run Once for All Items`
   - `Language`: `JavaScript`

4. Paste this exact code:

```javascript
const items = $input.all();
const parsed = [];

for (const item of items) {
  const row = item.json;

  // Skip completely empty rows
  const values = Object.values(row).filter(v => String(v ?? '').trim() !== '');
  if (values.length === 0) continue;

  parsed.push({
    json: {
      CustomerID: String(row.CustomerID ?? '').trim(),
      OldEmail: String(row.OldEmail ?? '').trim(),
      FullName: String(row.FullName ?? '').trim(),
      LastServiceDate: String(row.LastServiceDate ?? '').trim(),
      SkinConcerns: String(row.SkinConcerns ?? '').trim(),
      NewsletterOptIn: String(row.NewsletterOptIn ?? '').trim(),
      SpendCategory: String(row.SpendCategory ?? '').trim(),
      AccountStatus: String(row.AccountStatus ?? '').trim(),
      _parsedAt: new Date().toISOString(),
    }
  });
}

return parsed;
```

## What this node returns

Every item is now a clean JSON object. All fields are guaranteed to be strings. The `_parsedAt` timestamp is appended for audit purposes.

---

# STEP 4: Split into batches

Processing 100+ records in one pass can overload memory and slow your workflow. Batching solves this.

## Add the SplitInBatches node

1. Connect `Code - Parse CSV to JSON` to a `SplitInBatches` node.
2. Rename it to:

```text
SplitInBatches - Process in Batches
```

## Configure the node

Set:

- `Batch Size`: `10`
- `Options → Reset`: leave unchecked

This processes 10 records at a time and loops automatically until all records are processed.

## How the loop works

The SplitInBatches node has two outputs:

- `output 1` = the current batch of items (connect this to the rest of your pipeline)
- `output 2` = fires when all batches are done (you can connect a final Set node here if you want a completion signal)

All downstream nodes connect from `output 1` only.

---

# STEP 5: Normalize all fields

This is the first transformation. It makes all field values predictable before any validation or logic runs.

## Add the Code node

1. Connect `output 1` of `SplitInBatches - Process in Batches` to a `Code` node.
2. Rename it to:

```text
Code - Normalize Fields
```

3. Set:
   - `Mode`: `Run Once for Each Item`
   - `Language`: `JavaScript`

4. Paste this exact code:

```javascript
const row = $json;

// Normalize NewsletterOptIn to true/false boolean
const optInRaw = String(row.NewsletterOptIn ?? '').trim().toLowerCase();
const newsletterOptIn = optInRaw === 'true' || optInRaw === '1' || optInRaw === 'yes';

// Normalize SpendCategory to title-case
const spendRaw = String(row.SpendCategory ?? '').trim();
const spendCategory = spendRaw.charAt(0).toUpperCase() + spendRaw.slice(1).toLowerCase();

// Normalize AccountStatus to title-case
const statusRaw = String(row.AccountStatus ?? '').trim();
const accountStatus = statusRaw.charAt(0).toUpperCase() + statusRaw.slice(1).toLowerCase();

// Trim full name
const fullName = String(row.FullName ?? '').trim().replace(/\s+/g, ' ');

// Email: lowercase and trim only — do NOT correct malformed emails here
const email = String(row.OldEmail ?? '').trim().toLowerCase();

return [{
  json: {
    CustomerID: String(row.CustomerID ?? '').trim(),
    OldEmail: email,
    FullName: fullName,
    LastServiceDate: String(row.LastServiceDate ?? '').trim(),
    SkinConcerns: String(row.SkinConcerns ?? '').trim(),
    NewsletterOptIn: newsletterOptIn,
    SpendCategory: spendCategory,
    AccountStatus: accountStatus,
    _parsedAt: row._parsedAt,
    _normalizedAt: new Date().toISOString(),
  }
}];
```

## What this node returns

All fields are now consistent. `NewsletterOptIn` is a boolean. `SpendCategory` and `AccountStatus` are title-cased. Email is lowercased but not yet validated.

---

# STEP 6: Check for missing required fields

Records with blank CustomerID, OldEmail, or FullName cannot be migrated. This IF node catches them early.

## Add the IF node

1. Connect `Code - Normalize Fields` to an `IF` node.
2. Rename it to:

```text
IF - Missing Fields?
```

## Configure the conditions

Set the combine mode to `AND`. Add three conditions:

**Condition 1:**

- `Left Value`: `={{ $json.CustomerID }}`
- `Operator`: `is not empty`

**Condition 2:**

- `Left Value`: `={{ $json.OldEmail }}`
- `Operator`: `is not empty`

**Condition 3:**

- `Left Value`: `={{ $json.FullName }}`
- `Operator`: `is not empty`

## How this IF works

- `TRUE` = all required fields are present → continue to validation
- `FALSE` = one or more required fields are blank → route to error branch

---

## 6A. Error branch: tag missing field records

1. Connect the `FALSE` output of `IF - Missing Fields?` to a `Set` node.
2. Rename it to:

```text
Set - Tag Missing Field Error
```

3. Add these assignments:

- `CustomerID` = `{{ $json.CustomerID || 'UNKNOWN' }}`
- `OldEmail` = `{{ $json.OldEmail || 'BLANK' }}`
- `ErrorMessage` = `One or more required fields are blank (CustomerID, OldEmail, or FullName)`
- `ErrorType` = `Missing Field`
- `Timestamp` = `{{ new Date().toISOString() }}`

---

## 6B. Log missing field errors to Google Sheets

1. Connect `Set - Tag Missing Field Error` to a `Google Sheets` node.
2. Rename it to:

```text
Google Sheets - Log Missing Field Error
```

## Configure the node

Set:

- `Credential to connect with`: your Google Sheets credential
- `Resource`: `Sheet Within Document`
- `Operation`: `Append Row`
- `Document ID`: your spreadsheet ID
- `Sheet Name`: `Migration Errors`

Map columns:

- `CustomerID` → `{{ $json["CustomerID"] }}`
- `OldEmail` → `{{ $json["OldEmail"] }}`
- `ErrorMessage` → `{{ $json["ErrorMessage"] }}`
- `ErrorType` → `{{ $json["ErrorType"] }}`
- `Timestamp` → `{{ $json["Timestamp"] }}`

In settings, enable:

- `Retry On Fail`: on
- `Max Tries`: `3`
- `Wait Between Tries (ms)`: `2000`

---

# STEP 7: Validate email syntax with regex

Records that pass the missing-field check move here. This Code node tests the email address against a regex pattern.

## Add the Code node

1. Connect the `TRUE` output of `IF - Missing Fields?` to a `Code` node.
2. Rename it to:

```text
Code - Regex Email Check
```

3. Set:
   - `Mode`: `Run Once for Each Item`
   - `Language`: `JavaScript`

4. Paste this exact code:

```javascript
const email = String($json.OldEmail ?? '').trim().toLowerCase();

// Standard email syntax regex
const emailRegex = /^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$/;

// Known disposable email domains
const disposableDomains = [
  'mailinator.com',
  'guerrillamail.com',
  'temp-mail.org',
  'throwaway.email',
  'yopmail.com',
  'trashmail.com',
  '10minutemail.com',
  'sharklasers.com',
  'getairmail.com',
  'fakeinbox.com',
];

const isSyntaxValid = emailRegex.test(email);
const domain = email.split('@')[1] ?? '';
const isDisposable = disposableDomains.includes(domain.toLowerCase());

return [{
  json: {
    ...$json,
    _emailSyntaxValid: isSyntaxValid,
    _emailDisposable: isDisposable,
    _emailDomain: domain,
    _syntaxCheckedAt: new Date().toISOString(),
  }
}];
```

## What this node returns

Each item now has three new fields:

- `_emailSyntaxValid` — `true` if the email passes the regex
- `_emailDisposable` — `true` if the domain is in the known disposable list
- `_emailDomain` — the extracted domain for downstream logging

---

# STEP 8: Route invalid syntax and disposable domains

## Add the IF node

1. Connect `Code - Regex Email Check` to an `IF` node.
2. Rename it to:

```text
IF - Invalid Syntax?
```

## Configure the conditions

Set the combine mode to `OR`. Add two conditions:

**Condition 1:**

- `Left Value`: `={{ $json._emailSyntaxValid }}`
- `Operator`: `is false`

**Condition 2:**

- `Left Value`: `={{ $json._emailDisposable }}`
- `Operator`: `is true`

## How this IF works

- `TRUE` = email is malformed or from a disposable domain → route to error branch
- `FALSE` = email looks valid so far → continue to API validation

---

## 8A. Tag syntax/disposable errors

1. Connect the `TRUE` output of `IF - Invalid Syntax?` to a `Set` node.
2. Rename it to:

```text
Set - Tag Syntax Error
```

3. Add these assignments:

- `CustomerID` = `{{ $json.CustomerID }}`
- `OldEmail` = `{{ $json.OldEmail }}`
- `ErrorMessage` = `={{ $json._emailDisposable ? 'Disposable domain detected: ' + $json._emailDomain : 'Invalid email syntax' }}`
- `ErrorType` = `={{ $json._emailDisposable ? 'Disposable Domain' : 'Invalid Syntax' }}`
- `Timestamp` = `{{ new Date().toISOString() }}`

---

## 8B. Log syntax errors to Google Sheets

1. Connect `Set - Tag Syntax Error` to a `Google Sheets` node.
2. Rename it to:

```text
Google Sheets - Log Syntax Error
```

Use the same configuration as `Google Sheets - Log Missing Field Error`:

- `Document ID`: your spreadsheet ID
- `Sheet Name`: `Migration Errors`
- `Operation`: `Append Row`

Map columns with the same five fields:

- `CustomerID` → `{{ $json["CustomerID"] }}`
- `OldEmail` → `{{ $json["OldEmail"] }}`
- `ErrorMessage` → `{{ $json["ErrorMessage"] }}`
- `ErrorType` → `{{ $json["ErrorType"] }}`
- `Timestamp` → `{{ $json["Timestamp"] }}`

---

# STEP 9: Live email validation via Abstract API

Records that pass regex and disposable checks are now sent to the Abstract API for a live deliverability check.

## Add the HTTP Request node

1. Connect the `FALSE` output of `IF - Invalid Syntax?` to an `HTTP Request` node.
2. Rename it to:

```text
HTTP Request - Validate Email API
```

## Configure the node

Set:

- `Method`: `GET`
- `URL`:

```text
https://emailvalidation.abstractapi.com/v1/
```

- `Authentication`: `None` (the API key goes in the query params)

Under `Query Parameters`, add:

| Name | Value |
|------|-------|
| `api_key` | `YOUR_ABSTRACT_API_KEY` |
| `email` | `={{ $json.OldEmail }}` |

In node settings, set:

- `On Error`: `Continue (Regular Output)`
- `Retry On Fail`: on
- `Max Tries`: `3`
- `Wait Between Tries (ms)`: `3000`

## What this node returns

The Abstract API returns a JSON object. The key field you care about is:

```json
{
  "deliverability": "DELIVERABLE",
  "is_valid_format": { "value": true },
  "is_disposable_email": { "value": false }
}
```

Values of `deliverability` can be: `DELIVERABLE`, `UNDELIVERABLE`, or `UNKNOWN`.

---

# STEP 10: Route based on API response

## Add the IF node

1. Connect `HTTP Request - Validate Email API` to an `IF` node.
2. Rename it to:

```text
IF - API Rejected Email?
```

## Configure the conditions

Set the combine mode to `OR`. Add two conditions:

**Condition 1:**

- `Left Value`: `={{ $json.deliverability }}`
- `Operator`: `equals`
- `Right Value`: `UNDELIVERABLE`

**Condition 2:**

- `Left Value`: `={{ $json.message }}`
- `Operator`: `is not empty`

The second condition catches HTTP or API-level errors by checking if the response contains an error message field.

## How this IF works

- `TRUE` = email is undeliverable or the API call itself failed → route to error branch
- `FALSE` = email is deliverable or unknown (treat unknown as passable) → continue to sanitization

---

## 10A. Tag API-rejected errors

1. Connect the `TRUE` output of `IF - API Rejected Email?` to a `Set` node.
2. Rename it to:

```text
Set - Tag API Error
```

3. Add these assignments:

- `CustomerID` = `{{ $('Code - Normalize Fields').item.json.CustomerID }}`
- `OldEmail` = `{{ $('Code - Normalize Fields').item.json.OldEmail }}`
- `ErrorMessage` = `={{ $json.message ? 'API call failed: ' + $json.message : 'Email marked UNDELIVERABLE by validation API' }}`
- `ErrorType` = `={{ $json.message ? 'API Failure' : 'Undeliverable Email' }}`
- `Timestamp` = `{{ new Date().toISOString() }}`

Note: these expressions pull from `Code - Normalize Fields` directly because the HTTP Request output shape may not include the original customer fields.

---

## 10B. Log API errors to Google Sheets

1. Connect `Set - Tag API Error` to a `Google Sheets` node.
2. Rename it to:

```text
Google Sheets - Log API Error
```

Use the same sheet configuration as the previous error loggers:

- `Document ID`: your spreadsheet ID
- `Sheet Name`: `Migration Errors`
- `Operation`: `Append Row`

Map the same five columns.

---

# STEP 11: Sanitize clean records

Records that pass all three validation stages arrive here. This node performs final data cleaning before the logic engine.

## Add the Code node

1. Connect the `FALSE` output of `IF - API Rejected Email?` to a `Code` node.
2. Rename it to:

```text
Code - Sanitize Fields
```

3. Set:
   - `Mode`: `Run Once for Each Item`
   - `Language`: `JavaScript`

4. Paste this exact code:

```javascript
// Pull from the normalizer — the HTTP Request node may have diluted the payload
const original = $('Code - Normalize Fields').item.json;

// Rebuild from the known-good normalized source
const email = String(original.OldEmail ?? '').trim().toLowerCase();
const fullName = String(original.FullName ?? '').trim().replace(/\s+/g, ' ');
const customerID = String(original.CustomerID ?? '').trim();
const lastServiceDate = String(original.LastServiceDate ?? '').trim();
const skinConcerns = String(original.SkinConcerns ?? '').trim();
const spendCategory = String(original.SpendCategory ?? '');
const accountStatus = String(original.AccountStatus ?? '');
const newsletterOptIn = original.NewsletterOptIn;

// Validate and parse LastServiceDate
let parsedDate = null;
let dateIsValid = false;
const dateAttempt = new Date(lastServiceDate);
if (!isNaN(dateAttempt.getTime())) {
  parsedDate = dateAttempt.toISOString().split('T')[0]; // YYYY-MM-DD
  dateIsValid = true;
}

return [{
  json: {
    CustomerID: customerID,
    Email: email,
    FullName: fullName,
    LastServiceDate: parsedDate ?? lastServiceDate,
    _lastServiceDateValid: dateIsValid,
    SkinConcerns: skinConcerns,
    NewsletterOptIn: newsletterOptIn,
    SpendCategory: spendCategory,
    AccountStatus: accountStatus,
    _sanitizedAt: new Date().toISOString(),
  }
}];
```

## Why this node pulls from Code - Normalize Fields

The HTTP Request node replaces most of the item's JSON with the API response body. That means the original customer fields are gone from `$json` by the time data reaches this node. Pulling from the named node reference (`$('Code - Normalize Fields').item.json`) is the correct pattern here.

---

# STEP 12: Deduplicate records

This node checks whether the same email has already been processed in this batch run.

## Add the Code node

1. Connect `Code - Sanitize Fields` to a `Code` node.
2. Rename it to:

```text
Code - Deduplicate Check
```

3. Set:
   - `Mode`: `Run Once for All Items`
   - `Language`: `JavaScript`

4. Paste this exact code:

```javascript
const items = $input.all();
const seen = new Map(); // email -> first CustomerID seen
const results = [];

for (const item of items) {
  const email = String(item.json.Email ?? '').toLowerCase();
  const customerID = String(item.json.CustomerID ?? '');

  if (seen.has(email)) {
    results.push({
      json: {
        ...item.json,
        _isDuplicate: true,
        _duplicateOf: seen.get(email),
        _duplicateReason: `Email already seen for CustomerID ${seen.get(email)}`,
      }
    });
  } else {
    seen.set(email, customerID);
    results.push({
      json: {
        ...item.json,
        _isDuplicate: false,
        _duplicateOf: null,
        _duplicateReason: null,
      }
    });
  }
}

return results;
```

---

# STEP 13: Route duplicates

## Add the IF node

1. Connect `Code - Deduplicate Check` to an `IF` node.
2. Rename it to:

```text
IF - Is Duplicate?
```

## Configure the condition

- `Left Value`: `={{ $json._isDuplicate }}`
- `Operator`: `is true`

## How this IF works

- `TRUE` = duplicate email → log to Duplicate Tracker sheet and stop
- `FALSE` = unique email → continue to the logic engine

---

## 13A. Log duplicates to Google Sheets

1. Connect the `TRUE` output of `IF - Is Duplicate?` to a `Google Sheets` node.
2. Rename it to:

```text
Google Sheets - Log Duplicate
```

Set:

- `Document ID`: your spreadsheet ID
- `Sheet Name`: `Duplicate Tracker`
- `Operation`: `Append Row`

Map columns:

- `CustomerID` → `{{ $json["CustomerID"] }}`
- `Email` → `{{ $json["Email"] }}`
- `FullName` → `{{ $json["FullName"] }}`
- `DetectedAt` → `{{ new Date().toISOString() }}`
- `Reason` → `{{ $json["_duplicateReason"] }}`

---

# STEP 14: Tier routing — Active vs Lapsed

This is the core of the logic engine. Records are split based on how recently the client visited.

Active = last visit within the past 90 days → Airtable CRM
Lapsed = last visit more than 90 days ago → Google Sheets re-engagement list

## Add the IF node

1. Connect the `FALSE` output of `IF - Is Duplicate?` to an `IF` node.
2. Rename it to:

```text
IF - Active or Lapsed?
```

## Configure the condition

Use a JavaScript expression in the `Left Value`:

- `Left Value`:

```javascript
={{ (() => {
  const lastService = new Date($json.LastServiceDate);
  const today = new Date();
  const diffMs = today.getTime() - lastService.getTime();
  const diffDays = Math.floor(diffMs / (1000 * 60 * 60 * 24));
  return diffDays;
})() }}
```

- `Operator`: `smaller or equal`
- `Right Value`: `90`

## How this IF works

- `TRUE` = visited within the last 90 days → Tier 1 Active → Airtable
- `FALSE` = visited more than 90 days ago → Tier 2 Lapsed → Google Sheets

---

# STEP 15: Personalization routing by skin concern

After tier routing, each record is tagged with a skin-concern category. This Switch node creates four personalization paths.

## Add the Switch node

1. Connect the `TRUE` output of `IF - Active or Lapsed?` to a `Switch` node.

> **Important:** You also need to connect the `FALSE` output (Lapsed path) to this same Switch node in a moment. Add the Switch first, then draw both connections.

2. Rename it to:

```text
Switch - Route by Skin Concern
```

## Configure the rules

Create four rules:

**Rule 1 — Anti-Aging:**

- `Left Value`: `={{ $json.SkinConcerns }}`
- `Operator`: `equals`
- `Right Value`: `Anti-Aging`

**Rule 2 — Acne-Prone:**

- `Left Value`: `={{ $json.SkinConcerns }}`
- `Operator`: `equals`
- `Right Value`: `Acne-Prone`

**Rule 3 — Sensitive:**

- `Left Value`: `={{ $json.SkinConcerns }}`
- `Operator`: `equals`
- `Right Value`: `Sensitive`

**Rule 4 (Fallback) — Other:**

Turn on the fallback output. This catches all other skin concern values: Rosacea, Dehydration, Oily Skin, Fine Lines, Hyperpigmentation, etc.

## Connect BOTH tier outputs

Draw the connection from:

- `TRUE` output of `IF - Active or Lapsed?` → `Switch - Route by Skin Concern`
- `FALSE` output of `IF - Active or Lapsed?` → `Switch - Route by Skin Concern`

Both tiers pass through the same Switch. The tier (Active vs Lapsed) is carried in the item's `AccountStatus` and `LastServiceDate` fields, so the downstream destination nodes can still route correctly.

---

## 15A. Add a Set node for each skin concern branch

Create four `Set` nodes, one for each Switch output. These nodes attach the skin tier label before the record enters the destination system.

### Anti-Aging branch

1. Connect Switch output 1 to a `Set` node.
2. Rename it to:

```text
Set - Tag Anti-Aging
```

3. Add these assignments:

- `SkinTier` = `Anti-Aging`
- `CustomerID` = `{{ $json.CustomerID }}`
- `Email` = `{{ $json.Email }}`
- `FullName` = `{{ $json.FullName }}`
- `LastServiceDate` = `{{ $json.LastServiceDate }}`
- `SkinConcerns` = `{{ $json.SkinConcerns }}`
- `NewsletterOptIn` = `{{ $json.NewsletterOptIn }}`
- `SpendCategory` = `{{ $json.SpendCategory }}`
- `AccountStatus` = `{{ $json.AccountStatus }}`
- `MigratedAt` = `{{ new Date().toISOString() }}`

### Acne-Prone branch

1. Connect Switch output 2 to a `Set` node.
2. Rename it to:

```text
Set - Tag Acne-Prone
```

3. Use the same assignments as above, but set:

- `SkinTier` = `Acne-Prone`

### Sensitive branch

1. Connect Switch output 3 to a `Set` node.
2. Rename it to:

```text
Set - Tag Sensitive
```

3. Use the same assignments, but set:

- `SkinTier` = `Sensitive`

### Other / Default branch

1. Connect the Switch fallback output to a `Set` node.
2. Rename it to:

```text
Set - Tag Other
```

3. Use the same assignments, but set:

- `SkinTier` = `Other`

---

# STEP 16: VIP detection

This node checks two conditions that define VIP status.

## Add the IF node

1. Connect all four skin-tag Set nodes (`Set - Tag Anti-Aging`, `Set - Tag Acne-Prone`, `Set - Tag Sensitive`, `Set - Tag Other`) into a single `IF` node.
2. Rename it to:

```text
IF - Is VIP?
```

## Configure the conditions

Set the combine mode to `OR`. Add two conditions:

**Condition 1:**

- `Left Value`: `={{ $json.SpendCategory }}`
- `Operator`: `equals`
- `Right Value`: `VIP`

**Condition 2:**

- `Left Value`: `={{ $json.AccountStatus }}`
- `Operator`: `equals`
- `Right Value`: `VIP`

## How this IF works

- `TRUE` = client is VIP → send Gmail notification AND continue to destination
- `FALSE` = not VIP → skip notification and go straight to destination

---

# STEP 17: Send VIP Gmail notification

## Add the Gmail node

1. Connect the `TRUE` output of `IF - Is VIP?` to a `Gmail` node.
2. Rename it to:

```text
Gmail - Send VIP Notification
```

## Configure the node

Set:

- `Credential to connect with`: your Gmail OAuth2 credential
- `Resource`: `Message`
- `Operation`: `Send`
- `To`: your internal notification email address (e.g. your spa manager's email)
- `Subject`:

```javascript
=⭐ VIP Client Migrated: {{ $json.FullName }} ({{ $json.CustomerID }})
```

- `Email Type`: `HTML`
- `Message`:

```html
={{ `
<!DOCTYPE html>
<html>
<body style="font-family: Arial, sans-serif; background: #f9f5f2; padding: 30px;">
  <div style="max-width: 560px; margin: auto; background: #fff; border-radius: 10px; padding: 30px; border-top: 5px solid #c9a96e;">
    <h2 style="color: #4a3728; margin-bottom: 4px;">⭐ VIP Client Migration Alert</h2>
    <p style="color: #888; font-size: 13px; margin-top: 0;">Lume & Lather Med Spa — Data Migration Engine</p>
    <hr style="border: none; border-top: 1px solid #eee; margin: 20px 0;">
    <table style="width: 100%; font-size: 15px; color: #333;">
      <tr>
        <td style="padding: 8px 0; font-weight: bold; width: 40%;">Client Name</td>
        <td style="padding: 8px 0;">${$json.FullName}</td>
      </tr>
      <tr style="background: #fdf9f5;">
        <td style="padding: 8px 0; font-weight: bold;">Customer ID</td>
        <td style="padding: 8px 0;">${$json.CustomerID}</td>
      </tr>
      <tr>
        <td style="padding: 8px 0; font-weight: bold;">Email</td>
        <td style="padding: 8px 0;">${$json.Email}</td>
      </tr>
      <tr style="background: #fdf9f5;">
        <td style="padding: 8px 0; font-weight: bold;">Skin Concern</td>
        <td style="padding: 8px 0;">${$json.SkinConcerns}</td>
      </tr>
      <tr>
        <td style="padding: 8px 0; font-weight: bold;">Skin Tier</td>
        <td style="padding: 8px 0;">${$json.SkinTier}</td>
      </tr>
      <tr style="background: #fdf9f5;">
        <td style="padding: 8px 0; font-weight: bold;">Spend Category</td>
        <td style="padding: 8px 0;">${$json.SpendCategory}</td>
      </tr>
      <tr>
        <td style="padding: 8px 0; font-weight: bold;">Last Service</td>
        <td style="padding: 8px 0;">${$json.LastServiceDate}</td>
      </tr>
      <tr style="background: #fdf9f5;">
        <td style="padding: 8px 0; font-weight: bold;">Account Status</td>
        <td style="padding: 8px 0;">${$json.AccountStatus}</td>
      </tr>
      <tr>
        <td style="padding: 8px 0; font-weight: bold;">Migrated At</td>
        <td style="padding: 8px 0;">${$json.MigratedAt}</td>
      </tr>
    </table>
    <hr style="border: none; border-top: 1px solid #eee; margin: 24px 0;">
    <p style="font-size: 12px; color: #aaa; text-align: center;">This alert was generated automatically by the Lume & Lather Migration Engine.</p>
  </div>
</body>
</html>
` }}
```

In node settings, enable:

- `Retry On Fail`: on
- `Max Tries`: `3`
- `Wait Between Tries (ms)`: `2000`

---

# STEP 18: Route to destination systems

After VIP detection, every record must arrive at the correct destination: Airtable for Active, Google Sheets for Lapsed.

Both the `TRUE` and `FALSE` outputs of `IF - Is VIP?` feed into this logic. The VIP notification was a side-action — destination routing happens for all records regardless.

Add one more `IF` node to split by tier:

## Add the IF node

1. Connect both outputs of `IF - Is VIP?` to a single new `IF` node.

> To connect two outputs to the same node, draw the connection from `TRUE` first, then draw the `FALSE` connection to the same node.

2. Rename it to:

```text
IF - Route to Destination?
```

## Configure the condition

- `Left Value`: `={{ $json.AccountStatus }}`
- `Operator`: `equals`
- `Right Value`: `Active`

## How this IF works

- `TRUE` = Active client → Airtable
- `FALSE` = Lapsed or Inactive client → Google Sheets

---

# STEP 19: Write Active clients to Airtable

## Add the Airtable node

1. Connect the `TRUE` output of `IF - Route to Destination?` to an `Airtable` node.
2. Rename it to:

```text
Airtable - Create Active Client Record
```

## Configure the node

Set:

- `Authentication`: Airtable Personal Access Token
- `Resource`: `Record`
- `Operation`: `Create`
- `Base`: your Airtable base ID
- `Table`: your table ID for `Active Clients`

Map these fields:

- `CustomerID` → `{{ $json["CustomerID"] }}`
- `Email` → `{{ $json["Email"] }}`
- `FullName` → `{{ $json["FullName"] }}`
- `LastServiceDate` → `{{ $json["LastServiceDate"] }}`
- `SkinConcerns` → `{{ $json["SkinConcerns"] }}`
- `NewsletterOptIn` → `{{ $json["NewsletterOptIn"] }}`
- `SpendCategory` → `{{ $json["SpendCategory"] }}`
- `AccountStatus` → `{{ $json["AccountStatus"] }}`
- `SkinTier` → `{{ $json["SkinTier"] }}`
- `MigratedAt` → `{{ $json["MigratedAt"] }}`

In node settings:

- `On Error`: `Continue (Regular Output)`
- `Retry On Fail`: on
- `Max Tries`: `3`
- `Wait Between Tries (ms)`: `3000`

---

# STEP 20: Write Lapsed clients to Google Sheets

## Add the Google Sheets node

1. Connect the `FALSE` output of `IF - Route to Destination?` to a `Google Sheets` node.
2. Rename it to:

```text
Google Sheets - Append Lapsed Client
```

## Configure the node

Set:

- `Credential to connect with`: your Google Sheets credential
- `Resource`: `Sheet Within Document`
- `Operation`: `Append Row`
- `Document ID`: your spreadsheet ID
- `Sheet Name`: `Re-engagement`

Map columns:

- `CustomerID` → `{{ $json["CustomerID"] }}`
- `Email` → `{{ $json["Email"] }}`
- `FullName` → `{{ $json["FullName"] }}`
- `LastServiceDate` → `{{ $json["LastServiceDate"] }}`
- `SkinConcerns` → `{{ $json["SkinConcerns"] }}`
- `NewsletterOptIn` → `{{ $json["NewsletterOptIn"] }}`
- `SpendCategory` → `{{ $json["SpendCategory"] }}`
- `AccountStatus` → `{{ $json["AccountStatus"] }}`
- `SkinTier` → `{{ $json["SkinTier"] }}`
- `MigratedAt` → `{{ $json["MigratedAt"] }}`

In settings, enable:

- `Retry On Fail`: on
- `Max Tries`: `3`
- `Wait Between Tries (ms)`: `2000`

---

# STEP 21: Mark migration complete

Both destination nodes flow into a final cleanup node.

## Add the Set node

1. Connect both `Airtable - Create Active Client Record` and `Google Sheets - Append Lapsed Client` to a single `Set` node.
2. Rename it to:

```text
Set - Migration Complete
```

3. Add these assignments:

- `CustomerID` = `{{ $json.CustomerID }}`
- `Email` = `{{ $json.Email }}`
- `FullName` = `{{ $json.FullName }}`
- `Destination` = `={{ $json.AccountStatus === 'Active' ? 'Airtable - Active Clients' : 'Google Sheets - Re-engagement' }}`
- `Status` = `migrated`
- `CompletedAt` = `{{ new Date().toISOString() }}`

This node acts as a clean, observable endpoint for each successfully migrated record.

---

# STEP 22: Build the separate error workflow

This second workflow catches any node-level execution failure in the main workflow.

## 22A. Add the Error Trigger node

Inside `Lume & Lather - Migration Error Logger`, add an `Error Trigger` node.

Rename it to:

```text
Error Trigger - Global
```

No configuration is required. This node activates automatically when the main workflow fails.

---

## 22B. Normalize the error payload

1. Add a `Set` node after `Error Trigger - Global`.
2. Rename it to:

```text
Set - Prepare Error Log
```

3. Add these assignments:

- `Timestamp` = `{{ new Date().toISOString() }}`
- `WorkflowName` = `={{ $json.workflow?.name || 'Lume & Lather - Email Migration Engine' }}`
- `NodeName` = `={{ $json.execution?.lastNodeExecuted || 'Unknown Node' }}`
- `ErrorMessage` = `={{ $json.execution?.error?.message || $json.error?.message || $json.execution?.data?.resultData?.error?.message || 'Unknown execution error' }}`
- `ExecutionTime` = `={{ $json.execution?.startedAt || $json.execution?.stoppedAt || new Date().toISOString() }}`

---

## 22C. Write the error to Google Sheets

1. Connect `Set - Prepare Error Log` to a `Google Sheets` node.
2. Rename it to:

```text
Google Sheets - Log Execution Error
```

Set:

- `Document ID`: your spreadsheet ID (same spreadsheet)
- `Sheet Name`: `Error Logs`
- `Operation`: `Append Row`

Map columns:

- `Timestamp` → `{{ $json["Timestamp"] }}`
- `WorkflowName` → `{{ $json["WorkflowName"] }}`
- `NodeName` → `{{ $json["NodeName"] }}`
- `ErrorMessage` → `{{ $json["ErrorMessage"] }}`
- `ExecutionTime` → `{{ $json["ExecutionTime"] }}`

In settings, enable:

- `Retry On Fail`: on
- `Max Tries`: `3`
- `Wait Between Tries (ms)`: `2000`

---

## 22D. Link the error workflow to the main workflow

After saving both workflows:

1. Open `Lume & Lather - Email Migration Engine`.
2. Open workflow settings.
3. In the `Error Workflow` field, select:

```text
Lume & Lather - Migration Error Logger
```

4. Save.

---

# STEP 23: Complete connection map

Use this map to verify every connection on your canvas before testing.

```text
Manual Trigger - Start Migration
  → Spreadsheet File - Read CSV
  → Code - Parse CSV to JSON
  → SplitInBatches - Process in Batches (output 1)
  → Code - Normalize Fields
  → IF - Missing Fields?

IF - Missing Fields? FALSE
  → Set - Tag Missing Field Error
  → Google Sheets - Log Missing Field Error
  [END — error branch]

IF - Missing Fields? TRUE
  → Code - Regex Email Check
  → IF - Invalid Syntax?

IF - Invalid Syntax? TRUE
  → Set - Tag Syntax Error
  → Google Sheets - Log Syntax Error
  [END — error branch]

IF - Invalid Syntax? FALSE
  → HTTP Request - Validate Email API
  → IF - API Rejected Email?

IF - API Rejected Email? TRUE
  → Set - Tag API Error
  → Google Sheets - Log API Error
  [END — error branch]

IF - API Rejected Email? FALSE
  → Code - Sanitize Fields
  → Code - Deduplicate Check
  → IF - Is Duplicate?

IF - Is Duplicate? TRUE
  → Google Sheets - Log Duplicate
  [END — duplicate branch]

IF - Is Duplicate? FALSE
  → IF - Active or Lapsed?

IF - Active or Lapsed? TRUE (≤90 days)
  → Switch - Route by Skin Concern

IF - Active or Lapsed? FALSE (>90 days)
  → Switch - Route by Skin Concern

Switch - Route by Skin Concern output 1 (Anti-Aging)
  → Set - Tag Anti-Aging
  → IF - Is VIP?

Switch - Route by Skin Concern output 2 (Acne-Prone)
  → Set - Tag Acne-Prone
  → IF - Is VIP?

Switch - Route by Skin Concern output 3 (Sensitive)
  → Set - Tag Sensitive
  → IF - Is VIP?

Switch - Route by Skin Concern fallback (Other)
  → Set - Tag Other
  → IF - Is VIP?

IF - Is VIP? TRUE
  → Gmail - Send VIP Notification
  → IF - Route to Destination?

IF - Is VIP? FALSE
  → IF - Route to Destination?

IF - Route to Destination? TRUE (Active)
  → Airtable - Create Active Client Record
  → Set - Migration Complete

IF - Route to Destination? FALSE (Lapsed/Inactive)
  → Google Sheets - Append Lapsed Client
  → Set - Migration Complete
```

Error workflow:

```text
Error Trigger - Global
  → Set - Prepare Error Log
  → Google Sheets - Log Execution Error
```

---

# STEP 24: Recommended canvas layout

Lay the canvas out left to right in five horizontal lanes:

- **Top lane** (error branches): all error Set and Google Sheets log nodes
- **Lane 2** (ingestion): Manual Trigger → Spreadsheet File → Code Parse → SplitInBatches
- **Lane 3** (validation): Code Normalize → IF Missing → Code Regex → IF Syntax → HTTP Request → IF API
- **Lane 4** (sanitization + logic): Code Sanitize → Code Dedup → IF Duplicate → IF Tier → Switch Skin → IF VIP → IF Destination
- **Bottom lane** (destinations): Airtable and Google Sheets destination nodes → Set Migration Complete

Keep all error branches connecting upward into the top lane. This makes the happy path across the middle visually clean and easy to trace.

---

# STEP 25: Testing guide

Test one layer at a time. Do not run the full workflow until each layer works independently.

---

## Test 1: Valid Active VIP record

Use this record:

```json
{
  "CustomerID": "LL-0001",
  "OldEmail": "sarah.j@gmail.com",
  "FullName": "Sarah Jenkins",
  "LastServiceDate": "2024-05-20",
  "SkinConcerns": "Anti-Aging",
  "NewsletterOptIn": "True",
  "SpendCategory": "VIP",
  "AccountStatus": "Active"
}
```

Expected path:
1. Passes missing field check
2. Passes regex check
3. API confirms deliverable
4. Sanitized cleanly
5. Not a duplicate (first run)
6. Tiered as Active (≤90 days from run date)
7. Tagged as Anti-Aging
8. Detected as VIP → Gmail notification fires
9. Written to Airtable `Active Clients`
10. `Set - Migration Complete` outputs `status = migrated`

Expected Gmail notification subject:

```text
⭐ VIP Client Migrated: Sarah Jenkins (LL-0001)
```

Expected Airtable record:

```json
{
  "CustomerID": "LL-0001",
  "Email": "sarah.j@gmail.com",
  "FullName": "Sarah Jenkins",
  "SkinTier": "Anti-Aging",
  "MigratedAt": "<current ISO timestamp>"
}
```

---

## Test 2: Invalid syntax email

Use this record:

```json
{
  "CustomerID": "LL-0004",
  "OldEmail": "dthompson.com",
  "FullName": "David Thompson",
  "LastServiceDate": "2024-01-05",
  "SkinConcerns": "Hyperpigmentation",
  "NewsletterOptIn": "True",
  "SpendCategory": "Low",
  "AccountStatus": "Inactive"
}
```

Expected path:
1. Passes missing field check
2. Fails regex check (no `@` symbol)
3. Routes to `Set - Tag Syntax Error`
4. Written to `Migration Errors` sheet with `ErrorType = Invalid Syntax`
5. Does not reach Airtable or Google Sheets destination

---

## Test 3: Disposable domain email

Use this record:

```json
{
  "CustomerID": "LL-0003",
  "OldEmail": "elena.rod@mailinator.com",
  "FullName": "Elena Rodriguez",
  "LastServiceDate": "2023-11-15",
  "SkinConcerns": "Sensitive",
  "NewsletterOptIn": "False",
  "SpendCategory": "High",
  "AccountStatus": "Lapsed"
}
```

Expected path:
1. Passes missing field check
2. Regex passes (syntax is valid)
3. Disposable domain check triggers
4. Routes to `Set - Tag Syntax Error` with `ErrorType = Disposable Domain`
5. Written to `Migration Errors` sheet

---

## Test 4: Double-@@ syntax error

Use this record:

```json
{
  "CustomerID": "LL-0010",
  "OldEmail": "igarcia@@icloud.com",
  "FullName": "Isabella Garcia",
  "LastServiceDate": "2024-06-10",
  "SkinConcerns": "Sensitive",
  "NewsletterOptIn": "True",
  "SpendCategory": "Medium",
  "AccountStatus": "Active"
}
```

Expected path:
1. Passes missing field check
2. Fails regex (double `@@` is invalid syntax)
3. Routes to error branch with `ErrorType = Invalid Syntax`

---

## Test 5: Lapsed client to Google Sheets

Use this record with a service date well over 90 days ago:

```json
{
  "CustomerID": "LL-0009",
  "OldEmail": "loconnor@protonmail.com",
  "FullName": "Liam O'Connor",
  "LastServiceDate": "2023-08-30",
  "SkinConcerns": "Oily Skin",
  "NewsletterOptIn": "False",
  "SpendCategory": "Low",
  "AccountStatus": "Lapsed"
}
```

Expected path:
1. Passes all three validation stages
2. Sanitized and confirmed as not duplicate
3. Tiered as Lapsed (>90 days)
4. Routes through Switch to `Set - Tag Other` (Oily Skin is not a named branch)
5. Not VIP → no Gmail
6. Written to Google Sheets `Re-engagement` tab

Expected Google Sheets row:

```text
LL-0009 | loconnor@protonmail.com | Liam O'Connor | 2023-08-30 | Oily Skin | false | Low | Lapsed | Other | <timestamp>
```

---

## Test 6: Duplicate record detection

Run the migration with two records that share the same email:

```json
[
  { "CustomerID": "LL-0021", "OldEmail": "spa_goer@protonmail.com", "FullName": "Ava DuVernay", ... },
  { "CustomerID": "LL-0023", "OldEmail": "spa_goer@protonmail.com", "FullName": "Elena Rodriguez", ... }
]
```

Expected path:
1. `LL-0021` processes normally
2. `LL-0023` reaches `Code - Deduplicate Check`
3. Its email is flagged as duplicate of `LL-0021`
4. Written to `Duplicate Tracker` sheet

---

## Test 7: Batch processing with full 150-record CSV

Load the full 150-row CSV file.

Expected behavior:
1. SplitInBatches processes 15 batches of 10 records each
2. All 150 records are processed sequentially
3. Valid records reach Airtable or Google Sheets
4. Invalid records land in Migration Errors
5. Duplicates land in Duplicate Tracker
6. No records are silently dropped

---

## Test 8: API failure simulation

To test the API failure fallback:

1. Temporarily change the Abstract API key to an invalid value.
2. Run a valid record through.
3. The HTTP Request node should fail.
4. The `IF - API Rejected Email?` node catches the error message.
5. Record is routed to `Set - Tag API Error` and logged in `Migration Errors` with `ErrorType = API Failure`.

---

## Test 9: Error workflow trigger

To test the error workflow:

1. Temporarily break a required node — for example, put an invalid expression in `Code - Sanitize Fields`.
2. Run the workflow.
3. n8n should trigger `Lume & Lather - Migration Error Logger`.
4. Confirm a row appears in the `Error Logs` tab.

---

# STEP 26: Expected outputs summary

After a clean run of the full 150-record dataset, you should see:

### Airtable — Active Clients

Records from clients whose last service date was within 90 days of the migration run date and whose email passed all three validation stages.

Example:

```text
CustomerID: LL-0001
Email: sarah.j@gmail.com
FullName: Sarah Jenkins
SkinTier: Anti-Aging
MigratedAt: 2026-06-15T10:30:00.000Z
```

---

### Google Sheets — Re-engagement

Records from Lapsed or Inactive clients who passed validation.

Example:

```text
LL-0009 | loconnor@protonmail.com | Liam O'Connor | 2023-08-30 | Oily Skin | FALSE | Low | Lapsed | Other | 2026-06-15T10:30:00.000Z
```

---

### Google Sheets — Migration Errors

Records that failed validation at any stage.

Example rows:

```text
LL-0004 | dthompson.com | Invalid email syntax | Invalid Syntax | 2026-06-15T10:30:00.000Z
LL-0003 | elena.rod@mailinator.com | Disposable domain detected: mailinator.com | Disposable Domain | 2026-06-15T10:30:00.000Z
LL-0010 | igarcia@@icloud.com | Invalid email syntax | Invalid Syntax | 2026-06-15T10:30:00.000Z
```

---

### Google Sheets — Duplicate Tracker

Records where the email was already processed by a prior record in the same run.

Example:

```text
LL-0023 | spa_goer@protonmail.com | Elena Rodriguez | 2026-06-15T10:30:00.000Z | Email already seen for CustomerID LL-0021
```

---

### Gmail VIP notifications

One HTML email per VIP client that passed validation. Example subject line:

```text
⭐ VIP Client Migrated: Sarah Jenkins (LL-0001)
```

---

# STEP 27: Best practices and naming conventions

## Node naming

Every node name in this workflow follows this pattern:

```text
[NodeType] - [SpecificAction]
```

Examples:

```text
IF - Missing Fields?
Code - Normalize Fields
Google Sheets - Log Migration Error
Airtable - Create Active Client Record
```

This naming makes the canvas readable at a glance and makes error logs meaningful when a node name appears in an execution failure message.

---

## Error handling philosophy

Every branch in this workflow that can fail has an explicit landing point. There are no dead ends. A record that fails validation does not disappear — it lands in a log sheet with a clear error type and timestamp.

Always set `On Error: Continue (Regular Output)` on nodes that call external services (Airtable, HTTP Request, Gmail). This prevents one bad record from stopping the entire batch.

---

## Data sourcing discipline

Never read `$json` blindly after nodes that replace the payload (like `HTTP Request`). Always use named-node references such as:

```javascript
$('Code - Normalize Fields').item.json.CustomerID
```

This is especially important in the sanitization node and the API error tag node, where the HTTP Request response body overwrites the original customer data.

---

## Debugging tips

- Use the `Output` panel after each node during development. Click a node, run a test, and inspect what JSON it produces before connecting the next node.
- Add a temporary `Set` node at any point in the chain to inspect the exact shape of the data at that moment. Delete it when you are done.
- The SplitInBatches node can be tricky during testing — if you only have 3 test records, it will fire output 1 once and output 2 once. Connect a `No Operation` node to output 2 if you don't want it to error during testing.
- Always check `Migration Errors` in Google Sheets after a test run before claiming the run was clean. Silent drops never appear in the Airtable or Sheets destination — they only appear in the error log.

---

## Scaling the batch size

The batch size of `10` is conservative and safe for API-rate-limited runs. If your Abstract API plan allows higher throughput, you can increase it to `25` or `50`.

Do not exceed `50` per batch unless you have confirmed your external API rate limits can handle it.

---

# STEP 28: Activation checklist

Before activating either workflow in production:

1. Confirm the Google Sheets spreadsheet has all four tabs with exact header rows.
2. Confirm the Airtable base and table exist with exact field names.
3. Confirm the Gmail credential is valid and the sending address is verified.
4. Confirm the Abstract API key is active and has remaining quota.
5. Run all nine test cases in the n8n editor (not in production execution).
6. Verify the error workflow is assigned to the main workflow in settings.
7. Confirm at least one error-branch test produced a row in `Migration Errors`.
8. Activate `Lume & Lather - Migration Error Logger` first.
9. Activate `Lume & Lather - Email Migration Engine` second.

---

# Final result

Once built, this system gives Lume & Lather a robust, observable, and scalable data migration pipeline:

- Every client record is validated in three layers before any write happens
- Active clients land cleanly in Airtable for CRM use
- Lapsed clients land in Google Sheets for re-engagement campaigns
- VIP clients trigger an immediate internal notification
- Every invalid record is logged with a clear reason, not silently dropped
- Duplicate emails within a single run are caught and quarantined separately
- Any execution-level failure is captured by the error workflow
- The batching design means the workflow handles 100+ records or 10,000+ records without modification

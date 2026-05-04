# Pacific Peak Immigration Law — Legal Lead Intake Automation

This guide is written so you can build the workflow manually in n8n yourself, step by step, without depending on a JSON import.

It follows the same build-it-yourself style as other workflow guides in this project folder.

It assumes:

- You are using a recent n8n version with the current `Google Sheets Trigger`, `Code`, `Set`, `IF`, `Switch`, `Merge`, `SplitInBatches`, `Wait`, `Airtable`, `Slack`, `Gmail`, and `Error Trigger` nodes.
- You already have these credentials available inside n8n:
  - `Google Sheets OAuth2`
  - `Airtable Personal Access Token`
  - `Slack OAuth2`
  - `Gmail OAuth2`

Use these exact workflow names:

- `Pacific Peak - Legal Lead Intake Automation`
- `Pacific Peak - Error Logger`

---

## What this workflow does

1. Watches a Google Sheet for new lead rows submitted through any intake form.
2. Splits records into individual items and processes them one by one.
3. Normalizes all field values — email casing, case type spelling, court date format.
4. Validates each record: checks for missing fields and runs email regex.
5. Routes invalid records to a dedicated error-logging branch.
6. Runs the priority engine on valid records:
   - checks if Urgency Level is "Urgent" or "High"
   - calculates days until the court date
   - merges both signals into a single priority flag
7. Creates a record in Airtable for every valid lead.
8. If the lead is priority, sends a Slack message and an HTML email to the legal team.
9. Logs any Airtable failures, missing fields, or invalid emails to a Google Sheets error log.
10. Uses a second error workflow to catch any unexpected execution-level failures.

---

## Before you build

Create the assets first so you do not have to stop halfway.

### 1. Google Sheets spreadsheet

Create one spreadsheet. Name it:

```
Pacific Peak - Lead Management
```

Create these tabs:

1. `Lead Intake` — the source sheet your trigger will watch
2. `Error Log` — for validation and system failures

#### Lead Intake tab headers

```
Submission ID | Lead Name | Email | Case Type | Urgency Level | Court Date | Submission Date
```

These match your `pacific_peak_leads_starter.csv` exactly. Import that CSV into this tab to populate your test data.

#### Error Log tab headers

```
Timestamp | Lead Name | Email | Error Type | Error Message
```

Keep the spreadsheet ID ready. You will paste it into every Google Sheets node.

### 2. Airtable base

Create or open an Airtable base. Recommended name:

```
Pacific Peak CRM
```

Create one table:

```
Leads
```

Create these exact fields in that table:

```
Name           (Single line text)
Email          (Email)
Case Type      (Single line text)
Priority       (Single line text)
Court Date     (Date)
Status         (Single line text)
Priority Flag  (Checkbox or Single line text)
Submission ID  (Single line text)
```

Keep the base ID and table ID ready.

### 3. Slack channel

Create or choose a channel. Recommended:

```
#urgent-legal-leads
```

Keep the channel ID and Slack OAuth credential ready.

### 4. Gmail sender

You need a Gmail account authorized in n8n via OAuth2. This will be the "from" address for priority email alerts. Keep the credential ready.

### 5. Test row reference

Here is the exact row shape your workflow will process:

```
PPL-0001 | Elena Rodriguez | e.rodriguez@example.com | Deportation Defense | Urgent | 2024-11-10 | 2024-10-25
```

---

## STEP 0: Create the two workflows

### 0A. Create the main workflow

1. Open n8n.
2. Click `Create Workflow`.
3. Rename it to:

```
Pacific Peak - Legal Lead Intake Automation
```

4. Save once before adding any nodes.

### 0B. Create the error workflow

1. Create a second workflow.
2. Rename it to:

```
Pacific Peak - Error Logger
```

3. Save once.

---

## STEP 1: Plan the node order

You will build the main workflow in this order. Do not skip ahead — each node feeds the next.

```
01  Google Sheets Trigger - Watch Lead Intake
02  Manual Trigger - Test Fallback
03  SplitInBatches - Process One Lead at a Time
04  Set - Rename Raw Fields
05  Code - Normalize Field Values
06  Code - Parse and Format Court Date
07  IF - Check Required Fields Present
08  Code - Validate Email Format
09  IF - Email Valid?
10  Google Sheets - Log Validation Error
11  Set - Mark Valid Lead
12  IF - Urgency Is High Priority?
13  Code - Calculate Days Until Court Date
14  IF - Court Date Within 30 Days?
15  Merge - Combine Priority Signals
16  Set - Assign Priority Flag
17  Airtable - Create Lead Record
18  IF - Is Priority Lead?
19  Slack - Send Priority Alert
20  Gmail - Send Priority Email
21  Wait - Throttle Between Records
```

This gives you one clean entry point, a full validation gate, a multi-node priority engine, a CRM write, and a dual-channel notification system.

---

## STEP 2: Add the triggers

### Node 01 — Google Sheets Trigger - Watch Lead Intake

This is the live production trigger. It polls your sheet every minute and fires when a new row appears.

1. Add a `Google Sheets Trigger` node.
2. Rename it to:

```
Google Sheets Trigger - Watch Lead Intake
```

3. Configure:

- `Credential`: select your `Google Sheets OAuth2` credential
- `Trigger On`: `Row Added`
- `Document ID`: paste your spreadsheet ID
- `Sheet Name`: `Lead Intake`
- `Poll Time`: `Every Minute`

> **Why every minute?** Immigration cases can be time-sensitive. A one-minute polling interval means no lead sits unseen for more than 60 seconds after submission.

The node will output one item per new row it detects. Each item contains all column values as JSON fields.

Expected output shape:

```json
{
  "Submission ID": "PPL-0001",
  "Lead Name": "Elena Rodriguez",
  "Email": "e.rodriguez@example.com",
  "Case Type": "Deportation Defense",
  "Urgency Level": "Urgent",
  "Court Date": "2024-11-10",
  "Submission Date": "2024-10-25"
}
```

---

### Node 02 — Manual Trigger - Test Fallback

This is for testing only. You will use it to fire the workflow manually while building.

1. Add a `Manual Trigger` node.
2. Rename it to:

```
Manual Trigger - Test Fallback
```

3. Do not configure anything. This node takes no inputs.

Connect both `Google Sheets Trigger - Watch Lead Intake` and `Manual Trigger - Test Fallback` to the same next node: `SplitInBatches - Process One Lead at a Time`.

> When using the Manual Trigger during development, you will need to inject a test item. The easiest approach: temporarily add a `Set` node after the manual trigger that hardcodes a sample row, then remove it before activating.

---

## STEP 3: Batch the records

### Node 03 — SplitInBatches - Process One Lead at a Time

The Google Sheets Trigger may return multiple new rows in a single poll. This node ensures each row is processed individually and sequentially, which prevents race conditions in Airtable and makes errors traceable to a single lead.

1. Add a `SplitInBatches` node.
2. Rename it to:

```
SplitInBatches - Process One Lead at a Time
```

3. Configure:

- `Batch Size`: `1`

This means only one lead moves through the pipeline at a time. After the full workflow processes that lead, it loops back and picks up the next.

> **Scaling note:** Even though your starter dataset is 30 rows, this architecture supports thousands of rows without change. Increase batch size to 5 or 10 later if processing speed becomes a bottleneck.

---

## STEP 4: Rename and clean the raw fields

### Node 04 — Set - Rename Raw Fields

Google Sheets column names may contain spaces, inconsistent casing, or special characters. This node gives every field a clean internal name that is safe to use in expressions throughout the workflow.

1. Add a `Set` node.
2. Rename it to:

```
Set - Rename Raw Fields
```

3. Set `Mode` to `Manual`.
4. Add these assignments one by one:

| Output Field Name | Expression |
|---|---|
| `submission_id` | `{{ $json["Submission ID"] }}` |
| `lead_name` | `{{ $json["Lead Name"] }}` |
| `email_raw` | `{{ $json["Email"] }}` |
| `case_type_raw` | `{{ $json["Case Type"] }}` |
| `urgency_raw` | `{{ $json["Urgency Level"] }}` |
| `court_date_raw` | `{{ $json["Court Date"] }}` |
| `submission_date_raw` | `{{ $json["Submission Date"] }}` |

5. Turn on `Keep Only Set`: **No** — keep all existing fields in addition to these.

After this node, all downstream nodes will use the snake_case names, not the original column headers.

---

## STEP 5: Normalize field values

### Node 05 — Code - Normalize Field Values

This node standardizes the three fields most likely to arrive with inconsistent formatting: the email address, the case type label, and the urgency level.

1. Add a `Code` node.
2. Rename it to:

```
Code - Normalize Field Values
```

3. Set:
   - `Mode`: `Run Once for Each Item`
   - `Language`: `JavaScript`

4. Paste this exact code:

```javascript
const item = $json;

// Normalize email: lowercase and trim whitespace
const emailNormalized = String(item.email_raw ?? '').toLowerCase().trim();

// Normalize case type: trim whitespace, title-case each word
const rawCaseType = String(item.case_type_raw ?? '').trim();
const caseTypeNormalized = rawCaseType
  .split(' ')
  .map(word => word.charAt(0).toUpperCase() + word.slice(1).toLowerCase())
  .join(' ');

// Normalize urgency: trim and title-case
const rawUrgency = String(item.urgency_raw ?? '').trim();
const urgencyNormalized = rawUrgency.charAt(0).toUpperCase() + rawUrgency.slice(1).toLowerCase();

return [{
  json: {
    ...item,
    email: emailNormalized,
    case_type: caseTypeNormalized,
    urgency_level: urgencyNormalized,
  },
}];
```

Expected output additions:

```json
{
  "email": "e.rodriguez@example.com",
  "case_type": "Deportation Defense",
  "urgency_level": "Urgent"
}
```

---

## STEP 6: Parse and format the court date

### Node 06 — Code - Parse and Format Court Date

Court dates arrive as plain strings like `2024-11-10`. This node converts them into:

1. A proper ISO 8601 timestamp (for Airtable)
2. A human-readable display string (for notifications)
3. A raw `Date` object epoch (for days-remaining math in the priority engine)

1. Add a `Code` node.
2. Rename it to:

```
Code - Parse and Format Court Date
```

3. Set:
   - `Mode`: `Run Once for Each Item`
   - `Language`: `JavaScript`

4. Paste this exact code:

```javascript
const item = $json;

const rawDate = String(item.court_date_raw ?? '').trim();

// Attempt to parse as a date
const parsedDate = new Date(rawDate);
const isValid = !Number.isNaN(parsedDate.getTime());

// ISO format for Airtable
const courtDateISO = isValid ? parsedDate.toISOString().split('T')[0] : null;

// Human-readable for Slack and email
const courtDateDisplay = isValid
  ? parsedDate.toLocaleDateString('en-US', {
      timeZone: 'UTC',
      year: 'numeric',
      month: 'long',
      day: 'numeric',
    })
  : 'Invalid Date';

// Epoch milliseconds for downstream math
const courtDateEpoch = isValid ? parsedDate.getTime() : null;

return [{
  json: {
    ...item,
    court_date_iso: courtDateISO,
    court_date_display: courtDateDisplay,
    court_date_epoch: courtDateEpoch,
    court_date_valid: isValid,
  },
}];
```

Expected output additions:

```json
{
  "court_date_iso": "2024-11-10",
  "court_date_display": "November 10, 2024",
  "court_date_epoch": 1731196800000,
  "court_date_valid": true
}
```

---

## STEP 7: Validate required fields

### Node 07 — IF - Check Required Fields Present

Before doing any CRM writes or priority calculations, check that the lead record is complete. A missing name, email, or case type makes the record unusable.

1. Add an `IF` node.
2. Rename it to:

```
IF - Check Required Fields Present
```

3. Set `Combine Conditions` to `AND` — all three must pass.

4. Add these three conditions:

**Condition 1**

- `Value 1`: `{{ $json["lead_name"] }}`
- `Operation`: `Is Not Empty`

**Condition 2**

- `Value 1`: `{{ $json["email"] }}`
- `Operation`: `Is Not Empty`

**Condition 3**

- `Value 1`: `{{ $json["case_type"] }}`
- `Operation`: `Is Not Empty`

**TRUE output** → connect to `Code - Validate Email Format`

**FALSE output** → connect to `Google Sheets - Log Validation Error`

---

## STEP 8: Validate the email address

### Node 08 — Code - Validate Email Format

A filled-in email field is not enough. This node runs a proper regex check to confirm the value looks like a real email address before the record enters the CRM.

1. Add a `Code` node.
2. Rename it to:

```
Code - Validate Email Format
```

3. Set:
   - `Mode`: `Run Once for Each Item`
   - `Language`: `JavaScript`

4. Paste this exact code:

```javascript
const item = $json;

const email = String(item.email ?? '').trim();

// Standard email regex — covers the vast majority of real addresses
const emailRegex = /^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$/;

const emailIsValid = emailRegex.test(email);

return [{
  json: {
    ...item,
    email_valid: emailIsValid,
    email_validation_checked: true,
  },
}];
```

Expected output additions:

```json
{
  "email_valid": true,
  "email_validation_checked": true
}
```

---

### Node 09 — IF - Email Valid?

This node reads the `email_valid` flag and routes the item.

1. Add an `IF` node.
2. Rename it to:

```
IF - Email Valid?
```

3. Add one condition:

- `Value 1`: `{{ $json["email_valid"] }}`
- `Operation`: `Is True`

**TRUE output** → connect to `Set - Mark Valid Lead`

**FALSE output** → connect to `Google Sheets - Log Validation Error`

---

## STEP 9: Log validation errors

### Node 10 — Google Sheets - Log Validation Error

Any record that failed required-field check or email validation lands here. It is written to your `Error Log` sheet so the legal team can manually follow up.

1. Add a `Google Sheets` node.
2. Rename it to:

```
Google Sheets - Log Validation Error
```

3. Configure:

- `Credential`: your `Google Sheets OAuth2`
- `Document ID`: your spreadsheet ID
- `Sheet Name`: `Error Log`
- `Operation`: `Append Row`

4. Map these columns:

| Sheet Column | Expression |
|---|---|
| `Timestamp` | `{{ new Date().toISOString() }}` |
| `Lead Name` | `{{ $json["lead_name"] }}` |
| `Email` | `{{ $json["email_raw"] }}` |
| `Error Type` | `Validation Failure` |
| `Error Message` | `{{ $json["email_valid"] === false ? "Invalid email format" : "Missing required field" }}` |

5. In node settings, turn on:

- `Retry On Fail`: Yes
- `Max Tries`: `3`
- `Wait Between Tries (ms)`: `2000`

This branch terminates here. Invalid records do not continue into the priority engine or CRM.

---

## STEP 10: Mark valid leads

### Node 11 — Set - Mark Valid Lead

This node stamps a `record_valid` flag on every item that passed validation. It also sets the initial status to `Pending` which will be written to Airtable later.

1. Add a `Set` node.
2. Rename it to:

```
Set - Mark Valid Lead
```

3. Add these assignments:

| Field | Value |
|---|---|
| `record_valid` | `true` |
| `consultation_status` | `Pending` |

4. Leave `Keep Only Set` as **No** — keep all upstream fields.

---

## STEP 11: Build the priority engine

The priority engine has three nodes. Do not compress them into one — they need to be separate so you can debug each signal independently.

### Node 12 — IF - Urgency Is High Priority?

This node checks whether the urgency level warrants immediate attention.

1. Add an `IF` node.
2. Rename it to:

```
IF - Urgency Is High Priority?
```

3. Set `Combine Conditions` to `OR`.

4. Add two conditions:

**Condition 1**

- `Value 1`: `{{ $json["urgency_level"] }}`
- `Operation`: `Equals`
- `Value 2`: `Urgent`

**Condition 2**

- `Value 1`: `{{ $json["urgency_level"] }}`
- `Operation`: `Equals`
- `Value 2`: `High`

**TRUE output** → connect to `Merge - Combine Priority Signals` (input 0)

**FALSE output** → also connect to `Merge - Combine Priority Signals` (input 0, different branch — explained in Step 11C)

> **Note:** You will connect both TRUE and FALSE outputs of this node to the Merge node. The `urgency_triggered` flag written in the Code node below is what carries the actual true/false logic into the merge.

Before connecting, add an intermediate Set node on each branch to stamp the result:

**On the TRUE branch**, add a `Set` node named:

```
Set - Urgency Flag True
```

Add one assignment:

- `urgency_triggered`: `true`

**On the FALSE branch**, add a `Set` node named:

```
Set - Urgency Flag False
```

Add one assignment:

- `urgency_triggered`: `false`

Connect both of these Set nodes into the Merge node.

---

### Node 13 — Code - Calculate Days Until Court Date

This node does the date math that determines whether a court date is dangerously close.

1. Add a `Code` node **in parallel** to the urgency IF branch — it should receive its input from `Set - Mark Valid Lead` as well.

> **Architecture note:** In n8n, you can split one node's output into two branches by connecting it to two different nodes. Both `IF - Urgency Is High Priority?` and `Code - Calculate Days Until Court Date` receive the same item from `Set - Mark Valid Lead`.

2. Rename this node to:

```
Code - Calculate Days Until Court Date
```

3. Set:
   - `Mode`: `Run Once for Each Item`
   - `Language`: `JavaScript`

4. Paste this exact code:

```javascript
const item = $json;

const courtEpoch = item.court_date_epoch;
const today = new Date();
today.setHours(0, 0, 0, 0);
const todayEpoch = today.getTime();

let daysUntilCourt = null;
let courtDateTriggered = false;

if (courtEpoch !== null && item.court_date_valid === true) {
  daysUntilCourt = Math.ceil((courtEpoch - todayEpoch) / (1000 * 60 * 60 * 24));
  // If court date is within 30 calendar days (including past-due)
  courtDateTriggered = daysUntilCourt <= 30;
}

return [{
  json: {
    ...item,
    days_until_court: daysUntilCourt,
    court_date_triggered: courtDateTriggered,
  },
}];
```

Expected output additions:

```json
{
  "days_until_court": 16,
  "court_date_triggered": true
}
```

---

### Node 14 — IF - Court Date Within 30 Days?

1. Add an `IF` node after `Code - Calculate Days Until Court Date`.
2. Rename it to:

```
IF - Court Date Within 30 Days?
```

3. Add one condition:

- `Value 1`: `{{ $json["court_date_triggered"] }}`
- `Operation`: `Is True`

**TRUE output** → add a `Set` node named `Set - Court Date Flag True` with assignment `court_triggered: true`, then connect to `Merge - Combine Priority Signals` (input 1)

**FALSE output** → add a `Set` node named `Set - Court Date Flag False` with assignment `court_triggered: false`, then connect to `Merge - Combine Priority Signals` (input 1)

---

### Node 15 — Merge - Combine Priority Signals

This is the core of the priority engine. It receives two separate items — one from the urgency branch and one from the court date branch — and combines them.

1. Add a `Merge` node.
2. Rename it to:

```
Merge - Combine Priority Signals
```

3. Configure:

- `Mode`: `Combine`
- `Combination Mode`: `Merge By Position`

This means item 0 from input 0 merges with item 0 from input 1, producing a single item that contains all fields from both branches.

After the merge, every item will have both `urgency_triggered` and `court_triggered` available for the final decision.

---

### Node 16 — Set - Assign Priority Flag

This node reads both signals and writes the final `is_priority` boolean that all downstream nodes use.

1. Add a `Set` node after `Merge - Combine Priority Signals`.
2. Rename it to:

```
Set - Assign Priority Flag
```

3. Add one assignment using a JavaScript expression:

| Field | Expression |
|---|---|
| `is_priority` | `{{ $json["urgency_triggered"] === true \|\| $json["court_triggered"] === true }}` |

After this node, every valid lead has a clean `is_priority: true` or `is_priority: false` flag.

---

## STEP 12: Write to Airtable

### Node 17 — Airtable - Create Lead Record

Every valid lead — priority or not — gets written to Airtable here.

1. Add an `Airtable` node.
2. Rename it to:

```
Airtable - Create Lead Record
```

3. Configure:

- `Credential`: your `Airtable Personal Access Token`
- `Operation`: `Create Record`
- `Base ID`: your Airtable base ID
- `Table ID`: your `Leads` table ID

4. Map these fields:

| Airtable Field | Expression |
|---|---|
| `Name` | `{{ $json["lead_name"] }}` |
| `Email` | `{{ $json["email"] }}` |
| `Case Type` | `{{ $json["case_type"] }}` |
| `Priority` | `{{ $json["urgency_level"] }}` |
| `Court Date` | `{{ $json["court_date_iso"] }}` |
| `Status` | `{{ $json["consultation_status"] }}` |
| `Priority Flag` | `{{ $json["is_priority"] }}` |
| `Submission ID` | `{{ $json["submission_id"] }}` |

5. In node settings, turn on:

- `Retry On Fail`: Yes
- `Max Tries`: `3`
- `Wait Between Tries (ms)`: `3000`

> If Airtable fails after three retries, the node throws an error. That error will be caught by the global error workflow and logged to your `Error Log` sheet.

---

## STEP 13: Route priority leads to notifications

### Node 18 — IF - Is Priority Lead?

1. Add an `IF` node after `Airtable - Create Lead Record`.
2. Rename it to:

```
IF - Is Priority Lead?
```

3. Add one condition:

- `Value 1`: `{{ $json["is_priority"] }}`
- `Operation`: `Is True`

**TRUE output** → connect to both `Slack - Send Priority Alert` and `Gmail - Send Priority Email`

**FALSE output** → connect to `Wait - Throttle Between Records`

---

## STEP 14: Send the Slack notification

### Node 19 — Slack - Send Priority Alert

1. Add a `Slack` node.
2. Rename it to:

```
Slack - Send Priority Alert
```

3. Configure:

- `Credential`: your `Slack OAuth2`
- `Operation`: `Send Message`
- `Channel`: your `#urgent-legal-leads` channel ID

4. Set the `Message` field to this exact template:

```
🚨 *PRIORITY LEGAL LEAD — IMMEDIATE ATTENTION REQUIRED*

*Lead Name:* {{ $json["lead_name"] }}
*Case Type:* {{ $json["case_type"] }}
*Urgency Level:* {{ $json["urgency_level"] }}
*Court Date:* {{ $json["court_date_display"] }}
*Days Until Court:* {{ $json["days_until_court"] !== null ? $json["days_until_court"] + " days" : "Unknown" }}
*Email:* {{ $json["email"] }}
*Submission ID:* {{ $json["submission_id"] }}

{{ $json["urgency_triggered"] ? "⚠️ Urgency level flagged as high priority." : "" }}
{{ $json["court_triggered"] ? "📅 Court date is within 30 days." : "" }}

Please review and assign to a consultant immediately.
```

5. In node settings, turn on:

- `Retry On Fail`: Yes
- `Max Tries`: `2`
- `Wait Between Tries (ms)`: `2000`

---

## STEP 15: Send the email notification

### Node 20 — Gmail - Send Priority Email

1. Add a `Gmail` node.
2. Rename it to:

```
Gmail - Send Priority Email
```

3. Configure:

- `Credential`: your `Gmail OAuth2`
- `Operation`: `Send`
- `To`: the legal team's intake email (e.g. `intake@pacificpeaklaw.com`)
- `Subject`:

```
🚨 Priority Lead Alert — {{ $json["lead_name"] }} | {{ $json["case_type"] }}
```

4. Set `Body Type` to `HTML`.

5. Paste this exact HTML template into the body field:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>Priority Lead Alert</title>
</head>
<body style="font-family: Arial, sans-serif; background-color: #f4f4f4; margin: 0; padding: 20px;">
  <div style="max-width: 600px; margin: 0 auto; background-color: #ffffff; border-radius: 8px; overflow: hidden; box-shadow: 0 2px 8px rgba(0,0,0,0.1);">

    <div style="background-color: #b91c1c; padding: 24px 32px;">
      <h1 style="color: #ffffff; margin: 0; font-size: 20px;">🚨 Priority Legal Lead Alert</h1>
      <p style="color: #fecaca; margin: 8px 0 0 0; font-size: 14px;">Pacific Peak Immigration Law — Intake Automation</p>
    </div>

    <div style="padding: 32px;">
      <p style="font-size: 16px; color: #111827; margin-top: 0;">A new lead has been flagged as <strong>high priority</strong> and requires immediate attention.</p>

      <table style="width: 100%; border-collapse: collapse; margin-top: 16px;">
        <tr style="border-bottom: 1px solid #e5e7eb;">
          <td style="padding: 12px 0; font-weight: bold; color: #374151; width: 40%;">Lead Name</td>
          <td style="padding: 12px 0; color: #111827;">{{ $json["lead_name"] }}</td>
        </tr>
        <tr style="border-bottom: 1px solid #e5e7eb;">
          <td style="padding: 12px 0; font-weight: bold; color: #374151;">Email</td>
          <td style="padding: 12px 0; color: #111827;">{{ $json["email"] }}</td>
        </tr>
        <tr style="border-bottom: 1px solid #e5e7eb;">
          <td style="padding: 12px 0; font-weight: bold; color: #374151;">Case Type</td>
          <td style="padding: 12px 0; color: #111827;">{{ $json["case_type"] }}</td>
        </tr>
        <tr style="border-bottom: 1px solid #e5e7eb;">
          <td style="padding: 12px 0; font-weight: bold; color: #374151;">Urgency Level</td>
          <td style="padding: 12px 0; color: #b91c1c;"><strong>{{ $json["urgency_level"] }}</strong></td>
        </tr>
        <tr style="border-bottom: 1px solid #e5e7eb;">
          <td style="padding: 12px 0; font-weight: bold; color: #374151;">Court Date</td>
          <td style="padding: 12px 0; color: #111827;">{{ $json["court_date_display"] }}</td>
        </tr>
        <tr style="border-bottom: 1px solid #e5e7eb;">
          <td style="padding: 12px 0; font-weight: bold; color: #374151;">Days Until Court</td>
          <td style="padding: 12px 0; color: #111827;">{{ $json["days_until_court"] !== null ? $json["days_until_court"] + " days" : "Unknown" }}</td>
        </tr>
        <tr>
          <td style="padding: 12px 0; font-weight: bold; color: #374151;">Submission ID</td>
          <td style="padding: 12px 0; color: #111827;">{{ $json["submission_id"] }}</td>
        </tr>
      </table>

      <div style="margin-top: 24px; background-color: #fef2f2; border-left: 4px solid #b91c1c; padding: 16px; border-radius: 4px;">
        <p style="margin: 0; color: #7f1d1d; font-size: 14px;">
          {{ $json["urgency_triggered"] ? "⚠️ This lead was flagged because their urgency level is Urgent or High." : "" }}<br/>
          {{ $json["court_triggered"] ? "📅 This lead was flagged because their court date is within 30 days." : "" }}
        </p>
      </div>

      <p style="margin-top: 24px; font-size: 14px; color: #6b7280;">
        This notification was generated automatically by the Pacific Peak Lead Intake system.<br/>
        Please log in to Airtable to assign a consultant and update the lead status.
      </p>
    </div>

    <div style="background-color: #f9fafb; padding: 16px 32px; text-align: center;">
      <p style="margin: 0; font-size: 12px; color: #9ca3af;">Pacific Peak Immigration Law — Automated Intake System</p>
    </div>

  </div>
</body>
</html>
```

6. In node settings, turn on:

- `Retry On Fail`: Yes
- `Max Tries`: `2`
- `Wait Between Tries (ms)`: `2000`

---

## STEP 16: Throttle between records

### Node 21 — Wait - Throttle Between Records

After each lead is fully processed, add a brief pause before the SplitInBatches loop picks up the next one. This protects against API rate limits.

1. Add a `Wait` node after `IF - Is Priority Lead?` FALSE output and after the notification nodes.
2. Rename it to:

```
Wait - Throttle Between Records
```

3. Configure:

- `Resume`: `After Time Interval`
- `Wait Amount`: `1`
- `Wait Unit`: `Seconds`

Connect the output of this node back to the input of `SplitInBatches - Process One Lead at a Time` to complete the loop.

---

## STEP 17: Build the error workflow

This is a completely separate workflow. Its only job is to catch n8n execution-level failures in the main workflow and write them to the `Error Log` sheet.

### 17A — Add the Error Trigger node

Inside the `Pacific Peak - Error Logger` workflow:

1. Add an `Error Trigger` node.
2. Rename it to:

```
Error Trigger - Global
```

This node fires automatically whenever the main workflow throws an uncaught error.

---

### 17B — Add the Set node for error normalization

1. Add a `Set` node after `Error Trigger - Global`.
2. Rename it to:

```
Set - Prepare Error Log Entry
```

3. Add these assignments:

| Field | Expression |
|---|---|
| `Timestamp` | `{{ new Date().toISOString() }}` |
| `Lead Name` | `{{ $json.execution?.data?.resultData?.runData?.["Set - Rename Raw Fields"]?.[0]?.data?.main?.[0]?.[0]?.json?.lead_name \|\| "Unknown" }}` |
| `Email` | `{{ $json.execution?.data?.resultData?.runData?.["Set - Rename Raw Fields"]?.[0]?.data?.main?.[0]?.[0]?.json?.email_raw \|\| "Unknown" }}` |
| `Error Type` | `Execution Failure` |
| `Error Message` | `{{ $json.execution?.error?.message \|\| $json.execution?.data?.resultData?.error?.message \|\| "Unknown error" }}` |

> The Lead Name and Email expressions attempt to pull data from the last successfully completed Set node. They fall back to `"Unknown"` if the workflow crashed before that node ran.

---

### 17C — Add the Google Sheets error logger

1. Add a `Google Sheets` node after `Set - Prepare Error Log Entry`.
2. Rename it to:

```
Google Sheets - Log Execution Error
```

3. Configure:

- `Credential`: your `Google Sheets OAuth2`
- `Document ID`: your spreadsheet ID
- `Sheet Name`: `Error Log`
- `Operation`: `Append Row`

4. Map:

| Sheet Column | Expression |
|---|---|
| `Timestamp` | `{{ $json["Timestamp"] }}` |
| `Lead Name` | `{{ $json["Lead Name"] }}` |
| `Email` | `{{ $json["Email"] }}` |
| `Error Type` | `{{ $json["Error Type"] }}` |
| `Error Message` | `{{ $json["Error Message"] }}` |

5. In settings, turn on:

- `Retry On Fail`: Yes
- `Max Tries`: `3`
- `Wait Between Tries (ms)`: `2000`

---

### 17D — Link the error workflow to the main workflow

After both workflows are saved:

1. Open `Pacific Peak - Legal Lead Intake Automation`.
2. Open workflow settings (gear icon or top menu).
3. Find the **Error Workflow** dropdown.
4. Select:

```
Pacific Peak - Error Logger
```

5. Save.

This is the step that activates the error workflow connection. Without it, the error logger never fires.

---

## STEP 18: Connection map

Use this as your final sanity-check. Every arrow below is one connection you must draw on the canvas.

```
Google Sheets Trigger - Watch Lead Intake
  -> SplitInBatches - Process One Lead at a Time

Manual Trigger - Test Fallback
  -> SplitInBatches - Process One Lead at a Time

SplitInBatches - Process One Lead at a Time
  -> Set - Rename Raw Fields

Set - Rename Raw Fields
  -> Code - Normalize Field Values

Code - Normalize Field Values
  -> Code - Parse and Format Court Date

Code - Parse and Format Court Date
  -> IF - Check Required Fields Present

IF - Check Required Fields Present [TRUE]
  -> Code - Validate Email Format

IF - Check Required Fields Present [FALSE]
  -> Google Sheets - Log Validation Error

Code - Validate Email Format
  -> IF - Email Valid?

IF - Email Valid? [TRUE]
  -> Set - Mark Valid Lead

IF - Email Valid? [FALSE]
  -> Google Sheets - Log Validation Error

Set - Mark Valid Lead
  -> IF - Urgency Is High Priority?       (branch A)
  -> Code - Calculate Days Until Court Date (branch B)

IF - Urgency Is High Priority? [TRUE]
  -> Set - Urgency Flag True
  -> Merge - Combine Priority Signals [input 0]

IF - Urgency Is High Priority? [FALSE]
  -> Set - Urgency Flag False
  -> Merge - Combine Priority Signals [input 0]

Code - Calculate Days Until Court Date
  -> IF - Court Date Within 30 Days?

IF - Court Date Within 30 Days? [TRUE]
  -> Set - Court Date Flag True
  -> Merge - Combine Priority Signals [input 1]

IF - Court Date Within 30 Days? [FALSE]
  -> Set - Court Date Flag False
  -> Merge - Combine Priority Signals [input 1]

Merge - Combine Priority Signals
  -> Set - Assign Priority Flag

Set - Assign Priority Flag
  -> Airtable - Create Lead Record

Airtable - Create Lead Record
  -> IF - Is Priority Lead?

IF - Is Priority Lead? [TRUE]
  -> Slack - Send Priority Alert
  -> Gmail - Send Priority Email
  -> Wait - Throttle Between Records

IF - Is Priority Lead? [FALSE]
  -> Wait - Throttle Between Records

Wait - Throttle Between Records
  -> SplitInBatches - Process One Lead at a Time  [loop back]
```

Error workflow:

```
Error Trigger - Global
  -> Set - Prepare Error Log Entry
  -> Google Sheets - Log Execution Error
```

---

## STEP 19: Recommended canvas layout

Lay the canvas out left to right with three horizontal lanes:

- **Top lane**: urgency detection branch (IF - Urgency Is High Priority?, Set - Urgency Flag True/False)
- **Middle lane**: the main spine (triggers → split → normalize → validate → merge → Airtable → notify)
- **Bottom lane**: court date calculation branch (Code - Calculate Days Until Court Date, IF - Court Date Within 30 Days?, Set flags)

Recommended anchor positions:

- Both triggers at the far left
- SplitInBatches slightly right, centered between the two triggers
- The four normalization nodes strung left to right across the middle
- IF - Check Required Fields branches up to the validation error log below the main lane
- The Merge node in the center of the canvas where both priority branches converge
- Airtable to the right of Merge
- Slack and Gmail above Airtable at the far right
- Wait node at the far right looping back left

This layout makes the priority engine the visual center of gravity, which reflects its importance to the business logic.

---

## STEP 20: Full field mapping reference

| Google Sheets Column | Internal Name (after Set node) | Airtable Field | Notes |
|---|---|---|---|
| Submission ID | `submission_id` | `Submission ID` | Passed through unchanged |
| Lead Name | `lead_name` | `Name` | Passed through unchanged |
| Email | `email` (from `email_raw`) | `Email` | Lowercased and trimmed |
| Case Type | `case_type` (from `case_type_raw`) | `Case Type` | Title-cased |
| Urgency Level | `urgency_level` (from `urgency_raw`) | `Priority` | Title-cased |
| Court Date | `court_date_iso` (from `court_date_raw`) | `Court Date` | ISO 8601 format |
| — | `consultation_status` | `Status` | Hardcoded to `Pending` |
| — | `is_priority` | `Priority Flag` | Computed by priority engine |

---

## STEP 21: How a lead moves through the system

This walkthrough follows PPL-0001, Elena Rodriguez.

**Source row:**

```
PPL-0001 | Elena Rodriguez | e.rodriguez@example.com | Deportation Defense | Urgent | 2024-11-10 | 2024-10-25
```

1. `Google Sheets Trigger` detects the new row and outputs one item.
2. `SplitInBatches` takes that item and sends it into the pipeline alone.
3. `Set - Rename Raw Fields` renames columns to snake_case.
4. `Code - Normalize Field Values` lowercases the email, title-cases the case type.
5. `Code - Parse and Format Court Date` converts `2024-11-10` to ISO and human-readable strings, and computes the epoch timestamp.
6. `IF - Check Required Fields Present` confirms `lead_name`, `email`, and `case_type` are all non-empty. They are. Routes to TRUE.
7. `Code - Validate Email Format` runs the regex on `e.rodriguez@example.com`. Passes. Sets `email_valid: true`.
8. `IF - Email Valid?` sees `true`. Routes to `Set - Mark Valid Lead`.
9. `Set - Mark Valid Lead` stamps `record_valid: true` and `consultation_status: Pending`.
10. **Priority engine starts:**
    - `IF - Urgency Is High Priority?` sees `Urgent`. Routes TRUE. `Set - Urgency Flag True` stamps `urgency_triggered: true`.
    - `Code - Calculate Days Until Court Date` computes days between today and November 10, 2024. If within 30 days, `court_date_triggered: true`.
    - `IF - Court Date Within 30 Days?` routes accordingly.
    - `Merge - Combine Priority Signals` joins both branches.
    - `Set - Assign Priority Flag` evaluates `urgency_triggered || court_triggered`. Result: `is_priority: true`.
11. `Airtable - Create Lead Record` writes the full record to the `Leads` table.
12. `IF - Is Priority Lead?` sees `true`. Routes to both notification nodes.
13. `Slack - Send Priority Alert` posts the message to `#urgent-legal-leads`.
14. `Gmail - Send Priority Email` sends the HTML alert to the intake team.
15. `Wait - Throttle Between Records` pauses 1 second.
16. `SplitInBatches` picks up the next item — PPL-0002 — and the cycle repeats.

---

## STEP 22: Testing checklist

Test one branch at a time. Do not activate the live trigger until all five tests pass.

### Test 1: Normal (non-priority) lead

Use this sample row manually in your test trigger:

```json
{
  "Submission ID": "PPL-0002",
  "Lead Name": "Mateo Chen",
  "Email": "mchen88@provider.net",
  "Case Type": "H-1B Visa",
  "Urgency Level": "Low",
  "Court Date": "2025-05-12",
  "Submission Date": "2024-10-25"
}
```

Expected:

1. Passes validation.
2. `urgency_triggered: false`, `court_triggered: false` (court date far in future).
3. `is_priority: false`.
4. Airtable record created with `Priority Flag: false`.
5. No Slack or Gmail notification.

---

### Test 2: Urgent lead

```json
{
  "Submission ID": "PPL-0001",
  "Lead Name": "Elena Rodriguez",
  "Email": "e.rodriguez@example.com",
  "Case Type": "Deportation Defense",
  "Urgency Level": "Urgent",
  "Court Date": "2024-11-10",
  "Submission Date": "2024-10-25"
}
```

Expected:

1. Passes validation.
2. `urgency_triggered: true`.
3. `is_priority: true`.
4. Airtable record created with `Priority Flag: true`.
5. Slack message posted.
6. HTML email sent.

---

### Test 3: Court date within 30 days (non-urgent urgency level)

```json
{
  "Submission ID": "PPL-TEST",
  "Lead Name": "Test Lead",
  "Email": "test@example.com",
  "Case Type": "Asylum Claim",
  "Urgency Level": "Medium",
  "Court Date": "{{ date 15 days from today in YYYY-MM-DD format }}",
  "Submission Date": "2024-10-31"
}
```

Replace the Court Date placeholder with an actual date 15 days from when you are testing.

Expected:

1. Passes validation.
2. `urgency_triggered: false`.
3. `court_date_triggered: true`.
4. `is_priority: true` (because court trigger is true).
5. Both notifications fire.

---

### Test 4: Invalid email

```json
{
  "Submission ID": "PPL-ERR-01",
  "Lead Name": "Bad Email Lead",
  "Email": "not-an-email",
  "Case Type": "Naturalization",
  "Urgency Level": "Low",
  "Court Date": "2025-06-01",
  "Submission Date": "2024-10-31"
}
```

Expected:

1. Passes required-field check.
2. `email_valid: false`.
3. Routes to `Google Sheets - Log Validation Error`.
4. Error row written to `Error Log` sheet with message `"Invalid email format"`.
5. No Airtable record created.

---

### Test 5: Missing required field

```json
{
  "Submission ID": "PPL-ERR-02",
  "Lead Name": "",
  "Email": "someone@example.com",
  "Case Type": "H-1B Visa",
  "Urgency Level": "Low",
  "Court Date": "2025-04-01",
  "Submission Date": "2024-10-31"
}
```

Expected:

1. `IF - Check Required Fields Present` sees empty `lead_name`. Routes FALSE.
2. Error row written to `Error Log` sheet.
3. No further processing.

---

### Test 6: Forced Airtable failure

1. Temporarily change the Airtable table ID to an invalid value.
2. Send the Urgent lead payload from Test 2.
3. Watch the node retry three times with 3-second waits between each.
4. After the third failure, the node throws an error.
5. The `Pacific Peak - Error Logger` workflow fires and writes a row to `Error Log`.

After confirming, restore the correct table ID.

---

## STEP 23: Expected results summary

After running all six tests successfully:

| Test | Airtable Record | Slack Message | Email | Error Log |
|---|---|---|---|---|
| Normal lead | ✅ Created | ❌ None | ❌ None | ❌ None |
| Urgent lead | ✅ Created | ✅ Sent | ✅ Sent | ❌ None |
| Court ≤30 days | ✅ Created | ✅ Sent | ✅ Sent | ❌ None |
| Invalid email | ❌ None | ❌ None | ❌ None | ✅ Logged |
| Missing field | ❌ None | ❌ None | ❌ None | ✅ Logged |
| Airtable failure | ❌ Failed | ❌ None | ❌ None | ✅ Logged |

---

## STEP 24: Best practices

### Naming conventions

Every node name in this workflow follows the pattern:

```
[Node Type] - [What It Does]
```

Examples: `IF - Email Valid?`, `Code - Calculate Days Until Court Date`, `Set - Assign Priority Flag`.

This convention makes the canvas readable at a glance and makes error logs meaningful — n8n includes the failing node name in error messages, so descriptive names save debugging time.

### Node grouping

On the n8n canvas, use the built-in sticky note feature to label the three logical sections:

- **Data Preparation Zone** — from `Set - Rename Raw Fields` to `Code - Parse and Format Court Date`
- **Validation Gate** — from `IF - Check Required Fields Present` to `Set - Mark Valid Lead`
- **Priority Engine** — from the two parallel branches through `Set - Assign Priority Flag`

This makes it obvious which part of the canvas to look at when a specific type of problem is reported.

### Debugging techniques

1. **Run with a single test row first.** Use the Manual Trigger and a hardcoded Set node. Only switch to the live Google Sheets trigger after every node is confirmed working.

2. **Pin output at each Code node.** In n8n, you can pin the output of a node during testing. This lets you inspect what each Code node produces without re-running upstream nodes.

3. **Check the execution log.** Every workflow run is stored in n8n's execution history. If a node fails, open the execution, click the failing node, and read the exact error in the output panel.

4. **Test the Merge node carefully.** Merge nodes behave unexpectedly if one input branch produces zero items. Always confirm both branches of the priority engine produce exactly one item before connecting them to the Merge node.

5. **Validate your Airtable field names.** Airtable is case-sensitive. If the node maps to `"name"` but the field is named `"Name"`, the record will be created with a blank field and no error will appear. Open your Airtable table and copy field names exactly as they appear.

---

## STEP 25: Activation checklist

Before activating the main workflow, confirm every item on this list:

1. `Lead Intake` sheet exists with correct headers and test data.
2. `Error Log` sheet exists with correct headers.
3. All five Airtable fields exist and names match exactly.
4. Slack credential is valid and channel ID is correct.
5. Gmail credential is valid and the "to" address is reachable.
6. All six test scenarios passed successfully in the editor.
7. The `Pacific Peak - Error Logger` workflow is assigned in the main workflow's error workflow setting.
8. The error workflow is saved and activated separately.
9. Only after all of the above: activate `Pacific Peak - Legal Lead Intake Automation`.

---

## Final result

Once built and activated, this system gives Pacific Peak Immigration Law a production-ready lead pipeline:

- Every new sheet row triggers the intake pipeline within one minute.
- All data is normalized and validated before any CRM write happens.
- The dual-signal priority engine catches urgent cases by urgency level, by court proximity, or by both — so no critical case slips through.
- Every priority lead triggers both a Slack message and a formatted HTML email simultaneously.
- Every invalid record is routed to the error log rather than silently dropped.
- Execution-level failures are captured by the dedicated error workflow.
- The SplitInBatches architecture ensures the system scales to any volume of incoming leads without redesign.

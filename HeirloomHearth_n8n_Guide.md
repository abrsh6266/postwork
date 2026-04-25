# Multi-Channel Dietary Safety & Logistics Sync for Heirloom & Hearth Catering

---

> **Document Type:** Full n8n Implementation Guide  
> **Audience:** n8n beginners to intermediate users  
> **Estimated Build Time:** 3–5 hours  
> **Node Count:** 38 nodes across 2 workflows  
> **Version:** n8n 1.x (Cloud or Self-hosted)

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Full Node Architecture Diagram](#2-full-node-architecture-diagram)
3. [Credentials Required](#3-credentials-required)
4. [Google Sheets Setup](#4-google-sheets-setup)
5. [Step-by-Step Node Build Guide — Main Workflow](#5-step-by-step-node-build-guide--main-workflow)
6. [Step-by-Step Node Build Guide — Error Handling Workflow](#6-step-by-step-node-build-guide--error-handling-workflow)
7. [Connection Map](#7-connection-map)
8. [Testing Scenarios](#8-testing-scenarios)
9. [Execution Proof Guide](#9-execution-proof-guide)
10. [Final Deliverables & Activation](#10-final-deliverables--activation)

---

## 1. Project Overview

### The Problem

Luxury catering events serving hundreds of guests face a life-threatening operational gap: dietary restriction requests submitted during RSVP often fail to reach the kitchen team with the urgency and clarity required. A single missed peanut allergy can be fatal. A Celiac cross-contamination incident destroys reputation and creates legal liability.

### The Solution

This n8n workflow creates a **fully automated, multi-channel dietary safety and logistics synchronization system** for **Heirloom & Hearth Catering**. Every guest submission is:

- Cleaned and normalized
- Assigned a unique Safety ID
- Checked for duplicates
- Risk-categorized (High / Medium / Low)
- Routed to the appropriate communication channels (Slack, SMS, Discord, Email)
- Logged to a master Google Sheets manifest with role-specific tabs
- Confirmed back to the guest with a luxury-brand assurance email

Errors are caught in a second workflow and reported to operations management.

### Business Value

| Metric | Before | After |
|---|---|---|
| Kitchen awareness of high-risk allergies | Manual, inconsistent | Instant, multi-channel |
| Duplicate submissions | No protection | Duplicate-blocked |
| Guest confirmation | None | Automated luxury email |
| Audit trail | None | Full Google Sheets log |
| Error visibility | Silent failures | Immediate ops alert |

---

## 2. Full Node Architecture Diagram

```
MAIN WORKFLOW
═══════════════════════════════════════════════════════════════════

[01] Webhook Trigger
        │
        ▼
[02] Set Node — Rename & Map Fields
        │
        ▼
[03] Function Node — Capitalize Names
        │
        ▼
[04] Function Node — Normalize Dietary Restrictions
        │
        ▼
[05] Date & Time Node — Add Timestamp
        │
        ▼
[06] Function Node — Generate Safety ID
        │
        ▼
[07] Google Sheets — Lookup Existing Submissions (Duplicate Check)
        │
        ▼
[08] IF Node — Is Duplicate?
        │               │
      YES              NO
        │               │
        ▼               ▼
[09] NoOp           [10] Merge Node — Continue Pipeline
(Stop/Log)               │
                         ▼
                   [11] Code Node — Calculate Risk Score
                         │
                         ▼
                   [12] Switch Node — Risk Category Router
                         │
            ┌────────────┼────────────┐
          HIGH         MEDIUM        LOW
            │             │            │
            ▼             ▼            ▼
      [13] Wait       [21] Wait    [27] NoOp
       (2 sec)        (2 sec)      Low Risk
            │             │            │
            ▼             ▼            │
      [14] Slack      [22] Slack       │
      High Alert      Medium Warn      │
            │             │            │
            ▼             │            │
      [15] Twilio         │            │
       SMS Alert          │            │
            │             │            │
            ▼             │            │
      [16] Discord        │            │
       Alert              │            │
            │             │            │
            ▼             ▼            ▼
      [17] Google    [23] Google  [28] Google
       Sheets —       Sheets —    Sheets —
       Master         Master      Master
       Manifest       Manifest    Manifest
            │             │            │
            ▼             │            │
      [18] Google         │            │
       Sheets —           │            │
       High Risk          │            │
       Queue              │            │
            │             │            │
            ▼             ▼            ▼
      [19] Set        [24] Set    [29] Set
       Status=         Status=    Status=
       Confirmed       Confirmed  Confirmed
            │             │            │
            ▼             ▼            ▼
      [20] Gmail     [25] Gmail  [30] Gmail
       Assurance      Assurance  Assurance
       Email          Email      Email
            │             │            │
            └──────────────┴────────────┘
                           │
                           ▼
                   [31] Function Node — Generate Kitchen Label
                           │
                           ▼
                   [32] Google Sheets — Audit Log Entry
                           │
                           ▼
                   [33] Set Node — Final Status
                           │
                           ▼
                   [34] NoOp — Workflow Complete

═══════════════════════════════════════════════════════════════════
ERROR HANDLING WORKFLOW (Separate Workflow)
═══════════════════════════════════════════════════════════════════

[E01] Error Trigger
         │
         ▼
[E02] Set Node — Extract Error Details
         │
         ▼
[E03] Function Node — Format Error Report
         │
         ▼
[E04] Gmail Node — Alert Operations Manager
         │
         ▼
[E05] Google Sheets — Log to Audit Log (Failure Row)
         │
         ▼
[E06] Slack Node — Ops Channel Alert
         │
         ▼
[E07] NoOp — Error Handled
```

---

## 3. Credentials Required

Before building any node, set up the following credentials in n8n under **Settings → Credentials**.

### 3.1 Google OAuth2

Used by: Google Sheets nodes, Gmail nodes

**Steps:**
1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Create a new project or select existing
3. Enable APIs: **Google Sheets API**, **Gmail API**
4. Create OAuth2 credentials (Desktop App)
5. Download the client ID and secret
6. In n8n → Credentials → New → Google OAuth2 API
7. Paste Client ID and Client Secret
8. Click **Sign in with Google** and authorize

**Credential Name to use in n8n:** `Google OAuth2 – Heirloom`

---

### 3.2 Slack API

Used by: Slack nodes (High Alert, Medium Warning, Error Ops)

**Steps:**
1. Go to [api.slack.com/apps](https://api.slack.com/apps)
2. Create New App → From Scratch → name it `Heirloom Safety Bot`
3. Under **OAuth & Permissions**, add Bot Token Scopes:
   - `chat:write`
   - `chat:write.public`
4. Install to Workspace → copy Bot User OAuth Token
5. In n8n → Credentials → New → Slack API
6. Paste Bot Token

**Credential Name:** `Slack – Heirloom Bot`

---

### 3.3 Twilio

Used by: SMS node

**Steps:**
1. Log in to [twilio.com/console](https://twilio.com/console)
2. Copy your **Account SID** and **Auth Token**
3. Get a Twilio phone number (SMS capable)
4. In n8n → Credentials → New → Twilio API
5. Paste Account SID and Auth Token

**Credential Name:** `Twilio – Heirloom SMS`

---

### 3.4 Discord Webhook (Optional)

Used by: Discord node

**Steps:**
1. Open Discord server → channel settings → Integrations
2. Create Webhook → copy URL
3. In n8n → Credentials → New → Discord Webhook
4. Paste the Webhook URL

**Credential Name:** `Discord – Heirloom Kitchen`

---

## 4. Google Sheets Setup

Create **one Google Spreadsheet** named: `Heirloom Catering Manifest`

### 4.1 Tabs to Create

| Tab Name | Purpose |
|---|---|
| `Master Manifest` | Every submission |
| `High Risk Queue` | High risk rows only |
| `Audit Log` | All system events including errors |

---

### 4.2 Master Manifest — Column Headers (Row 1)

| A | B | C | D | E | F | G | H | I | J | K |
|---|---|---|---|---|---|---|---|---|---|---|
| Timestamp | Safety ID | Full Name | Email | Phone | Event Name | Table Number | Dietary Restriction | Severity | Notes | Status |

---

### 4.3 High Risk Queue — Column Headers (Row 1)

| A | B | C | D | E | F | G | H | I |
|---|---|---|---|---|---|---|---|---|
| Timestamp | Safety ID | Full Name | Table Number | Dietary Restriction | Severity | Email | Phone | Status |

---

### 4.4 Audit Log — Column Headers (Row 1)

| A | B | C | D | E | F |
|---|---|---|---|---|---|
| Timestamp | Event Type | Safety ID | Details | Status | Error Message |

---

## 5. Step-by-Step Node Build Guide — Main Workflow

Open n8n. Click **+ New Workflow**. Name it: `Heirloom Dietary Safety — Main`

---

### Node 01: Webhook Trigger

**Node Type:** `Webhook`  
**Node Name:** `Webhook — Guest Submission`

**Parameters:**

| Field | Value |
|---|---|
| HTTP Method | `POST` |
| Path | `heirloom-dietary` |
| Response Mode | `Respond with Workflow Data` |
| Authentication | None (or Basic Auth if desired) |

**What it does:** Listens for incoming dietary restriction form submissions sent via POST request. This is the workflow entry point.

**Expected Input Body (JSON):**
```json
{
  "full_name": "Sarah Smith",
  "email": "sarah.smith@example.com",
  "phone": "+14155551234",
  "event_name": "Thornton Wedding Gala",
  "table_number": "12",
  "dietary_restriction": "Peanut Severe",
  "severity": "Anaphylactic",
  "notes": "Carries EpiPen. Must not share cooking surfaces."
}
```

**Webhook URL will be:**
```
https://your-n8n-instance.com/webhook/heirloom-dietary
```

**Output Example:**
```json
{
  "body": {
    "full_name": "Sarah Smith",
    "email": "sarah.smith@example.com",
    ...
  },
  "headers": {...},
  "params": {}
}
```

---

### Node 02: Set Node — Rename & Map Fields

**Node Type:** `Set`  
**Node Name:** `Set — Map Incoming Fields`

**Connect from:** Node 01 (Webhook)

**What it does:** Extracts the body fields into clean, consistently named variables for the rest of the workflow. This prevents expression errors if the input source changes.

**Parameters → Add Fields (one per row):**

| Name | Type | Value Expression |
|---|---|---|
| `full_name` | String | `{{ $json.body.full_name }}` |
| `email` | String | `{{ $json.body.email }}` |
| `phone` | String | `{{ $json.body.phone }}` |
| `event_name` | String | `{{ $json.body.event_name }}` |
| `table_number` | String | `{{ $json.body.table_number }}` |
| `dietary_restriction` | String | `{{ $json.body.dietary_restriction }}` |
| `severity` | String | `{{ $json.body.severity }}` |
| `notes` | String | `{{ $json.body.notes }}` |

**Settings:**
- Keep Only Set Fields: **ON**

---

### Node 03: Function Node — Capitalize Names

**Node Type:** `Code` (use the Code node in n8n 1.x)  
**Node Name:** `Code — Capitalize Name`

**Connect from:** Node 02

**What it does:** Properly capitalizes the guest's full name and event name regardless of how they were submitted (ALL CAPS, lowercase, etc.).

**Language:** JavaScript

**Code:**
```javascript
const items = $input.all();

for (const item of items) {
  const capitalizeName = (str) => {
    if (!str) return '';
    return str
      .toLowerCase()
      .split(' ')
      .map(word => word.charAt(0).toUpperCase() + word.slice(1))
      .join(' ');
  };

  item.json.full_name = capitalizeName(item.json.full_name);
  item.json.event_name = capitalizeName(item.json.event_name);
  item.json.notes = item.json.notes || '';
}

return items;
```

**Output:** `full_name` will be `"Sarah Smith"` instead of `"SARAH SMITH"` or `"sarah smith"`.

---

### Node 04: Code Node — Normalize Dietary Restrictions

**Node Type:** `Code`  
**Node Name:** `Code — Normalize Restrictions`

**Connect from:** Node 03

**What it does:** Standardizes shorthand dietary submissions into the official internal terminology used across all communications and the manifest.

**Code:**
```javascript
const items = $input.all();

const restrictionMap = {
  'gf': 'Gluten Free',
  'gluten free': 'Gluten Free',
  'gluten-free': 'Gluten Free',
  'vegan meal': 'Vegan',
  'vegan': 'Vegan',
  'vegetarian': 'Vegetarian',
  'veggie': 'Vegetarian',
  'nut allergy': 'Tree Nut Allergy',
  'tree nut': 'Tree Nut Allergy',
  'tree nut allergy': 'Tree Nut Allergy',
  'peanut': 'Peanut Allergy',
  'peanut allergy': 'Peanut Allergy',
  'peanut severe': 'Peanut Allergy — Severe',
  'shellfish': 'Shellfish Allergy',
  'shellfish emergency': 'Shellfish Allergy — Emergency',
  'dairy free': 'Dairy Free',
  'lactose': 'Lactose Intolerant',
  'celiac': 'Celiac Disease',
  'coeliac': 'Celiac Disease',
  'halal': 'Halal',
  'kosher': 'Kosher',
  'no preference': 'No Dietary Restriction',
  'none': 'No Dietary Restriction',
};

const severityMap = {
  'anaphylactic': 'Anaphylactic',
  'life-threatening': 'Life-Threatening',
  'life threatening': 'Life-Threatening',
  'severe': 'Severe',
  'shellfish emergency': 'Emergency',
  'peanut severe': 'Severe',
  'allergy': 'Allergy',
  'medical restriction': 'Medical Restriction',
  'celiac': 'Medical Restriction',
  'vegetarian': 'Preference',
  'vegan': 'Preference',
  'halal': 'Preference',
  'kosher': 'Preference',
  'no preference': 'None',
  'none': 'None',
};

for (const item of items) {
  const rawRestriction = (item.json.dietary_restriction || '').trim().toLowerCase();
  const rawSeverity = (item.json.severity || '').trim().toLowerCase();

  item.json.dietary_restriction_normalized = restrictionMap[rawRestriction] || item.json.dietary_restriction;
  item.json.severity_normalized = severityMap[rawSeverity] || item.json.severity;
}

return items;
```

---

### Node 05: Date & Time Node — Add Timestamp

**Node Type:** `Date & Time`  
**Node Name:** `DateTime — Add Timestamp`

**Connect from:** Node 04

**What it does:** Adds the current ISO timestamp and a formatted human-readable date for use in the Safety ID and the manifest.

**Parameters:**

| Field | Value |
|---|---|
| Operation | `Format a Date` |
| Date | `{{ $now }}` |
| Format | `YYYY-MM-DD HH:mm:ss` |
| Output Field Name | `formatted_timestamp` |

Then add a second operation (or use a Code node below if chaining isn't supported):
- Output the date portion only as `date_for_id` in format `YYYYMMDD`

> **Tip:** If the Date & Time node only supports one operation, use this Code node instead:

**Alternative Code Node — Add Timestamp:**
```javascript
const items = $input.all();
const now = new Date();

const pad = (n) => String(n).padStart(2, '0');

const year = now.getFullYear();
const month = pad(now.getMonth() + 1);
const day = pad(now.getDate());
const hours = pad(now.getHours());
const minutes = pad(now.getMinutes());
const seconds = pad(now.getSeconds());

for (const item of items) {
  item.json.formatted_timestamp = `${year}-${month}-${day} ${hours}:${minutes}:${seconds}`;
  item.json.date_for_id = `${year}${month}${day}`;
}

return items;
```

---

### Node 06: Code Node — Generate Safety ID

**Node Type:** `Code`  
**Node Name:** `Code — Generate Safety ID`

**Connect from:** Node 05

**What it does:** Creates a unique Safety ID in the format `HH-YYYYMMDD-XXXXXX` where the last 6 digits are random. This ID follows the submission through every system.

**Code:**
```javascript
const items = $input.all();

for (const item of items) {
  const date = item.json.date_for_id || new Date().toISOString().slice(0, 10).replace(/-/g, '');
  const random6 = Math.floor(100000 + Math.random() * 900000).toString();
  item.json.safety_id = `HH-${date}-${random6}`;
}

return items;
```

**Output Example:**
```
safety_id: "HH-20260425-381922"
```

---

### Node 07: Google Sheets — Lookup Existing Submissions

**Node Type:** `Google Sheets`  
**Node Name:** `Sheets — Duplicate Check Lookup`

**Connect from:** Node 06

**Credentials:** `Google OAuth2 – Heirloom`

**Parameters:**

| Field | Value |
|---|---|
| Operation | `Read Rows` |
| Spreadsheet | `Heirloom Catering Manifest` |
| Sheet | `Master Manifest` |
| Filters → Column | `Email` |
| Filters → Value | `{{ $json.email }}` |

**What it does:** Searches the Master Manifest for any existing row with the same email address. If found, the submission is a duplicate.

---

### Node 08: IF Node — Is Duplicate?

**Node Type:** `IF`  
**Node Name:** `IF — Is Duplicate?`

**Connect from:** Node 07

**What it does:** Checks whether the Google Sheets lookup returned any results (meaning a prior submission exists for this email).

**Conditions:**

| Field | Operator | Value |
|---|---|---|
| `{{ $json.length }}` | Greater than | `0` |

> **Note:** Use `{{ $input.all().length }}` if checking the count of returned rows.

**True Branch (YES = Duplicate):** Connect to Node 09  
**False Branch (NO = New Submission):** Connect to Node 10

---

### Node 09: NoOp — Stop Duplicate

**Node Type:** `NoOp`  
**Node Name:** `NoOp — Duplicate Blocked`

**Connect from:** Node 08 (True branch)

**What it does:** Silently ends the workflow for duplicate submissions. In production, you could add a "Duplicate Notice" email here.

> **Optional Enhancement:** Add a Gmail node here to notify the guest that their request was already received.

---

### Node 10: Merge Node — Continue Pipeline

**Node Type:** `Merge`  
**Node Name:** `Merge — Continue After Duplicate Check`

**Connect from:** Node 08 (False branch — no duplicate)

**Mode:** `Pass-through` (simply forwards the data)

**What it does:** Acts as a clean merge/join point so the workflow architecture remains clear. This node can also be used in bulk-import scenarios to merge multiple batches.

---

### Node 11: Code Node — Calculate Risk Score

**Node Type:** `Code`  
**Node Name:** `Code — Risk Score Calculator`

**Connect from:** Node 10

**What it does:** Assigns a numeric risk score to each submission based on severity. This score is stored in the manifest and can be used for analytics.

**Code:**
```javascript
const items = $input.all();

const highKeywords = [
  'anaphylactic', 'life-threatening', 'life threatening',
  'emergency', 'peanut allergy — severe', 'shellfish allergy — emergency',
  'severe'
];

const mediumKeywords = [
  'allergy', 'medical restriction', 'celiac disease', 'celiac',
  'tree nut allergy', 'gluten free', 'dairy free', 'lactose intolerant'
];

for (const item of items) {
  const severity = (item.json.severity_normalized || '').toLowerCase();
  const restriction = (item.json.dietary_restriction_normalized || '').toLowerCase();
  const combined = severity + ' ' + restriction;

  let riskScore = 20;
  let riskCategory = 'LOW';

  const isHigh = highKeywords.some(kw => combined.includes(kw));
  const isMedium = mediumKeywords.some(kw => combined.includes(kw));

  if (isHigh) {
    riskScore = 100;
    riskCategory = 'HIGH';
  } else if (isMedium) {
    riskScore = 60;
    riskCategory = 'MEDIUM';
  }

  item.json.risk_score = riskScore;
  item.json.risk_category = riskCategory;
}

return items;
```

---

### Node 12: Switch Node — Risk Category Router

**Node Type:** `Switch`  
**Node Name:** `Switch — Risk Category Router`

**Connect from:** Node 11

**What it does:** Routes the workflow into three branches based on `risk_category`.

**Parameters:**

| Field | Value |
|---|---|
| Mode | `Rules` |
| Routing Expression | `{{ $json.risk_category }}` |

**Rules:**

| Rule Name | Condition | Output |
|---|---|---|
| HIGH | Equals `HIGH` | Output 1 |
| MEDIUM | Equals `MEDIUM` | Output 2 |
| LOW | Equals `LOW` | Output 3 |
| Fallback | All other values | Output 3 (default to LOW) |

---

## HIGH RISK BRANCH (Switch Output 1)

---

### Node 13: Wait Node — High Risk Delay

**Node Type:** `Wait`  
**Node Name:** `Wait — High Risk 2sec`

**Connect from:** Node 12 (Output 1 — HIGH)

**Parameters:**

| Field | Value |
|---|---|
| Resume | `After Time Interval` |
| Amount | `2` |
| Unit | `Seconds` |

**What it does:** Adds a 2-second delay before firing notifications, allowing any final data transformations upstream to complete.

---

### Node 14: Slack Node — High Risk Alert

**Node Type:** `Slack`  
**Node Name:** `Slack — HIGH RISK Kitchen Alert`

**Connect from:** Node 13

**Credentials:** `Slack – Heirloom Bot`

**Parameters:**

| Field | Value |
|---|---|
| Operation | `Send a Message` |
| Channel | `#kitchen-alerts` |
| Message Type | `Block Kit` |

**Block Kit JSON:**
```json
[
  {
    "type": "header",
    "text": {
      "type": "plain_text",
      "text": "🚨 HIGH PRIORITY DIETARY ALERT",
      "emoji": true
    }
  },
  {
    "type": "section",
    "fields": [
      {
        "type": "mrkdwn",
        "text": "*Guest Name:*\n{{ $json.full_name }}"
      },
      {
        "type": "mrkdwn",
        "text": "*Table Number:*\n{{ $json.table_number }}"
      }
    ]
  },
  {
    "type": "section",
    "fields": [
      {
        "type": "mrkdwn",
        "text": "*Dietary Restriction:*\n{{ $json.dietary_restriction_normalized }}"
      },
      {
        "type": "mrkdwn",
        "text": "*Severity:*\n{{ $json.severity_normalized }}"
      }
    ]
  },
  {
    "type": "section",
    "fields": [
      {
        "type": "mrkdwn",
        "text": "*Safety ID:*\n`{{ $json.safety_id }}`"
      },
      {
        "type": "mrkdwn",
        "text": "*Event:*\n{{ $json.event_name }}"
      }
    ]
  },
  {
    "type": "section",
    "text": {
      "type": "mrkdwn",
      "text": "*Notes:*\n{{ $json.notes }}"
    }
  },
  {
    "type": "divider"
  },
  {
    "type": "context",
    "elements": [
      {
        "type": "mrkdwn",
        "text": "⚠️ *ACTION REQUIRED* — Verify kitchen prep protocols immediately. Do not proceed without Head Chef confirmation."
      }
    ]
  }
]
```

---

### Node 15: Twilio Node — Emergency SMS

**Node Type:** `Twilio`  
**Node Name:** `Twilio — Emergency SMS to Kitchen Manager`

**Connect from:** Node 14

**Credentials:** `Twilio – Heirloom SMS`

**Parameters:**

| Field | Value |
|---|---|
| Operation | `Send SMS` |
| From | Your Twilio number, e.g. `+18005550100` |
| To | Kitchen Manager number (store as env var or hardcode) e.g. `+14155559999` |
| Message | See below |

**Message Expression:**
```
URGENT Allergy Alert — Table {{ $json.table_number }}.
Guest: {{ $json.full_name }}.
{{ $json.dietary_restriction_normalized }} / {{ $json.severity_normalized }}.
Safety ID: {{ $json.safety_id }}.
Event: {{ $json.event_name }}.
IMMEDIATE ACTION REQUIRED.
```

> **Pro Tip:** Store the kitchen manager phone number as an n8n environment variable named `KITCHEN_MANAGER_PHONE` and reference it as `{{ $env.KITCHEN_MANAGER_PHONE }}`.

---

### Node 16: Discord Node — Secondary Alert

**Node Type:** `Discord`  
**Node Name:** `Discord — Kitchen Alert (Secondary)`

**Connect from:** Node 15

**Credentials:** `Discord – Heirloom Kitchen`

**Parameters:**

| Field | Value |
|---|---|
| Operation | `Send a Message` |
| Webhook URL | `{{ $credentials.webhookUrl }}` |
| Content | See below |

**Message:**
```
🚨 **HIGH RISK DIETARY ALERT**
**Guest:** {{ $json.full_name }}
**Table:** {{ $json.table_number }}
**Restriction:** {{ $json.dietary_restriction_normalized }}
**Severity:** {{ $json.severity_normalized }}
**Safety ID:** `{{ $json.safety_id }}`
**Event:** {{ $json.event_name }}
**Notes:** {{ $json.notes }}
> ⚠️ Immediate kitchen protocol review required.
```

---

### Node 17: Google Sheets — Write to Master Manifest (HIGH)

**Node Type:** `Google Sheets`  
**Node Name:** `Sheets — Master Manifest (HIGH)`

**Connect from:** Node 16

**Credentials:** `Google OAuth2 – Heirloom`

**Parameters:**

| Field | Value |
|---|---|
| Operation | `Append Row` |
| Spreadsheet | `Heirloom Catering Manifest` |
| Sheet | `Master Manifest` |

**Column Mappings:**

| Column | Expression |
|---|---|
| Timestamp | `{{ $json.formatted_timestamp }}` |
| Safety ID | `{{ $json.safety_id }}` |
| Full Name | `{{ $json.full_name }}` |
| Email | `{{ $json.email }}` |
| Phone | `{{ $json.phone }}` |
| Event Name | `{{ $json.event_name }}` |
| Table Number | `{{ $json.table_number }}` |
| Dietary Restriction | `{{ $json.dietary_restriction_normalized }}` |
| Severity | `{{ $json.severity_normalized }}` |
| Notes | `{{ $json.notes }}` |
| Status | `Pending` |

---

### Node 18: Google Sheets — Write to High Risk Queue

**Node Type:** `Google Sheets`  
**Node Name:** `Sheets — High Risk Queue`

**Connect from:** Node 17

**Credentials:** `Google OAuth2 – Heirloom`

**Parameters:**

| Field | Value |
|---|---|
| Operation | `Append Row` |
| Spreadsheet | `Heirloom Catering Manifest` |
| Sheet | `High Risk Queue` |

**Column Mappings:**

| Column | Expression |
|---|---|
| Timestamp | `{{ $json.formatted_timestamp }}` |
| Safety ID | `{{ $json.safety_id }}` |
| Full Name | `{{ $json.full_name }}` |
| Table Number | `{{ $json.table_number }}` |
| Dietary Restriction | `{{ $json.dietary_restriction_normalized }}` |
| Severity | `{{ $json.severity_normalized }}` |
| Email | `{{ $json.email }}` |
| Phone | `{{ $json.phone }}` |
| Status | `URGENT — Requires Kitchen Confirmation` |

---

### Node 19: Set Node — Mark Confirmed (HIGH)

**Node Type:** `Set`  
**Node Name:** `Set — Status Confirmed (HIGH)`

**Connect from:** Node 18

**Parameters → Add Field:**

| Name | Type | Value |
|---|---|---|
| `status` | String | `Confirmed` |
| `branch` | String | `HIGH` |

---

### Node 20: Gmail Node — Guest Assurance Email (HIGH)

**Node Type:** `Gmail`  
**Node Name:** `Gmail — Assurance Email (HIGH)`

**Connect from:** Node 19

**Credentials:** `Google OAuth2 – Heirloom`

**Parameters:**

| Field | Value |
|---|---|
| Operation | `Send` |
| To | `{{ $json.email }}` |
| Subject | `Your Dietary Safety Request Has Been Confirmed — Heirloom & Hearth` |
| Email Type | `HTML` |
| Body | See below |

**HTML Body:**
```html
<!DOCTYPE html>
<html>
<head>
  <style>
    body { font-family: Georgia, serif; color: #2c2c2c; background: #fafaf7; margin: 0; padding: 0; }
    .container { max-width: 600px; margin: 40px auto; padding: 40px; background: #fff; border-top: 4px solid #8b6914; }
    .logo { font-size: 22px; color: #8b6914; letter-spacing: 2px; margin-bottom: 30px; }
    h2 { color: #3a3a3a; font-weight: normal; }
    .detail-box { background: #fdf8ee; border-left: 3px solid #8b6914; padding: 15px 20px; margin: 20px 0; }
    .safety-id { font-family: monospace; font-size: 16px; color: #8b6914; }
    .footer { font-size: 12px; color: #888; margin-top: 30px; border-top: 1px solid #eee; padding-top: 15px; }
  </style>
</head>
<body>
  <div class="container">
    <div class="logo">HEIRLOOM & HEARTH CATERING</div>
    <h2>Dear {{ $json.full_name }},</h2>
    <p>We have received and confirmed your dietary safety request for the upcoming <strong>{{ $json.event_name }}</strong>. Your wellbeing is our highest priority, and our culinary team has been personally notified.</p>
    <div class="detail-box">
      <p><strong>Dietary Requirement:</strong> {{ $json.dietary_restriction_normalized }}</p>
      <p><strong>Table Assignment:</strong> {{ $json.table_number }}</p>
      <p><strong>Your Safety ID:</strong> <span class="safety-id">{{ $json.safety_id }}</span></p>
    </div>
    <p>Our Head Chef and kitchen management team have been alerted to your specific requirements. Every dish prepared for your table will be carefully reviewed against your dietary needs. Please do not hesitate to remind our service staff of your Safety ID upon arrival.</p>
    <p>If you have any questions or wish to speak with our culinary team directly, please reply to this email or call our events line.</p>
    <p>With warm regards,<br><strong>The Heirloom & Hearth Culinary Team</strong></p>
    <div class="footer">This confirmation was generated automatically. Safety ID {{ $json.safety_id }} is on file with our kitchen operations team.</div>
  </div>
</body>
</html>
```

---

## MEDIUM RISK BRANCH (Switch Output 2)

---

### Node 21: Wait Node — Medium Risk Delay

**Node Type:** `Wait`  
**Node Name:** `Wait — Medium Risk 2sec`

**Connect from:** Node 12 (Output 2 — MEDIUM)

**Parameters:**

| Field | Value |
|---|---|
| Resume | `After Time Interval` |
| Amount | `2` |
| Unit | `Seconds` |

---

### Node 22: Slack Node — Medium Risk Warning

**Node Type:** `Slack`  
**Node Name:** `Slack — MEDIUM RISK Kitchen Warning`

**Connect from:** Node 21

**Credentials:** `Slack – Heirloom Bot`

**Parameters:**

| Field | Value |
|---|---|
| Operation | `Send a Message` |
| Channel | `#kitchen-prep` |
| Message Type | `Block Kit` |

**Block Kit JSON:**
```json
[
  {
    "type": "header",
    "text": {
      "type": "plain_text",
      "text": "⚠️ Dietary Requirement Notice",
      "emoji": true
    }
  },
  {
    "type": "section",
    "fields": [
      {
        "type": "mrkdwn",
        "text": "*Guest:*\n{{ $json.full_name }}"
      },
      {
        "type": "mrkdwn",
        "text": "*Table:*\n{{ $json.table_number }}"
      }
    ]
  },
  {
    "type": "section",
    "fields": [
      {
        "type": "mrkdwn",
        "text": "*Restriction:*\n{{ $json.dietary_restriction_normalized }}"
      },
      {
        "type": "mrkdwn",
        "text": "*Category:*\n{{ $json.severity_normalized }}"
      }
    ]
  },
  {
    "type": "section",
    "text": {
      "type": "mrkdwn",
      "text": "*Safety ID:* `{{ $json.safety_id }}` | *Event:* {{ $json.event_name }}"
    }
  },
  {
    "type": "context",
    "elements": [
      {
        "type": "mrkdwn",
        "text": "Medium priority — please confirm prep protocols before service."
      }
    ]
  }
]
```

---

### Node 23: Google Sheets — Write to Master Manifest (MEDIUM)

**Node Type:** `Google Sheets`  
**Node Name:** `Sheets — Master Manifest (MEDIUM)`

**Connect from:** Node 22

**Credentials:** `Google OAuth2 – Heirloom`

**Parameters:** *(Same as Node 17 — Master Manifest)*

| Field | Value |
|---|---|
| Operation | `Append Row` |
| Spreadsheet | `Heirloom Catering Manifest` |
| Sheet | `Master Manifest` |

**Column Mappings:** *(Identical to Node 17)*

| Column | Expression |
|---|---|
| Timestamp | `{{ $json.formatted_timestamp }}` |
| Safety ID | `{{ $json.safety_id }}` |
| Full Name | `{{ $json.full_name }}` |
| Email | `{{ $json.email }}` |
| Phone | `{{ $json.phone }}` |
| Event Name | `{{ $json.event_name }}` |
| Table Number | `{{ $json.table_number }}` |
| Dietary Restriction | `{{ $json.dietary_restriction_normalized }}` |
| Severity | `{{ $json.severity_normalized }}` |
| Notes | `{{ $json.notes }}` |
| Status | `Pending` |

---

### Node 24: Set Node — Mark Confirmed (MEDIUM)

**Node Type:** `Set`  
**Node Name:** `Set — Status Confirmed (MEDIUM)`

**Connect from:** Node 23

**Parameters → Add Field:**

| Name | Type | Value |
|---|---|---|
| `status` | String | `Confirmed` |
| `branch` | String | `MEDIUM` |

---

### Node 25: Gmail Node — Guest Assurance Email (MEDIUM)

**Node Type:** `Gmail`  
**Node Name:** `Gmail — Assurance Email (MEDIUM)`

**Connect from:** Node 24

**Credentials:** `Google OAuth2 – Heirloom`

**Parameters:**

| Field | Value |
|---|---|
| To | `{{ $json.email }}` |
| Subject | `Your Dietary Safety Request Has Been Confirmed — Heirloom & Hearth` |
| Email Type | `HTML` |
| Body | *(Use the same HTML template as Node 20 — it is universal for all risk levels)* |

---

## LOW RISK BRANCH (Switch Output 3)

---

### Node 27: NoOp — Low Risk Acknowledged

**Node Type:** `NoOp`  
**Node Name:** `NoOp — Low Risk (No Alert Required)`

**Connect from:** Node 12 (Output 3 — LOW)

**What it does:** Documents the decision point that no urgent alert is required for low-risk preferences. The workflow continues to logging and email.

---

### Node 28: Google Sheets — Write to Master Manifest (LOW)

**Node Type:** `Google Sheets`  
**Node Name:** `Sheets — Master Manifest (LOW)`

**Connect from:** Node 27

**Credentials:** `Google OAuth2 – Heirloom`

**Parameters:** *(Identical column mappings to Node 17)*

| Field | Value |
|---|---|
| Operation | `Append Row` |
| Spreadsheet | `Heirloom Catering Manifest` |
| Sheet | `Master Manifest` |

---

### Node 29: Set Node — Mark Confirmed (LOW)

**Node Type:** `Set`  
**Node Name:** `Set — Status Confirmed (LOW)`

**Connect from:** Node 28

**Parameters → Add Field:**

| Name | Type | Value |
|---|---|---|
| `status` | String | `Confirmed` |
| `branch` | String | `LOW` |

---

### Node 30: Gmail Node — Guest Assurance Email (LOW)

**Node Type:** `Gmail`  
**Node Name:** `Gmail — Assurance Email (LOW)`

**Connect from:** Node 29

**Credentials:** `Google OAuth2 – Heirloom`

**Parameters:** *(Same as Node 20 and Node 25)*

| Field | Value |
|---|---|
| To | `{{ $json.email }}` |
| Subject | `Your Dietary Preference Has Been Noted — Heirloom & Hearth` |
| Email Type | `HTML` |

**Modified HTML Body for LOW (lighter tone):**
```html
<!DOCTYPE html>
<html>
<head>
  <style>
    body { font-family: Georgia, serif; color: #2c2c2c; background: #fafaf7; margin: 0; padding: 0; }
    .container { max-width: 600px; margin: 40px auto; padding: 40px; background: #fff; border-top: 4px solid #8b6914; }
    .logo { font-size: 22px; color: #8b6914; letter-spacing: 2px; margin-bottom: 30px; }
    .detail-box { background: #fdf8ee; border-left: 3px solid #8b6914; padding: 15px 20px; margin: 20px 0; }
    .safety-id { font-family: monospace; font-size: 16px; color: #8b6914; }
    .footer { font-size: 12px; color: #888; margin-top: 30px; border-top: 1px solid #eee; padding-top: 15px; }
  </style>
</head>
<body>
  <div class="container">
    <div class="logo">HEIRLOOM & HEARTH CATERING</div>
    <p>Dear {{ $json.full_name }},</p>
    <p>Thank you for sharing your dining preference with us for the <strong>{{ $json.event_name }}</strong>. We have noted your preference and our kitchen will do everything possible to accommodate you beautifully.</p>
    <div class="detail-box">
      <p><strong>Dining Preference:</strong> {{ $json.dietary_restriction_normalized }}</p>
      <p><strong>Table Assignment:</strong> {{ $json.table_number }}</p>
      <p><strong>Your Reference ID:</strong> <span class="safety-id">{{ $json.safety_id }}</span></p>
    </div>
    <p>We look forward to crafting a memorable culinary experience for you.</p>
    <p>Warm regards,<br><strong>The Heirloom & Hearth Culinary Team</strong></p>
    <div class="footer">Reference ID: {{ $json.safety_id }}</div>
  </div>
</body>
</html>
```

---

## POST-BRANCH CONVERGING NODES

*(These nodes process all branches after their respective email nodes)*

---

### Node 31: Code Node — Generate Kitchen Label

**Node Type:** `Code`  
**Node Name:** `Code — Generate Kitchen Label Text`

**Connect from:** Nodes 20, 25, and 30 (connect all three Gmail nodes into this one)

> **How to connect multiple inputs:** In n8n, you can connect multiple nodes to a single node's input. Right-click the input port of Node 31 and connect each Gmail output.

**What it does:** Generates a formatted plain-text label that kitchen staff can print and attach to the guest's table setting or meal ticket.

**Code:**
```javascript
const items = $input.all();

for (const item of items) {
  const riskEmoji = {
    'HIGH': '🚨',
    'MEDIUM': '⚠️',
    'LOW': '✅'
  }[item.json.risk_category] || '✅';

  const label = `
╔════════════════════════════════╗
║   HEIRLOOM & HEARTH CATERING  ║
╠════════════════════════════════╣
║ Safety ID: ${item.json.safety_id}
║ Guest: ${item.json.full_name}
║ Table: ${item.json.table_number}
║ Event: ${item.json.event_name}
╠════════════════════════════════╣
║ ${riskEmoji} DIETARY: ${item.json.dietary_restriction_normalized}
║ Severity: ${item.json.severity_normalized}
╠════════════════════════════════╣
║ Notes: ${item.json.notes || 'None'}
║ Processed: ${item.json.formatted_timestamp}
╚════════════════════════════════╝
  `.trim();

  item.json.kitchen_label = label;
}

return items;
```

---

### Node 32: Google Sheets — Audit Log Entry

**Node Type:** `Google Sheets`  
**Node Name:** `Sheets — Audit Log`

**Connect from:** Node 31

**Credentials:** `Google OAuth2 – Heirloom`

**Parameters:**

| Field | Value |
|---|---|
| Operation | `Append Row` |
| Spreadsheet | `Heirloom Catering Manifest` |
| Sheet | `Audit Log` |

**Column Mappings:**

| Column | Expression |
|---|---|
| Timestamp | `{{ $json.formatted_timestamp }}` |
| Event Type | `Submission Processed` |
| Safety ID | `{{ $json.safety_id }}` |
| Details | `Risk: {{ $json.risk_category }} | Branch: {{ $json.branch }} | Score: {{ $json.risk_score }}` |
| Status | `{{ $json.status }}` |
| Error Message | *(leave blank)* |

---

### Node 33: Set Node — Final Status

**Node Type:** `Set`  
**Node Name:** `Set — Workflow Final Status`

**Connect from:** Node 32

**Parameters → Add Fields:**

| Name | Type | Value |
|---|---|---|
| `workflow_status` | String | `Complete` |
| `workflow_name` | String | `Heirloom Dietary Safety — Main` |

---

### Node 34: NoOp — Workflow Complete

**Node Type:** `NoOp`  
**Node Name:** `NoOp — ✅ Workflow Complete`

**Connect from:** Node 33

**What it does:** Visual endpoint marking successful completion of the entire workflow. Useful during debugging to confirm execution reached the end.

---

## 6. Step-by-Step Node Build Guide — Error Handling Workflow

Create a **second, separate workflow** in n8n.

Click **+ New Workflow** → Name it: `Heirloom — Error Handler`

---

### Node E01: Error Trigger

**Node Type:** `Error Trigger`  
**Node Name:** `Error Trigger — Main Workflow Failure`

**What it does:** This node fires automatically whenever the **Main Workflow** encounters an unhandled node error.

**Important Setup Step:**
1. Go to the **Main Workflow** settings (gear icon or Settings tab)
2. Under **Error Workflow**, select `Heirloom — Error Handler`
3. Save the main workflow

From this point forward, any node failure in the main workflow will trigger this error workflow.

**Output of Error Trigger (auto-populated):**
```json
{
  "workflow": {
    "id": "...",
    "name": "Heirloom Dietary Safety — Main"
  },
  "execution": {
    "id": "...",
    "mode": "trigger",
    "retryOf": null,
    "error": {
      "name": "NodeOperationError",
      "message": "Could not send SMS: Invalid phone number",
      "node": {
        "name": "Twilio — Emergency SMS to Kitchen Manager",
        "type": "n8n-nodes-base.twilio"
      }
    },
    "lastNodeExecuted": "Twilio — Emergency SMS to Kitchen Manager"
  }
}
```

---

### Node E02: Set Node — Extract Error Details

**Node Type:** `Set`  
**Node Name:** `Set — Extract Error Details`

**Connect from:** Node E01

**Parameters → Add Fields:**

| Name | Type | Value |
|---|---|---|
| `error_workflow_name` | String | `{{ $json.workflow.name }}` |
| `error_node_name` | String | `{{ $json.execution.error.node.name }}` |
| `error_message` | String | `{{ $json.execution.error.message }}` |
| `error_timestamp` | String | `{{ $now.toISO() }}` |
| `execution_id` | String | `{{ $json.execution.id }}` |

---

### Node E03: Code Node — Format Error Report

**Node Type:** `Code`  
**Node Name:** `Code — Format Error Report`

**Connect from:** Node E02

**Code:**
```javascript
const items = $input.all();

for (const item of items) {
  item.json.error_report_html = `
    <h2 style="color: #cc0000;">⚠️ Workflow Error Alert — Heirloom & Hearth</h2>
    <table style="border-collapse:collapse; width:100%;">
      <tr><td style="padding:8px; font-weight:bold; background:#f5f5f5;">Workflow</td><td style="padding:8px;">${item.json.error_workflow_name}</td></tr>
      <tr><td style="padding:8px; font-weight:bold; background:#f5f5f5;">Failed Node</td><td style="padding:8px;">${item.json.error_node_name}</td></tr>
      <tr><td style="padding:8px; font-weight:bold; background:#f5f5f5;">Error</td><td style="padding:8px; color:#cc0000;">${item.json.error_message}</td></tr>
      <tr><td style="padding:8px; font-weight:bold; background:#f5f5f5;">Timestamp</td><td style="padding:8px;">${item.json.error_timestamp}</td></tr>
      <tr><td style="padding:8px; font-weight:bold; background:#f5f5f5;">Execution ID</td><td style="padding:8px;">${item.json.execution_id}</td></tr>
    </table>
    <p style="margin-top:20px; color:#555;">Please review the n8n execution log immediately and manually verify any pending dietary safety submissions.</p>
  `;

  item.json.error_report_plain = `
WORKFLOW ERROR — Heirloom & Hearth Catering
Workflow: ${item.json.error_workflow_name}
Failed Node: ${item.json.error_node_name}
Error: ${item.json.error_message}
Timestamp: ${item.json.error_timestamp}
Execution ID: ${item.json.execution_id}
  `.trim();
}

return items;
```

---

### Node E04: Gmail Node — Alert Operations Manager

**Node Type:** `Gmail`  
**Node Name:** `Gmail — Ops Manager Error Alert`

**Connect from:** Node E03

**Credentials:** `Google OAuth2 – Heirloom`

**Parameters:**

| Field | Value |
|---|---|
| To | `operations@heirloomhearth.com` (your ops manager email) |
| Subject | `🚨 SYSTEM ERROR — Heirloom Dietary Workflow Failure` |
| Email Type | `HTML` |
| Body | `{{ $json.error_report_html }}` |

---

### Node E05: Google Sheets — Log Failure to Audit Log

**Node Type:** `Google Sheets`  
**Node Name:** `Sheets — Audit Log (Error Entry)`

**Connect from:** Node E04

**Credentials:** `Google OAuth2 – Heirloom`

**Parameters:**

| Field | Value |
|---|---|
| Operation | `Append Row` |
| Spreadsheet | `Heirloom Catering Manifest` |
| Sheet | `Audit Log` |

**Column Mappings:**

| Column | Expression |
|---|---|
| Timestamp | `{{ $json.error_timestamp }}` |
| Event Type | `WORKFLOW ERROR` |
| Safety ID | `N/A` |
| Details | `Node: {{ $json.error_node_name }} | Workflow: {{ $json.error_workflow_name }}` |
| Status | `FAILED` |
| Error Message | `{{ $json.error_message }}` |

---

### Node E06: Slack Node — Ops Channel Alert

**Node Type:** `Slack`  
**Node Name:** `Slack — Ops Error Alert`

**Connect from:** Node E05

**Credentials:** `Slack – Heirloom Bot`

**Parameters:**

| Field | Value |
|---|---|
| Channel | `#ops-alerts` |
| Message | See below |

**Message Expression:**
```
🚨 *WORKFLOW FAILURE — Heirloom Dietary Safety System*
*Workflow:* {{ $json.error_workflow_name }}
*Failed Node:* {{ $json.error_node_name }}
*Error:* {{ $json.error_message }}
*Time:* {{ $json.error_timestamp }}
*Execution ID:* {{ $json.execution_id }}
> Please check n8n execution logs and verify all pending dietary submissions manually.
```

---

### Node E07: NoOp — Error Handled

**Node Type:** `NoOp`  
**Node Name:** `NoOp — Error Handled`

**Connect from:** Node E06

**What it does:** Marks the end of the error handling workflow.

---

## 7. Connection Map

Use this as a reference when wiring nodes in the n8n canvas.

```
[01 Webhook]
    │
    └──► [02 Set — Map Fields]
              │
              └──► [03 Code — Capitalize Name]
                        │
                        └──► [04 Code — Normalize Restrictions]
                                  │
                                  └──► [05 Code — Add Timestamp]
                                            │
                                            └──► [06 Code — Generate Safety ID]
                                                      │
                                                      └──► [07 Sheets — Duplicate Check]
                                                                │
                                                                └──► [08 IF — Is Duplicate?]
                                                                          │
                                                               ┌──────────┴──────────┐
                                                             TRUE                  FALSE
                                                               │                    │
                                                         [09 NoOp             [10 Merge]
                                                          Duplicate]               │
                                                                                   └──► [11 Code — Risk Score]
                                                                                             │
                                                                                             └──► [12 Switch — Router]
                                                                                                       │
                                                          ┌───────────────────────────┬───────────────┘
                                                        HIGH                        MEDIUM             LOW
                                                          │                            │                │
                                                   [13 Wait]                    [21 Wait]         [27 NoOp]
                                                       │                            │                │
                                                   [14 Slack HIGH]          [22 Slack MED]    [28 Sheets Master]
                                                       │                            │                │
                                                   [15 Twilio SMS]          [23 Sheets Master] [29 Set Confirmed]
                                                       │                            │                │
                                                   [16 Discord]             [24 Set Confirmed] [30 Gmail LOW]
                                                       │                            │                │
                                                   [17 Sheets Master]       [25 Gmail MED]           │
                                                       │                            │                │
                                                   [18 Sheets HighRisk]            │                │
                                                       │                            │                │
                                                   [19 Set Confirmed]               │                │
                                                       │                            │                │
                                                   [20 Gmail HIGH]                  │                │
                                                       │                            │                │
                                                       └────────────────────────────┴────────────────┘
                                                                                    │
                                                                             [31 Code — Kitchen Label]
                                                                                    │
                                                                             [32 Sheets — Audit Log]
                                                                                    │
                                                                             [33 Set — Final Status]
                                                                                    │
                                                                             [34 NoOp — Complete]
```

---

## 8. Testing Scenarios

Use these three test scenarios to validate your workflow. In n8n, open Node 01 (Webhook Trigger), click **Test Step**, and paste the JSON body below.

Alternatively, use a tool like Postman, Insomnia, or `curl` to POST to your webhook URL.

---

### Test 1 — HIGH RISK: Peanut Anaphylactic

**POST to:** `https://your-n8n-instance.com/webhook/heirloom-dietary`

**Body:**
```json
{
  "full_name": "sarah smith",
  "email": "sarah.smith@testguest.com",
  "phone": "+14155551234",
  "event_name": "thornton wedding gala",
  "table_number": "12",
  "dietary_restriction": "Peanut Severe",
  "severity": "Anaphylactic",
  "notes": "Carries EpiPen. Must not share cooking surfaces or utensils. Confirmed severe anaphylactic reaction history."
}
```

**Expected Results:**

| Check | Expected Output |
|---|---|
| Name Capitalized | `Sarah Smith` |
| Restriction Normalized | `Peanut Allergy — Severe` |
| Severity Normalized | `Anaphylactic` |
| Safety ID Format | `HH-20260425-XXXXXX` |
| Risk Score | `100` |
| Risk Category | `HIGH` |
| Slack #kitchen-alerts | 🚨 HIGH PRIORITY message received |
| Twilio SMS | SMS sent to kitchen manager |
| Discord | Alert posted |
| Master Manifest Row | Row appended with all fields |
| High Risk Queue Row | Row appended |
| Guest Email | Luxury assurance email received |
| Audit Log | Row with `Submission Processed` |

---

### Test 2 — MEDIUM RISK: Celiac Disease

**Body:**
```json
{
  "full_name": "JAMES CHEN",
  "email": "james.chen@testguest.com",
  "phone": "+16505559876",
  "event_name": "morrison corporate dinner",
  "table_number": "7",
  "dietary_restriction": "Celiac",
  "severity": "Celiac",
  "notes": "Diagnosed Celiac. Strict gluten-free required. Even cross-contamination is harmful."
}
```

**Expected Results:**

| Check | Expected Output |
|---|---|
| Name Capitalized | `James Chen` |
| Restriction Normalized | `Celiac Disease` |
| Risk Score | `60` |
| Risk Category | `MEDIUM` |
| Slack #kitchen-prep | ⚠️ Warning message received |
| Twilio SMS | NOT sent |
| Discord | NOT sent |
| Master Manifest Row | Row appended |
| High Risk Queue Row | NOT written |
| Guest Email | Assurance email received |

---

### Test 3 — LOW RISK: Vegetarian Preference

**Body:**
```json
{
  "full_name": "emily rodriguez",
  "email": "emily.rodriguez@testguest.com",
  "phone": "+17185554321",
  "event_name": "Hartley Anniversary Celebration",
  "table_number": "3",
  "dietary_restriction": "Vegetarian",
  "severity": "No Preference",
  "notes": "No meat please. Fish and seafood are fine."
}
```

**Expected Results:**

| Check | Expected Output |
|---|---|
| Restriction Normalized | `Vegetarian` |
| Risk Score | `20` |
| Risk Category | `LOW` |
| Slack | NOT sent |
| Twilio SMS | NOT sent |
| Discord | NOT sent |
| Master Manifest Row | Row appended |
| High Risk Queue Row | NOT written |
| Guest Email | Preference note email received |

---

### Test 4 — Duplicate Protection Test

**Submit Test 1 again (same email):**

```json
{
  "full_name": "Sarah Smith",
  "email": "sarah.smith@testguest.com",
  "phone": "+14155551234",
  "event_name": "thornton wedding gala",
  "table_number": "12",
  "dietary_restriction": "Peanut Severe",
  "severity": "Anaphylactic",
  "notes": "Duplicate submission test."
}
```

**Expected Result:**

| Check | Expected Output |
|---|---|
| IF Duplicate | Routes to `TRUE` branch |
| NoOp Reached | `NoOp — Duplicate Blocked` executed |
| No Slack Alert | Correct — not re-sent |
| No new Sheet Row | Correct — not duplicated |

---

## 9. Execution Proof Guide

After successfully completing all three test scenarios, capture the following as proof of working automation for client delivery.

### Screenshot 1 — Full Canvas View

- Zoom out to show the entire workflow
- All nodes visible with connecting arrows
- Capture as PNG at full resolution

### Screenshot 2 — Successful Execution

- Open **Executions** tab in n8n
- Click the most recent successful run
- Show green checkmarks on all nodes
- Expand Node 12 (Switch) to show the routing decision

### Screenshot 3 — Google Sheets — Master Manifest

- Open `Heirloom Catering Manifest` → `Master Manifest` tab
- Show all three test rows (HIGH, MEDIUM, LOW)
- All columns populated

### Screenshot 4 — Google Sheets — High Risk Queue

- Show `High Risk Queue` tab
- One row (Sarah Smith — Peanut Severe)
- Status column showing `URGENT — Requires Kitchen Confirmation`

### Screenshot 5 — Slack Alert

- Open `#kitchen-alerts` Slack channel
- Show the Block Kit formatted HIGH PRIORITY message
- Include Safety ID visible in the message

### Screenshot 6 — Gmail Confirmation

- Show the received confirmation email in the test guest inbox
- Subject line visible: `Your Dietary Safety Request Has Been Confirmed`
- Safety ID visible in the email body

### Screenshot 7 — Node Input/Output Panel

- Click Node 11 (Risk Score Calculator)
- Show the Input panel on the left and Output panel on the right
- Risk score and category visible in the output

### Screenshot 8 — Error Workflow (Optional)

- Temporarily disconnect a node to force an error
- Show the error email received by the ops manager
- Show the Audit Log failure row in Google Sheets

---

## 10. Final Deliverables & Activation

### 10.1 Export the Workflow as JSON

1. Open the **Main Workflow** canvas
2. Click the **three-dot menu** (top right) → **Export**
3. Choose **Download as JSON**
4. Save file as: `heirloom-dietary-safety-main.json`
5. Repeat for the Error Handling Workflow → Save as: `heirloom-error-handler.json`

> Store both JSON files in a shared Google Drive folder named `Heirloom n8n Workflows` for version control and backup.

---

### 10.2 Activate the Workflow

1. In n8n, open the Main Workflow
2. Toggle the **Active** switch (top right of canvas) to **ON**
3. The Webhook URL is now live and listening
4. Confirm activation: the toggle turns blue/green
5. Repeat for the Error Handling Workflow

> ⚠️ **Important:** The Error Handling Workflow must be active BEFORE the Main Workflow to catch any startup errors.

---

### 10.3 Sharing with the Client

**Option A — n8n Cloud Sharing**
1. Go to workflow settings → **Share**
2. Enter the client's n8n account email
3. Set permission level to `Editor` or `Viewer`

**Option B — JSON File Delivery**
1. Share both exported JSON files via email or Drive
2. Client installs n8n (cloud or self-hosted)
3. Client imports workflows via **Import from File**
4. Client adds their own credentials

**Option C — Self-Hosted Instance Handoff**
1. Export workflow JSON files
2. Provide the client with n8n Docker setup instructions
3. Walk through credential setup in a shared screen session
4. Transfer the Google Sheets template (share the spreadsheet with client's Google account)

---

### 10.4 Post-Activation Checklist

| Item | Done? |
|---|---|
| Main workflow active | ☐ |
| Error workflow active | ☐ |
| Error workflow set in main workflow settings | ☐ |
| Slack bot added to #kitchen-alerts channel | ☐ |
| Slack bot added to #kitchen-prep channel | ☐ |
| Slack bot added to #ops-alerts channel | ☐ |
| Twilio number verified | ☐ |
| Kitchen Manager phone number stored | ☐ |
| Gmail authorized | ☐ |
| Google Sheet created with 3 tabs + correct headers | ☐ |
| Test 1 (HIGH RISK) passed | ☐ |
| Test 2 (MEDIUM RISK) passed | ☐ |
| Test 3 (LOW RISK) passed | ☐ |
| Duplicate protection test passed | ☐ |
| Error workflow tested | ☐ |
| Client handed credentials setup guide | ☐ |
| Workflows exported and backed up | ☐ |

---

## Appendix A — Node Quick Reference

| # | Node Name | Type | Branch |
|---|---|---|---|
| 01 | Webhook — Guest Submission | Webhook | All |
| 02 | Set — Map Incoming Fields | Set | All |
| 03 | Code — Capitalize Name | Code | All |
| 04 | Code — Normalize Restrictions | Code | All |
| 05 | Code — Add Timestamp | Code | All |
| 06 | Code — Generate Safety ID | Code | All |
| 07 | Sheets — Duplicate Check Lookup | Google Sheets | All |
| 08 | IF — Is Duplicate? | IF | All |
| 09 | NoOp — Duplicate Blocked | NoOp | Duplicate |
| 10 | Merge — Continue After Duplicate Check | Merge | All |
| 11 | Code — Risk Score Calculator | Code | All |
| 12 | Switch — Risk Category Router | Switch | All |
| 13 | Wait — High Risk 2sec | Wait | HIGH |
| 14 | Slack — HIGH RISK Kitchen Alert | Slack | HIGH |
| 15 | Twilio — Emergency SMS | Twilio | HIGH |
| 16 | Discord — Kitchen Alert | Discord | HIGH |
| 17 | Sheets — Master Manifest (HIGH) | Google Sheets | HIGH |
| 18 | Sheets — High Risk Queue | Google Sheets | HIGH |
| 19 | Set — Status Confirmed (HIGH) | Set | HIGH |
| 20 | Gmail — Assurance Email (HIGH) | Gmail | HIGH |
| 21 | Wait — Medium Risk 2sec | Wait | MEDIUM |
| 22 | Slack — MEDIUM RISK Kitchen Warning | Slack | MEDIUM |
| 23 | Sheets — Master Manifest (MEDIUM) | Google Sheets | MEDIUM |
| 24 | Set — Status Confirmed (MEDIUM) | Set | MEDIUM |
| 25 | Gmail — Assurance Email (MEDIUM) | Gmail | MEDIUM |
| 27 | NoOp — Low Risk | NoOp | LOW |
| 28 | Sheets — Master Manifest (LOW) | Google Sheets | LOW |
| 29 | Set — Status Confirmed (LOW) | Set | LOW |
| 30 | Gmail — Assurance Email (LOW) | Gmail | LOW |
| 31 | Code — Generate Kitchen Label | Code | All |
| 32 | Sheets — Audit Log | Google Sheets | All |
| 33 | Set — Workflow Final Status | Set | All |
| 34 | NoOp — ✅ Workflow Complete | NoOp | All |
| E01 | Error Trigger | Error Trigger | Error WF |
| E02 | Set — Extract Error Details | Set | Error WF |
| E03 | Code — Format Error Report | Code | Error WF |
| E04 | Gmail — Ops Manager Error Alert | Gmail | Error WF |
| E05 | Sheets — Audit Log (Error Entry) | Google Sheets | Error WF |
| E06 | Slack — Ops Error Alert | Slack | Error WF |
| E07 | NoOp — Error Handled | NoOp | Error WF |

---

## Appendix B — Common Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| Webhook not receiving data | Workflow not active | Toggle Active ON |
| Google Sheets node fails | OAuth expired | Re-authenticate credential |
| Slack message not appearing | Bot not in channel | `/invite @HeirloomSafetyBot` in channel |
| SMS not delivering | Invalid Twilio number format | Ensure E.164 format: `+1XXXXXXXXXX` |
| Duplicate not detected | Email column name mismatch | Verify column header is exactly `Email` |
| Switch not routing correctly | Risk category string case mismatch | Ensure Switch rule values match exactly: `HIGH`, `MEDIUM`, `LOW` |
| Error workflow not triggering | Error workflow not set in main settings | Go to main workflow → Settings → Error Workflow → Select error handler |
| All nodes green but no email received | Gmail auth scope missing | Re-auth with `gmail.send` scope enabled |

---

*End of Implementation Guide*  
*Heirloom & Hearth Catering — Multi-Channel Dietary Safety & Logistics Sync*  
*Built with n8n v1.x | Last Updated: April 2026*

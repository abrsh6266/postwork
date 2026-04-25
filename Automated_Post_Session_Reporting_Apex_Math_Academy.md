# Automated Post-Session Reporting & Alert System for Apex Math Academy

> **Document Type:** Step-by-Step n8n Workflow Implementation Guide
> **Version:** 1.0.0
> **Last Updated:** 2026-04-24
> **Target Platform:** n8n (Self-Hosted or n8n Cloud)
> **Difficulty Level:** Beginner-Friendly · Production-Quality

---

## Table of Contents

1. [System Architecture Overview](#system-architecture-overview)
2. [Prerequisites & Setup](#prerequisites--setup)
3. [Layer 1 — Google Sheets Trigger](#layer-1--google-sheets-trigger)
4. [Layer 2 — Data Normalization](#layer-2--data-normalization)
5. [Layer 3 — Data Validation](#layer-3--data-validation)
6. [Layer 4 — Airtable Logging](#layer-4--airtable-logging)
7. [Layer 5 — Confidence Classification](#layer-5--confidence-classification)
8. [Layer 6 — Low Confidence Branch (Slack Alerts)](#layer-6--low-confidence-branch-slack-alerts)
9. [Layer 7 — High Confidence Branch (Gmail)](#layer-7--high-confidence-branch-gmail)
10. [Layer 8 — Error Handling System](#layer-8--error-handling-system)
11. [Layer 9 — Logging & Observability](#layer-9--logging--observability)
12. [Full Connection Map](#full-connection-map)
13. [Testing Section](#testing-section)
14. [n8n Workflow JSON Export](#n8n-workflow-json-export)
15. [Airtable Schema](#airtable-schema)
16. [Proof of Execution Guide](#proof-of-execution-guide)
17. [Acceptance Checklist](#acceptance-checklist)

---

## System Architecture Overview

```
[Google Sheets Trigger]
         │
         ▼
[Set: Normalize Fields]
         │
         ▼
[IF: Name Exists?]──No──► [Set: Error Payload] ──► [Slack: Error Alert]
         │ Yes
         ▼
[IF: Email Valid?]──No───► [Set: Error Payload] ──► [Slack: Error Alert]
         │ Yes
         ▼
[IF: Rating 1–5?]──No────► [Set: Error Payload] ──► [Slack: Error Alert]
         │ Yes
         ▼
[Function: Clean Summary]
         │
         ▼
[Set: Add Timestamp]
         │
         ▼
[Function: Attach Metadata]
         │
         ▼
[Airtable: Create Record]──Fail─► [Set: Airtable Error] ──► [Slack: Error Alert]
         │ Success
         ▼
[IF: Rating <= 2?]
    │           │
  Yes (Low)   No (High)
    │           │
    ▼           ▼
[Set: Format  [Function: Build
 Slack Msg]    HTML Email]
    │           │
    ▼           ▼
[Function:   [Gmail: Send
 Add Urgency]  Email]
    │
    ▼
[Slack: Send
 Founder Alert]
```

---

## Prerequisites & Setup

### Required Accounts & Credentials

| Service | Purpose | Where to Get |
|---|---|---|
| n8n | Workflow automation | n8n.io or self-hosted |
| Google Sheets | Form data source | Google Workspace |
| Airtable | Lesson record database | airtable.com |
| Slack | Internal alerts | slack.com |
| Gmail | Parent email notifications | Google Workspace |

### n8n Credential Setup

Before building the workflow, register all credentials in n8n:

#### 1. Google Sheets OAuth2

1. Open n8n → **Settings** → **Credentials** → **Add Credential**
2. Search for **Google Sheets OAuth2 API**
3. Click **Connect my account** and sign in with your Google account
4. Grant access to Sheets
5. Name it: `Google Sheets - Apex Math`

#### 2. Airtable API Key

1. Go to [airtable.com/account](https://airtable.com/account) → API section
2. Generate a Personal Access Token with scopes: `data.records:read`, `data.records:write`
3. In n8n → **Settings** → **Credentials** → **Add Credential** → **Airtable API**
4. Paste your token
5. Name it: `Airtable - Apex Math`

#### 3. Slack OAuth

1. Go to [api.slack.com/apps](https://api.slack.com/apps) → Create New App
2. Enable **Bot Token Scopes**: `chat:write`, `chat:write.public`
3. Install app to your workspace
4. Copy the **Bot User OAuth Token** (starts with `xoxb-`)
5. In n8n → **Credentials** → **Slack OAuth2 API**
6. Paste the token
7. Name it: `Slack - Apex Math`

#### 4. Gmail OAuth2

1. In n8n → **Credentials** → **Gmail OAuth2**
2. Click **Connect my account** → Sign in with the sender Gmail account
3. Name it: `Gmail - Apex Math`

### Google Sheet Setup

Create a Google Sheet called **Apex Math Lesson Reports** with the following columns in Row 1:

| A | B | C | D | E |
|---|---|---|---|---|
| Student Name | Parent Email | Lesson Summary | Confidence Rating | Timestamp |

> **Tip:** If you are using a Google Form, link it to this sheet — responses will populate automatically.

---

## Layer 1 — Google Sheets Trigger

### Node 1: Google Sheets Trigger

This is the entry point of the entire workflow. It polls the Google Sheet every minute and fires whenever a new row is added.

**Node Type:** `Google Sheets Trigger`
**Node Name:** `Trigger: New Lesson Report`

#### Configuration

| Setting | Value |
|---|---|
| Credential | `Google Sheets - Apex Math` |
| Trigger On | `Row Added` |
| Spreadsheet | Select `Apex Math Lesson Reports` |
| Sheet | `Sheet1` |
| Poll Times | Every `1` minute |

#### How to Add This Node

1. Click the **+** button on the n8n canvas
2. Search for `Google Sheets Trigger`
3. Click to add it
4. In the right panel, select `Google Sheets - Apex Math` as the credential
5. Under **Trigger On**, select `Row Added`
6. Click the Spreadsheet field → navigate and select `Apex Math Lesson Reports`
7. Select `Sheet1`
8. Under **Poll Times** → Every → `1 Minute`
9. Click **Save**

#### Sample Output Data

When this node fires, it produces the following data structure:

```json
{
  "row_number": 2,
  "Student Name": "Jamie Torres",
  "Parent Email": "torres.family@gmail.com",
  "Lesson Summary": "Worked on long division. Jamie struggled with remainders but improved by end of session.",
  "Confidence Rating": "2",
  "Timestamp": "2026-04-24T09:15:00Z"
}
```

> **Note:** All values from Google Sheets come in as strings, even numeric ones like `Confidence Rating`. This is why we normalize in the next node.

---

## Layer 2 — Data Normalization

### Node 2: Set — Normalize Incoming Data

This node standardizes the field names and types from the trigger output. It ensures the rest of the workflow uses consistent, predictable variable names regardless of how the sheet columns are named.

**Node Type:** `Set`
**Node Name:** `Set: Normalize Fields`

#### Configuration

Set the following fields by clicking **Add Value** for each:

| Field Name | Type | Value / Expression |
|---|---|---|
| `studentName` | String | `={{ $json["Student Name"] }}` |
| `parentEmail` | String | `={{ $json["Parent Email"] }}` |
| `lessonSummary` | String | `={{ $json["Lesson Summary"] }}` |
| `confidenceRating` | Number | `={{ Number($json["Confidence Rating"]) }}` |
| `rawTimestamp` | String | `={{ $json["Timestamp"] || $now }}` |
| `rowNumber` | Number | `={{ $json["row_number"] }}` |

**Options:**
- Set **Include Other Input Fields** to `false` (we only want our normalized fields going forward)

#### How to Add This Node

1. Click the **+** connector coming from `Trigger: New Lesson Report`
2. Search for `Set`
3. Add it and name it `Set: Normalize Fields`
4. Click **Add Value** → **String** for each string field above
5. For `confidenceRating`, choose **Number** and use the `Number()` expression to cast the string
6. Set **Include Other Input Fields** to `false`
7. Click **Save**

#### Sample Output Data

```json
{
  "studentName": "Jamie Torres",
  "parentEmail": "torres.family@gmail.com",
  "lessonSummary": "Worked on long division. Jamie struggled with remainders but improved by end of session.",
  "confidenceRating": 2,
  "rawTimestamp": "2026-04-24T09:15:00Z",
  "rowNumber": 2
}
```

---

## Layer 3 — Data Validation

This layer contains four sequential validation nodes. Each checks one condition. If the condition fails, data is routed to the Error branch. If it passes, data continues to the next check.

---

### Node 3: IF — Check Student Name Exists

**Node Type:** `IF`
**Node Name:** `IF: Name Exists?`

#### Configuration

| Setting | Value |
|---|---|
| Condition Type | String |
| Value 1 | `={{ $json.studentName }}` |
| Operation | `is not empty` |

#### Connections

- **True (name present):** → connects to `IF: Email Valid?`
- **False (name missing):** → connects to `Set: Error — Missing Name`

#### How to Add This Node

1. Connect from `Set: Normalize Fields`
2. Search for `IF` and add it
3. Name it `IF: Name Exists?`
4. Under **Conditions** → **Add Condition** → **String**
5. Value 1: `={{ $json.studentName }}`
6. Operation: `Is Not Empty`
7. Click **Save**

---

### Node 4: IF — Validate Email Format

**Node Type:** `IF`
**Node Name:** `IF: Email Valid?`

#### Configuration

| Setting | Value |
|---|---|
| Condition Type | String |
|Value 1 | `={{ $json.parentEmail }}` |
| Operation | `Regex` |
| Value 2 (Regex) | `^[^\s@]+@[^\s@]+\.[^\s@]+$` |

#### Connections

- **True (valid email):** → connects to `IF: Rating In Range?`
- **False (invalid email):** → connects to `Set: Error — Bad Email`

#### How to Add This Node

1. Connect from the **True** output of `IF: Name Exists?`
2. Add an `IF` node named `IF: Email Valid?`
3. Add Condition → String
4. Value 1: `={{ $json.parentEmail }}`
5. Operation: select `Regex`
6. Value 2: `^[^\s@]+@[^\s@]+\.[^\s@]+$`
7. Click **Save**

---

### Node 5: IF — Validate Rating Range

**Node Type:** `IF`
**Node Name:** `IF: Rating In Range?`

#### Configuration (Two conditions, both must be true)

| Setting | Value |
|---|---|
| Combine | `ALL` (AND logic) |
| Condition 1 Type | Number |
| Condition 1 Value 1 | `={{ $json.confidenceRating }}` |
| Condition 1 Operation | `≥` |
| Condition 1 Value 2 | `1` |
| Condition 2 Type | Number |
| Condition 2 Value 1 | `={{ $json.confidenceRating }}` |
| Condition 2 Operation | `≤` |
| Condition 2 Value 2 | `5` |

#### Connections

- **True (1–5):** → connects to `Function: Clean Summary`
- **False (out of range):** → connects to `Set: Error — Bad Rating`

#### How to Add This Node

1. Connect from **True** output of `IF: Email Valid?`
2. Add `IF` node named `IF: Rating In Range?`
3. Click **Add Condition** twice (one for ≥ 1, one for ≤ 5)
4. Set **Combine** at the top of the conditions block to `ALL`
5. Configure each condition as specified above
6. Click **Save**

---

### Node 6: Function — Clean Lesson Summary

**Node Type:** `Code` (previously called Function node)
**Node Name:** `Function: Clean Summary`

This node sanitizes the lesson summary text: trims whitespace, removes double spaces, strips dangerous characters, and enforces a maximum length of 2000 characters.

#### Configuration

**Language:** JavaScript

```javascript
// Get the lesson summary from previous node
let summary = $input.first().json.lessonSummary || "";

// Step 1: Trim leading and trailing whitespace
summary = summary.trim();

// Step 2: Collapse multiple spaces into single spaces
summary = summary.replace(/\s+/g, ' ');

// Step 3: Remove any HTML tags (prevent injection in emails)
summary = summary.replace(/<[^>]*>/g, '');

// Step 4: Remove non-printable characters
summary = summary.replace(/[^\x20-\x7E\n]/g, '');

// Step 5: Enforce max length of 2000 characters
if (summary.length > 2000) {
  summary = summary.substring(0, 1997) + '...';
}

// Step 6: If summary is now empty, flag it
if (summary.length === 0) {
  summary = '[No lesson summary provided]';
}

// Return the entire payload with the cleaned summary
return {
  json: {
    ...$input.first().json,
    lessonSummary: summary,
    summaryLength: summary.length,
    summaryCleaned: true
  }
};
```

#### Connections

- **Output:** → connects to `Set: Add Timestamp`

#### How to Add This Node

1. Connect from **True** output of `IF: Rating In Range?`
2. Add a `Code` node named `Function: Clean Summary`
3. Set language to `JavaScript`
4. Paste the code above
5. Click **Save**

---

## Layer 4 — Airtable Logging

Before logging, we add timestamps and metadata.

### Node 7: Set — Add Timestamp

**Node Type:** `Set`
**Node Name:** `Set: Add Timestamp`

#### Configuration

| Field Name | Type | Value |
|---|---|---|
| `submissionTimestamp` | String | `={{ $now.toISO() }}` |
| `submissionDate` | String | `={{ $now.toFormat('yyyy-MM-dd') }}` |
| `submissionTime` | String | `={{ $now.toFormat('HH:mm:ss') }}` |

**Options:** Set **Include Other Input Fields** to `true` (keep all previous fields)

#### How to Add This Node

1. Connect from `Function: Clean Summary`
2. Add `Set` node named `Set: Add Timestamp`
3. Add each field with **Include Other Input Fields** = `true`
4. Click **Save**

---

### Node 8: Function — Attach Execution Metadata

**Node Type:** `Code`
**Node Name:** `Function: Attach Metadata`

```javascript
const item = $input.first().json;

return {
  json: {
    ...item,
    executionId: $execution.id,
    workflowId: $workflow.id,
    workflowName: $workflow.name,
    processingEnvironment: 'production',
    dataSchemaVersion: '1.0'
  }
};
```

#### Connections

- **Output:** → connects to `Airtable: Create Lesson Record`

---

### Node 9: Airtable — Create Lesson Record

**Node Type:** `Airtable`
**Node Name:** `Airtable: Create Lesson Record`

#### Airtable Base Setup

Before configuring this node, set up your Airtable base:

1. Go to [airtable.com](https://airtable.com) and create a new Base called **Apex Math Academy**
2. Rename the default table to **Lesson Reports**
3. Create the following fields:

| Field Name | Airtable Field Type | Notes |
|---|---|---|
| `Student Name` | Single line text | Primary field |
| `Parent Email` | Email | |
| `Lesson Summary` | Long text | |
| `Confidence Rating` | Number | Precision: Integer |
| `Submission Timestamp` | Date & Time | Include time |
| `Submission Date` | Single line text | |
| `Execution ID` | Single line text | For audit trail |
| `Processing Status` | Single select | Options: Pending, Processed, Error |

4. Copy the **Base ID** from the URL: `https://airtable.com/appXXXXXXXXXXXXXX/...`

#### Node Configuration

| Setting | Value |
|---|---|
| Credential | `Airtable - Apex Math` |
| Operation | `Create` |
| Base ID | `appXXXXXXXXXXXXXX` (your base ID) |
| Table | `Lesson Reports` |

**Fields to Map:**

Click **Add Field** for each:

| Airtable Field | Value / Expression |
|---|---|
| Student Name | `={{ $json.studentName }}` |
| Parent Email | `={{ $json.parentEmail }}` |
| Lesson Summary | `={{ $json.lessonSummary }}` |
| Confidence Rating | `={{ $json.confidenceRating }}` |
| Submission Timestamp | `={{ $json.submissionTimestamp }}` |
| Submission Date | `={{ $json.submissionDate }}` |
| Execution ID | `={{ $json.executionId }}` |
| Processing Status | `Pending` |

#### Error Handling on This Node

1. Click the Airtable node settings (⚙️ icon)
2. Under **On Error**, select `Continue (using error output)`
3. This routes Airtable failures to the error branch automatically

#### Connections

- **Success output:** → connects to `IF: Rating Low?`
- **Error output:** → connects to `Set: Airtable Error Payload`

#### How to Add This Node

1. Connect from `Function: Attach Metadata`
2. Add `Airtable` node named `Airtable: Create Lesson Record`
3. Select `Create` operation
4. Enter your Base ID and Table name
5. Map all fields as specified
6. Configure error handling
7. Click **Save**

---

## Layer 5 — Confidence Classification

### Node 10: IF — Rating Low Confidence?

**Node Type:** `IF`
**Node Name:** `IF: Rating Low?`

This is the branching point that separates urgent (low confidence) from positive (high confidence) outcomes.

#### Configuration

| Setting | Value |
|---|---|
| Condition Type | Number |
| Value 1 | `={{ $json.confidenceRating }}` |
| Operation | `≤` |
| Value 2 | `2` |

#### Connections

- **True (rating is 1 or 2):** → connects to `Set: Format Slack Alert`
- **False (rating is 3, 4, or 5):** → connects to `Function: Build HTML Email`

#### How to Add This Node

1. Connect from the **Success** output of `Airtable: Create Lesson Record`
2. Add `IF` node named `IF: Rating Low?`
3. Configure the number comparison as above
4. Click **Save**

---

## Layer 6 — Low Confidence Branch (Slack Alerts)

This branch handles students with a confidence rating of 1 or 2. It sends an urgent alert to a Slack channel monitored by the Apex Math founder.

---

### Node 11: Set — Format Slack Alert Message

**Node Type:** `Set`
**Node Name:** `Set: Format Slack Alert`

This node prepares the structured data needed for the Slack message.

#### Configuration

| Field Name | Type | Value |
|---|---|---|
| `alertStudentName` | String | `={{ $json.studentName }}` |
| `alertRating` | Number | `={{ $json.confidenceRating }}` |
| `alertSummary` | String | `={{ $json.lessonSummary }}` |
| `alertDate` | String | `={{ $json.submissionDate }}` |
| `alertTime` | String | `={{ $json.submissionTime }}` |
| `alertRatingLabel` | String | `={{ $json.confidenceRating === 1 ? 'Critical (1/5)' : 'Very Low (2/5)' }}` |

**Options:** Include Other Input Fields = `true`

---

### Node 12: Function — Add Urgency Flag

**Node Type:** `Code`
**Node Name:** `Function: Add Urgency Flag`

This node builds the final Slack message text with urgency indicators.

```javascript
const item = $input.first().json;

// Determine urgency level
const rating = item.alertRating;
let urgencyEmoji = '🟡';
let urgencyLevel = 'MEDIUM';

if (rating === 1) {
  urgencyEmoji = '🔴';
  urgencyLevel = 'CRITICAL';
} else if (rating === 2) {
  urgencyEmoji = '🟠';
  urgencyLevel = 'HIGH';
}

// Build the Slack message
const slackMessage = `${urgencyEmoji} *TUTOR ALERT — ${urgencyLevel} PRIORITY* ${urgencyEmoji}

*Student:* ${item.alertStudentName}
*Confidence Rating:* ${item.alertRatingLabel}
*Session Date:* ${item.alertDate} at ${item.alertTime}

*Lesson Summary:*
> ${item.alertSummary}

*Recommended Action:*
• Contact parent within 24 hours
• Review upcoming lesson plan
• Consider scheduling additional session

_This alert was generated automatically by the Apex Math Reporting System._`;

return {
  json: {
    ...item,
    urgencyLevel,
    urgencyEmoji,
    slackMessage
  }
};
```

---

### Node 13: Slack — Send Founder Alert

**Node Type:** `Slack`
**Node Name:** `Slack: Send Founder Alert`

#### Configuration

| Setting | Value |
|---|---|
| Credential | `Slack - Apex Math` |
| Resource | `Message` |
| Operation | `Send` |
| Channel | `#founder-alerts` |
| Text | `={{ $json.slackMessage }}` |

**Additional Options:**

| Option | Value |
|---|---|
| Username | `Apex Math Bot` |
| Icon Emoji | `:warning:` |

#### How to Add This Node

1. Connect from `Function: Add Urgency Flag`
2. Add `Slack` node named `Slack: Send Founder Alert`
3. Select your `Slack - Apex Math` credential
4. Set Resource to `Message`, Operation to `Send`
5. Channel: `#founder-alerts`
6. Text field: `={{ $json.slackMessage }}`
7. Expand **Additional Fields** and set Username and Icon Emoji
8. Click **Save**

#### Sample Slack Message Output

```
🔴 TUTOR ALERT — CRITICAL PRIORITY 🔴

*Student:* Jamie Torres
*Confidence Rating:* Critical (1/5)
*Session Date:* 2026-04-24 at 09:15:00

*Lesson Summary:*
> Worked on long division. Jamie struggled with remainders but improved by end of session.

*Recommended Action:*
• Contact parent within 24 hours
• Review upcoming lesson plan
• Consider scheduling additional session

_This alert was generated automatically by the Apex Math Reporting System._
```

---

## Layer 7 — High Confidence Branch (Gmail)

This branch handles students with a confidence rating of 3, 4, or 5. It sends a warm, HTML-formatted progress email to the parent.

---

### Node 14: Function — Build HTML Email

**Node Type:** `Code`
**Node Name:** `Function: Build HTML Email`

This node assembles a fully styled HTML email body.

```javascript
const item = $input.first().json;

const rating = item.confidenceRating;
const stars = '⭐'.repeat(rating);
const emptyStars = '☆'.repeat(5 - rating);

// Determine tone based on rating
let toneMessage = '';
if (rating === 3) {
  toneMessage = 'making solid progress and continuing to build their skills';
} else if (rating === 4) {
  toneMessage = 'doing really well and showing great understanding of the material';
} else if (rating === 5) {
  toneMessage = 'excelling brilliantly — a truly outstanding session!';
}

const htmlBody = `
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Session Progress Update</title>
  <style>
    body {
      font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
      background-color: #f4f7fb;
      margin: 0;
      padding: 0;
      color: #333333;
    }
    .email-wrapper {
      max-width: 600px;
      margin: 30px auto;
      background-color: #ffffff;
      border-radius: 12px;
      overflow: hidden;
      box-shadow: 0 4px 20px rgba(0,0,0,0.08);
    }
    .header {
      background: linear-gradient(135deg, #1e3a5f 0%, #2d6a9f 100%);
      padding: 32px 40px;
      text-align: center;
    }
    .header h1 {
      color: #ffffff;
      font-size: 24px;
      margin: 0;
      letter-spacing: 1px;
    }
    .header p {
      color: #a8d4f5;
      margin: 6px 0 0;
      font-size: 14px;
    }
    .body {
      padding: 36px 40px;
    }
    .greeting {
      font-size: 18px;
      color: #1e3a5f;
      font-weight: 600;
      margin-bottom: 16px;
    }
    .intro {
      font-size: 15px;
      line-height: 1.7;
      color: #555555;
      margin-bottom: 24px;
    }
    .rating-box {
      background-color: #eaf4ff;
      border-left: 5px solid #2d6a9f;
      border-radius: 8px;
      padding: 16px 20px;
      margin-bottom: 24px;
    }
    .rating-box .label {
      font-size: 12px;
      text-transform: uppercase;
      letter-spacing: 1px;
      color: #2d6a9f;
      font-weight: 700;
    }
    .rating-box .stars {
      font-size: 22px;
      margin: 6px 0 2px;
    }
    .rating-box .score {
      font-size: 13px;
      color: #666;
    }
    .summary-box {
      background-color: #f9f9f9;
      border-radius: 8px;
      padding: 20px 24px;
      margin-bottom: 24px;
    }
    .summary-box .label {
      font-size: 12px;
      text-transform: uppercase;
      letter-spacing: 1px;
      color: #999;
      font-weight: 700;
      margin-bottom: 10px;
    }
    .summary-box p {
      font-size: 15px;
      line-height: 1.7;
      color: #444;
      margin: 0;
    }
    .cta {
      text-align: center;
      margin: 28px 0 0;
    }
    .cta p {
      font-size: 14px;
      color: #888;
    }
    .footer {
      background-color: #f0f4f8;
      padding: 20px 40px;
      text-align: center;
      font-size: 12px;
      color: #aaaaaa;
      border-top: 1px solid #e0e7ef;
    }
    .footer a {
      color: #2d6a9f;
      text-decoration: none;
    }
  </style>
</head>
<body>
  <div class="email-wrapper">
    <div class="header">
      <h1>📐 Apex Math Academy</h1>
      <p>Session Progress Update · ${item.submissionDate}</p>
    </div>
    <div class="body">
      <div class="greeting">Dear ${item.studentName.split(' ')[0]}'s Family,</div>
      <p class="intro">
        We're happy to share that <strong>${item.studentName}</strong> is ${toneMessage}. 
        Here's a summary of today's tutoring session along with their confidence rating.
      </p>

      <div class="rating-box">
        <div class="label">Confidence Rating</div>
        <div class="stars">${stars}${emptyStars}</div>
        <div class="score">${rating} out of 5 — ${rating >= 4 ? 'Excellent' : 'Good Progress'}</div>
      </div>

      <div class="summary-box">
        <div class="label">Session Summary</div>
        <p>${item.lessonSummary}</p>
      </div>

      <div class="cta">
        <p>If you have any questions or would like to discuss your child's progress further, 
        please don't hesitate to reach out to us directly.</p>
      </div>
    </div>
    <div class="footer">
      <p>Sent by <strong>Apex Math Academy</strong> Automated Reporting System<br>
      Session logged on ${item.submissionDate} at ${item.submissionTime}</p>
    </div>
  </div>
</body>
</html>
`;

const emailSubject = `✅ Progress Update for ${item.studentName} — Session ${item.submissionDate}`;

return {
  json: {
    ...item,
    emailHtmlBody: htmlBody,
    emailSubject,
    emailRecipient: item.parentEmail
  }
};
```

---

### Node 15: Gmail — Send Progress Email

**Node Type:** `Gmail`
**Node Name:** `Gmail: Send Progress Email`

#### Configuration

| Setting | Value |
|---|---|
| Credential | `Gmail - Apex Math` |
| Resource | `Message` |
| Operation | `Send` |
| To | `={{ $json.emailRecipient }}` |
| Subject | `={{ $json.emailSubject }}` |
| Email Type | `HTML` |
| Message | `={{ $json.emailHtmlBody }}` |

#### Additional Fields

| Option | Value |
|---|---|
| Reply To | `admin@apexmathacademy.com` |
| BCC | `records@apexmathacademy.com` |

#### How to Add This Node

1. Connect from `Function: Build HTML Email`
2. Add `Gmail` node named `Gmail: Send Progress Email`
3. Select `Gmail - Apex Math` credential
4. Set Resource: `Message`, Operation: `Send`
5. Map To, Subject, and Message fields as above
6. Expand **Additional Fields** → add Reply To and BCC
7. Click **Save**

---

## Layer 8 — Error Handling System

Every failure point in the workflow routes to a specific error Set node that packages the error context, then sends it to Slack.

---

### Error Path A: Missing Student Name

#### Node 16: Set — Error Payload (Missing Name)

**Node Type:** `Set`
**Node Name:** `Set: Error — Missing Name`

#### Configuration

| Field | Type | Value |
|---|---|---|
| `errorType` | String | `MISSING_STUDENT_NAME` |
| `errorMessage` | String | `Student name is empty or was not submitted` |
| `errorSeverity` | String | `HIGH` |
| `rawPayload` | String | `={{ JSON.stringify($json) }}` |
| `errorTimestamp` | String | `={{ $now.toISO() }}` |

**Connect from:** False output of `IF: Name Exists?`
**Connect to:** `Slack: Send Error Alert`

---

### Error Path B: Invalid Email

#### Node 17: Set — Error Payload (Bad Email)

**Node Type:** `Set`
**Node Name:** `Set: Error — Bad Email`

#### Configuration

| Field | Type | Value |
|---|---|---|
| `errorType` | String | `INVALID_EMAIL_FORMAT` |
| `errorMessage` | String | `Parent email failed regex validation` |
| `errorSeverity` | String | `HIGH` |
| `offendingValue` | String | `={{ $json.parentEmail }}` |
| `rawPayload` | String | `={{ JSON.stringify($json) }}` |
| `errorTimestamp` | String | `={{ $now.toISO() }}` |

**Connect from:** False output of `IF: Email Valid?`
**Connect to:** `Slack: Send Error Alert`

---

### Error Path C: Rating Out of Range

#### Node 18: Set — Error Payload (Bad Rating)

**Node Type:** `Set`
**Node Name:** `Set: Error — Bad Rating`

#### Configuration

| Field | Type | Value |
|---|---|---|
| `errorType` | String | `RATING_OUT_OF_RANGE` |
| `errorMessage` | String | `Confidence rating must be between 1 and 5` |
| `errorSeverity` | String | `MEDIUM` |
| `offendingValue` | Number | `={{ $json.confidenceRating }}` |
| `rawPayload` | String | `={{ JSON.stringify($json) }}` |
| `errorTimestamp` | String | `={{ $now.toISO() }}` |

**Connect from:** False output of `IF: Rating In Range?`
**Connect to:** `Slack: Send Error Alert`

---

### Error Path D: Airtable Failure

#### Node 19: Set — Error Payload (Airtable Failure)

**Node Type:** `Set`
**Node Name:** `Set: Airtable Error Payload`

#### Configuration

| Field | Type | Value |
|---|---|---|
| `errorType` | String | `AIRTABLE_WRITE_FAILURE` |
| `errorMessage` | String | `Failed to create record in Airtable Lesson Reports table` |
| `errorSeverity` | String | `CRITICAL` |
| `airtableError` | String | `={{ $json.error?.message || 'Unknown Airtable error' }}` |
| `rawPayload` | String | `={{ JSON.stringify($json) }}` |
| `errorTimestamp` | String | `={{ $now.toISO() }}` |

**Connect from:** Error output of `Airtable: Create Lesson Record`
**Connect to:** `Slack: Send Error Alert`

---

### Node 20: Function — Build Error Slack Message

**Node Type:** `Code`
**Node Name:** `Function: Build Error Message`

Connect all four error Set nodes into this single Function node using a **Merge** node first (see below).

```javascript
const item = $input.first().json;

const severity = item.errorSeverity || 'UNKNOWN';
const severityEmoji = severity === 'CRITICAL' ? '💀' : severity === 'HIGH' ? '🔴' : '🟡';

const errorMessage = `${severityEmoji} *SYSTEM ERROR — ${severity} SEVERITY*

*Error Type:* \`${item.errorType}\`
*Message:* ${item.errorMessage}
*Timestamp:* ${item.errorTimestamp}

${item.offendingValue !== undefined ? `*Offending Value:* \`${item.offendingValue}\`` : ''}
${item.airtableError ? `*Airtable Error:* ${item.airtableError}` : ''}

*Raw Submission Payload:*
\`\`\`
${item.rawPayload ? item.rawPayload.substring(0, 800) : 'No payload available'}
\`\`\`

_Please investigate immediately and correct the data source._`;

return {
  json: {
    ...item,
    errorSlackMessage: errorMessage
  }
};
```

---

### Node 20B: Merge — Combine Error Paths

**Node Type:** `Merge`
**Node Name:** `Merge: All Error Paths`

Since all four error paths need to go to the same Slack alert, use a Merge node:

1. Add a `Merge` node named `Merge: All Error Paths`
2. Set **Mode** to `Merge By Position` (or `Append` — either works here)
3. Connect all four error Set nodes into the inputs of this Merge node
4. Connect Merge output → `Function: Build Error Message`

---

### Node 21: Slack — Send Error Alert

**Node Type:** `Slack`
**Node Name:** `Slack: Send Error Alert`

#### Configuration

| Setting | Value |
|---|---|
| Credential | `Slack - Apex Math` |
| Resource | `Message` |
| Operation | `Send` |
| Channel | `#system-errors` |
| Text | `={{ $json.errorSlackMessage }}` |

**Additional Fields:**

| Option | Value |
|---|---|
| Username | `Apex Math Error Bot` |
| Icon Emoji | `:rotating_light:` |

---

## Layer 9 — Logging & Observability

### Node 22: Airtable — Log Execution Record (Optional)

**Node Type:** `Airtable`
**Node Name:** `Airtable: Log Execution`

Create a second table in your Airtable base called **Execution Logs** with these fields:

| Field Name | Type |
|---|---|
| `Execution ID` | Single line text |
| `Workflow Name` | Single line text |
| `Student Name` | Single line text |
| `Outcome` | Single select: Success-Low / Success-High / Error |
| `Error Type` | Single line text |
| `Timestamp` | Date & Time |

#### Node Configuration for Success Path

Add this node after each final output node (Slack alert and Gmail). Set:

| Field | Value |
|---|---|
| Execution ID | `={{ $json.executionId }}` |
| Workflow Name | `={{ $json.workflowName }}` |
| Student Name | `={{ $json.studentName }}` |
| Outcome | `Success-Low` or `Success-High` (hardcode per branch) |
| Timestamp | `={{ $json.submissionTimestamp }}` |

---

## Full Connection Map

Below is a complete list of every connection in the workflow, in execution order:

```
Node 1: Trigger: New Lesson Report
  └─[output]──► Node 2: Set: Normalize Fields

Node 2: Set: Normalize Fields
  └─[output]──► Node 3: IF: Name Exists?

Node 3: IF: Name Exists?
  ├─[True]───► Node 4: IF: Email Valid?
  └─[False]──► Node 16: Set: Error — Missing Name

Node 4: IF: Email Valid?
  ├─[True]───► Node 5: IF: Rating In Range?
  └─[False]──► Node 17: Set: Error — Bad Email

Node 5: IF: Rating In Range?
  ├─[True]───► Node 6: Function: Clean Summary
  └─[False]──► Node 18: Set: Error — Bad Rating

Node 6: Function: Clean Summary
  └─[output]──► Node 7: Set: Add Timestamp

Node 7: Set: Add Timestamp
  └─[output]──► Node 8: Function: Attach Metadata

Node 8: Function: Attach Metadata
  └─[output]──► Node 9: Airtable: Create Lesson Record

Node 9: Airtable: Create Lesson Record
  ├─[Success]──► Node 10: IF: Rating Low?
  └─[Error]────► Node 19: Set: Airtable Error Payload

Node 10: IF: Rating Low?
  ├─[True]────► Node 11: Set: Format Slack Alert
  └─[False]───► Node 14: Function: Build HTML Email

Node 11: Set: Format Slack Alert
  └─[output]──► Node 12: Function: Add Urgency Flag

Node 12: Function: Add Urgency Flag
  └─[output]──► Node 13: Slack: Send Founder Alert

Node 14: Function: Build HTML Email
  └─[output]──► Node 15: Gmail: Send Progress Email

Node 16: Set: Error — Missing Name    ─┐
Node 17: Set: Error — Bad Email       ─┤──► Node 20B: Merge: All Error Paths
Node 18: Set: Error — Bad Rating      ─┤
Node 19: Set: Airtable Error Payload  ─┘

Node 20B: Merge: All Error Paths
  └─[output]──► Node 20: Function: Build Error Message

Node 20: Function: Build Error Message
  └─[output]──► Node 21: Slack: Send Error Alert
```

---

## Testing Section

### ✅ Test Case A — Low Confidence Student

**Purpose:** Verify that a rating of 1 or 2 triggers a Slack alert and logs to Airtable.

**Test Input Row in Google Sheet:**

| Student Name | Parent Email | Lesson Summary | Confidence Rating |
|---|---|---|---|
| Jamie Torres | torres.family@gmail.com | Struggled with long division remainders. Needs follow-up practice on carrying numbers. | 2 |

**Steps to Run:**

1. Add this row to your Google Sheet
2. Wait up to 1 minute for the trigger to fire (or click **Execute Workflow** manually)
3. Watch the n8n execution view

**Expected Results:**

| Check | Expected | Pass/Fail |
|---|---|---|
| IF: Name Exists? | Routes True | ✅ |
| IF: Email Valid? | Routes True | ✅ |
| IF: Rating In Range? | Routes True (2 is valid) | ✅ |
| Airtable record created | Row appears in Lesson Reports | ✅ |
| IF: Rating Low? | Routes True (2 ≤ 2) | ✅ |
| Slack message sent | Visible in #founder-alerts | ✅ |
| Gmail NOT sent | No email dispatched | ✅ |

**Verification:**
- Open Airtable → Lesson Reports → confirm new row with Confidence Rating = 2
- Open Slack → #founder-alerts → confirm message with 🟠 TUTOR ALERT — HIGH PRIORITY

---

### ✅ Test Case B — High Confidence Student

**Purpose:** Verify that a rating of 3–5 sends an HTML email and logs to Airtable.

**Test Input Row:**

| Student Name | Parent Email | Lesson Summary | Confidence Rating |
|---|---|---|---|
| Alex Patel | patel.home@outlook.com | Mastered quadratic equations today. Alex solved all 10 problems independently with no hints needed. Exceptional session! | 5 |

**Expected Results:**

| Check | Expected | Pass/Fail |
|---|---|---|
| IF: Name Exists? | Routes True | ✅ |
| IF: Email Valid? | Routes True | ✅ |
| IF: Rating In Range? | Routes True | ✅ |
| Airtable record created | Row appears in Lesson Reports | ✅ |
| IF: Rating Low? | Routes False (5 > 2) | ✅ |
| Gmail HTML email sent | Email with 5 stars visible | ✅ |
| Slack alert NOT sent | No message in #founder-alerts | ✅ |

**Verification:**
- Check `patel.home@outlook.com` inbox for styled HTML email with ⭐⭐⭐⭐⭐
- Open Airtable → confirm record with Rating = 5

---

### ❌ Test Case C — Missing Student Name

**Purpose:** Verify that missing required fields trigger the error Slack alert.

**Test Input Row:**

| Student Name | Parent Email | Lesson Summary | Confidence Rating |
|---|---|---|---|
| _(empty)_ | incomplete@test.com | Some lesson notes. | 3 |

**Expected Results:**

| Check | Expected |
|---|---|
| IF: Name Exists? | Routes False |
| Error Set node fires | `MISSING_STUDENT_NAME` payload built |
| Slack error alert sent | Visible in #system-errors |
| Airtable NOT written | No partial record created |
| Gmail NOT sent | No email sent |

---

### ❌ Test Case D — Invalid Email

**Test Input Row:**

| Student Name | Parent Email | Lesson Summary | Confidence Rating |
|---|---|---|---|
| Sam Lee | not-an-email | Good session today. | 4 |

**Expected Results:**

| Check | Expected |
|---|---|
| IF: Name Exists? | Routes True |
| IF: Email Valid? | Routes False |
| Error: `INVALID_EMAIL_FORMAT` | Sent to #system-errors |

---

### ❌ Test Case E — Rating Out of Range

**Test Input Row:**

| Student Name | Parent Email | Lesson Summary | Confidence Rating |
|---|---|---|---|
| Morgan Smith | smith@gmail.com | Reviewed algebra basics. | 7 |

**Expected Results:**

| Check | Expected |
|---|---|
| IF: Rating In Range? | Routes False (7 > 5) |
| Error: `RATING_OUT_OF_RANGE` | Sent to #system-errors |

---

## n8n Workflow JSON Export

Below is the full n8n workflow JSON. You can import this directly into n8n via **Workflows → Import from JSON**.

> **Before importing:** You must update all credential IDs to match those in your own n8n instance. Search for `"credentials"` blocks in the JSON and replace the IDs with your actual credential IDs.

```json
{
  "name": "Apex Math Academy - Post-Session Reporting",
  "nodes": [
    {
      "parameters": {
        "pollTimes": {
          "item": [{ "mode": "everyMinute" }]
        },
        "triggerOn": "rowAdded",
        "sheetName": "Sheet1"
      },
      "id": "node-001",
      "name": "Trigger: New Lesson Report",
      "type": "n8n-nodes-base.googleSheetsTrigger",
      "typeVersion": 1,
      "position": [250, 300],
      "credentials": {
        "googleSheetsOAuth2Api": {
          "id": "REPLACE_WITH_YOUR_CREDENTIAL_ID",
          "name": "Google Sheets - Apex Math"
        }
      }
    },
    {
      "parameters": {
        "values": {
          "string": [
            { "name": "studentName", "value": "={{ $json['Student Name'] }}" },
            { "name": "parentEmail", "value": "={{ $json['Parent Email'] }}" },
            { "name": "lessonSummary", "value": "={{ $json['Lesson Summary'] }}" },
            { "name": "rawTimestamp", "value": "={{ $json['Timestamp'] || $now }}" }
          ],
          "number": [
            { "name": "confidenceRating", "value": "={{ Number($json['Confidence Rating']) }}" },
            { "name": "rowNumber", "value": "={{ $json['row_number'] }}" }
          ]
        },
        "options": { "includeOtherFields": false }
      },
      "id": "node-002",
      "name": "Set: Normalize Fields",
      "type": "n8n-nodes-base.set",
      "typeVersion": 3,
      "position": [450, 300]
    },
    {
      "parameters": {
        "conditions": {
          "string": [
            {
              "value1": "={{ $json.studentName }}",
              "operation": "isNotEmpty"
            }
          ]
        }
      },
      "id": "node-003",
      "name": "IF: Name Exists?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2,
      "position": [650, 300]
    },
    {
      "parameters": {
        "conditions": {
          "string": [
            {
              "value1": "={{ $json.parentEmail }}",
              "operation": "regex",
              "value2": "^[^\\s@]+@[^\\s@]+\\.[^\\s@]+$"
            }
          ]
        }
      },
      "id": "node-004",
      "name": "IF: Email Valid?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2,
      "position": [850, 200]
    },
    {
      "parameters": {
        "conditions": {
          "number": [
            {
              "value1": "={{ $json.confidenceRating }}",
              "operation": "gte",
              "value2": 1
            },
            {
              "value1": "={{ $json.confidenceRating }}",
              "operation": "lte",
              "value2": 5
            }
          ]
        },
        "combineOperation": "all"
      },
      "id": "node-005",
      "name": "IF: Rating In Range?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2,
      "position": [1050, 200]
    },
    {
      "parameters": {
        "jsCode": "const item = $input.first().json;\nlet summary = item.lessonSummary || '';\nsummary = summary.trim().replace(/\\s+/g, ' ').replace(/<[^>]*>/g, '').replace(/[^\\x20-\\x7E\\n]/g, '');\nif (summary.length > 2000) summary = summary.substring(0, 1997) + '...';\nif (summary.length === 0) summary = '[No lesson summary provided]';\nreturn { json: { ...item, lessonSummary: summary, summaryCleaned: true } };"
      },
      "id": "node-006",
      "name": "Function: Clean Summary",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [1250, 200]
    },
    {
      "parameters": {
        "values": {
          "string": [
            { "name": "submissionTimestamp", "value": "={{ $now.toISO() }}" },
            { "name": "submissionDate", "value": "={{ $now.toFormat('yyyy-MM-dd') }}" },
            { "name": "submissionTime", "value": "={{ $now.toFormat('HH:mm:ss') }}" }
          ]
        },
        "options": { "includeOtherFields": true }
      },
      "id": "node-007",
      "name": "Set: Add Timestamp",
      "type": "n8n-nodes-base.set",
      "typeVersion": 3,
      "position": [1450, 200]
    },
    {
      "parameters": {
        "jsCode": "const item = $input.first().json;\nreturn { json: { ...item, executionId: $execution.id, workflowId: $workflow.id, workflowName: $workflow.name, processingEnvironment: 'production', dataSchemaVersion: '1.0' } };"
      },
      "id": "node-008",
      "name": "Function: Attach Metadata",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [1650, 200]
    },
    {
      "parameters": {
        "operation": "create",
        "base": { "value": "REPLACE_WITH_YOUR_BASE_ID", "mode": "id" },
        "table": { "value": "Lesson Reports", "mode": "name" },
        "columns": {
          "mappingMode": "defineBelow",
          "value": {
            "Student Name": "={{ $json.studentName }}",
            "Parent Email": "={{ $json.parentEmail }}",
            "Lesson Summary": "={{ $json.lessonSummary }}",
            "Confidence Rating": "={{ $json.confidenceRating }}",
            "Submission Timestamp": "={{ $json.submissionTimestamp }}",
            "Submission Date": "={{ $json.submissionDate }}",
            "Execution ID": "={{ $json.executionId }}",
            "Processing Status": "Pending"
          }
        }
      },
      "id": "node-009",
      "name": "Airtable: Create Lesson Record",
      "type": "n8n-nodes-base.airtable",
      "typeVersion": 2,
      "position": [1850, 200],
      "onError": "continueErrorOutput",
      "credentials": {
        "airtableTokenApi": {
          "id": "REPLACE_WITH_YOUR_CREDENTIAL_ID",
          "name": "Airtable - Apex Math"
        }
      }
    },
    {
      "parameters": {
        "conditions": {
          "number": [
            {
              "value1": "={{ $json.confidenceRating }}",
              "operation": "lte",
              "value2": 2
            }
          ]
        }
      },
      "id": "node-010",
      "name": "IF: Rating Low?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2,
      "position": [2050, 200]
    }
  ],
  "connections": {
    "Trigger: New Lesson Report": {
      "main": [[{ "node": "Set: Normalize Fields", "type": "main", "index": 0 }]]
    },
    "Set: Normalize Fields": {
      "main": [[{ "node": "IF: Name Exists?", "type": "main", "index": 0 }]]
    },
    "IF: Name Exists?": {
      "main": [
        [{ "node": "IF: Email Valid?", "type": "main", "index": 0 }],
        [{ "node": "Set: Error — Missing Name", "type": "main", "index": 0 }]
      ]
    },
    "IF: Email Valid?": {
      "main": [
        [{ "node": "IF: Rating In Range?", "type": "main", "index": 0 }],
        [{ "node": "Set: Error — Bad Email", "type": "main", "index": 0 }]
      ]
    },
    "IF: Rating In Range?": {
      "main": [
        [{ "node": "Function: Clean Summary", "type": "main", "index": 0 }],
        [{ "node": "Set: Error — Bad Rating", "type": "main", "index": 0 }]
      ]
    },
    "Function: Clean Summary": {
      "main": [[{ "node": "Set: Add Timestamp", "type": "main", "index": 0 }]]
    },
    "Set: Add Timestamp": {
      "main": [[{ "node": "Function: Attach Metadata", "type": "main", "index": 0 }]]
    },
    "Function: Attach Metadata": {
      "main": [[{ "node": "Airtable: Create Lesson Record", "type": "main", "index": 0 }]]
    },
    "Airtable: Create Lesson Record": {
      "main": [
        [{ "node": "IF: Rating Low?", "type": "main", "index": 0 }],
        [{ "node": "Set: Airtable Error Payload", "type": "main", "index": 0 }]
      ]
    },
    "IF: Rating Low?": {
      "main": [
        [{ "node": "Set: Format Slack Alert", "type": "main", "index": 0 }],
        [{ "node": "Function: Build HTML Email", "type": "main", "index": 0 }]
      ]
    }
  },
  "settings": {
    "executionOrder": "v1",
    "saveManualExecutions": true,
    "callerPolicy": "workflowsFromSameOwner",
    "errorWorkflow": ""
  },
  "staticData": null,
  "tags": ["apex-math", "education", "reporting", "production"],
  "triggerCount": 1,
  "updatedAt": "2026-04-24T00:00:00.000Z",
  "versionId": "1.0.0"
}
```

> **Note:** This JSON is a representative export. After importing, use the n8n canvas to add and connect any remaining nodes (Slack, Gmail, error paths) as described in this guide.

---

## Airtable Schema

### Table: Lesson Reports

| Field Name | Field Type | Options / Notes |
|---|---|---|
| `Student Name` | Single line text | Primary field — required |
| `Parent Email` | Email | Validates email format in Airtable |
| `Lesson Summary` | Long text | Supports up to 100k characters |
| `Confidence Rating` | Number | Integer, 1–5 |
| `Submission Timestamp` | Date & Time | ISO 8601, include time field |
| `Submission Date` | Single line text | Format: YYYY-MM-DD |
| `Execution ID` | Single line text | n8n workflow execution ID |
| `Processing Status` | Single select | Options: Pending, Processed, Error |

### Table: Execution Logs

| Field Name | Field Type | Options / Notes |
|---|---|---|
| `Execution ID` | Single line text | Primary field |
| `Workflow Name` | Single line text | |
| `Student Name` | Single line text | |
| `Outcome` | Single select | Success-Low / Success-High / Error |
| `Error Type` | Single line text | Blank if success |
| `Timestamp` | Date & Time | |

---

## Proof of Execution Guide

### 1. Screenshot the Workflow Canvas

1. Open n8n → navigate to your **Apex Math Academy** workflow
2. Press `Ctrl+Shift+F` (or `Cmd+Shift+F` on Mac) to fit the workflow to screen
3. Use your OS screenshot tool:
   - Windows: `Win+Shift+S`
   - Mac: `Cmd+Shift+4`
4. Capture the full canvas showing all nodes and connections

### 2. Record a Test Run

1. In n8n, click **Execute Workflow** (▶ button top right)
2. Add a test row to your Google Sheet
3. Watch the execution log panel on the right
4. Each node that fires will highlight green (success) or red (error)
5. Click any node to inspect its **Input** and **Output** data tabs
6. Use Loom, OBS, or Quicktime to record the screen during execution

### 3. Verify Slack Output

**For Low Confidence Alert:**
1. Open Slack → navigate to `#founder-alerts`
2. Screenshot the alert message showing the student name, rating, and urgency emoji
3. Confirm timestamp matches the test submission time

**For Error Alerts:**
1. Open Slack → navigate to `#system-errors`
2. Screenshot the error message showing the error type and raw payload

### 4. Verify Gmail Output

1. Log in to the parent email address used in Test Case B
2. Open the inbox and find the email from Apex Math Academy
3. Screenshot:
   - Email subject line showing student name and date
   - Full HTML body with star rating and lesson summary
   - Confirm the email renders correctly on both desktop and mobile

### 5. Verify Airtable Output

1. Open your Airtable base → **Lesson Reports** table
2. Confirm the test records appear with all fields populated
3. Screenshot the grid view showing the new rows
4. Check that `Processing Status` is set to `Pending`

---

## Acceptance Checklist

Use this checklist to confirm the workflow is production-ready before going live.

### Core Functionality

- [ ] Google Sheets Trigger fires within 1 minute of a new row being added
- [ ] All incoming data is correctly normalized (strings cast, fields renamed)
- [ ] Student name validation works — empty name routes to error branch
- [ ] Email validation works — malformed email routes to error branch
- [ ] Rating range validation works — value outside 1–5 routes to error branch
- [ ] Lesson summary is cleaned (HTML stripped, whitespace normalized)
- [ ] Submission timestamp is added in ISO 8601 format
- [ ] Execution metadata (ID, workflow name) is attached to payload

### Airtable Logging

- [ ] Airtable record is created successfully for valid submissions
- [ ] All 8 fields are populated in the record
- [ ] Airtable failure routes to the error Slack alert (not a silent failure)

### Confidence Routing

- [ ] Rating ≤ 2 routes to Slack alert branch ✅
- [ ] Rating ≥ 3 routes to Gmail branch ✅

### Slack Alerts

- [ ] Slack alert is ONLY sent for ratings 1 or 2 ✅
- [ ] Alert includes student name, rating, summary, and urgency level
- [ ] 🔴 emoji for rating = 1, 🟠 emoji for rating = 2
- [ ] Alert appears in `#founder-alerts` channel

### Gmail Communication

- [ ] Gmail is ONLY sent for ratings 3, 4, or 5 ✅
- [ ] Email uses correct recipient (parent email from submission)
- [ ] Email subject includes student name and session date
- [ ] HTML email renders correctly with styled header, rating stars, and summary box ✅
- [ ] BCC is sent to records@apexmathacademy.com

### Error Handling

- [ ] Missing name → error Slack alert with `MISSING_STUDENT_NAME` ✅
- [ ] Bad email → error Slack alert with `INVALID_EMAIL_FORMAT` ✅
- [ ] Out-of-range rating → error Slack alert with `RATING_OUT_OF_RANGE` ✅
- [ ] Airtable failure → error Slack alert with `AIRTABLE_WRITE_FAILURE` ✅
- [ ] All error alerts appear in `#system-errors` channel
- [ ] Raw payload is included in error message for debugging

### Logging & Observability

- [ ] Execution logs table is populated after each run
- [ ] Outcome field reflects correct branch taken
- [ ] No orphaned or partial records in Airtable

---

*End of Implementation Guide — Apex Math Academy Automated Post-Session Reporting & Alert System*

*Version 1.0.0 · Built for n8n · Beginner-Friendly · Production-Ready*

# Fabric Donation Routing for Sprout & Sew
### Complete n8n Automation Implementation Guide

---

> **Difficulty:** Beginner-Intermediate  
> **Estimated Build Time:** 3–5 hours  
> **Nodes in Workflow:** 16  
> **Tools Required:** n8n, Google Forms, Google Sheets, Slack  

---

## Table of Contents

1. [Project Title & Summary](#1-project-title--summary)
2. [Business Goal](#2-business-goal)
3. [Full Workflow Architecture](#3-full-workflow-architecture)
4. [Required Accounts & Credentials](#4-required-accounts--credentials)
5. [Google Form Setup](#5-google-form-setup)
6. [Google Sheet Setup](#6-google-sheet-setup)
7. [Full n8n Build Guide – Node by Node](#7-full-n8n-build-guide--node-by-node)
8. [Exact Field Mapping Reference](#8-exact-field-mapping-reference)
9. [Slack Message Formatting](#9-slack-message-formatting)
10. [Sustainability Metric Logic](#10-sustainability-metric-logic)
11. [Error Handling Section](#11-error-handling-section)
12. [Testing Procedure](#12-testing-procedure)
13. [Final Deliverables Checklist](#13-final-deliverables-checklist)
14. [Troubleshooting Section](#14-troubleshooting-section)
15. [Pro Tips](#15-pro-tips)

---

# 1. Project Title & Summary

**Project Name:** Fabric Donation Routing Automation  
**Client:** Sprout & Sew – Sustainable Textile Boutique  
**Built With:** n8n (self-hosted or cloud), Google Forms, Google Sheets, Slack  

**One-Line Summary:**  
An automated pipeline that captures fabric donation offers from a Google Form, validates and enriches the data, sends structured Slack notifications to the production team, logs every donation to Google Sheets, and tracks estimated sustainability impact in real time — all without any manual intervention.

---

# 2. Business Goal

## The Problem

Sprout & Sew runs a **Fabric Donation Program** that allows community members to donate vintage linen and other textiles. These donations are raw materials the boutique upcycles into children's clothing.

Currently, the donation intake process is broken:

- Donors submit interest via email or a loosely formatted contact form
- Emails pile up in a shared inbox and are frequently missed
- There is no consistent record of who donated what, when, or how much
- The production team is often unaware that new donations are available for pickup
- There is no way to measure sustainability impact (CO2 diverted, waste avoided)

## The Solution

This n8n automation creates a **fully automated intake pipeline** that:

| Problem | Solution |
|---|---|
| Donations lost in email | Structured Google Form captures all data in a consistent format |
| Team unaware of new offers | Instant Slack notification sent to production channel on each submission |
| No donation records | Every submission is automatically logged to Google Sheets |
| Duplicate submissions | Automated duplicate detection prevents double-logging |
| No sustainability tracking | CO2 saved estimate calculated and stored per donation |
| No timestamps | Every record includes ISO timestamp and generated submission ID |

## Business Impact

- **Zero** donation offers missed
- Production team notified in **under 30 seconds** of any new offer
- Full donation history searchable in Google Sheets
- Sustainability impact report data available at any time
- Entire pipeline runs **24/7 without staff involvement**

---

# 3. Full Workflow Architecture

## Visual Node Flow

```
[Node 1]  Google Forms Trigger
              ↓
[Node 2]  Set Node – Normalize & Clean Data
              ↓
[Node 3]  Code Node – Generate Submission ID + Timestamp
              ↓
[Node 4]  IF Node – Validate Required Fields (Name + Email + Fabric Type)
              ↓                              ↓
         [VALID]                        [INVALID]
              ↓                              ↓
[Node 5]  IF Node – Pickup Needed?    [Node 5b] Slack – Send Validation Failure Alert
              ↓              ↓
        [YES]            [NO]
              ↓              ↓
[Node 6a] Set – Build   [Node 6b] Set – Build
          Slack Msg          Slack Msg
          (with pickup)      (no pickup)
              ↓              ↓
[Node 7]  Merge Node – Rejoin Pickup Branches
              ↓
[Node 8]  Slack Node – Send Donation Alert to #donations Channel
              ↓
[Node 9]  Google Sheets – Search Existing Row by Email
              ↓
[Node 10] IF Node – Duplicate Found?
              ↓                    ↓
        [NOT DUPLICATE]       [DUPLICATE]
              ↓                    ↓
[Node 11] Code Node –       [Node 11b] Slack – Send Duplicate
          Calculate CO2              Warning Message
          Saved Estimate
              ↓
[Node 12] Google Sheets – Append New Row
              ↓
[Node 13] Set Node – Build Success Summary
              ↓
[Node 14] Slack Node – Send Confirmation to #donations
              ↓
[Node 15] NoOp – End / Workflow Complete
```

## Node Summary Table

| # | Node Name | Type | Purpose |
|---|---|---|---|
| 1 | Google Forms Trigger | Trigger | Fires when a new form response arrives |
| 2 | Normalize Submission Data | Set | Cleans whitespace, normalizes field names |
| 3 | Generate Submission ID | Code | Creates unique ID and ISO timestamp |
| 4 | Validate Required Fields | IF | Routes invalid submissions to alert |
| 5a | Validate Failure Alert | Slack | Notifies team of bad submission |
| 5b | Pickup Branch Router | IF | Splits YES/NO pickup flows |
| 6a | Build Slack Msg – Pickup Yes | Set | Formats message with pickup details |
| 6b | Build Slack Msg – Pickup No | Set | Formats message without pickup details |
| 7 | Merge Pickup Branches | Merge | Rejoins pickup/no-pickup flows |
| 8 | Send Donation Alert | Slack | Posts donation offer to Slack channel |
| 9 | Search Sheet by Email | Google Sheets | Checks if donor email already logged today |
| 10 | Duplicate Check Router | IF | Routes duplicate vs. new submissions |
| 11a | Calculate CO2 Saved | Code | Computes textile diversion estimate |
| 11b | Send Duplicate Warning | Slack | Alerts team of possible duplicate |
| 12 | Append Row to Sheet | Google Sheets | Writes full record to spreadsheet |
| 13 | Build Success Summary | Set | Formats final summary fields |
| 14 | Send Confirmation Message | Slack | Posts success confirmation to Slack |
| 15 | End – Workflow Complete | NoOp | Terminates the workflow cleanly |

---

# 4. Required Accounts & Credentials

## 4.1 Google Account

You need a standard Google account (free Gmail is sufficient).

All Google services used — Forms, Sheets, and the trigger integration — operate under the same Google account.

**Action:** Go to [https://accounts.google.com](https://accounts.google.com) and confirm you are logged in.

---

## 4.2 Google Forms

No special setup is required. Google Forms is included free with any Google account.

**Action:** Go to [https://forms.google.com](https://forms.google.com) — you will create the form in Section 5 of this guide.

---

## 4.3 Google Sheets

Google Sheets is free with any Google account.

**Action:** Go to [https://sheets.google.com](https://sheets.google.com) — you will create the sheet in Section 6 of this guide.

---

## 4.4 Connecting Google to n8n

In n8n, you will use **OAuth2** to connect your Google account.

**Steps:**

1. Open n8n and go to **Settings → Credentials**
2. Click **New Credential**
3. Search for `Google Sheets OAuth2 API`
4. Click **Connect with Google**
5. Sign in with your Google account and grant n8n permission
6. Name this credential: `Google – Sprout and Sew`
7. Click **Save**

> ⚠️ You will use this same credential for both the Google Forms Trigger and the Google Sheets nodes.

---

## 4.5 Slack Workspace

You need access to a Slack workspace where you can install apps.

If you do not have one:  
1. Go to [https://slack.com/get-started](https://slack.com/get-started)
2. Create a free workspace (e.g., `sprout-and-sew.slack.com`)

**Create a channel** inside that workspace called `#donations`.

---

## 4.6 Slack Bot Token

You need to create a **Slack App** and generate a Bot Token.

**Steps:**

1. Go to [https://api.slack.com/apps](https://api.slack.com/apps)
2. Click **Create New App → From Scratch**
3. Name: `Sprout Sew Bot`
4. Select your workspace → Click **Create App**
5. In the left sidebar, click **OAuth & Permissions**
6. Scroll to **Scopes → Bot Token Scopes** → Add these scopes:
   - `chat:write`
   - `chat:write.public`
   - `channels:read`
7. Scroll up and click **Install to Workspace** → Allow
8. Copy the **Bot OAuth Token** (starts with `xoxb-...`)

**Add to n8n:**

1. In n8n → **Settings → Credentials**
2. Click **New Credential** → Search `Slack`
3. Select **Slack API**
4. Paste the Bot Token
5. Name it: `Slack – Sprout Sew Bot`
6. Click **Save**

---

## 4.7 n8n Instance

You need a running n8n instance. Options:

| Option | URL | Notes |
|---|---|---|
| n8n Cloud | [https://app.n8n.cloud](https://app.n8n.cloud) | Easiest, paid after trial |
| Self-hosted (Docker) | `localhost:5678` | Free, requires Docker |
| Self-hosted (npm) | `localhost:5678` | Free, requires Node.js |

For self-hosting with Docker, run:

```bash
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v ~/.n8n:/home/node/.n8n \
  n8nio/n8n
```

Then open [http://localhost:5678](http://localhost:5678) in your browser.

---

# 5. Google Form Setup

## 5.1 Create the Form

1. Go to [https://forms.google.com](https://forms.google.com)
2. Click the **+** button to create a blank form
3. Title the form: `Fabric Donation – Sprout & Sew`
4. Add a short description: *"Thank you for donating! Please fill in the details below so our team can arrange collection."*

---

## 5.2 Add These Fields (In Order)

### Field 1 – Full Name

- **Question:** `Full Name`
- **Type:** Short answer
- **Required:** ✅ Yes

---

### Field 2 – Email Address

- **Question:** `Email`
- **Type:** Short answer
- **Required:** ✅ Yes
- **Validation:** Click the three-dot menu → **Response Validation** → Email

---

### Field 3 – Phone Number

- **Question:** `Phone Number`
- **Type:** Short answer
- **Required:** ❌ No

---

### Field 4 – Fabric Type

- **Question:** `Fabric Type`
- **Type:** Dropdown
- **Options:**
  - Cotton Linen
  - Pure Linen
  - Cotton
  - Wool
  - Denim
  - Silk
  - Mixed / Other
- **Required:** ✅ Yes

---

### Field 5 – Fabric Condition

- **Question:** `Fabric Condition`
- **Type:** Multiple choice
- **Options:**
  - Excellent – Like new
  - Good – Minor wear
  - Fair – Visible wear, still usable
  - Poor – Heavily worn
- **Required:** ✅ Yes

---

### Field 6 – Estimated Weight (kg)

- **Question:** `Estimated Weight (kg)`
- **Type:** Short answer
- **Required:** ✅ Yes
- **Validation:** Click three-dot menu → **Response Validation** → Number → Greater than → `0`

---

### Field 7 – Pickup Needed?

- **Question:** `Pickup Needed?`
- **Type:** Multiple choice
- **Options:**
  - Yes – Please collect from me
  - No – I will drop it off
- **Required:** ✅ Yes

---

### Field 8 – Preferred Pickup Date

- **Question:** `Preferred Pickup Date`
- **Type:** Date
- **Required:** ❌ No
- **Note:** Add a description: *"Only required if you selected Yes above"*

---

### Field 9 – Additional Notes

- **Question:** `Additional Notes`
- **Type:** Paragraph
- **Required:** ❌ No

---

## 5.3 Link Form to Google Sheets

After creating the form:

1. Click the **Responses** tab at the top of the form
2. Click the **Google Sheets icon** (Create Spreadsheet)
3. Choose **Create a new spreadsheet**
4. Name it: `Sprout & Sew – Donation Log`
5. Click **Create**

> This creates a linked sheet at `Sheet1` with one tab titled `Form Responses 1`. You will use a **second tab** for the processed log. Keep this sheet open — you'll configure it in Section 6.

---

## 5.4 Get the Form Trigger URL (for n8n)

n8n's Google Forms Trigger works by monitoring the **linked Google Sheet** for new rows, not by connecting directly to the form. You will use the Spreadsheet ID from the sheet URL.

**Find your Spreadsheet ID:**

Open the linked spreadsheet. The URL will look like:

```
https://docs.google.com/spreadsheets/d/1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OgVE2upms/edit
```

The Spreadsheet ID is: `1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OgVE2upms`

Copy and save this. You will need it in Node 1.

---

# 6. Google Sheet Setup

## 6.1 Structure Overview

You will use **two tabs** in the same Google Sheet:

| Tab Name | Purpose |
|---|---|
| `Form Responses 1` | Auto-generated by Google Forms — do NOT edit |
| `Donation Log` | Your processed, structured log — created manually |

---

## 6.2 Create the Donation Log Tab

1. At the bottom of the sheet, click the **+** button
2. Right-click the new tab → **Rename** → Type `Donation Log`
3. Click into cell **A1** and add the following headers across Row 1:

| Column | Header Text |
|---|---|
| A | Timestamp |
| B | Submission ID |
| C | Full Name |
| D | Email |
| E | Phone |
| F | Fabric Type |
| G | Condition |
| H | Weight KG |
| I | Pickup Needed |
| J | Pickup Date |
| K | Notes |
| L | CO2 Saved Estimate (kg) |
| M | Status |

---

## 6.3 Format the Header Row

1. Select Row 1 (click the row number `1` on the left)
2. Set **background color** to `#2E7D32` (dark green) to match Sprout & Sew branding
3. Set **font color** to white
4. Set **font weight** to Bold
5. Click **View → Freeze → 1 row** to pin the header while scrolling

---

## 6.4 Column Format Settings

| Column | Format |
|---|---|
| A (Timestamp) | Format → Number → Date time |
| H (Weight KG) | Format → Number → Number (2 decimal places) |
| L (CO2 Saved) | Format → Number → Number (2 decimal places) |

---

## 6.5 Share the Sheet with n8n

If you're using **n8n Cloud**, no extra sharing step is needed after OAuth2 authentication.

If you're **self-hosting**, ensure your Google OAuth2 app has access to the sheet by confirming credentials are connected as described in Section 4.4.

---

# 7. Full n8n Build Guide – Node by Node

Open n8n and create a **new workflow**. Name it:  
`Fabric Donation Routing – Sprout & Sew`

---

## Node 1: Google Forms Trigger

**Node Type:** `Google Sheets Trigger`  
*(n8n uses Google Sheets Trigger to detect new form responses)*

**Rename this node to:** `Google Forms – New Donation Submission`

### Configuration

| Parameter | Value |
|---|---|
| Credential | `Google – Sprout and Sew` |
| Trigger On | `Row Added` |
| Document | Your linked spreadsheet (search by name or paste ID) |
| Sheet | `Form Responses 1` |
| Polling Interval | Every 1 minute |

### What It Does

This node watches the `Form Responses 1` tab of your linked Google Sheet. Every time someone submits the Google Form, a new row appears in that sheet. n8n detects this row and triggers the entire workflow.

### Output Example

```json
{
  "Timestamp": "2024-11-15 10:32:44",
  "Full Name": "Sarah Keller",
  "Email": "sarah.keller@email.com",
  "Phone Number": "07712 345678",
  "Fabric Type": "Cotton Linen",
  "Fabric Condition": "Good – Minor wear",
  "Estimated Weight (kg)": "8",
  "Pickup Needed?": "Yes – Please collect from me",
  "Preferred Pickup Date": "2024-11-20",
  "Additional Notes": "Stored in bags in my garage."
}
```

---

## Node 2: Normalize Submission Data

**Node Type:** `Set`

**Rename this node to:** `Normalize Submission Data`

### What It Does

Cleans up incoming data: trims whitespace, normalizes the pickup field to a simple `Yes` or `No`, and converts weight from string to number. This makes all downstream expressions reliable.

### Configuration

Click **Add Value** for each of the following:

| Field Name | Type | Value |
|---|---|---|
| `name` | String | `{{ $json["Full Name"].trim() }}` |
| `email` | String | `{{ $json["Email"].trim().toLowerCase() }}` |
| `phone` | String | `{{ $json["Phone Number"] ? $json["Phone Number"].trim() : "Not provided" }}` |
| `fabricType` | String | `{{ $json["Fabric Type"].trim() }}` |
| `condition` | String | `{{ $json["Fabric Condition"].trim() }}` |
| `weightKg` | Number | `{{ parseFloat($json["Estimated Weight (kg)"]) }}` |
| `pickupRaw` | String | `{{ $json["Pickup Needed?"].trim() }}` |
| `pickupNeeded` | String | `{{ $json["Pickup Needed?"].includes("Yes") ? "Yes" : "No" }}` |
| `pickupDate` | String | `{{ $json["Preferred Pickup Date"] ? $json["Preferred Pickup Date"] : "N/A" }}` |
| `notes` | String | `{{ $json["Additional Notes"] ? $json["Additional Notes"].trim() : "None" }}` |

**Settings tab:**  
Set **Include Other Input Fields** → `false` (output only your clean fields)

### Output Example

```json
{
  "name": "Sarah Keller",
  "email": "sarah.keller@email.com",
  "phone": "07712 345678",
  "fabricType": "Cotton Linen",
  "condition": "Good – Minor wear",
  "weightKg": 8,
  "pickupNeeded": "Yes",
  "pickupDate": "2024-11-20",
  "notes": "Stored in bags in my garage."
}
```

---

## Node 3: Generate Submission ID + Timestamp

**Node Type:** `Code`

**Rename this node to:** `Generate Submission ID`

### What It Does

Creates a unique Submission ID (e.g., `SS-20241115-A3F7`) and captures the current ISO timestamp. These are essential for logging and deduplication.

### Code (JavaScript)

Paste this exactly into the code editor:

```javascript
const items = $input.all();

for (const item of items) {
  // Generate a short random alphanumeric suffix
  const randomSuffix = Math.random().toString(36).substring(2, 6).toUpperCase();

  // Get current date in YYYYMMDD format
  const now = new Date();
  const year = now.getFullYear();
  const month = String(now.getMonth() + 1).padStart(2, '0');
  const day = String(now.getDate()).padStart(2, '0');
  const dateStr = `${year}${month}${day}`;

  // Build Submission ID
  item.json.submissionId = `SS-${dateStr}-${randomSuffix}`;

  // ISO timestamp for logging
  item.json.isoTimestamp = now.toISOString();

  // Human-readable timestamp for Slack
  item.json.readableTimestamp = now.toLocaleString('en-GB', {
    day: '2-digit',
    month: 'long',
    year: 'numeric',
    hour: '2-digit',
    minute: '2-digit',
    timeZone: 'Europe/London'
  });
}

return items;
```

### Output Example

```json
{
  "name": "Sarah Keller",
  "email": "sarah.keller@email.com",
  "submissionId": "SS-20241115-A3F7",
  "isoTimestamp": "2024-11-15T10:32:44.000Z",
  "readableTimestamp": "15 November 2024, 10:32"
}
```

---

## Node 4: Validate Required Fields

**Node Type:** `IF`

**Rename this node to:** `Validate Required Fields`

### What It Does

Checks that all three critical fields are present and non-empty: name, email, and weightKg. If any are missing or invalid, the item is routed to the failure alert path.

### Configuration

**Combine:** `AND` (all conditions must be true)

| Condition | Left Value | Operation | Right Value |
|---|---|---|---|
| 1 | `{{ $json.name }}` | Is not empty | |
| 2 | `{{ $json.email }}` | Contains | `@` |
| 3 | `{{ $json.weightKg }}` | Is greater than | `0` |

- **True branch** → connects to Node 5b (Pickup Branch Router)
- **False branch** → connects to Node 5a (Validation Failure Alert)

---

## Node 5a: Validation Failure Alert

**Node Type:** `Slack`

**Rename this node to:** `Slack – Validation Failure Alert`

> This node is on the **FALSE** branch of Node 4.

### Configuration

| Parameter | Value |
|---|---|
| Credential | `Slack – Sprout Sew Bot` |
| Operation | `Send a message` |
| Channel | `#donations` |
| Message | *(see below)* |

### Message Text

```
⚠️ *Incomplete Donation Submission*

A form response was received but failed validation.

*Submission received at:* {{ $json.readableTimestamp }}
*Name provided:* {{ $json.name || "MISSING" }}
*Email provided:* {{ $json.email || "MISSING" }}
*Weight provided:* {{ $json.weightKg || "MISSING or ZERO" }}

Please check the form responses sheet manually and follow up if needed.
```

After this node → connect to the **End node** (Node 15) or simply leave unconnected (the branch terminates here).

---

## Node 5b: Pickup Branch Router

**Node Type:** `IF`

**Rename this node to:** `Pickup Branch Router`

> This node is on the **TRUE** branch of Node 4.

### What It Does

Routes the submission into one of two Slack message builder paths depending on whether pickup is needed.

### Configuration

| Condition | Left Value | Operation | Right Value |
|---|---|---|---|
| 1 | `{{ $json.pickupNeeded }}` | Is equal to | `Yes` |

- **True branch** → Node 6a (Build Slack Message – With Pickup)
- **False branch** → Node 6b (Build Slack Message – Drop Off)

---

## Node 6a: Build Slack Message – With Pickup

**Node Type:** `Set`

**Rename this node to:** `Build Slack Msg – Pickup Yes`

> This node is on the **TRUE** branch of Node 5b.

### Configuration

Add one field:

| Field Name | Type | Value |
|---|---|---|
| `slackMessage` | String | *(see below)* |

### Message Value

```
📦 *New Fabric Donation Offer – Sprout & Sew*

*Submission ID:* {{ $json.submissionId }}
*Received:* {{ $json.readableTimestamp }}

*Donor:* {{ $json.name }}
*Email:* {{ $json.email }}
*Phone:* {{ $json.phone }}

*Fabric Type:* {{ $json.fabricType }}
*Condition:* {{ $json.condition }}
*Estimated Weight:* {{ $json.weightKg }} kg

🚗 *Pickup Required:* Yes
📅 *Preferred Pickup Date:* {{ $json.pickupDate }}

📝 *Notes:* {{ $json.notes }}

Please respond to donor and arrange collection.
```

**Settings:** Keep input fields (do NOT exclude other fields)

---

## Node 6b: Build Slack Message – Drop Off

**Node Type:** `Set`

**Rename this node to:** `Build Slack Msg – Drop Off`

> This node is on the **FALSE** branch of Node 5b.

### Configuration

Add one field:

| Field Name | Type | Value |
|---|---|---|
| `slackMessage` | String | *(see below)* |

### Message Value

```
📦 *New Fabric Donation Offer – Sprout & Sew*

*Submission ID:* {{ $json.submissionId }}
*Received:* {{ $json.readableTimestamp }}

*Donor:* {{ $json.name }}
*Email:* {{ $json.email }}
*Phone:* {{ $json.phone }}

*Fabric Type:* {{ $json.fabricType }}
*Condition:* {{ $json.condition }}
*Estimated Weight:* {{ $json.weightKg }} kg

🏬 *Pickup Required:* No – Donor will drop off

📝 *Notes:* {{ $json.notes }}

Please contact donor to confirm drop-off details.
```

---

## Node 7: Merge Pickup Branches

**Node Type:** `Merge`

**Rename this node to:** `Merge Pickup Branches`

### What It Does

Rejoins the two Slack message builder paths into a single stream so downstream nodes don't need to be duplicated.

### Configuration

| Parameter | Value |
|---|---|
| Mode | `Append` |

**Input 1:** Output of Node 6a  
**Input 2:** Output of Node 6b

> In n8n, drag a connection from both Node 6a's output and Node 6b's output into the two input handles of this Merge node.

---

## Node 8: Send Donation Alert to Slack

**Node Type:** `Slack`

**Rename this node to:** `Slack – Send Donation Alert`

### Configuration

| Parameter | Value |
|---|---|
| Credential | `Slack – Sprout Sew Bot` |
| Operation | `Send a message` |
| Channel | `#donations` |
| Message | `{{ $json.slackMessage }}` |

> This sends the pre-built `slackMessage` field (from either Node 6a or 6b) to the #donations Slack channel.

---

## Node 9: Search Sheet for Existing Email

**Node Type:** `Google Sheets`

**Rename this node to:** `Sheets – Check for Duplicate`

### What It Does

Searches the `Donation Log` tab for any existing row where the **Email** column matches the current donor's email. This catches duplicate submissions.

### Configuration

| Parameter | Value |
|---|---|
| Credential | `Google – Sprout and Sew` |
| Operation | `Lookup` |
| Document | `Sprout & Sew – Donation Log` |
| Sheet | `Donation Log` |
| Lookup Column | `D` (Email column) |
| Lookup Value | `{{ $json.email }}` |

---

## Node 10: Duplicate Check Router

**Node Type:** `IF`

**Rename this node to:** `Duplicate Check Router`

### What It Does

If the Google Sheets Lookup returned any results, it means a row with this email already exists — this is a potential duplicate. Route accordingly.

### Configuration

| Condition | Left Value | Operation | Right Value |
|---|---|---|---|
| 1 | `{{ $json["Email"] }}` | Is not empty | |

> If the Sheets lookup returned rows, the output will contain the Email field from the found row. If empty, no match was found.

- **True branch** (email found = DUPLICATE) → Node 11b (Send Duplicate Warning)
- **False branch** (no match = NEW) → Node 11a (Calculate CO2)

---

## Node 11a: Calculate CO2 Saved Estimate

**Node Type:** `Code`

**Rename this node to:** `Calculate CO2 Saved`

> This node is on the **FALSE** branch of Node 10 (new submissions only).

### What It Does

Calculates an estimated CO2 / textile waste diversion figure based on submitted weight.

**Formula:** `CO2 Saved (kg) = Weight (kg) × 3.2`

### Code

```javascript
const items = $input.all();

for (const item of items) {
  const weight = parseFloat(item.json.weightKg) || 0;

  // 1 kg of donated fabric = 3.2 kg of textile waste diverted
  const co2Estimate = Math.round(weight * 3.2 * 100) / 100;

  item.json.co2SavedKg = co2Estimate;
  item.json.status = "New";
}

return items;
```

### Output Example

If `weightKg = 8`:

```json
{
  "co2SavedKg": 25.6,
  "status": "New"
}
```

---

## Node 11b: Send Duplicate Warning

**Node Type:** `Slack`

**Rename this node to:** `Slack – Duplicate Warning`

> This node is on the **TRUE** branch of Node 10.

### Configuration

| Parameter | Value |
|---|---|
| Credential | `Slack – Sprout Sew Bot` |
| Operation | `Send a message` |
| Channel | `#donations` |
| Message | *(see below)* |

### Message

```
🔁 *Possible Duplicate Donation Submission*

A new submission was received from an email address already in the Donation Log.

*Donor:* {{ $json.name }}
*Email:* {{ $json.email }}
*Submission ID:* {{ $json.submissionId }}
*Received:* {{ $json.readableTimestamp }}

Please review the Donation Log and confirm whether this is a new donation or a duplicate entry.
```

> After this node, connect to Node 15 (End) so the duplicate branch terminates cleanly.

---

## Node 12: Append Row to Google Sheets

**Node Type:** `Google Sheets`

**Rename this node to:** `Sheets – Append Donation Row`

> This node follows Node 11a (new submissions only).

### Configuration

| Parameter | Value |
|---|---|
| Credential | `Google – Sprout and Sew` |
| Operation | `Append or Update Row` |
| Document | `Sprout & Sew – Donation Log` |
| Sheet | `Donation Log` |
| Column Matching | `No matching column (always append)` |

### Field Mapping

Click **Add Field** for each row:

| Column Header | Expression |
|---|---|
| `Timestamp` | `{{ $json.isoTimestamp }}` |
| `Submission ID` | `{{ $json.submissionId }}` |
| `Full Name` | `{{ $json.name }}` |
| `Email` | `{{ $json.email }}` |
| `Phone` | `{{ $json.phone }}` |
| `Fabric Type` | `{{ $json.fabricType }}` |
| `Condition` | `{{ $json.condition }}` |
| `Weight KG` | `{{ $json.weightKg }}` |
| `Pickup Needed` | `{{ $json.pickupNeeded }}` |
| `Pickup Date` | `{{ $json.pickupDate }}` |
| `Notes` | `{{ $json.notes }}` |
| `CO2 Saved Estimate (kg)` | `{{ $json.co2SavedKg }}` |
| `Status` | `{{ $json.status }}` |

---

## Node 13: Build Success Summary

**Node Type:** `Set`

**Rename this node to:** `Build Success Summary`

### What It Does

Prepares a clean summary string for the final Slack confirmation message.

### Configuration

| Field Name | Type | Value |
|---|---|---|
| `successMessage` | String | *(see below)* |

### Value

```
✅ *Donation Record Logged Successfully*

*Submission ID:* {{ $json.submissionId }}
*Donor:* {{ $json.name }} ({{ $json.email }})
*Fabric:* {{ $json.fabricType }} – {{ $json.weightKg }} kg
*Condition:* {{ $json.condition }}
*CO2 Waste Diverted:* ~{{ $json.co2SavedKg }} kg

Row added to Donation Log ✔
```

---

## Node 14: Send Confirmation Message

**Node Type:** `Slack`

**Rename this node to:** `Slack – Send Confirmation`

### Configuration

| Parameter | Value |
|---|---|
| Credential | `Slack – Sprout Sew Bot` |
| Operation | `Send a message` |
| Channel | `#donations` |
| Message | `{{ $json.successMessage }}` |

---

## Node 15: End – Workflow Complete

**Node Type:** `NoOp`

**Rename this node to:** `End – Workflow Complete`

### What It Does

Acts as a clean terminal node. All active branches that reach their conclusion connect here. This makes the workflow canvas easier to read and confirms all paths have an intentional endpoint.

### Configuration

No configuration needed. Simply place this node and connect:
- Output of Node 14 → this node
- Output of Node 5a (validation failure) → this node
- Output of Node 11b (duplicate warning) → this node

---

# 8. Exact Field Mapping Reference

Use this reference table any time you need an expression in a node.

## From Google Form (Node 1 raw output)

| Form Field | n8n Expression |
|---|---|
| Full Name | `{{ $json["Full Name"] }}` |
| Email | `{{ $json["Email"] }}` |
| Phone Number | `{{ $json["Phone Number"] }}` |
| Fabric Type | `{{ $json["Fabric Type"] }}` |
| Fabric Condition | `{{ $json["Fabric Condition"] }}` |
| Estimated Weight (kg) | `{{ $json["Estimated Weight (kg)"] }}` |
| Pickup Needed? | `{{ $json["Pickup Needed?"] }}` |
| Preferred Pickup Date | `{{ $json["Preferred Pickup Date"] }}` |
| Additional Notes | `{{ $json["Additional Notes"] }}` |

## From Normalized Data (Node 2 onwards)

| Clean Field | n8n Expression |
|---|---|
| Name | `{{ $json.name }}` |
| Email | `{{ $json.email }}` |
| Phone | `{{ $json.phone }}` |
| Fabric Type | `{{ $json.fabricType }}` |
| Condition | `{{ $json.condition }}` |
| Weight | `{{ $json.weightKg }}` |
| Pickup Needed | `{{ $json.pickupNeeded }}` |
| Pickup Date | `{{ $json.pickupDate }}` |
| Notes | `{{ $json.notes }}` |
| Submission ID | `{{ $json.submissionId }}` |
| ISO Timestamp | `{{ $json.isoTimestamp }}` |
| Readable Timestamp | `{{ $json.readableTimestamp }}` |
| CO2 Saved | `{{ $json.co2SavedKg }}` |
| Status | `{{ $json.status }}` |
| Slack Message | `{{ $json.slackMessage }}` |

## Date & Time Utilities

| Use Case | Expression |
|---|---|
| Current ISO timestamp | `{{ $now.toISO() }}` |
| Today's date only | `{{ $today.toFormat('yyyy-MM-dd') }}` |
| Formatted readable | `{{ $now.toFormat('dd MMMM yyyy, HH:mm') }}` |

---

# 9. Slack Message Formatting

## Full Donation Alert (Pickup = Yes)

```
📦 *New Fabric Donation Offer – Sprout & Sew*

*Submission ID:* SS-20241115-A3F7
*Received:* 15 November 2024, 10:32

*Donor:* Sarah Keller
*Email:* sarah.keller@email.com
*Phone:* 07712 345678

*Fabric Type:* Cotton Linen
*Condition:* Good – Minor wear
*Estimated Weight:* 8 kg

🚗 *Pickup Required:* Yes
📅 *Preferred Pickup Date:* 2024-11-20

📝 *Notes:* Stored in bags in my garage.

Please respond to donor and arrange collection.
```

## Full Donation Alert (Pickup = No)

```
📦 *New Fabric Donation Offer – Sprout & Sew*

*Submission ID:* SS-20241115-B9Q2
*Received:* 15 November 2024, 14:15

*Donor:* Tom Nguyen
*Email:* tom.nguyen@email.com
*Phone:* Not provided

*Fabric Type:* Pure Linen
*Condition:* Excellent – Like new
*Estimated Weight:* 15 kg

🏬 *Pickup Required:* No – Donor will drop off

📝 *Notes:* Two large bin bags, very clean.

Please contact donor to confirm drop-off details.
```

## Success Confirmation

```
✅ *Donation Record Logged Successfully*

*Submission ID:* SS-20241115-A3F7
*Donor:* Sarah Keller (sarah.keller@email.com)
*Fabric:* Cotton Linen – 8 kg
*Condition:* Good – Minor wear
*CO2 Waste Diverted:* ~25.6 kg

Row added to Donation Log ✔
```

---

## Slack Formatting Reference

| Syntax | Effect |
|---|---|
| `*text*` | **Bold** |
| `_text_` | *Italic* |
| `` `text` `` | `Inline code` |
| `\n` | New line |
| `>text` | Block quote |

---

# 10. Sustainability Metric Logic

## Formula

```
CO2 Saved Estimate (kg) = Donated Weight (kg) × 3.2
```

## Background

The United Nations Environment Programme estimates that approximately **3.2 kg of CO2-equivalent emissions** are avoided per kilogram of textile waste that is diverted from landfill through reuse or upcycling. This is a conservative industry benchmark commonly used in sustainability reporting.

## Examples

| Donated Weight | CO2 / Waste Diverted |
|---|---|
| 1 kg | 3.2 kg |
| 5 kg | 16.0 kg |
| 10 kg | 32.0 kg |
| 15 kg | 48.0 kg |
| 25 kg | 80.0 kg |

## Code Used in Node 11a

```javascript
const weight = parseFloat(item.json.weightKg) || 0;
const co2Estimate = Math.round(weight * 3.2 * 100) / 100;
item.json.co2SavedKg = co2Estimate;
```

`Math.round(value * 100) / 100` rounds the result to exactly 2 decimal places to keep the spreadsheet values clean.

## How to Display Cumulative Impact

In Google Sheets, add this formula to a summary cell on the `Donation Log` tab:

```
=SUM(L2:L9999) & " kg of textile waste diverted 🌱"
```

You can place this in a dedicated **Dashboard** tab to give stakeholders a live sustainability score.

---

# 11. Error Handling Section

## 11.1 Enable Workflow Error Trigger

n8n allows you to create a **separate error workflow** that fires whenever any node in the main workflow fails.

**Steps:**

1. Create a new workflow called `Donation Routing – Error Handler`
2. Add an **Error Trigger** node (node type: `n8n Trigger → On Error`)
3. Add a **Slack node** after it
4. Configure the Slack message:

```
🔴 *Workflow Error – Fabric Donation Routing*

*Workflow:* {{ $json.workflow.name }}
*Failed Node:* {{ $json.execution.lastNodeExecuted }}
*Error:* {{ $json.error.message }}
*Time:* {{ $now.toISO() }}

Please check n8n execution logs immediately.
```

5. Send to: `#donations` or a private `#n8n-alerts` channel

6. Go back to your main workflow → **Settings** → **Error Workflow** → Select `Donation Routing – Error Handler`

---

## 11.2 Retry Strategy

For Google Sheets and Slack nodes specifically:

1. Click the node
2. Go to **Settings** tab
3. Set **Retry on Fail** → `On`
4. Set **Max Tries** → `3`
5. Set **Wait Between Tries** → `5000` ms (5 seconds)

This handles transient API rate limits or temporary connectivity drops.

---

## 11.3 Node-Level Try/Catch

In the Code nodes (Node 3, Node 11a), wrap code in try/catch:

```javascript
try {
  // your logic here
  return items;
} catch (error) {
  throw new Error(`Code node failed: ${error.message}`);
}
```

This surfaces clean error messages in the n8n execution log rather than cryptic internal errors.

---

# 12. Testing Procedure

## Step 1 – Activate the Workflow

1. Open your workflow in n8n
2. Click the **Active** toggle in the top-right corner
3. Status should show **Active** (green indicator)

---

## Step 2 – Submit a Test Form Response

1. Open your Google Form
2. Fill in all fields with test data:
   - **Full Name:** `Test Donor`
   - **Email:** `test.donor@sproutsew.com`
   - **Phone:** `07700 000000`
   - **Fabric Type:** `Cotton Linen`
   - **Fabric Condition:** `Excellent – Like new`
   - **Estimated Weight:** `10`
   - **Pickup Needed?:** `Yes – Please collect from me`
   - **Preferred Pickup Date:** *(next week)*
   - **Additional Notes:** `This is a test submission.`
3. Click **Submit**

---

## Step 3 – Check n8n Execution

1. In n8n, go to the workflow and click **Executions** (clock icon)
2. Within ~1 minute, a new execution should appear
3. Click on it to expand the execution view
4. Verify each node shows a **green tick** (success)
5. Click on any node to inspect its input/output data

**Expected execution path for a valid, non-duplicate submission:**

```
Node 1 ✅ → Node 2 ✅ → Node 3 ✅ → Node 4 ✅ (TRUE) →
Node 5b ✅ (TRUE/FALSE based on pickup) →
Node 6a or 6b ✅ → Node 7 ✅ → Node 8 ✅ →
Node 9 ✅ → Node 10 ✅ (FALSE – no duplicate) →
Node 11a ✅ → Node 12 ✅ → Node 13 ✅ → Node 14 ✅ → Node 15 ✅
```

---

## Step 4 – Verify Slack Message

1. Open your Slack workspace
2. Go to `#donations`
3. You should see:
   - One **donation alert** message (from Node 8)
   - One **confirmation message** (from Node 14)

---

## Step 5 – Verify Google Sheets Row

1. Open your `Sprout & Sew – Donation Log` spreadsheet
2. Click the `Donation Log` tab
3. A new row should appear with all 13 columns filled in
4. Confirm the `CO2 Saved Estimate (kg)` column shows `32` (10 kg × 3.2)
5. Confirm `Submission ID` is populated (e.g., `SS-20241115-A3F7`)

---

## Step 6 – Test Duplicate Detection

1. Submit the exact same form again using the same email address
2. Wait for the workflow to trigger
3. Check Slack `#donations` — you should see a **Duplicate Warning** message
4. Check Google Sheets — **no new row should be added**

---

## Step 7 – Test Validation Failure

1. Submit the form with:
   - Leave **Estimated Weight** blank *(or enter 0)*
2. Check Slack — you should see a **Validation Failure Alert** message
3. Google Sheets should not receive a new row

---

# 13. Final Deliverables Checklist

Complete these steps before presenting to a client or submitting as a project:

| # | Deliverable | How to Get It |
|---|---|---|
| 1 | Workflow JSON export | In n8n: top-right menu → Download → Save as `.json` |
| 2 | Workflow canvas screenshot | Zoom out to see all nodes → Take screenshot |
| 3 | Successful execution screenshot | Executions tab → click last run → screenshot all green nodes |
| 4 | Slack alert screenshot | Screenshot the #donations channel showing both Slack messages |
| 5 | Google Sheets screenshot | Screenshot the Donation Log tab with at least one row of data |
| 6 | Error handler workflow screenshot | Screenshot the error handler workflow canvas |

**Recommended naming convention for screenshots:**

```
sprout-sew-01-workflow-canvas.png
sprout-sew-02-execution-success.png
sprout-sew-03-slack-donation-alert.png
sprout-sew-04-slack-confirmation.png
sprout-sew-05-google-sheets-row.png
sprout-sew-06-error-handler.png
```

---

# 14. Troubleshooting Section

## Problem: Google Forms Trigger Not Firing

**Symptom:** You submit the form but n8n execution never starts.

**Causes and Fixes:**

| Cause | Fix |
|---|---|
| Workflow is inactive | Toggle the workflow to **Active** in n8n |
| Wrong sheet selected | Ensure the trigger is pointing to `Form Responses 1`, not `Donation Log` |
| Wrong spreadsheet ID | Double-check the ID in the URL of your linked sheet |
| OAuth2 expired | Go to Credentials → re-authenticate your Google account |
| Polling delay | Wait up to 2 minutes — n8n polls every 1 minute by default |

---

## Problem: Slack Auth Failed

**Symptom:** Slack node shows `invalid_auth` or `not_authed` error.

**Fixes:**

1. Go to [https://api.slack.com/apps](https://api.slack.com/apps)
2. Select your app → **OAuth & Permissions**
3. Confirm the bot token still starts with `xoxb-`
4. Re-install the app to workspace if needed
5. In n8n Credentials → delete and re-create the Slack credential
6. Paste the fresh token

---

## Problem: Google Sheets Permission Denied

**Symptom:** Google Sheets node returns `403 Forbidden` or `The caller does not have permission`.

**Fixes:**

1. Confirm your Google OAuth2 credential is authenticated with the account that **owns** the spreadsheet
2. Go to the spreadsheet → Share → confirm the authenticated email has Editor access
3. Re-authenticate the Google credential in n8n

---

## Problem: Weight Value Read as Text, Not Number

**Symptom:** The IF validation node returns false even when a weight is entered, or CO2 calculation returns `NaN`.

**Fix:**

Node 2 (Normalize Data) handles this with `parseFloat()`. Confirm:

```
{{ parseFloat($json["Estimated Weight (kg)"]) }}
```

If the value is still a string after Node 2, check that the Set node field type is set to **Number**, not **String**.

---

## Problem: Expressions Returning `undefined`

**Symptom:** Slack message shows `undefined` where a value should appear, or Google Sheets row shows blank cells.

**Causes and Fixes:**

| Cause | Fix |
|---|---|
| Referencing pre-normalize field names after Node 2 | Use `$json.name` not `$json["Full Name"]` after Node 2 |
| Node output is empty array | Check upstream IF node — item may be on the wrong branch |
| Merge node not connected correctly | Ensure both inputs of the Merge node are wired from Nodes 6a and 6b |
| Code node not returning `items` | Confirm your Code node ends with `return items;` |

---

## Problem: Duplicate Check Always Returns True

**Symptom:** Every submission is flagged as a duplicate, even first-time donors.

**Fix:**

The Google Sheets Lookup node returns an empty array `[]` if no match is found. The IF node checks `$json["Email"]` — if the lookup failed completely (wrong column, wrong sheet), the field may appear populated with an error object.

1. Click Node 9 → run manually with a test email → inspect the raw output
2. Confirm the `Lookup Column` is set to `D` (which is the Email column)
3. Confirm `Sheet` is `Donation Log` (not `Form Responses 1`)

---

## Problem: CO2 Calculation Showing 0

**Symptom:** The `co2SavedKg` field is `0` even with a valid weight.

**Fix:**

Check that `weightKg` is arriving at Node 11a as a **number**. If it arrives as a string (e.g. `"8"`), `parseFloat()` in the Code node still handles it. If `weightKg` itself is missing:

1. Open Node 11a → inspect the input data
2. Trace back to Node 2 and confirm the `weightKg` field is being set
3. Ensure the IF node (Node 4) is routing valid items to the TRUE path correctly

---

# 15. Pro Tips

## 15.1 Add a Workflow Description

In n8n, click the workflow name at the top → **Add Description** → Write:

```
Automated fabric donation intake pipeline for Sprout & Sew.
Captures Google Form submissions, sends Slack alerts, checks for duplicates,
logs to Google Sheets, and calculates CO2 sustainability estimates.
Version 1.0 | Built: November 2024
```

This looks extremely professional when you hand over the workflow JSON to a client.

---

## 15.2 Use Colour-Coded Node Backgrounds

In n8n, right-click any node → **Change Colour**. Use this system:

| Colour | Meaning |
|---|---|
| 🟢 Green | Data input / triggers |
| 🔵 Blue | Data transformation / formatting |
| 🟡 Yellow | Decision / routing nodes |
| 🟣 Purple | Slack notifications |
| 🔴 Red | Error / validation failure paths |

This makes your workflow canvas immediately readable and looks highly professional in client demos.

---

## 15.3 Add Sticky Notes to Your Canvas

n8n allows sticky notes (right-click on canvas → **Add Sticky Note**). Add at least:

- A title note at the top: `Sprout & Sew – Fabric Donation Routing v1.0`
- A note above the IF nodes explaining what they check
- A note on the CO2 node explaining the formula source

---

## 15.4 Tag All Credentials Consistently

Name all credentials with a prefix so they're easy to identify:

```
Google – Sprout and Sew
Slack – Sprout Sew Bot
```

This matters when you have multiple clients — credentials without context labels become confusing quickly.

---

## 15.5 Schedule a Weekly Summary (Bonus Automation)

Add a separate workflow that runs every Monday at 8:00 AM:

1. **Schedule Trigger** → every Monday 08:00
2. **Google Sheets** → Read all rows from `Donation Log`
3. **Code Node** → Sum total weight and CO2 from the past 7 days
4. **Slack Node** → Post to `#donations`:

```
📊 *Weekly Donation Summary – Sprout & Sew*

This week's intake:
• Total donations received: X
• Total fabric weight: X kg
• Estimated CO2 diverted: X kg

Keep up the great work! 🌿
```

This single extra workflow adds enormous perceived value to a client and takes under 30 minutes to build.

---

## 15.6 Version Control Your Workflow JSON

After every significant change:

1. Export the workflow JSON (top-right menu → Download)
2. Save it in a folder with a version number:
   ```
   donations-workflow-v1.0.json
   donations-workflow-v1.1-added-duplicate-check.json
   ```
3. Optionally push to a GitHub repo — this shows clients you apply engineering best practices

---

## 15.7 Build a Handover Document

When handing this project to a client, provide:

1. This implementation guide (what you're reading)
2. The workflow JSON file
3. A 1-page "How to Manage This Workflow" summary covering:
   - How to activate/deactivate the workflow
   - What to do if Slack messages stop arriving
   - How to check execution history in n8n
   - Contact info for support

Providing structured handover documentation elevates a freelance delivery from "good" to "premium agency-level."

---

*End of Implementation Guide*

---

**Document Version:** 1.0  
**Prepared for:** Sprout & Sew Fabric Donation Program  
**Built with:** n8n Automation Platform  
**Guide Author:** n8n Automation Architect  

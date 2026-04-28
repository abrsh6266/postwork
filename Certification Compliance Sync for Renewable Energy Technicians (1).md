# Certification Compliance Sync for Renewable Energy Technicians
### A Production-Grade n8n Automation Guide for Helios Current
**Version:** 1.0 | **Nodes:** 24 | **Difficulty:** Intermediate–Advanced | **Platform:** n8n (Cloud or Self-Hosted)

---

## Table of Contents

1. [Project Title & Overview](#1-project-title--overview)
2. [Business Problem Solved](#2-business-problem-solved)
3. [Full Workflow Architecture](#3-full-workflow-architecture)
4. [Required Accounts & Credentials](#4-required-accounts--credentials)
5. [Starter File Setup](#5-starter-file-setup)
6. [Airtable Dashboard Setup](#6-airtable-dashboard-setup)
7. [Slack Lead Routing Setup](#7-slack-lead-routing-setup)
8. [Full n8n Build Guide – Node by Node](#8-full-n8n-build-guide--node-by-node)
9. [Exact Field Mapping Reference](#9-exact-field-mapping-reference)
10. [Date Logic Section](#10-date-logic-section)
11. [Slack Message Templates](#11-slack-message-templates)
12. [Airtable Upsert Logic](#12-airtable-upsert-logic)
13. [Missing Date Handling](#13-missing-date-handling)
14. [Executive Daily Summary](#14-executive-daily-summary)
15. [Testing Procedure](#15-testing-procedure)
16. [Deliverables Checklist](#16-deliverables-checklist)
17. [Troubleshooting Guide](#17-troubleshooting-guide)
18. [Pro Freelancer Tips](#18-pro-freelancer-tips)

---

## 1. Project Title & Overview

**Project Name:** Certification Compliance Sync for Renewable Energy Technicians
**Client:** Helios Current — Regional Solar & Battery Installation Firm
**Automation Platform:** n8n (version 1.x+)

### What This Workflow Does

This automation monitors every certification expiry date across all Helios Current field technicians in four regions (North, South, East, West). It runs automatically every morning at 7:00 AM and whenever the source spreadsheet is updated. It reads every technician record, computes how many days remain before each certification expires, assigns a compliance status, writes everything to a central Airtable dashboard, fires Slack alerts to the correct regional leads, and sends a CEO-level daily summary to a leadership channel — all without a human touching a spreadsheet.

### Technician Certification Types Tracked

| Certification | Abbreviation |
|---|---|
| PV Installation | PV |
| OSHA 30 | OSHA |
| Battery Storage Specialist | BSS |
| NFPA 70E Electrical Safety | NFPA |
| Fall Protection | FP |
| EVSE Level 2 | EVSE |
| NABCEP PV Associate | NABCEP |
| Lead-Acid Safety | LAS |
| First Aid / CPR | FA |
| Rigging & Signalling | RS |

---

## 2. Business Problem Solved

### Current State (Before Automation)

Helios Current's HR Manager maintains a master Excel spreadsheet with 75+ certification rows spanning 12 technicians across four regions. Every Monday morning, the HR team manually opens the file, scrolls through expiry dates, and tries to flag upcoming expirations. Problems include:

| Problem | Business Impact |
|---|---|
| Expired certs discovered on job site | Safety liability, insurance void, potential OSHA violation |
| Missed renewal reminders | Technicians lapse, cannot legally work on PV systems |
| No regional routing | Regional leads don't receive alerts; only HR does |
| No executive visibility | Leadership has no dashboard view of compliance health |
| Spreadsheet chaos | Multiple copies, version conflicts, no audit trail |
| Manual effort | ~3 hours/week wasted on a process that can be automated |

### Future State (After Automation)

- ✅ Every certification is checked daily at 7:00 AM automatically
- ✅ Airtable is the single source of truth, always up to date
- ✅ Regional leads receive Slack alerts only for their technicians
- ✅ Leadership gets a daily digest with compliance counts
- ✅ Missing expiry dates are flagged — no silent gaps
- ✅ Zero risk of expired certs being overlooked
- ✅ Full audit trail via Airtable timestamps

---

## 3. Full Workflow Architecture

```
┌─────────────────────────────────────────────────────────┐
│                  TRIGGER LAYER                          │
│  [01] Cron Trigger – Daily 7 AM                         │
│  [02] Manual Trigger – On-Demand                        │
│  [03] Google Sheets Trigger – On Change (optional)      │
│              ↓                                          │
│  [04] Merge Node – Combine Trigger Sources              │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│                  DATA INGESTION LAYER                   │
│  [05] Read Spreadsheet – Load CSV / Google Sheet Data   │
│  [06] Set Node – Normalize & Clean Headers              │
│  [07] Split In Batches – Process Records One by One     │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│                  VALIDATION LAYER                       │
│  [08] IF Node – Expiry Date Exists?                     │
│         ├─ YES → [09] Code Node – Calculate Days        │
│         └─ NO  → [10] Set Node – Flag Missing Date      │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│                  ROUTING LAYER                          │
│  [11] Switch Node – Status Router                       │
│         ├─ Valid        → [12] Airtable Upsert Valid    │
│         ├─ Expiring Soon→ [13] Airtable Upsert Expiring │
│         ├─ Expired      → [14] Airtable Upsert Expired  │
│         └─ Missing Date → [15] Airtable Upsert Missing  │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│                  ALERTING LAYER                         │
│  [16] Set Node – Build Slack Alert (Expiring)           │
│  [17] Set Node – Region Channel Resolver                │
│  [18] Slack Node – Send Expiring Alert                  │
│  [19] Set Node – Build Slack Alert (Expired)            │
│  [20] Slack Node – Send Expired Alert                   │
│  [21] Set Node – Admin Notice (Missing Date)            │
│  [22] Slack Node – Admin Missing Date Alert             │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│                  SUMMARY LAYER                          │
│  [23] Merge Node – Rejoin All Branches                  │
│  [24] Aggregate Node – Daily Summary Counts             │
│  [25] Set Node – Format Executive Summary               │
│  [26] Slack Node – Executive Summary to #leadership     │
│  [27] NoOp Node – Final Success Logger                  │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│                  ERROR HANDLING LAYER                   │
│  [28] Error Trigger – Catch All Failures                │
│  [29] Set Node – Format Error Message                   │
│  [30] Slack Node – Send Failure Alert to #ops-alerts    │
└─────────────────────────────────────────────────────────┘
```

**Total Nodes: 30**

---

## 4. Required Accounts & Credentials

### 4.1 n8n

| Requirement | Detail |
|---|---|
| Version | 1.0+ (Cloud or self-hosted via Docker) |
| Plan | Starter or higher for Slack + Airtable integrations |
| URL | https://app.n8n.cloud or your self-hosted URL |

**Self-hosted quick start (Docker):**
```bash
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v ~/.n8n:/home/node/.n8n \
  n8nio/n8n
```
Then open `http://localhost:5678` in your browser.

---

### 4.2 Airtable

1. Go to [https://airtable.com](https://airtable.com) → create a free account
2. Navigate to **Account → Developer Hub → Personal Access Tokens**
3. Click **Create New Token**
4. Give it a name: `n8n-helios-token`
5. Scopes required:
   - `data.records:read`
   - `data.records:write`
   - `schema.bases:read`
6. Under Workspaces, select the workspace where your base will live
7. Click **Create Token** and copy the token (you will not see it again)

**In n8n:**
- Go to **Settings → Credentials → Add Credential**
- Search: `Airtable`
- Paste your Personal Access Token
- Name it: `Airtable – Helios Current`

---

### 4.3 Slack

1. Go to [https://api.slack.com/apps](https://api.slack.com/apps) → **Create New App → From Scratch**
2. Name: `Helios Compliance Bot`
3. Select your Slack workspace
4. Under **OAuth & Permissions**, add these Bot Token Scopes:
   - `chat:write`
   - `chat:write.public`
   - `channels:read`
5. Click **Install to Workspace** → **Allow**
6. Copy the **Bot User OAuth Token** (starts with `xoxb-`)

**In n8n:**
- Go to **Settings → Credentials → Add Credential**
- Search: `Slack`
- Select **Access Token** method
- Paste your Bot Token
- Name it: `Slack – Helios Current Bot`

**Create these Slack channels in your workspace:**

| Channel | Purpose |
|---|---|
| `#north-compliance` | North region alerts |
| `#south-compliance` | South region alerts |
| `#east-compliance` | East region alerts |
| `#west-compliance` | West region alerts |
| `#leadership` | Executive daily summary |
| `#ops-alerts` | Error/failure alerts |

Invite the Helios Compliance Bot to all six channels by typing `/invite @HeliosComplianceBot` in each.

---

### 4.4 Google Sheets (Optional — for live trigger)

1. Go to [https://console.cloud.google.com](https://console.cloud.google.com)
2. Create a new project: `Helios Compliance`
3. Enable APIs: **Google Sheets API** and **Google Drive API**
4. Go to **Credentials → Create Credentials → OAuth 2.0 Client ID**
5. Application type: **Web application**
6. Authorized redirect URI: `https://your-n8n-domain.com/rest/oauth2-credential/callback`
7. Download the JSON credentials

**In n8n:**
- Add Credential → `Google Sheets OAuth2 API`
- Paste Client ID and Client Secret
- Click **Sign in with Google** and authorize
- Name it: `Google Sheets – Helios Current`

---

## 5. Starter File Setup

The starter file is:

**`helios_compliance_sync_starter.csv`**

It contains 75 rows of technician certification data with these columns:

| Column | Example Value |
|---|---|
| Employee ID | HE-00001 |
| Technician Name | Marcus Thorne |
| Region | North |
| Certification Type | PV Installation |
| Expiry Date | 2023-11-12 |
| Supervisor Email | lead.North@helios-current.com |

> ⚠️ **Note:** Row HE-00008 (Carlos Mendez – Lead-Acid Safety) has a **blank Expiry Date** intentionally. Your workflow must handle this gracefully.

---

### Option A: Import into Google Sheets (Recommended for Live Trigger)

1. Go to [https://sheets.google.com](https://sheets.google.com) → **Blank spreadsheet**
2. Rename it: `Helios Compliance Master`
3. Go to **File → Import → Upload**
4. Upload `helios_compliance_sync_starter.csv`
5. Import Location: **Replace current sheet**
6. Separator: **Comma**
7. Click **Import Data**
8. Verify all 75 rows and 6 columns are present
9. Copy the Spreadsheet ID from the URL:
   `https://docs.google.com/spreadsheets/d/`**`YOUR_SPREADSHEET_ID`**`/edit`
10. You will paste this ID into Node 5 (Read Spreadsheet node)

---

### Option B: Use Read Binary File + Spreadsheet File Node in n8n

If you prefer not to use Google Sheets:

1. In n8n, add a **Read/Write Files from Disk** node (or use **HTTP Request** to fetch from a URL)
2. Upload your CSV to your n8n server at a known path, e.g., `/data/helios_compliance_sync_starter.csv`
3. Use the **Read/Write Files from Disk** node with:
   - **Operation:** `Read File`
   - **File Path:** `/data/helios_compliance_sync_starter.csv`
4. Pipe the output into a **Spreadsheet File** node:
   - **Operation:** `From File`
   - **File Format:** `CSV`
   - This converts the binary data into JSON rows automatically

For n8n Cloud users: Upload the CSV via the **n8n Form Trigger** or host it publicly and use **HTTP Request** to download it.

---

## 6. Airtable Dashboard Setup

### Step 1: Create the Base

1. Log in to Airtable → **Add a base → Start from scratch**
2. Name the base: `Compliance Dashboard`
3. Rename the default table to: `Technician Compliance`

### Step 2: Create the Fields

Delete all default fields except the first one (which becomes "Record ID"). Then add:

| # | Field Name | Airtable Field Type | Notes |
|---|---|---|---|
| 1 | Record ID | Single line text | Unique key: `{Employee ID}_{Certification Type}` |
| 2 | Technician Name | Single line text | From CSV |
| 3 | Region | Single select | Options: North, South, East, West |
| 4 | Certification Type | Single line text | From CSV |
| 5 | Expiry Date | Date | Date only (no time) |
| 6 | Days Remaining | Number | Integer |
| 7 | Compliance Status | Single select | Options: Valid, Expiring Soon, Expired, Missing Date |
| 8 | Supervisor Email | Email | From CSV |
| 9 | Last Synced | Date and time | Set by n8n on each run |
| 10 | Alert Sent | Checkbox | True when Slack alert fired |
| 11 | Action Required | Long text | Human-readable action note |

### Step 3: Configure Single Select Colors

In the **Compliance Status** field, set colors:

| Status | Color |
|---|---|
| Valid | Green |
| Expiring Soon | Yellow/Orange |
| Expired | Red |
| Missing Date | Gray |

### Step 4: Get Your Base ID and Table Name

- Base ID: Found in the Airtable API docs URL or in the URL when viewing the base: `https://airtable.com/`**`appXXXXXXXX`**`/...`
- Table Name: `Technician Compliance` (exactly as created above)

---

## 7. Slack Lead Routing Setup

### Channel Routing Table

| Region | Slack Channel | Supervisor Email |
|---|---|---|
| North | `#north-compliance` | lead.North@helios-current.com |
| South | `#south-compliance` | lead.South@helios-current.com |
| East | `#east-compliance` | lead.East@helios-current.com |
| West | `#west-compliance` | lead.West@helios-current.com |

### Option A: Route to Regional Slack Channels

This is the recommended approach. Node 17 (Region Channel Resolver) maps the Region field to the correct Slack channel name using a JavaScript expression. The Slack node then uses `{{$json["slackChannel"]}}` as the channel target.

### Option B: Route via Direct Message to Supervisor Email

If supervisors prefer DMs:
1. Use the Slack **Conversations** method to look up a user by email
2. Use the returned user ID as the channel in the Slack Send Message node
3. This requires the `users:read.email` scope in your Slack app

For this guide, we implement **Option A (channel routing)** as it provides a shared team record of alerts.

---

## 8. Full n8n Build Guide – Node by Node

> 🔧 **How to Add a Node in n8n:**  
> Click the **+** icon on any connection arrow, or press `Tab` to open the node picker. Search by name. After adding, rename every node by clicking its title at the top of the panel.

---

### Node 01: Cron Trigger – Daily 7 AM

| Setting | Value |
|---|---|
| **Node Type** | Schedule Trigger |
| **Rename To** | `⏰ Cron – Daily 7 AM` |
| **Credentials** | None required |

**Parameters:**

- Trigger Interval: `Custom (Cron Expression)`
- Cron Expression: `0 7 * * *`
  - This fires every day at 7:00 AM in the n8n server's timezone
- Timezone: Set your n8n instance timezone to `America/New_York` (or your business timezone) under **Settings → General**

**Purpose:**  
This is the primary automated trigger. Every weekday morning, compliance data is pulled, classified, and reported before the first field crew assignment goes out.

**Example Output:**
```json
{
  "timestamp": "2024-05-20T07:00:00.000Z"
}
```

**Why needed:**  
Automation fails if it only runs on-demand. The daily cron ensures zero human intervention is needed to maintain compliance visibility.

---

### Node 02: Manual Trigger – On-Demand Run

| Setting | Value |
|---|---|
| **Node Type** | Manual Trigger |
| **Rename To** | `🖐 Manual Trigger – On Demand` |
| **Credentials** | None required |

**Parameters:**  
No configuration needed. This trigger fires when you click **Execute Workflow** in the n8n canvas.

**Purpose:**  
Used for testing, ad-hoc runs, or when HR needs to re-sync after updating the spreadsheet immediately.

**Why needed:**  
During development and testing, you will click this hundreds of times. It also gives the HR Manager a button to re-run the sync instantly after editing a record.

---

### Node 03: Google Sheets Trigger – On Change

| Setting | Value |
|---|---|
| **Node Type** | Google Sheets Trigger |
| **Rename To** | `📄 GSheets Trigger – On Change` |
| **Credentials** | `Google Sheets – Helios Current` |

**Parameters:**

- **Event:** `Row Added or Updated`
- **Spreadsheet:** Paste your Google Sheets URL or Spreadsheet ID
- **Sheet:** `Sheet1` (or whatever your sheet tab is named)
- **Poll Time:** `Every 15 Minutes`

> ℹ️ **Note:** Google Sheets Trigger works via polling (n8n checks every N minutes). It is not a true webhook unless you add an Apps Script. The 15-minute interval is a good balance between responsiveness and API quota.

**Purpose:**  
If the HR Manager updates a date or adds a row in Google Sheets, this trigger catches the change and re-syncs within 15 minutes.

**Why needed:**  
Provides near-real-time sync without requiring HR to manually trigger the workflow after every edit.

---

### Node 04: Merge – Combine Trigger Sources

| Setting | Value |
|---|---|
| **Node Type** | Merge |
| **Rename To** | `🔀 Merge – Combine Triggers` |
| **Credentials** | None |

**Parameters:**

- **Mode:** `Append`
- Connect **all three trigger nodes** (Nodes 01, 02, 03) as inputs to this Merge node

**Purpose:**  
n8n workflows can only have one active path at a time. This Merge node collects output from whichever trigger fired and passes a single execution signal downstream. In Append mode, it passes through whatever data arrives from any connected input.

**Why needed:**  
Without this node, you would need three separate duplicate workflows — one per trigger. The Merge node unifies them into one maintainable pipeline.

> 💡 **Tip:** After connecting all three triggers, right-click the Merge node and select **Set as Start** so the canvas renders cleanly.

---

### Node 05: Read Spreadsheet – Load Technician Data

| Setting | Value |
|---|---|
| **Node Type** | `Google Sheets` (if using Google Sheets) OR `Spreadsheet File` (if using CSV) |
| **Rename To** | `📥 Read Sheet – Technician Data` |
| **Credentials** | `Google Sheets – Helios Current` |

**Parameters (Google Sheets method):**

- **Operation:** `Get Many Rows`
- **Spreadsheet:** [Your Spreadsheet ID]
- **Sheet:** `Sheet1`
- **Return All:** `ON` (toggle enabled)
- **Options → First Row is Header Row:** `ON`

**Parameters (CSV / Spreadsheet File method):**

- **Node Type:** Spreadsheet File
- **Operation:** `From File`
- **Input Binary Field:** `data`
- **File Format:** `CSV`

**Example Output (first row):**
```json
{
  "Employee ID": "HE-00001",
  "Technician Name": "Marcus Thorne",
  "Region": "North",
  "Certification Type": "PV Installation",
  "Expiry Date": "2023-11-12",
  "Supervisor Email": "lead.North@helios-current.com"
}
```

**Why needed:**  
This is the source of truth. All 75 rows are loaded here and flow downstream for processing.

---

### Node 06: Set – Normalize & Clean Headers

| Setting | Value |
|---|---|
| **Node Type** | Set |
| **Rename To** | `🧹 Set – Normalize Headers` |
| **Credentials** | None |

**Parameters:**

Enable **"Keep Only Set"** toggle = **OFF** (keep all existing fields, just add/rename)

Add these fields (use **Expression mode** for values):

| Field Name | Value Expression |
|---|---|
| `employeeId` | `{{$json["Employee ID"]}}` |
| `technicianName` | `{{$json["Technician Name"]}}` |
| `region` | `{{$json["Region"]}}` |
| `certType` | `{{$json["Certification Type"]}}` |
| `expiryDate` | `{{$json["Expiry Date"]}}` |
| `supervisorEmail` | `{{$json["Supervisor Email"]}}` |
| `recordId` | `{{$json["Employee ID"] + "_" + $json["Certification Type"].replace(/ /g, "_")}}` |

**Purpose:**  
CSV headers with spaces are error-prone inside n8n expressions. This node creates clean camelCase aliases and generates a unique `recordId` key used for Airtable upsert deduplication.

**Example Output (added fields only):**
```json
{
  "recordId": "HE-00001_PV_Installation",
  "technicianName": "Marcus Thorne",
  "region": "North",
  "certType": "PV Installation",
  "expiryDate": "2023-11-12",
  "supervisorEmail": "lead.North@helios-current.com"
}
```

**Why needed:**  
Prevents `undefined` errors downstream when referencing fields with spaces. The `recordId` field is critical for deduplication logic.

---

### Node 07: Split In Batches – Process One Record at a Time

| Setting | Value |
|---|---|
| **Node Type** | Split In Batches |
| **Rename To** | `📦 Split – One Record at a Time` |
| **Credentials** | None |

**Parameters:**

- **Batch Size:** `1`
- **Options → Reset:** `OFF`

**Purpose:**  
Processes each of the 75 technician-certification rows individually so each record can flow independently through the IF node, Code node, Switch node, and Airtable upsert.

**Why needed:**  
Without batching, all 75 rows hit every node simultaneously. n8n handles them in parallel but with batch size 1 you get clean per-record routing and error isolation — if one record fails, the others continue.

---

### Node 08: IF – Expiry Date Exists?

| Setting | Value |
|---|---|
| **Node Type** | IF |
| **Rename To** | `❓ IF – Expiry Date Exists?` |
| **Credentials** | None |

**Parameters:**

- **Condition:** `String`
- **Value 1:** `{{$json["expiryDate"]}}`
- **Operation:** `is not empty`

**True Branch (output 1):** Record has an expiry date → go to Code Node  
**False Branch (output 2):** Record has no expiry date → go to Missing Date Set Node

**Why needed:**  
The CSV contains at least one intentionally blank expiry date (HE-00008, Carlos Mendez). Without this guard, the Code Node would crash on `null` date arithmetic. This IF node routes bad data safely.

---

### Node 09: Code – Calculate Days Remaining

| Setting | Value |
|---|---|
| **Node Type** | Code |
| **Rename To** | `🧮 Code – Calculate Days Remaining` |
| **Credentials** | None |
| **Language** | JavaScript |

**Parameters:**

- **Mode:** `Run Once for Each Item`

**Code:**

```javascript
const items = $input.all();
const results = [];

for (const item of items) {
  const data = { ...item.json };
  
  // Parse the expiry date from the record
  const expiryRaw = data.expiryDate;
  
  if (!expiryRaw || expiryRaw.trim() === '') {
    // Safety net — should be caught by IF node, but handle gracefully
    data.daysRemaining = null;
    data.complianceStatus = 'Missing Date';
    data.actionRequired = 'Expiry date not recorded. Please update the spreadsheet.';
    results.push({ json: data });
    continue;
  }
  
  // Parse dates — strip time component for comparison
  const today = new Date();
  today.setHours(0, 0, 0, 0);
  
  const expiry = new Date(expiryRaw);
  expiry.setHours(0, 0, 0, 0);
  
  // Check for invalid date
  if (isNaN(expiry.getTime())) {
    data.daysRemaining = null;
    data.complianceStatus = 'Missing Date';
    data.actionRequired = `Expiry date "${expiryRaw}" could not be parsed. Fix format to YYYY-MM-DD.`;
    results.push({ json: data });
    continue;
  }
  
  // Calculate difference in whole days
  const msPerDay = 1000 * 60 * 60 * 24;
  const daysRemaining = Math.round((expiry.getTime() - today.getTime()) / msPerDay);
  
  data.daysRemaining = daysRemaining;
  
  // Assign compliance status
  if (daysRemaining < 0) {
    data.complianceStatus = 'Expired';
    data.actionRequired = `Certification expired ${Math.abs(daysRemaining)} day(s) ago. Remove technician from active scheduling immediately.`;
  } else if (daysRemaining <= 30) {
    data.complianceStatus = 'Expiring Soon';
    data.actionRequired = `Certification expires in ${daysRemaining} day(s). Schedule renewal now.`;
  } else {
    data.complianceStatus = 'Valid';
    data.actionRequired = 'No action required.';
  }
  
  // Add sync timestamp
  data.lastSynced = new Date().toISOString();
  data.alertSent = false;
  
  results.push({ json: data });
}

return results;
```

**Example Output (for Marcus Thorne, assuming today is 2024-05-20):**
```json
{
  "technicianName": "Marcus Thorne",
  "region": "North",
  "certType": "PV Installation",
  "expiryDate": "2023-11-12",
  "daysRemaining": -189,
  "complianceStatus": "Expired",
  "actionRequired": "Certification expired 189 day(s) ago. Remove technician from active scheduling immediately.",
  "lastSynced": "2024-05-20T07:00:00.000Z",
  "alertSent": false
}
```

**Why needed:**  
All classification logic lives here. The Switch node (Node 11) reads `complianceStatus` set by this code. Centralizing the logic in one Code Node makes it easy to adjust thresholds (e.g., change "Expiring Soon" from 30 to 45 days) without touching multiple nodes.

---

### Node 10: Set – Flag Missing Date

| Setting | Value |
|---|---|
| **Node Type** | Set |
| **Rename To** | `⚠️ Set – Flag Missing Date` |
| **Credentials** | None |

This node receives records from the **False branch** of Node 08.

**Parameters:**

Enable **"Keep Only Set"** = **OFF** (preserve all existing fields)

Add these fields:

| Field Name | Value |
|---|---|
| `daysRemaining` | `null` (leave blank / no value) |
| `complianceStatus` | `Missing Date` (fixed string) |
| `actionRequired` | `Expiry date not recorded. Please update the master spreadsheet.` |
| `lastSynced` | `{{new Date().toISOString()}}` |
| `alertSent` | `false` |

**Why needed:**  
Records without expiry dates bypass the Code Node. This Set Node stamps them with the `Missing Date` status so they flow correctly into the Switch Node and are upserted to Airtable with a clear status.

---

### Node 11: Switch – Status Router

| Setting | Value |
|---|---|
| **Node Type** | Switch |
| **Rename To** | `🔀 Switch – Status Router` |
| **Credentials** | None |

**Parameters:**

- **Value:** `{{$json["complianceStatus"]}}`
- **Routing Mode:** `Rules`

| Rule # | Operation | Value | Output |
|---|---|---|---|
| 1 | `equals` | `Valid` | Output 1 |
| 2 | `equals` | `Expiring Soon` | Output 2 |
| 3 | `equals` | `Expired` | Output 3 |
| 4 | `equals` | `Missing Date` | Output 4 |

- **Fallback Output:** Enable "Send data to fallback output" = ON (catches any unexpected values)

**Why needed:**  
This is the core router. Every record exits through exactly one of four outputs, sending it to the correct Airtable upsert node and alert path.

---

### Node 12: Airtable – Upsert Valid

| Setting | Value |
|---|---|
| **Node Type** | Airtable |
| **Rename To** | `✅ Airtable – Upsert Valid` |
| **Credentials** | `Airtable – Helios Current` |

**Parameters:**

- **Operation:** `Upsert`
- **Base:** `Compliance Dashboard`
- **Table:** `Technician Compliance`
- **Fields to Match On:** `Record ID` → `{{$json["recordId"]}}`

**Fields to Send:**

| Airtable Field | n8n Expression |
|---|---|
| Record ID | `{{$json["recordId"]}}` |
| Technician Name | `{{$json["technicianName"]}}` |
| Region | `{{$json["region"]}}` |
| Certification Type | `{{$json["certType"]}}` |
| Expiry Date | `{{$json["expiryDate"]}}` |
| Days Remaining | `{{$json["daysRemaining"]}}` |
| Compliance Status | `{{$json["complianceStatus"]}}` |
| Supervisor Email | `{{$json["supervisorEmail"]}}` |
| Last Synced | `{{$json["lastSynced"]}}` |
| Alert Sent | `false` |
| Action Required | `{{$json["actionRequired"]}}` |

**Connected to:** Output 1 of Node 11 (Switch – Valid branch)

**Why needed:**  
Valid records are upserted (created if new, updated if existing) so the Airtable dashboard always shows current status. No Slack alert is sent for valid records.

---

### Node 13: Airtable – Upsert Expiring Soon

| Setting | Value |
|---|---|
| **Node Type** | Airtable |
| **Rename To** | `⚠️ Airtable – Upsert Expiring Soon` |
| **Credentials** | `Airtable – Helios Current` |

**Parameters:** Identical to Node 12 with one difference:
- **Alert Sent field:** `true` (this record will trigger a Slack alert)

**Connected to:** Output 2 of Node 11 (Switch – Expiring Soon branch)

After this node, connect to **Node 16 (Set – Build Slack Alert Expiring)**.

---

### Node 14: Airtable – Upsert Expired

| Setting | Value |
|---|---|
| **Node Type** | Airtable |
| **Rename To** | `🚨 Airtable – Upsert Expired` |
| **Credentials** | `Airtable – Helios Current` |

**Parameters:** Identical to Node 12. Alert Sent = `true`.

**Connected to:** Output 3 of Node 11 (Switch – Expired branch)

After this node, connect to **Node 19 (Set – Build Slack Alert Expired)**.

---

### Node 15: Airtable – Upsert Missing Date

| Setting | Value |
|---|---|
| **Node Type** | Airtable |
| **Rename To** | `❓ Airtable – Upsert Missing Date` |
| **Credentials** | `Airtable – Helios Current` |

**Parameters:** Identical to Node 12. Expiry Date and Days Remaining fields should be left blank/null.

**Connected to:** Output 4 of Node 11 (Switch – Missing Date branch)

After this node, connect to **Node 21 (Set – Admin Notice Missing Date)**.

---

### Node 16: Set – Build Slack Alert (Expiring Soon)

| Setting | Value |
|---|---|
| **Node Type** | Set |
| **Rename To** | `📝 Set – Build Expiring Alert` |
| **Credentials** | None |

**Parameters:**

Keep existing fields. Add:

| Field | Expression |
|---|---|
| `slackAlertType` | `expiring` |
| `slackMessage` | See expression below |

**Slack Message Expression (paste into Expression editor):**
```
⚠️ *Certification Expiring Soon*

*Technician:* {{$json["technicianName"]}}
*Employee ID:* {{$json["employeeId"]}}
*Region:* {{$json["region"]}}
*Certification:* {{$json["certType"]}}
*Expiry Date:* {{$json["expiryDate"]}}
*Expires In:* {{$json["daysRemaining"]}} days
*Supervisor:* {{$json["supervisorEmail"]}}

*Action Required:* Schedule renewal immediately. Technician must not be assigned to jobs after expiry.
```

**Connected to:** Output of Node 13 (Airtable Expiring Soon)

---

### Node 17: Set – Region Channel Resolver

| Setting | Value |
|---|---|
| **Node Type** | Set |
| **Rename To** | `🗺️ Set – Region Channel Resolver` |
| **Credentials** | None |

**Parameters:**

Add field `slackChannel` with this expression:

```javascript
{{
  $json["region"] === "North" ? "#north-compliance" :
  $json["region"] === "South" ? "#south-compliance" :
  $json["region"] === "East" ? "#east-compliance" :
  $json["region"] === "West" ? "#west-compliance" :
  "#ops-alerts"
}}
```

**Connect Node 16 → Node 17 → Node 18**
**Also connect Node 19 → Node 17 (reuse for expired alerts)**

> 💡 Use a **Merge node** before Node 17 to combine the Expiring and Expired alert paths into one, then resolve the channel for both in a single node.

**Why needed:**  
Dynamically routes each alert to the correct regional channel without hardcoding. If a region value is unexpected, it falls back to `#ops-alerts`.

---

### Node 18: Slack – Send Expiring Alert

| Setting | Value |
|---|---|
| **Node Type** | Slack |
| **Rename To** | `💬 Slack – Send Expiring Alert` |
| **Credentials** | `Slack – Helios Current Bot` |

**Parameters:**

- **Resource:** `Message`
- **Operation:** `Send`
- **Channel:** `{{$json["slackChannel"]}}`
- **Text:** `{{$json["slackMessage"]}}`
- **Options:**
  - **Include Link to Workflow:** OFF
  - **Username:** `Helios Compliance Bot`

**Why needed:**  
This fires the actual Slack message to the regional channel. Each expiring record produces one alert message. The regional channel ensures the right lead sees it.

---

### Node 19: Set – Build Slack Alert (Expired)

| Setting | Value |
|---|---|
| **Node Type** | Set |
| **Rename To** | `📝 Set – Build Expired Alert` |
| **Credentials** | None |

**Parameters:**

Add field `slackMessage`:

```
🚨 *Certification EXPIRED — Immediate Action Required*

*Technician:* {{$json["technicianName"]}}
*Employee ID:* {{$json["employeeId"]}}
*Region:* {{$json["region"]}}
*Certification:* {{$json["certType"]}}
*Expired:* {{Math.abs($json["daysRemaining"])}} days ago ({{$json["expiryDate"]}})
*Supervisor:* {{$json["supervisorEmail"]}}

*⛔ Action Required:* Remove from active scheduling immediately. Do NOT assign to job sites until certification is renewed and on file.
```

**Connected to:** Output of Node 14 (Airtable Expired) → Node 17 → Node 18

---

### Node 20: Slack – Send Expired Alert

This node is shared with the expiring alert path. After Node 17 resolves the channel, both expiring and expired alerts flow through a **single Slack send node** because the channel resolution and send logic are identical. Rename Node 18 to handle both:

| Setting | Value |
|---|---|
| **Rename To** | `💬 Slack – Send Compliance Alert` |

**This handles both Expiring Soon and Expired alerts.** The message content differs because it was set in Node 16 (expiring) or Node 19 (expired) before reaching Node 17.

---

### Node 21: Set – Admin Notice (Missing Date)

| Setting | Value |
|---|---|
| **Node Type** | Set |
| **Rename To** | `📝 Set – Missing Date Notice` |
| **Credentials** | None |

**Parameters:**

Add field `slackMessage`:

```
🔔 *Missing Certification Date — Admin Notice*

*Technician:* {{$json["technicianName"]}}
*Employee ID:* {{$json["employeeId"]}}
*Region:* {{$json["region"]}}
*Certification:* {{$json["certType"]}}
*Supervisor:* {{$json["supervisorEmail"]}}

*Action:* No expiry date is recorded for this certification. Please update the master spreadsheet as soon as possible. This record cannot be compliance-checked until a date is provided.
```

Add field `slackChannel`: `#ops-alerts`

**Connected to:** Output of Node 15 (Airtable Missing Date)

---

### Node 22: Slack – Admin Missing Date Alert

| Setting | Value |
|---|---|
| **Node Type** | Slack |
| **Rename To** | `💬 Slack – Missing Date Alert` |
| **Credentials** | `Slack – Helios Current Bot` |

**Parameters:**

- **Channel:** `{{$json["slackChannel"]}}`  (resolves to `#ops-alerts`)
- **Text:** `{{$json["slackMessage"]}}`

**Connected to:** Output of Node 21

---

### Node 23: Merge – Rejoin All Branches

| Setting | Value |
|---|---|
| **Node Type** | Merge |
| **Rename To** | `🔀 Merge – Rejoin All Branches` |
| **Credentials** | None |

**Parameters:**

- **Mode:** `Append`
- **Number of Inputs:** `4`

Connect the outputs of:
- Node 12 (Airtable Valid)
- Node 18 (Slack Expiring/Expired)
- Node 22 (Slack Missing Date)
- Node 15 (Airtable Missing Date)

**Purpose:**  
After processing, all four branches rejoin here. The aggregation node downstream needs all records to compute totals.

**Why needed:**  
Without rejoining, the Summary node cannot see records from all branches. This Merge collects every processed record into one stream.

---

### Node 24: Aggregate – Daily Summary Counts

| Setting | Value |
|---|---|
| **Node Type** | Aggregate |
| **Rename To** | `📊 Aggregate – Daily Summary` |
| **Credentials** | None |

**Parameters:**

- **Aggregate:** `All Item Data (Into a Single List)`
- **Put Output in Field:** `allRecords`

**Then add a Code Node immediately after with this logic:**

```javascript
const allRecords = $input.first().json.allRecords;

let valid = 0;
let expiringSoon = 0;
let expired = 0;
let missingDate = 0;

for (const record of allRecords) {
  const status = record.complianceStatus;
  if (status === 'Valid') valid++;
  else if (status === 'Expiring Soon') expiringSoon++;
  else if (status === 'Expired') expired++;
  else if (status === 'Missing Date') missingDate++;
}

const total = valid + expiringSoon + expired + missingDate;
const runDate = new Date().toLocaleDateString('en-US', {
  weekday: 'long', year: 'numeric', month: 'long', day: 'numeric'
});

return [{
  json: {
    valid,
    expiringSoon,
    expired,
    missingDate,
    total,
    runDate,
    summaryMessage: `📋 *Helios Current — Daily Compliance Summary*\n*Date:* ${runDate}\n*Total Records Checked:* ${total}\n\n✅ Valid: ${valid}\n⚠️ Expiring Soon (≤30 days): ${expiringSoon}\n🚨 Expired: ${expired}\n❓ Missing Date: ${missingDate}\n\n${expired > 0 || expiringSoon > 0 ? '⚠️ Action required — see regional channels for details.' : '🎉 All certifications are current. Great job!'}`
  }
}];
```

**Rename this Code Node:** `🧮 Code – Build Summary Message`

---

### Node 25: Set – Format Executive Summary

| Setting | Value |
|---|---|
| **Node Type** | Set |
| **Rename To** | `📋 Set – Format Executive Summary` |
| **Credentials** | None |

This node is optional if the Code Node above already formats the full message. Use it to add any additional metadata:

| Field | Value |
|---|---|
| `executiveChannel` | `#leadership` |
| `reportReady` | `true` |

---

### Node 26: Slack – Executive Summary

| Setting | Value |
|---|---|
| **Node Type** | Slack |
| **Rename To** | `📊 Slack – Executive Summary` |
| **Credentials** | `Slack – Helios Current Bot` |

**Parameters:**

- **Channel:** `#leadership`
- **Text:** `{{$json["summaryMessage"]}}`

**Example Output in Slack:**
```
📋 Helios Current — Daily Compliance Summary
Date: Monday, May 20, 2024
Total Records Checked: 75

✅ Valid: 22
⚠️ Expiring Soon (≤30 days): 14
🚨 Expired: 38
❓ Missing Date: 1

⚠️ Action required — see regional channels for details.
```

**Why needed:**  
Leadership needs a single daily message that summarizes compliance health across all regions. This avoids executives having to check four separate regional channels.

---

### Node 27: NoOp – Final Success Logger

| Setting | Value |
|---|---|
| **Node Type** | NoOp (No Operation) |
| **Rename To** | `✅ Logger – Workflow Complete` |

**Purpose:**  
A visual endpoint for the workflow. When you see green on this node, the full run completed successfully. Useful for monitoring in n8n's execution history.

**Why needed:**  
Clean workflows always have a defined end. This also serves as an anchor point if you later want to add post-processing (e.g., write a log file, trigger another workflow).

---

### Node 28: Error Trigger – Catch All Failures

| Setting | Value |
|---|---|
| **Node Type** | Error Trigger |
| **Rename To** | `🚨 Error Trigger – Catch Failures` |

**Parameters:**

This node does not need configuration. It automatically fires when any node in the workflow throws an uncaught error.

**How to activate:**
1. Go to your workflow **Settings** (gear icon in top toolbar)
2. Under **Error Workflow**, select this workflow itself (or create a separate error-handling sub-workflow)

**Why needed:**  
Without error handling, a failure (e.g., Airtable API rate limit, Slack token expiry) silently stops the workflow. This node ensures failures are always reported.

---

### Node 29: Set – Format Error Message

| Setting | Value |
|---|---|
| **Node Type** | Set |
| **Rename To** | `📝 Set – Format Error Message` |

**Parameters:**

Add field `errorSlackMessage`:

```
🔴 *Helios Compliance Workflow — FAILURE ALERT*

*Workflow:* Certification Compliance Sync
*Time:* {{new Date().toISOString()}}
*Error:* {{$json["error"]["message"]}}
*Node:* {{$json["error"]["node"]["name"]}}

*Action:* Check n8n execution history immediately and investigate the failed node. Compliance sync did NOT complete for this run.
```

Add field `errorChannel`: `#ops-alerts`

---

### Node 30: Slack – Send Failure Alert

| Setting | Value |
|---|---|
| **Node Type** | Slack |
| **Rename To** | `💬 Slack – Failure Alert` |
| **Credentials** | `Slack – Helios Current Bot` |

**Parameters:**

- **Channel:** `#ops-alerts`
- **Text:** `{{$json["errorSlackMessage"]}}`

**Why needed:**  
The ops team must know immediately when the compliance workflow breaks. A silent failure could mean technicians go unmonitored for days.

---

## 9. Exact Field Mapping Reference

Use these exact n8n expressions throughout your workflow:

| Data Point | Expression |
|---|---|
| Employee ID | `{{$json["employeeId"]}}` |
| Technician Name | `{{$json["technicianName"]}}` |
| Region | `{{$json["region"]}}` |
| Certification Type | `{{$json["certType"]}}` |
| Expiry Date (raw) | `{{$json["expiryDate"]}}` |
| Days Remaining | `{{$json["daysRemaining"]}}` |
| Compliance Status | `{{$json["complianceStatus"]}}` |
| Supervisor Email | `{{$json["supervisorEmail"]}}` |
| Record ID (unique key) | `{{$json["recordId"]}}` |
| Last Synced | `{{$json["lastSynced"]}}` |
| Current timestamp | `{{$now.toISOString()}}` |
| Absolute days expired | `{{Math.abs($json["daysRemaining"])}}` |
| Slack channel | `{{$json["slackChannel"]}}` |
| Slack message | `{{$json["slackMessage"]}}` |

### CSV Raw Column Headers (before normalization in Node 06)

| CSV Header | Raw n8n Expression |
|---|---|
| Employee ID | `{{$json["Employee ID"]}}` |
| Technician Name | `{{$json["Technician Name"]}}` |
| Region | `{{$json["Region"]}}` |
| Certification Type | `{{$json["Certification Type"]}}` |
| Expiry Date | `{{$json["Expiry Date"]}}` |
| Supervisor Email | `{{$json["Supervisor Email"]}}` |

> ⚠️ Always use normalized expressions (camelCase) from Node 06 onwards. Never rely on raw CSV headers downstream.

---

## 10. Date Logic Section

### Overview of Thresholds

| Condition | Days Remaining | Status |
|---|---|---|
| Expires in 90+ days | `> 30` | `Valid` |
| Expires in 1–30 days | `>= 0` and `<= 30` | `Expiring Soon` |
| Expired | `< 0` | `Expired` |
| No date provided | null / empty string | `Missing Date` |
| Invalid date format | NaN from parse | `Missing Date` |

### Date Calculation Examples

Assuming today is **2024-05-20**:

| Record | Expiry Date | Calculation | Result |
|---|---|---|---|
| HE-00003 David Chen | 2025-08-22 | 459 days away | ✅ Valid |
| HE-00007 Yuki Tanaka | 2024-05-28 | 8 days away | ⚠️ Expiring Soon |
| HE-00001 Marcus Thorne | 2023-11-12 | 189 days ago | 🚨 Expired |
| HE-00008 Carlos Mendez | (blank) | No date | ❓ Missing Date |

### Full Code Node JavaScript (from Node 09)

```javascript
const items = $input.all();
const results = [];

for (const item of items) {
  const data = { ...item.json };
  const expiryRaw = data.expiryDate;
  
  if (!expiryRaw || expiryRaw.trim() === '') {
    data.daysRemaining = null;
    data.complianceStatus = 'Missing Date';
    data.actionRequired = 'Expiry date not recorded. Please update the spreadsheet.';
    results.push({ json: data });
    continue;
  }
  
  const today = new Date();
  today.setHours(0, 0, 0, 0);
  
  const expiry = new Date(expiryRaw);
  expiry.setHours(0, 0, 0, 0);
  
  if (isNaN(expiry.getTime())) {
    data.daysRemaining = null;
    data.complianceStatus = 'Missing Date';
    data.actionRequired = `Invalid date format: "${expiryRaw}". Expected YYYY-MM-DD.`;
    results.push({ json: data });
    continue;
  }
  
  const msPerDay = 1000 * 60 * 60 * 24;
  const daysRemaining = Math.round((expiry.getTime() - today.getTime()) / msPerDay);
  data.daysRemaining = daysRemaining;
  
  if (daysRemaining < 0) {
    data.complianceStatus = 'Expired';
    data.actionRequired = `Certification expired ${Math.abs(daysRemaining)} day(s) ago. Remove technician from active scheduling immediately.`;
  } else if (daysRemaining <= 30) {
    data.complianceStatus = 'Expiring Soon';
    data.actionRequired = `Expires in ${daysRemaining} day(s). Schedule renewal now.`;
  } else {
    data.complianceStatus = 'Valid';
    data.actionRequired = 'No action required.';
  }
  
  data.lastSynced = new Date().toISOString();
  data.alertSent = (data.complianceStatus !== 'Valid');
  
  results.push({ json: data });
}

return results;
```

### Adjusting the Expiring Soon Threshold

To change from 30 days to 45 days, simply change line:
```javascript
} else if (daysRemaining <= 30) {
```
to:
```javascript
} else if (daysRemaining <= 45) {
```

No other node needs to change.

---

## 11. Slack Message Templates

### Template A: Expiring Soon

```
⚠️ *Certification Expiring Soon*

*Technician:* {{$json["technicianName"]}}
*Employee ID:* {{$json["employeeId"]}}
*Region:* {{$json["region"]}}
*Certification:* {{$json["certType"]}}
*Expiry Date:* {{$json["expiryDate"]}}
*Expires In:* {{$json["daysRemaining"]}} days
*Supervisor:* {{$json["supervisorEmail"]}}

*Action Required:* Schedule renewal immediately. Technician must not be assigned to jobs after expiry date.
```

**Rendered Example:**
```
⚠️ Certification Expiring Soon

Technician: Yuki Tanaka
Employee ID: HE-00007
Region: West
Certification: NABCEP PV Associate
Expiry Date: 2024-05-28
Expires In: 8 days
Supervisor: lead.West@helios-current.com

Action Required: Schedule renewal immediately. Technician must not be assigned to jobs after expiry date.
```

---

### Template B: Expired

```
🚨 *Certification EXPIRED — Immediate Action Required*

*Technician:* {{$json["technicianName"]}}
*Employee ID:* {{$json["employeeId"]}}
*Region:* {{$json["region"]}}
*Certification:* {{$json["certType"]}}
*Expired:* {{Math.abs($json["daysRemaining"])}} days ago ({{$json["expiryDate"]}})
*Supervisor:* {{$json["supervisorEmail"]}}

*⛔ Action Required:* Remove from active scheduling immediately. Do NOT assign to job sites until certification is renewed and on file.
```

**Rendered Example:**
```
🚨 Certification EXPIRED — Immediate Action Required

Technician: Marcus Thorne
Employee ID: HE-00001
Region: North
Certification: PV Installation
Expired: 189 days ago (2023-11-12)
Supervisor: lead.North@helios-current.com

⛔ Action Required: Remove from active scheduling immediately. Do NOT assign to job sites until certification is renewed and on file.
```

---

### Template C: Missing Date (Admin Notice)

```
🔔 *Missing Certification Date — Admin Notice*

*Technician:* {{$json["technicianName"]}}
*Employee ID:* {{$json["employeeId"]}}
*Region:* {{$json["region"]}}
*Certification:* {{$json["certType"]}}
*Supervisor:* {{$json["supervisorEmail"]}}

*Action:* No expiry date is recorded for this certification. Please update the master spreadsheet. This record cannot be compliance-checked until a date is provided.
```

---

### Template D: Executive Daily Summary

```
📋 *Helios Current — Daily Compliance Summary*
*Date:* Monday, May 20, 2024
*Total Records Checked:* 75

✅ Valid: 22
⚠️ Expiring Soon (≤30 days): 14
🚨 Expired: 38
❓ Missing Date: 1

⚠️ Action required — see regional channels for details.
```

---

## 12. Airtable Upsert Logic

### Why Upsert (Not Create)?

If you use **Create** every time the workflow runs, you will accumulate duplicate rows — one per run, per technician, per certification. After 30 days, you would have 30× duplicates for each record. Upsert solves this.

### How Airtable Upsert Works in n8n

The Airtable node's **Upsert** operation:
1. Receives a unique key field (our `recordId`)
2. Searches for an existing Airtable row where that field matches
3. If found: **updates** that row with new data
4. If not found: **creates** a new row

### Unique Key Design

Our unique key combines `Employee ID` + `Certification Type`:

```
HE-00001_PV_Installation
HE-00008_Lead-Acid_Safety
```

This is set in Node 06:
```javascript
{{$json["Employee ID"] + "_" + $json["Certification Type"].replace(/ /g, "_")}}
```

This key is stored in the **Record ID** field in Airtable and used as the match field in all four Airtable upsert nodes.

### Airtable Node Upsert Settings

In each Airtable node (Nodes 12–15):

- **Operation:** `Upsert`
- **Fields to Match On:** Click **Add Field** → Select `Record ID` → Value: `{{$json["recordId"]}}`

> ⚠️ The "Fields to Match On" section is what tells Airtable which column to use as the unique key. Without this, Airtable will always create new rows.

---

## 13. Missing Date Handling

### The Problem

Row HE-00008 (Carlos Mendez, Lead-Acid Safety) has no expiry date in the CSV. Additional rows may also have blank dates due to data entry gaps.

### How the Workflow Handles It

```
Record loaded from spreadsheet
    ↓
Node 06: Set – recordId is built (even without expiry date)
    ↓
Node 07: Split In Batches
    ↓
Node 08: IF – Expiry Date Exists?
    ├─ YES → Node 09: Code – Calculate Days
    └─ NO  → Node 10: Set – Flag Missing Date
                ↓
             Node 15: Airtable Upsert Missing Date
                ↓
             Node 21: Set – Missing Date Notice
                ↓
             Node 22: Slack – Admin Alert to #ops-alerts
```

### Key Rules

- The workflow does **not crash** on missing dates
- Missing records are still written to Airtable with `Compliance Status = Missing Date`
- A Slack notice goes to `#ops-alerts` (not regional channels — it's an admin data quality issue, not a field ops issue)
- `daysRemaining` is stored as `null` in Airtable
- `actionRequired` reads: `Expiry date not recorded. Please update the master spreadsheet.`

---

## 14. Executive Daily Summary

### What the Summary Contains

The summary is generated once per workflow run (not once per record). It aggregates all 75 records and produces:

- Total records checked
- Count of Valid certifications
- Count of Expiring Soon
- Count of Expired
- Count of Missing Date
- Dynamic footer: either a warning or a celebration message

### Where It Goes

The summary is sent to `#leadership` in Slack every morning at 7:01 AM (immediately after all individual alerts fire).

### How to Set It Up

1. After Node 23 (Merge – Rejoin All Branches)
2. Add Node 24 (Aggregate – Collect All Records)
3. Add Code Node: `Code – Build Summary Message` (logic in Section 10)
4. Add Node 25 (Set – Format Executive Summary)
5. Add Node 26 (Slack – Executive Summary to `#leadership`)

### Sample Output

```
📋 Helios Current — Daily Compliance Summary
Date: Monday, May 20, 2024
Total Records Checked: 75

✅ Valid: 22
⚠️ Expiring Soon (≤30 days): 14
🚨 Expired: 38
❓ Missing Date: 1

⚠️ Action required — see regional channels for details.
```

> The counts above are realistic based on the starter CSV dates relative to 2024-05-20. Update the Expiry Dates in Google Sheets to current/future dates to see realistic Valid/Expiring Soon numbers in your own runs.

---

## 15. Testing Procedure

Follow these steps in order. Do not skip ahead.

### Step 1: Import the CSV

1. Open Google Sheets
2. Import `helios_compliance_sync_starter.csv`
3. Verify 75 rows + 1 header row = 76 total rows
4. Confirm HE-00008 (Carlos Mendez, Lead-Acid Safety) has a blank Expiry Date

### Step 2: Run Manually (First Test)

1. Open n8n and open your workflow
2. Click **Execute Workflow** (this triggers Node 02: Manual Trigger)
3. Watch the execution flow in real-time
4. Check for green (success) or red (error) on each node

### Step 3: Verify Switch Node Outputs

1. Click on Node 11 (Switch – Status Router) after execution
2. Click each output tab (1, 2, 3, 4)
3. Confirm records are in the correct output based on their compliance status
4. You should see at least one record in each output (given the CSV dates)

### Step 4: Verify Airtable Records

1. Open Airtable → Compliance Dashboard → Technician Compliance
2. Verify 75 rows exist (one per record)
3. Check that `Compliance Status`, `Days Remaining`, and `Last Synced` are populated
4. Confirm HE-00008 shows `Missing Date`
5. Run the workflow a second time — confirm row counts do NOT increase (upsert working)

### Step 5: Verify Slack Alerts

1. Open Slack
2. Check `#north-compliance`, `#south-compliance`, `#east-compliance`, `#west-compliance`
3. Verify that only Expiring Soon and Expired records generated messages
4. Check `#ops-alerts` for the Missing Date notice (HE-00008)
5. Check `#leadership` for the executive summary

### Step 6: Test Expired Record Scenario

1. In Google Sheets, change HE-00003 (David Chen, Battery Storage Specialist, 2025-08-22) to a date 5 days ago
2. Wait for the 15-minute Google Sheets trigger (or manually execute)
3. Verify the Airtable record updates to `Expired`
4. Verify a `🚨 Certification EXPIRED` Slack message appears in `#west-compliance`
5. Restore the date when done

### Step 7: Test Missing Date Scenario

1. Add a new row to Google Sheets with a blank Expiry Date
2. Execute the workflow
3. Verify the new record appears in Airtable with `Missing Date` status
4. Verify `#ops-alerts` received the admin notice

### Step 8: Test Error Handling

1. Temporarily invalidate your Airtable credentials (change one character in the token)
2. Execute the workflow
3. Verify that `#ops-alerts` receives the failure alert from Node 30
4. Restore the correct token

---

## 16. Deliverables Checklist

Use this checklist when submitting the project to a client:

### Workflow Files
- [ ] Export the n8n workflow as JSON: **Workflow → ... → Export**
- [ ] Name the file: `helios_compliance_sync_v1.json`

### Screenshots
- [ ] Full n8n canvas screenshot (all nodes visible, zoomed out)
- [ ] Successful execution screenshot (all nodes green)
- [ ] Node 09 (Code Node) output showing sample records with `complianceStatus`
- [ ] Node 11 (Switch) showing all 4 outputs populated
- [ ] Airtable dashboard screenshot showing 75 records with statuses and colors
- [ ] Slack screenshot: `#north-compliance` with expiring/expired alerts
- [ ] Slack screenshot: `#leadership` with executive summary message
- [ ] Slack screenshot: `#ops-alerts` with missing date notice

### Video Walkthrough
- [ ] Record a 2-minute Loom walkthrough showing:
  - Canvas overview and node naming
  - Manual trigger → execution flow
  - Airtable dashboard populated
  - Slack messages in all channels
  - Brief explanation of the Code Node logic

### Documentation
- [ ] This implementation guide (the `.md` file you are reading)
- [ ] Airtable base sharing link (set to Comment Only for client)
- [ ] Notes on how to update the expiry threshold (Section 10)

---

## 17. Troubleshooting Guide

### Problem: Airtable Authentication Failed

**Error:** `401 Unauthorized` or `You are not authenticated`

**Fix:**
1. In Airtable, go to Developer Hub → Personal Access Tokens
2. Confirm the token has `data.records:read`, `data.records:write`, `schema.bases:read`
3. Confirm the token is scoped to the correct workspace
4. In n8n Credentials, delete and recreate the Airtable credential
5. Paste the token fresh — do not copy from a document that may have added hidden characters

---

### Problem: Slack Invalid Token / Message Not Sent

**Error:** `invalid_auth` or `channel_not_found`

**Fix:**
1. Verify the Bot Token starts with `xoxb-` (not `xoxp-`)
2. In Slack API → Your App → OAuth & Permissions → confirm scopes: `chat:write`, `chat:write.public`
3. Reinstall the app to your workspace if scopes changed
4. Confirm the bot has been invited to the target channel (`/invite @HeliosComplianceBot`)
5. Channel names in n8n must include `#` prefix: `#north-compliance` (not `north-compliance`)

---

### Problem: Date Parsing Errors

**Error:** `Invalid Date` or `NaN` in `daysRemaining`

**Fix:**
1. Confirm all expiry dates in the spreadsheet use `YYYY-MM-DD` format
2. In Google Sheets: select the Expiry Date column → Format → Number → **Plain Text** or **Date (YYYY-MM-DD)**
3. European date formats (DD/MM/YYYY) will fail. Use find/replace or a Google Sheets formula to convert
4. Add this to the top of the Code Node to log bad dates:
   ```javascript
   console.log('Parsing date:', expiryRaw, '→', new Date(expiryRaw));
   ```

---

### Problem: CSV Headers Mismatch

**Error:** `undefined` values after Node 06

**Fix:**
1. Open the raw CSV and check for hidden characters in header names (especially a BOM character `﻿` at the start)
2. The starter CSV has a BOM character before `Employee ID`. Reference it as `$json["Employee ID"]` not `$json["﻿Employee ID"]`
3. In Node 06, use the **Input Data Preview** to see exactly what field names n8n received
4. Adjust expressions to match the exact names shown in the preview

---

### Problem: Undefined Expressions in Slack Message

**Error:** Slack message shows `undefined` where technician name should be

**Fix:**
1. Open Node 16 (Set – Build Expiring Alert) and click **Preview**
2. Verify `$json["technicianName"]` returns a value
3. Check that Node 06 ran before Node 16 (look at the node chain)
4. If using raw CSV headers: temporarily change `technicianName` to `Technician Name` in the Set node to confirm data is flowing

---

### Problem: Duplicate Rows in Airtable

**Symptom:** After multiple runs, Airtable has 150, 225, 300... rows instead of 75

**Fix:**
1. In Airtable Upsert nodes (12–15), confirm **Fields to Match On** is configured with `Record ID`
2. Verify Node 06 is correctly generating `recordId` (open node, click Input/Output to compare)
3. If duplicates already exist: in Airtable, filter by `Record ID`, sort, and manually delete duplicates
4. Going forward, confirm Operation is `Upsert` not `Create` in all four Airtable nodes

---

### Problem: Empty Switch Output

**Symptom:** One of the four Switch outputs has no records flowing through it

**Fix:**
1. This may be correct — your data may genuinely have no records in one category
2. To verify, add a `No Operation` node to that output and check its input count after execution
3. In n8n, Switch outputs with zero records don't cause errors — they simply don't execute downstream nodes

---

### Problem: Google Sheets Trigger Not Firing

**Symptom:** You update the spreadsheet but the workflow doesn't run

**Fix:**
1. The Google Sheets Trigger polls every N minutes — it will not fire instantly
2. Set poll interval to `Every 1 Minute` for testing, then change back to `Every 15 Minutes` for production
3. Confirm the trigger is active (toggle in the top-right of the workflow should be ON)
4. Check n8n Execution History — if the trigger ran but found no changes, it will show a successful but empty execution

---

## 18. Pro Freelancer Tips

### Canvas Organization

1. **Use Sticky Notes:** Right-click on the canvas → Add Sticky Note. Create one sticky note per workflow section (Trigger Layer, Data Layer, Routing Layer, etc.) and color-code them:
   - Blue = Trigger Layer
   - Yellow = Data Ingestion
   - Orange = Classification
   - Red = Alerting
   - Green = Summary

2. **Color-Code Nodes:** Right-click any node → Change Color. Match the sticky note colors for visual grouping.

3. **Consistent Node Naming:** Every node name should tell you exactly what it does in one glance:
   - ✅ `📥 Read Sheet – Technician Data`
   - ❌ `Google Sheets 1`

4. **Use Emojis as Visual Anchors:** The emoji prefix on each node name (📥, 🧮, 🔀, 💬) makes the canvas scannable at a glance without reading text.

### Reliability Improvements

5. **Add Retries:** On Airtable and Slack nodes, click **Settings (gear icon) → On Error → Retry on Fail** → set Max Tries to `3`, Wait Between Tries to `5000ms`. This handles transient API rate limits.

6. **Add a Wait Node:** Between the regional alerts and the executive summary, add a **Wait** node for 30 seconds. This ensures all Airtable writes complete before the summary count is taken.

7. **Use n8n's Pinned Data Feature:** During testing, pin the output of Node 05 (Read Sheet) so you don't hit the Google Sheets API 75 times per test. Pin it once, then unpin for production.

### Presentation for Clients

8. **Deliver a Live Demo:** During client handoff, run the workflow live with a screen share. Show the Slack messages appearing in real time.

9. **Annotate the Airtable Dashboard:** Add a Airtable **Gallery View** filtered to `Expiring Soon` and another filtered to `Expired`. Name them "Urgent Action Required" and "Needs Renewal". This visual layer impresses clients immediately.

10. **Include a Change Log:** Create a sticky note on the canvas titled `📋 Version History` and log:
    ```
    v1.0 – 2024-05-20 – Initial build by [Your Name]
    v1.1 – [Date] – Changed expiry threshold from 30 to 45 days
    ```

11. **Timestamp Your Exports:** Name your export files with the date:
    - `helios_compliance_sync_v1_20240520.json`

12. **Provide a README:** Write a 1-page summary document explaining:
    - What the workflow does
    - How to update the expiry threshold
    - How to add a new region
    - Who to contact if it breaks (you, for 30 days post-handoff)

13. **Test with Realistic Data:** Before the demo, manually adjust 5–6 CSV rows so there are records in all four categories (Valid, Expiring Soon, Expired, Missing Date). Nothing is worse than a demo where every record is "Valid" and no Slack alerts fire.

14. **Set Up a Monitoring Dashboard:** In n8n Cloud, use the **Executions** tab as your monitoring panel. Show the client where to see daily run history. If an execution shows red, they know to contact you.

---

## Appendix A: Node Connection Summary

| From Node | To Node | Connection Type |
|---|---|---|
| 01 Cron Trigger | 04 Merge Triggers | Main |
| 02 Manual Trigger | 04 Merge Triggers | Main |
| 03 GSheets Trigger | 04 Merge Triggers | Main |
| 04 Merge Triggers | 05 Read Sheet | Main |
| 05 Read Sheet | 06 Set Normalize | Main |
| 06 Set Normalize | 07 Split In Batches | Main |
| 07 Split In Batches | 08 IF Date Exists | Main |
| 08 IF Date Exists (TRUE) | 09 Code Calculate Days | Main |
| 08 IF Date Exists (FALSE) | 10 Set Flag Missing | Main |
| 09 Code Calculate Days | 11 Switch Status Router | Main |
| 10 Set Flag Missing | 11 Switch Status Router | Main |
| 11 Switch Output 1 (Valid) | 12 Airtable Valid | Main |
| 11 Switch Output 2 (Expiring) | 13 Airtable Expiring | Main |
| 11 Switch Output 3 (Expired) | 14 Airtable Expired | Main |
| 11 Switch Output 4 (Missing) | 15 Airtable Missing | Main |
| 13 Airtable Expiring | 16 Set Build Expiring Alert | Main |
| 14 Airtable Expired | 19 Set Build Expired Alert | Main |
| 16 Set Build Expiring | 17 Set Region Resolver | Main |
| 19 Set Build Expired | 17 Set Region Resolver | Main |
| 17 Set Region Resolver | 18 Slack Send Alert | Main |
| 15 Airtable Missing | 21 Set Admin Notice | Main |
| 21 Set Admin Notice | 22 Slack Missing Alert | Main |
| 12 Airtable Valid | 23 Merge Rejoin | Main |
| 18 Slack Send Alert | 23 Merge Rejoin | Main |
| 22 Slack Missing Alert | 23 Merge Rejoin | Main |
| 23 Merge Rejoin | 24 Aggregate Summary | Main |
| 24 Aggregate | Code Build Summary | Main |
| Code Build Summary | 25 Set Format Summary | Main |
| 25 Set Format Summary | 26 Slack Executive | Main |
| 26 Slack Executive | 27 NoOp Logger | Main |
| 28 Error Trigger | 29 Set Error Message | Main |
| 29 Set Error Message | 30 Slack Failure Alert | Main |

---

## Appendix B: Full Starter CSV Data Reference

The `helios_compliance_sync_starter.csv` file contains 75 records across 12 unique technicians and 10 certification types spanning all four regions.

| Employee ID | Technician Name | Region | Certification Type | Expiry Date |
|---|---|---|---|---|
| HE-00001 | Marcus Thorne | North | PV Installation | 2023-11-12 |
| HE-00002 | Elena Rodriguez | South | OSHA 30 | 2024-05-15 |
| HE-00003 | David Chen | West | Battery Storage Specialist | 2025-08-22 |
| HE-00004 | Sarah Jenkins | East | NFPA 70E Electrical Safety | 2024-05-02 |
| HE-00005 | Amara Okafor | North | Fall Protection | 2024-12-10 |
| HE-00006 | Liam O'Sullivan | South | EVSE Level 2 | 2023-02-14 |
| HE-00007 | Yuki Tanaka | West | NABCEP PV Associate | 2024-05-28 |
| HE-00008 | Carlos Mendez | East | Lead-Acid Safety | *(blank)* |
| HE-00009 | Sophie Bauer | North | First Aid/CPR | 2025-01-15 |
| HE-00010 | Jackson Reed | South | Rigging & Signalling | 2024-04-10 |
| … | … | … | … | … |
| HE-00075 | Isabella Conti | East | First Aid/CPR | 2025-04-18 |

> The full 75-row dataset is in `helios_compliance_sync_starter.csv`. Import it directly into Google Sheets as described in Section 5.

---

*End of Implementation Guide*

---

**Document:** Certification Compliance Sync for Renewable Energy Technicians.md  
**Prepared for:** Helios Current  
**Workflow Platform:** n8n  
**Total Nodes:** 30  
**Integrations:** Google Sheets · Airtable · Slack  
**Guide Version:** 1.0  

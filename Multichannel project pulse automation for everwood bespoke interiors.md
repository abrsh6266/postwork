# Multichannel Project Pulse Automation for Everwood Bespoke Interiors

> **Implementation Guide** — Node-by-Node n8n Workflow Build  
> **Version:** 1.0 | **Nodes:** 28 (Main) + 4 (Error Branch) = **32 Total**  
> **Trigger:** Google Sheets Row Update or Airtable Record Change  
> **Stack:** n8n · Google Sheets · Gmail · Slack · Google Drive  

---

## Table of Contents

1. [Project Title & Summary](#1-project-title--summary)
2. [Business Problem Solved](#2-business-problem-solved)
3. [Full Workflow Architecture](#3-full-workflow-architecture)
4. [Required Accounts & Credentials](#4-required-accounts--credentials)
5. [Starter File Setup](#5-starter-file-setup)
6. [Data Structure Setup](#6-data-structure-setup)
7. [Full n8n Build Guide — All 32 Nodes](#7-full-n8n-build-guide--all-32-nodes)
8. [Exact Expressions & Field Mapping](#8-exact-expressions--field-mapping)
9. [Projected Delivery Window Logic](#9-projected-delivery-window-logic)
10. [VIP Routing Logic](#10-vip-routing-logic)
11. [Fragile Routing Logic](#11-fragile-routing-logic)
12. [Gmail HTML Templates](#12-gmail-html-templates)
13. [Slack & Discord Alert Templates](#13-slack--discord-alert-templates)
14. [Error Handling Architecture](#14-error-handling-architecture)
15. [Logging & Tracking Enhancements](#15-logging--tracking-enhancements)
16. [Testing Procedure](#16-testing-procedure)
17. [Deliverables Checklist](#17-deliverables-checklist)
18. [Troubleshooting](#18-troubleshooting)
19. [Pro Freelancer Tips](#19-pro-freelancer-tips)

---

## 1. Project Title & Summary

**Project Name:** Multichannel Project Pulse Automation — Everwood Bespoke Interiors  
**Client Industry:** Luxury Kitchen & Bath Remodeling (Boutique Startup)  
**Automation Platform:** n8n (self-hosted or n8n Cloud)  
**Estimated Build Time:** 4–6 hours  
**Workflow Complexity:** Advanced (32 nodes, dual-trigger, multi-branch)

### What This Automation Does in Plain English

Every time a project row is updated in Google Sheets (or Airtable), this workflow wakes up, reads the project's status, figures out who the customer is and how important they are, calculates when their delivery window will be, then fires off a beautiful personalized email to the client, a Slack message to the install crew, and (if the project involves fragile custom items) an urgent management alert — all fully automatically, with every action logged and every failure caught.

---

## 2. Business Problem Solved

### The Manual Pain Points Everwood Faces Today

Everwood Bespoke Interiors is a boutique luxury remodeling startup. Their jobs involve high-end custom cabinetry, marble countertops, specialty bath fixtures, and white-glove fabrication. Their project list grows every week — and right now, every status update requires a team member to:

- Manually email the client with an update
- Separately text or call the install crew
- Remember to flag fragile delivery items for special transport
- Update a spreadsheet tracker by hand
- Hope nothing gets missed

This creates real business risk:

| Problem | Business Impact |
|---|---|
| VIP clients not getting priority communication | Lost trust, potential churn |
| Install crews missing delivery details | Delayed jobs, overtime costs |
| Fragile items shipped without warning | Breakage, liability, expensive remakes |
| No record of what notification was sent when | Disputes, confusion, poor accountability |
| Manual data entry errors in the tracker | Incorrect project timelines |

### What This Automation Solves

This n8n workflow eliminates all of that. It is a fully automated, intelligent notification engine that:

1. **Triggers instantly** when any project row changes status
2. **Validates data quality** before sending anything
3. **Calculates a real delivery window** (not just "+3 days" — it skips weekends)
4. **Routes VIP clients** to a premium concierge email template
5. **Routes fragile projects** to a management priority alert channel
6. **Notifies the correct install crew member** based on who is assigned
7. **Logs every action** so there is a permanent, searchable record
8. **Catches every failure** in a dedicated Error Trigger branch and logs it to a separate Google Sheet tab

The result: Everwood can scale their project volume without hiring more coordinators, and every client gets a consistent, luxury communication experience regardless of who is on duty.

---

## 3. Full Workflow Architecture

### Main Workflow — Visual Flow Map

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          TRIGGER LAYER                                       │
│                                                                               │
│  [Node 1] Google Sheets Trigger          [Node 2] Airtable Trigger          │
│      (Watch: Row Updated)                    (Watch: Record Changed)         │
│           │                                         │                        │
│           └──────────────┬──────────────────────────┘                       │
│                          ▼                                                   │
│                  [Node 3] Merge Trigger Sources                               │
│                    (Combine both sources)                                    │
└─────────────────────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                       DATA PREP LAYER                                        │
│                                                                               │
│          [Node 4] Set Node — Normalize & Standardize Fields                 │
│                           │                                                  │
│                           ▼                                                  │
│          [Node 5] IF Node — Required Fields Present?                        │
│              │ (YES)                          │ (NO)                         │
│              ▼                                ▼                              │
│    [Node 7] Code Node —          [Node 6] Google Sheets —                  │
│    Calculate Delivery Window      Log Missing/Invalid Row                   │
└─────────────────────────────────────────────────────────────────────────────┘
                           │ (after delivery window calc)
                           ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                       ROUTING LAYER                                          │
│                                                                               │
│          [Node 8] Switch Node — VIP vs Standard                             │
│              │ VIP                            │ Standard                    │
│              ▼                                ▼                              │
│  [Node 10] Set — Build VIP      [Node 9] Set — Build Standard               │
│      Email HTML                     Email HTML                               │
│              │                                │                              │
│              ▼                                ▼                              │
│  [Node 12] Gmail — Send VIP    [Node 11] Gmail — Send Standard              │
│      Email                          Email                                    │
│              │                                │                              │
│              └──────────────┬─────────────────┘                             │
│                             ▼                                                │
│          [Node 13] Merge Email Branches                                     │
└─────────────────────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      CREW ALERT LAYER                                        │
│                                                                               │
│          [Node 14] Set Node — Build Crew Slack Alert                        │
│                           │                                                  │
│          [Node 15] Slack — Notify Install Team                              │
│                           │                                                  │
│          [Node 16] IF Node — Item Fragility = High?                         │
│              │ (YES)                          │ (NO — skip)                 │
│              ▼                                ▼                              │
│  [Node 17] Set — Build Fragile    [Node 18] NoOp — Skip Fragile Alert     │
│      Alert Message                                                           │
│              │                                                               │
│              ▼                                                               │
│  [Node 19] Slack — Management Priority Alert                                │
└─────────────────────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                       LOGGING LAYER                                          │
│                                                                               │
│  [Node 20] Google Sheets — Update Notification Timestamp on Project Row     │
│                           │                                                  │
│  [Node 21] Merge Node — Rejoin All Branches                                 │
│                           │                                                  │
│  [Node 22] Set Node — Build Success Summary                                 │
│                           │                                                  │
│  [Node 23] Google Sheets — Append to Activity Log Tab                       │
│                           │                                                  │
│  [Node 24] Set Node — Final Completion Stamp                                │
│                           │                                                  │
│  [Node 25] NoOp — Workflow End                                              │
└─────────────────────────────────────────────────────────────────────────────┘

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
                        ERROR TRIGGER BRANCH (Separate)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  [Node 26] Error Trigger — Catch All Failures
                  │
  [Node 27] Set Node — Parse Failure Details
                  │
  [Node 28] Google Sheets — Log to Technical Errors Tab
                  │
  [Node 29] Slack — Admin Alert (DM to Workflow Owner)
```

---

## 4. Required Accounts & Credentials

### Account Requirements Summary

| Service | Required | Free Tier | Notes |
|---|---|---|---|
| n8n | Yes | Yes (Cloud trial / Self-hosted free) | v1.0+ recommended |
| Google Account | Yes | Yes | For Sheets + Gmail |
| Google Sheets | Yes | Yes | Project tracker + log |
| Gmail OAuth2 | Yes | Yes | Client email sending |
| Slack | Yes | Yes (free workspace) | Crew + admin alerts |
| Airtable | Optional | Yes (free plan) | Alternate trigger source |
| Google Drive | Optional | Yes | For backup exports |

---

### Setting Up Each Credential in n8n

#### Google Sheets + Gmail OAuth2

1. Go to [console.cloud.google.com](https://console.cloud.google.com)
2. Create a new project named **"Everwood Automation"**
3. Enable these APIs:
   - **Google Sheets API**
   - **Gmail API**
   - **Google Drive API** (optional)
4. Go to **Credentials** → **Create Credentials** → **OAuth 2.0 Client ID**
5. Application type: **Web Application**
6. Authorized redirect URIs — add:
   ```
   https://your-n8n-domain.com/rest/oauth2-credential/callback
   ```
   *(Replace with your actual n8n domain)*
7. Copy the **Client ID** and **Client Secret**
8. In n8n → **Credentials** → **New** → Search **"Google Sheets OAuth2 API"**
9. Paste Client ID and Client Secret → Click **Connect** → Authorize in the Google popup
10. Name the credential: `Everwood Google Sheets`
11. Repeat the same process for Gmail — name it: `Everwood Gmail`

#### Slack

1. Go to [api.slack.com/apps](https://api.slack.com/apps) → **Create New App** → **From Scratch**
2. App Name: **Everwood Project Bot**
3. Pick your Slack workspace
4. Go to **Incoming Webhooks** → Toggle **Activate Incoming Webhooks** → ON
5. Click **Add New Webhook to Workspace** → Choose channel **#install-crew**
6. Copy the webhook URL (it looks like: `https://hooks.slack.com/services/T.../B.../xxx`)
7. In n8n → **Credentials** → **New** → Search **"Slack"**
8. Select **Webhook** authentication
9. Paste the webhook URL → Name it: `Everwood Slack Crew Webhook`
10. Create a **second webhook** for the **#management-alerts** channel → name it: `Everwood Slack Admin Webhook`

#### Airtable (Optional)

1. Log into [airtable.com](https://airtable.com) → **Account** → **API** section
2. Generate a **Personal Access Token**
3. In n8n → **Credentials** → **New** → Search **"Airtable Token API"**
4. Paste the token → Name it: `Everwood Airtable`

---

## 5. Starter File Setup

The starter file **`everwood_project_pulse_starter.csv`** contains pre-seeded project data for building and testing the workflow. Here is exactly how to import it into each platform.

---

### Option A: Import into Google Sheets (Recommended)

1. Go to [sheets.google.com](https://sheets.google.com) → Click **Blank spreadsheet**
2. Name the file: `Everwood Project Pulse Tracker`
3. Go to **File** → **Import** → **Upload**
4. Drag and drop `everwood_project_pulse_starter.csv`
5. Import location: **Replace current sheet**
6. Separator type: **Comma**
7. Click **Import data**
8. You will see the columns populate automatically:

| A | B | C | D | E | F | G | H |
|---|---|---|---|---|---|---|---|
| Order_ID | Customer_Name | Contact_Email | Project_Type | Client_Tier | Item_Fragility | Current_Status | Lead_Installer |

9. Now add the recommended additional columns (see Section 6):
   - Column I: `Status_Date`
   - Column J: `Delivery_Window`
   - Column K: `Last_Notified`
   - Column L: `Error_Flag`
   - Column M: `Priority_Level`

10. Create a **second sheet tab** named: `Activity Log`
11. Create a **third sheet tab** named: `Technical Errors`

> **Important:** Copy the **Spreadsheet ID** from the URL:  
> `https://docs.google.com/spreadsheets/d/`**`YOUR_SPREADSHEET_ID_HERE`**`/edit`  
> You will paste this into every Google Sheets node.

---

### Option B: Import into Airtable

1. Log into [airtable.com](https://airtable.com) → **Add a base** → **Import a spreadsheet**
2. Upload `everwood_project_pulse_starter.csv`
3. Base Name: `Everwood Project Pulse`
4. Table Name: `Projects`
5. Airtable will auto-detect column types — verify:
   - Order_ID → **Single line text**
   - Customer_Name → **Single line text**
   - Contact_Email → **Email**
   - Project_Type → **Single select** (add: Kitchen, Bath)
   - Client_Tier → **Single select** (add: VIP, Standard)
   - Item_Fragility → **Single select** (add: High, Low)
   - Current_Status → **Single select** (add all status values)
   - Lead_Installer → **Single line text**
6. Add additional fields (from Section 6) manually

---

### Option C: Read CSV Directly Inside n8n (No Import Needed)

If you want to test the workflow without setting up Google Sheets or Airtable first:

1. In n8n, add a **Spreadsheet File** node at the start
2. Upload `everwood_project_pulse_starter.csv` as a binary file using the **Read Binary File** node pointed at a local path
3. Connect it to the **Spreadsheet File** node to parse it
4. Connect that to **Node 4 (Normalize Fields)** directly

> This is best for initial testing only. For production, use Google Sheets (Option A).

---

## 6. Data Structure Setup

### Existing Columns in the CSV

| Column | Type | Example Value | Purpose |
|---|---|---|---|
| `Order_ID` | Text | `EW-2024-001` | Unique project identifier |
| `Customer_Name` | Text | `Margaret Thornton` | Client full name |
| `Contact_Email` | Email | `mthornton@email.com` | Where to send client email |
| `Project_Type` | Select | `Kitchen` / `Bath` | Drives template selection |
| `Client_Tier` | Select | `VIP` / `Standard` | Drives routing logic |
| `Item_Fragility` | Select | `High` / `Low` | Triggers management alert |
| `Current_Status` | Select | `In Production` | Core trigger value |
| `Lead_Installer` | Text | `Marcus Webb` | Who gets the crew Slack alert |

---

### Recommended Additional Columns to Add

| Column | Type | Why It Matters |
|---|---|---|
| `Status_Date` | Date | The workflow reads this to calculate the delivery window. Without it, delivery window math is impossible. Set to today's date whenever status is updated. |
| `Delivery_Window` | Date | The workflow writes the calculated +3 business day date back here. Creates a permanent record. |
| `Last_Notified` | Timestamp | The workflow writes a timestamp here every time it sends a notification. Prevents duplicate sends. |
| `Error_Flag` | Checkbox | The workflow sets this to TRUE if a notification fails, so project managers can spot problem rows at a glance. |
| `Priority_Level` | Number (1–3) | Auto-calculated: VIP + High Fragility = 1 (highest). Used for sorting the dashboard view. |

---

### Valid Status Values

Configure these as a dropdown/select in your sheet or Airtable to prevent typos:

- `Design Approved`
- `In Production`
- `Ready for Delivery`
- `Install Scheduled`
- `Completed`
- `Delayed`

---

## 7. Full n8n Build Guide — All 32 Nodes

> **How to read this guide:**  
> Each node section tells you the exact node type, what to rename it, every setting to configure, and what the output data looks like. Follow top to bottom. Do not skip any node.

---

### ═══════════════════════════════════════
### TRIGGER LAYER
### ═══════════════════════════════════════

---

## Node 1: Google Sheets Trigger — Watch Project Updates

**Node Type:** `Google Sheets Trigger`  
**Rename To:** `GSheets — Watch Project Updates`  
**Position:** Top-left of canvas (start of main workflow)

### Configuration

| Setting | Value |
|---|---|
| Credential | `Everwood Google Sheets` |
| Trigger On | `Row Updated` |
| Spreadsheet | Paste your Spreadsheet ID |
| Sheet Name | `Sheet1` (or whatever you named the main project tab) |
| Columns to Watch | `Current_Status` |
| Poll Interval | `1 Minute` (for production) / `30 Seconds` (for testing) |

### Purpose

This node polls Google Sheets every 1 minute looking for any row where the `Current_Status` column has changed since the last check. When it detects a change, it passes the entire row as a JSON object downstream.

### Example Output

```json
{
  "Order_ID": "EW-2024-007",
  "Customer_Name": "Diana Prescott",
  "Contact_Email": "dprescott@gmail.com",
  "Project_Type": "Kitchen",
  "Client_Tier": "VIP",
  "Item_Fragility": "High",
  "Current_Status": "Ready for Delivery",
  "Lead_Installer": "Marcus Webb",
  "Status_Date": "2024-04-11",
  "Last_Notified": "",
  "Error_Flag": "FALSE",
  "Priority_Level": ""
}
```

---

## Node 2: Airtable Trigger — Watch Record Changes

**Node Type:** `Airtable Trigger`  
**Rename To:** `Airtable — Watch Record Changes`  
**Position:** Directly below Node 1 on the canvas  
**Note:** This is the alternate trigger. In a live deployment, only enable ONE trigger at a time. Keep both in the workflow for client flexibility — they can switch without rebuilding.

### Configuration

| Setting | Value |
|---|---|
| Credential | `Everwood Airtable` |
| Base ID | Copy from your Airtable base URL |
| Table | `Projects` |
| Trigger Field | `Current_Status` |
| Poll Interval | `1 Minute` |

### Purpose

Identical behavior to Node 1, but reads from Airtable instead. When a record's `Current_Status` field changes, this trigger fires and passes the Airtable record downstream.

### Important Note for Freelancer

Tell the client: "Both triggers are built in. Right now, Google Sheets is active and Airtable is disabled. If you ever switch to Airtable, just disable Node 1 and enable Node 2 — nothing else needs to change."

---

## Node 3: Merge — Combine Trigger Sources

**Node Type:** `Merge`  
**Rename To:** `Merge — Combine Trigger Sources`  
**Position:** Center canvas, receives connections from both Node 1 and Node 2

### Configuration

| Setting | Value |
|---|---|
| Mode | `Append` |
| Number of Inputs | `2` |

### How to Wire It

- Connect **Node 1 output** → **Merge Input 1**
- Connect **Node 2 output** → **Merge Input 2**

### Purpose

Regardless of whether the data came from Google Sheets or Airtable, this Merge node combines both streams into a single unified flow. The downstream nodes never need to know which source fired — they just see project data.

### Example Output

Identical structure to whatever trigger fired. The data passes through unchanged.

---

### ═══════════════════════════════════════
### DATA PREPARATION LAYER
### ═══════════════════════════════════════

---

## Node 4: Set — Normalize & Standardize Fields

**Node Type:** `Set`  
**Rename To:** `Set — Normalize Fields`  
**Position:** After the Merge node

### Configuration

Set Mode: `Keep Only Set`

Add the following fields:

| Field Name | Value / Expression |
|---|---|
| `order_id` | `{{$json["Order_ID"]}}` |
| `customer_name` | `{{$json["Customer_Name"]}}` |
| `contact_email` | `{{$json["Contact_Email"]}}` |
| `project_type` | `{{$json["Project_Type"]}}` |
| `client_tier` | `{{$json["Client_Tier"]}}` |
| `item_fragility` | `{{$json["Item_Fragility"]}}` |
| `current_status` | `{{$json["Current_Status"]}}` |
| `lead_installer` | `{{$json["Lead_Installer"]}}` |
| `status_date` | `{{$json["Status_Date"] || $now.toISODate()}}` |
| `source` | `{{$json["source"] || "Google Sheets"}}` |

### Purpose

This node does two critical things:

1. **Standardizes key names** — converts the original capitalized CSV column names (e.g., `Customer_Name`) into lowercase snake_case (`customer_name`) so every downstream node uses a consistent naming convention
2. **Sets a fallback for `status_date`** — if the cell is empty, it defaults to today's date so the delivery window calculation in Node 7 never breaks

### Example Output

```json
{
  "order_id": "EW-2024-007",
  "customer_name": "Diana Prescott",
  "contact_email": "dprescott@gmail.com",
  "project_type": "Kitchen",
  "client_tier": "VIP",
  "item_fragility": "High",
  "current_status": "Ready for Delivery",
  "lead_installer": "Marcus Webb",
  "status_date": "2024-04-11",
  "source": "Google Sheets"
}
```

---

## Node 5: IF — Required Fields Present?

**Node Type:** `IF`  
**Rename To:** `IF — Required Fields Present?`  
**Position:** After Node 4

### Configuration

**Combine:** `AND` (all conditions must be true)

| Condition | Value 1 | Operation | Value 2 |
|---|---|---|---|
| 1 | `{{$json["contact_email"]}}` | `is not empty` | — |
| 2 | `{{$json["order_id"]}}` | `is not empty` | — |
| 3 | `{{$json["current_status"]}}` | `is not empty` | — |
| 4 | `{{$json["client_tier"]}}` | `is not empty` | — |

### Output Branches

- **TRUE branch** → Continue to Node 7 (Calculate Delivery Window)
- **FALSE branch** → Route to Node 6 (Log Missing Fields)

### Purpose

Before the workflow wastes time sending emails or Slack messages, it validates that the minimum required data is present. A missing email or order ID would cause downstream failures. This IF node catches bad rows early.

---

## Node 6: Google Sheets — Log Missing / Invalid Row

**Node Type:** `Google Sheets`  
**Rename To:** `GSheets — Log Missing Field Row`  
**Position:** Connected to the FALSE branch of Node 5

### Configuration

| Setting | Value |
|---|---|
| Credential | `Everwood Google Sheets` |
| Operation | `Append Row` |
| Spreadsheet | Your Spreadsheet ID |
| Sheet Name | `Technical Errors` |

### Columns to Append

| Column | Expression |
|---|---|
| Timestamp | `{{$now}}` |
| Workflow Name | `Everwood Project Pulse` |
| Failed Node | `IF — Required Fields Present?` |
| Error Message | `Required field missing — check Order_ID, Contact_Email, Current_Status, Client_Tier` |
| Order ID | `{{$json["order_id"] || "UNKNOWN"}}` |
| Raw JSON | `{{JSON.stringify($json)}}` |

### Purpose

When a row fails validation, it is logged to the `Technical Errors` tab with a timestamp and full raw data. Project managers can review this tab to find data entry issues. The workflow then stops processing this row (no email is sent for invalid rows).

> **After this node:** Connect to a `NoOp` end node. This branch terminates here.

---

## Node 7: Code — Calculate Delivery Window (+3 Business Days)

**Node Type:** `Code`  
**Rename To:** `Code — Calculate Delivery Window`  
**Position:** Connected to the TRUE branch of Node 5

### Configuration

- Language: `JavaScript`
- Mode: `Run Once for Each Item`

### Exact Code

```javascript
// ─────────────────────────────────────────────────────────────
// EVERWOOD PROJECT PULSE — DELIVERY WINDOW CALCULATOR
// Adds 3 business days to status_date, skipping Sat + Sun
// ─────────────────────────────────────────────────────────────

const item = $input.item.json;

// Parse the status date — handle both ISO strings and Date objects
const rawDate = item.status_date || new Date().toISOString().split('T')[0];
let date = new Date(rawDate + 'T12:00:00Z'); // Use noon UTC to avoid timezone edge cases

if (isNaN(date.getTime())) {
  // If the date is invalid, use today
  date = new Date();
}

let businessDaysAdded = 0;
const targetDays = 3;

while (businessDaysAdded < targetDays) {
  date.setDate(date.getDate() + 1);
  const dayOfWeek = date.getUTCDay(); // 0 = Sunday, 6 = Saturday
  if (dayOfWeek !== 0 && dayOfWeek !== 6) {
    businessDaysAdded++;
  }
}

// Format as human-readable: "Mon Apr 15, 2024"
const formatted = date.toLocaleDateString('en-US', {
  weekday: 'short',
  month: 'short',
  day: 'numeric',
  year: 'numeric',
  timeZone: 'UTC'
});

// Format as ISO for writing back to sheet: "2024-04-15"
const isoFormatted = date.toISOString().split('T')[0];

return {
  ...item,
  delivery_window_display: formatted,       // "Mon Apr 15, 2024"
  delivery_window_iso: isoFormatted,        // "2024-04-15"
  days_added: targetDays,
  calculation_note: `Status date ${rawDate} + 3 business days = ${formatted}`
};
```

### Business Day Logic Explained

| Status Date | Day | +1 Day | +2 Days | +3 Business Days |
|---|---|---|---|---|
| Thursday Apr 11 | Thu | Fri (count 1) | Mon (skip Sat/Sun, count 2) | Tue Apr 16 (count 3) |
| Friday Apr 12 | Fri | Mon (skip Sat/Sun, count 1) | Tue (count 2) | Wed Apr 17 (count 3) |
| Monday Apr 15 | Mon | Tue (count 1) | Wed (count 2) | Thu Apr 18 (count 3) |

### Example Output

```json
{
  "order_id": "EW-2024-007",
  "customer_name": "Diana Prescott",
  "contact_email": "dprescott@gmail.com",
  "project_type": "Kitchen",
  "client_tier": "VIP",
  "item_fragility": "High",
  "current_status": "Ready for Delivery",
  "lead_installer": "Marcus Webb",
  "status_date": "2024-04-11",
  "delivery_window_display": "Tue, Apr 16, 2024",
  "delivery_window_iso": "2024-04-16",
  "days_added": 3,
  "calculation_note": "Status date 2024-04-11 + 3 business days = Tue, Apr 16, 2024"
}
```

---

### ═══════════════════════════════════════
### ROUTING LAYER — VIP vs STANDARD
### ═══════════════════════════════════════

---

## Node 8: Switch — VIP vs Standard Client

**Node Type:** `Switch`  
**Rename To:** `Switch — VIP vs Standard Client`  
**Position:** After Node 7

### Configuration

| Setting | Value |
|---|---|
| Mode | `Rules` |
| Value to Switch On | `{{$json["client_tier"]}}` |

### Rules

| Output | Condition | Routing |
|---|---|---|
| Output 1 (VIP) | Value equals `VIP` | → Node 10 (Build VIP Email HTML) |
| Output 2 (Standard) | Value equals `Standard` | → Node 9 (Build Standard Email HTML) |
| Fallback Output | All other values | → Node 9 (Standard — treat unknowns as standard) |

### Purpose

This is the primary branching point of the workflow. VIP clients receive a premium, concierge-toned email. Standard clients receive a clean professional email. The switch ensures no client ever receives the wrong template.

---

## Node 9: Set — Build Standard Email HTML

**Node Type:** `Set`  
**Rename To:** `Set — Build Standard Email HTML`  
**Position:** Connected to Switch Output 2 (Standard)

### Configuration

Add one field:

| Field Name | Type | Value |
|---|---|---|
| `email_subject` | String | `Project Update — Order #{{$json["order_id"]}} — {{$json["current_status"]}}` |
| `email_html` | String | *(paste the full Standard HTML template from Section 12)* |
| `email_type` | String | `standard` |

### Example Subject Output

```
Project Update — Order #EW-2024-012 — Ready for Delivery
```

---

## Node 10: Set — Build VIP Concierge Email HTML

**Node Type:** `Set`  
**Rename To:** `Set — Build VIP Concierge Email HTML`  
**Position:** Connected to Switch Output 1 (VIP)

### Configuration

| Field Name | Type | Value |
|---|---|---|
| `email_subject` | String | `🌟 VIP Project Update — Order #{{$json["order_id"]}} — {{$json["current_status"]}}` |
| `email_html` | String | *(paste the full VIP HTML template from Section 12)* |
| `email_type` | String | `vip` |

### Example Subject Output

```
🌟 VIP Project Update — Order #EW-2024-007 — Ready for Delivery
```

---

## Node 11: Gmail — Send Standard Client Email

**Node Type:** `Gmail`  
**Rename To:** `Gmail — Send Standard Email`  
**Position:** After Node 9

### Configuration

| Setting | Value |
|---|---|
| Credential | `Everwood Gmail` |
| Operation | `Send` |
| To | `{{$json["contact_email"]}}` |
| Subject | `{{$json["email_subject"]}}` |
| Email Type | `HTML` |
| Message | `{{$json["email_html"]}}` |
| From Name | `Everwood Bespoke Interiors` |

### Optional Settings

| Setting | Value |
|---|---|
| Reply To | `projects@everwoodinteriors.com` |
| CC | *(leave blank unless client requests)* |

---

## Node 12: Gmail — Send VIP Concierge Email

**Node Type:** `Gmail`  
**Rename To:** `Gmail — Send VIP Email`  
**Position:** After Node 10

### Configuration

| Setting | Value |
|---|---|
| Credential | `Everwood Gmail` |
| Operation | `Send` |
| To | `{{$json["contact_email"]}}` |
| Subject | `{{$json["email_subject"]}}` |
| Email Type | `HTML` |
| Message | `{{$json["email_html"]}}` |
| From Name | `Your Everwood Concierge Team` |
| Reply To | `concierge@everwoodinteriors.com` |

---

## Node 13: Merge — Rejoin Email Branches

**Node Type:** `Merge`  
**Rename To:** `Merge — Rejoin Email Branches`  
**Position:** After Nodes 11 and 12

### Configuration

| Setting | Value |
|---|---|
| Mode | `Append` |
| Number of Inputs | `2` |

### How to Wire It

- Connect **Node 11 output** → **Input 1**
- Connect **Node 12 output** → **Input 2**

### Purpose

After the VIP and Standard branches each send their emails, this Merge node brings them back together into a single flow so the crew alerts and logging logic can run once for both paths.

---

### ═══════════════════════════════════════
### CREW ALERT LAYER
### ═══════════════════════════════════════

---

## Node 14: Set — Build Install Crew Slack Alert

**Node Type:** `Set`  
**Rename To:** `Set — Build Crew Slack Alert`  
**Position:** After Node 13

### Configuration

Add one field:

| Field Name | Type |
|---|---|
| `crew_slack_message` | String |

### Value (paste exactly)

```
🔧 *INSTALL CREW UPDATE — EVERWOOD PROJECT*

*Order:* #{{$json["order_id"]}}
*Client:* {{$json["customer_name"]}}
*Project Type:* {{$json["project_type"]}}
*Status:* {{$json["current_status"]}}
*Lead Installer:* {{$json["lead_installer"]}}
*Delivery Window:* {{$json["delivery_window_display"]}}
*Client Tier:* {{$json["client_tier"]}}

_Please confirm receipt and update your schedule accordingly._
```

---

## Node 15: Slack — Notify Install Crew

**Node Type:** `Slack`  
**Rename To:** `Slack — Notify Install Crew`  
**Position:** After Node 14

### Configuration

| Setting | Value |
|---|---|
| Credential | `Everwood Slack Crew Webhook` |
| Operation | `Send a Message` |
| Authentication | `Webhook` |
| Webhook URL | *(auto-filled from credential)* |
| Message | `{{$json["crew_slack_message"]}}` |
| Channel | `#install-crew` |
| Username | `Everwood Project Bot` |
| Icon Emoji | `:hammer_and_wrench:` |

---

## Node 16: IF — Item Fragility = High?

**Node Type:** `IF`  
**Rename To:** `IF — Fragility High?`  
**Position:** After Node 15

### Configuration

| Condition | Value 1 | Operation | Value 2 |
|---|---|---|---|
| 1 | `{{$json["item_fragility"]}}` | `equals` | `High` |

### Output Branches

- **TRUE** → Node 17 (Build Fragile Alert Message)
- **FALSE** → Node 18 (NoOp — Skip Fragile Alert)

---

## Node 17: Set — Build Fragile Management Alert

**Node Type:** `Set`  
**Rename To:** `Set — Build Fragile Alert`  
**Position:** Connected to TRUE branch of Node 16

### Configuration

Add one field:

| Field Name | Type |
|---|---|
| `fragile_slack_message` | String |

### Value (paste exactly)

```
🚨 *SPECIALIZED HANDLING REQUIRED — MANAGEMENT ALERT*

⚠️ This project involves HIGH-FRAGILITY custom fabrication. Special transport and white-glove delivery protocols must be confirmed before dispatch.

*Order:* #{{$json["order_id"]}}
*Client:* {{$json["customer_name"]}} ({{$json["client_tier"]}})
*Project Type:* {{$json["project_type"]}}
*Status:* {{$json["current_status"]}}
*Lead Installer:* {{$json["lead_installer"]}}
*Delivery Window:* {{$json["delivery_window_display"]}}

*Required Actions:*
• Confirm padded transport vehicle available
• Assign minimum 2-person delivery team
• Notify warehouse manager before dispatch
• Client expects white-glove setup — no exceptions

_This alert was auto-generated by the Everwood Project Pulse system._
```

---

## Node 18: NoOp — Skip Fragile Alert

**Node Type:** `No Operation, do nothing`  
**Rename To:** `NoOp — Skip Fragile Alert`  
**Position:** Connected to FALSE branch of Node 16

### Purpose

When fragility is Low, the workflow still needs a node on the FALSE branch to terminate cleanly and rejoin the main flow. This NoOp placeholder prevents orphaned branches.

---

## Node 19: Slack — Management Priority Alert

**Node Type:** `Slack`  
**Rename To:** `Slack — Management Priority Alert`  
**Position:** After Node 17

### Configuration

| Setting | Value |
|---|---|
| Credential | `Everwood Slack Admin Webhook` |
| Operation | `Send a Message` |
| Webhook URL | *(auto-filled from credential)* |
| Message | `{{$json["fragile_slack_message"]}}` |
| Channel | `#management-alerts` |
| Username | `Everwood Priority Bot` |
| Icon Emoji | `:rotating_light:` |

---

### ═══════════════════════════════════════
### LOGGING LAYER
### ═══════════════════════════════════════

---

## Node 20: Google Sheets — Update Notification Timestamp on Row

**Node Type:** `Google Sheets`  
**Rename To:** `GSheets — Update Last Notified Timestamp`  
**Position:** After Node 19 (and NoOp Node 18)

### Configuration

| Setting | Value |
|---|---|
| Credential | `Everwood Google Sheets` |
| Operation | `Update Row` |
| Spreadsheet | Your Spreadsheet ID |
| Sheet Name | `Sheet1` |
| Matching Column | `Order_ID` |
| Matching Value | `{{$json["order_id"]}}` |

### Columns to Update

| Column | Expression |
|---|---|
| `Last_Notified` | `{{$now.toISO()}}` |
| `Delivery_Window` | `{{$json["delivery_window_iso"]}}` |
| `Priority_Level` | `{{$json["client_tier"] === "VIP" && $json["item_fragility"] === "High" ? 1 : $json["client_tier"] === "VIP" ? 2 : 3}}` |

### Purpose

After every successful notification run, this node writes back to the project row with:
- A timestamp so the team knows the last time this project was touched
- The calculated delivery window (so it's permanently stored in the sheet)
- A numeric priority level for dashboard sorting

---

## Node 21: Merge — Rejoin All Active Branches

**Node Type:** `Merge`  
**Rename To:** `Merge — Rejoin All Branches`  
**Position:** After Node 20

### Configuration

| Setting | Value |
|---|---|
| Mode | `Append` |
| Number of Inputs | `2` |

### How to Wire It

- Node 20 (update timestamp) → Input 1
- Node 18 (NoOp skip fragile) → Input 2

### Purpose

Ensures that whether or not the fragile alert branch ran, both paths converge here before the success log is written.

---

## Node 22: Set — Build Success Summary

**Node Type:** `Set`  
**Rename To:** `Set — Build Success Summary`  
**Position:** After Node 21

### Configuration

Add these fields:

| Field Name | Expression |
|---|---|
| `log_timestamp` | `{{$now.toISO()}}` |
| `log_order_id` | `{{$json["order_id"]}}` |
| `log_customer` | `{{$json["customer_name"]}}` |
| `log_status` | `{{$json["current_status"]}}` |
| `log_tier` | `{{$json["client_tier"]}}` |
| `log_fragility` | `{{$json["item_fragility"]}}` |
| `log_delivery` | `{{$json["delivery_window_display"]}}` |
| `log_email_sent` | `TRUE` |
| `log_crew_alerted` | `TRUE` |
| `log_fragile_alert` | `{{$json["item_fragility"] === "High" ? "TRUE" : "FALSE"}}` |
| `log_source` | `{{$json["source"]}}` |

---

## Node 23: Google Sheets — Append to Activity Log

**Node Type:** `Google Sheets`  
**Rename To:** `GSheets — Append Activity Log`  
**Position:** After Node 22

### Configuration

| Setting | Value |
|---|---|
| Credential | `Everwood Google Sheets` |
| Operation | `Append Row` |
| Spreadsheet | Your Spreadsheet ID |
| Sheet Name | `Activity Log` |

### Columns to Append

| Column Header | Expression |
|---|---|
| `Timestamp` | `{{$json["log_timestamp"]}}` |
| `Order_ID` | `{{$json["log_order_id"]}}` |
| `Customer` | `{{$json["log_customer"]}}` |
| `Status` | `{{$json["log_status"]}}` |
| `Client_Tier` | `{{$json["log_tier"]}}` |
| `Fragility` | `{{$json["log_fragility"]}}` |
| `Delivery_Window` | `{{$json["log_delivery"]}}` |
| `Email_Sent` | `{{$json["log_email_sent"]}}` |
| `Crew_Alerted` | `{{$json["log_crew_alerted"]}}` |
| `Fragile_Alert_Sent` | `{{$json["log_fragile_alert"]}}` |
| `Source` | `{{$json["log_source"]}}` |

### Purpose

Every successful workflow run appends one row to the Activity Log tab. This becomes an audit trail that Everwood can use to prove what notifications were sent and when — useful for client disputes, internal reviews, and performance tracking.

---

## Node 24: Set — Final Completion Stamp

**Node Type:** `Set`  
**Rename To:** `Set — Final Completion Stamp`  
**Position:** After Node 23

### Configuration

| Field | Value |
|---|---|
| `workflow_status` | `COMPLETED` |
| `completed_at` | `{{$now.toISO()}}` |
| `total_actions_taken` | `Email sent + Crew alerted + Log written` |

### Purpose

Creates a clean final data packet that confirms the workflow ran end-to-end successfully. Useful for monitoring dashboards and makes the execution logs in n8n easy to read at a glance.

---

## Node 25: NoOp — Workflow End

**Node Type:** `No Operation, do nothing`  
**Rename To:** `NoOp — Workflow Complete`  
**Position:** Final node of main workflow

### Purpose

Clean termination point. In n8n, having an explicit end node makes the workflow diagram easier to read and prevents n8n from showing the last active node as the "final" node in execution logs.

---

### ═══════════════════════════════════════
### ERROR TRIGGER BRANCH (Separate Workflow)
### ═══════════════════════════════════════

> **How to set up the Error Trigger:**  
> In n8n, go to the main Everwood workflow → **Settings** → **Error Workflow** → Select this separate error workflow from the dropdown.
>
> The Error Trigger workflow is a **separate workflow** in n8n, not connected by lines to the main workflow. n8n will automatically route failures to it.

---

## Node 26: Error Trigger — Catch All Failures

**Node Type:** `Error Trigger`  
**Rename To:** `Error Trigger — Catch Workflow Failures`  
**Position:** First node of the Error Trigger workflow

### Configuration

No configuration required. This node activates automatically whenever any node in the linked main workflow throws an error.

### Output Data Structure (Auto-provided by n8n)

```json
{
  "execution": {
    "id": "ex_abc123",
    "url": "https://your-n8n.com/execution/abc123",
    "retryOf": null,
    "error": {
      "message": "Request failed with status code 403",
      "stack": "Error: Request failed..."
    },
    "lastNodeExecuted": "Gmail — Send VIP Email",
    "mode": "trigger"
  },
  "workflow": {
    "id": "wf_xyz789",
    "name": "Everwood Project Pulse"
  }
}
```

---

## Node 27: Set — Parse Failure Details

**Node Type:** `Set`  
**Rename To:** `Set — Parse Failure Details`  
**Position:** After Node 26

### Configuration

Add these fields:

| Field Name | Expression |
|---|---|
| `error_timestamp` | `{{$now.toISO()}}` |
| `error_workflow` | `{{$json["workflow"]["name"]}}` |
| `error_node` | `{{$json["execution"]["lastNodeExecuted"]}}` |
| `error_message` | `{{$json["execution"]["error"]["message"]}}` |
| `error_execution_id` | `{{$json["execution"]["id"]}}` |
| `error_execution_url` | `{{$json["execution"]["url"]}}` |
| `raw_json` | `{{JSON.stringify($json)}}` |

---

## Node 28: Google Sheets — Log to Technical Errors Tab

**Node Type:** `Google Sheets`  
**Rename To:** `GSheets — Log Technical Error`  
**Position:** After Node 27

### Configuration

| Setting | Value |
|---|---|
| Credential | `Everwood Google Sheets` |
| Operation | `Append Row` |
| Spreadsheet | Your Spreadsheet ID |
| Sheet Name | `Technical Errors` |

### Columns to Append

| Column | Expression |
|---|---|
| `Timestamp` | `{{$json["error_timestamp"]}}` |
| `Workflow_Name` | `{{$json["error_workflow"]}}` |
| `Failed_Node` | `{{$json["error_node"]}}` |
| `Error_Message` | `{{$json["error_message"]}}` |
| `Execution_ID` | `{{$json["error_execution_id"]}}` |
| `Execution_URL` | `{{$json["error_execution_url"]}}` |
| `Raw_JSON` | `{{$json["raw_json"]}}` |

---

## Node 29: Slack — Admin Failure Alert

**Node Type:** `Slack`  
**Rename To:** `Slack — Admin Failure Alert`  
**Position:** After Node 28

### Configuration

| Setting | Value |
|---|---|
| Credential | `Everwood Slack Admin Webhook` |
| Operation | `Send a Message` |
| Channel | `#management-alerts` |
| Username | `Everwood Error Bot` |
| Icon Emoji | `:x:` |

### Message (paste exactly)

```
❌ *WORKFLOW FAILURE — IMMEDIATE ATTENTION REQUIRED*

*Workflow:* {{$json["error_workflow"]}}
*Failed Node:* {{$json["error_node"]}}
*Error:* {{$json["error_message"]}}
*Time:* {{$json["error_timestamp"]}}
*Execution ID:* {{$json["error_execution_id"]}}

🔗 View execution: {{$json["error_execution_url"]}}

_This error has been logged to the Technical Errors tab in Google Sheets._
```

---

## 8. Exact Expressions & Field Mapping

Use this reference table when building any node in the workflow.

### Core Data Expressions

| Field | Expression | Notes |
|---|---|---|
| Order ID | `{{$json["order_id"]}}` | Always lowercase after Node 4 |
| Customer Name | `{{$json["customer_name"]}}` | |
| Email Address | `{{$json["contact_email"]}}` | |
| Project Type | `{{$json["project_type"]}}` | Kitchen or Bath |
| Client Tier | `{{$json["client_tier"]}}` | VIP or Standard |
| Item Fragility | `{{$json["item_fragility"]}}` | High or Low |
| Current Status | `{{$json["current_status"]}}` | |
| Lead Installer | `{{$json["lead_installer"]}}` | |
| Status Date | `{{$json["status_date"]}}` | |
| Delivery Window (Display) | `{{$json["delivery_window_display"]}}` | After Node 7 only |
| Delivery Window (ISO) | `{{$json["delivery_window_iso"]}}` | After Node 7 only |

### Date & Time Expressions

| Use Case | Expression |
|---|---|
| Current timestamp (ISO) | `{{$now.toISO()}}` |
| Current date only | `{{$now.toISODate()}}` |
| Formatted date | `{{$now.toFormat('MMMM d, yyyy')}}` |
| Current year | `{{$now.year}}` |

### Conditional Expressions

| Use Case | Expression |
|---|---|
| Is VIP? | `{{$json["client_tier"] === "VIP"}}` |
| Is High Fragility? | `{{$json["item_fragility"] === "High"}}` |
| Is Kitchen? | `{{$json["project_type"] === "Kitchen"}}` |
| Priority number | `{{$json["client_tier"] === "VIP" && $json["item_fragility"] === "High" ? 1 : $json["client_tier"] === "VIP" ? 2 : 3}}` |

---

## 9. Projected Delivery Window Logic

### The Business Rule

When a project status changes, Everwood needs to communicate to the client **when they can expect their next milestone** — and that window is defined as **3 business days from the status update date**. Weekends cannot count because Everwood's fabrication partners and delivery teams do not operate on Saturday or Sunday.

### Why a Code Node (Not a Simple Formula)

n8n's built-in date operations don't natively skip weekends. A Code node in JavaScript gives us precise control to loop through days and count only Monday through Friday.

### The Algorithm Step by Step

```
1. Read status_date from the project row
2. If status_date is empty, use today's date
3. Parse the date as a UTC Date object (noon UTC to prevent off-by-one errors)
4. Start a counter: businessDaysAdded = 0
5. While businessDaysAdded < 3:
     a. Move forward 1 calendar day
     b. Check getUTCDay() — 0 = Sunday, 6 = Saturday
     c. If NOT Saturday and NOT Sunday: increment businessDaysAdded
6. Format the resulting date in two ways:
     - Human-readable: "Tue, Apr 16, 2024" (for emails and Slack messages)
     - ISO format: "2024-04-16" (for writing back to the sheet)
7. Return both formats plus the original calculation note
```

### Test Cases to Verify

| Status Date | Day | Expected Delivery Window |
|---|---|---|
| 2024-04-08 (Monday) | Mon | 2024-04-11 (Thursday) |
| 2024-04-10 (Wednesday) | Wed | 2024-04-15 (Monday) |
| 2024-04-11 (Thursday) | Thu | 2024-04-16 (Tuesday) |
| 2024-04-12 (Friday) | Fri | 2024-04-17 (Wednesday) |
| 2024-04-13 (Saturday) | Sat | 2024-04-18 (Thursday) |

---

## 10. VIP Routing Logic

### Decision Point: Node 8 (Switch)

The Switch node reads `client_tier` and routes to the correct branch. All values are case-sensitive in n8n expressions — make sure the sheet uses exactly `VIP` and `Standard`.

### What VIP Clients Receive

| Element | VIP | Standard |
|---|---|---|
| Email subject prefix | 🌟 VIP Project Update | Project Update |
| Salutation | `Dear [Name],` with full concierge tone | `Hello [Name],` |
| Body tone | White-glove luxury language | Professional, clear |
| Closing | Dedicated concierge team sign-off | Team sign-off |
| Response time messaging | "Our concierge team is personally overseeing your project" | Standard update language |
| Reply-to email | `concierge@everwoodinteriors.com` | `projects@everwoodinteriors.com` |

### VIP Subject Line Formula

```
🌟 VIP Project Update — Order #{{$json["order_id"]}} — {{$json["current_status"]}}
```

Example output:
```
🌟 VIP Project Update — Order #EW-2024-007 — Ready for Delivery
```

### VIP Tone Guidelines for Email Copy

Use language like:
- "We are personally overseeing every detail of your project"
- "Your dedicated concierge team"
- "As a valued Everwood client, your project receives our highest priority attention"
- "We will personally confirm with you before any delivery is scheduled"

Avoid:
- Generic "your order" language
- Any mention of timelines that sound automated
- Cold, transactional phrasing

---

## 11. Fragile Routing Logic

### Decision Point: Node 16 (IF)

After the email and initial crew Slack message, the workflow checks `item_fragility`. This is a completely separate routing decision from the VIP/Standard split.

### Fragility Matrix

| Client Tier | Fragility | Email Template | Crew Alert | Management Alert |
|---|---|---|---|---|
| VIP | High | VIP Template | Yes | **YES — Priority** |
| VIP | Low | VIP Template | Yes | No |
| Standard | High | Standard Template | Yes | **YES — Priority** |
| Standard | Low | Standard Template | Yes | No |

### Management Alert Triggers When

```
Item_Fragility = "High"
```

Regardless of client tier. A standard client's fragile marble countertop is just as breakable as a VIP client's.

### Alert Content Must Include

- Order ID
- Customer name + tier
- Lead installer name
- Delivery window
- Project type (Kitchen vs Bath)
- Required action checklist (padded transport, 2-person team, etc.)

### Common High-Fragility Project Examples at Everwood

- Large format marble or quartz slabs (kitchen countertops)
- Custom glass cabinet doors
- Specialty ceramic or porcelain bathroom tile installations
- Imported Italian stone vanity tops
- Hand-crafted wood veneers with lacquer finish

---

## 12. Gmail HTML Templates

### Standard Email Template

Paste this full HTML string into the `email_html` field of **Node 9** (Set — Build Standard Email HTML). Replace the `{{ }}` expressions with n8n expression syntax exactly as shown.

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Project Update — Everwood Bespoke Interiors</title>
</head>
<body style="margin:0;padding:0;background-color:#f5f5f0;font-family:Georgia,serif;">

  <!-- Header -->
  <table width="100%" cellpadding="0" cellspacing="0" border="0">
    <tr>
      <td align="center" style="padding:30px 0;">
        <table width="620" cellpadding="0" cellspacing="0" border="0" style="background-color:#1a1a1a;border-radius:4px 4px 0 0;">
          <tr>
            <td align="center" style="padding:32px 40px;">
              <p style="margin:0;font-family:Georgia,serif;font-size:22px;color:#c9a96e;letter-spacing:3px;text-transform:uppercase;">EVERWOOD</p>
              <p style="margin:4px 0 0;font-family:Georgia,serif;font-size:11px;color:#888;letter-spacing:2px;text-transform:uppercase;">BESPOKE INTERIORS</p>
            </td>
          </tr>
        </table>

        <!-- Body -->
        <table width="620" cellpadding="0" cellspacing="0" border="0" style="background-color:#ffffff;">
          <tr>
            <td style="padding:40px 48px 32px;">
              <p style="margin:0 0 8px;font-size:13px;color:#999;letter-spacing:1px;text-transform:uppercase;">Project Status Update</p>
              <h1 style="margin:0 0 28px;font-size:26px;color:#1a1a1a;font-weight:normal;">{{$json["current_status"]}}</h1>

              <p style="margin:0 0 20px;font-size:16px;color:#333;line-height:1.7;">Hello {{$json["customer_name"]}},</p>

              <p style="margin:0 0 20px;font-size:16px;color:#333;line-height:1.7;">
                We are pleased to share a progress update on your {{$json["project_type"]}} renovation project with Everwood Bespoke Interiors.
              </p>

              <!-- Status Box -->
              <table width="100%" cellpadding="0" cellspacing="0" border="0" style="margin:24px 0;">
                <tr>
                  <td style="background-color:#f9f7f3;border-left:4px solid #c9a96e;padding:20px 24px;border-radius:0 4px 4px 0;">
                    <p style="margin:0 0 6px;font-size:11px;color:#999;letter-spacing:1px;text-transform:uppercase;">Current Status</p>
                    <p style="margin:0 0 16px;font-size:18px;color:#1a1a1a;font-weight:bold;">{{$json["current_status"]}}</p>
                    <p style="margin:0 0 6px;font-size:11px;color:#999;letter-spacing:1px;text-transform:uppercase;">Order Number</p>
                    <p style="margin:0 0 16px;font-size:15px;color:#333;">#{{$json["order_id"]}}</p>
                    <p style="margin:0 0 6px;font-size:11px;color:#999;letter-spacing:1px;text-transform:uppercase;">Projected Delivery Window</p>
                    <p style="margin:0;font-size:15px;color:#333;">{{$json["delivery_window_display"]}}</p>
                  </td>
                </tr>
              </table>

              <p style="margin:0 0 20px;font-size:16px;color:#333;line-height:1.7;">
                Your dedicated project lead will be in contact to coordinate the next steps. If you have any questions in the meantime, please do not hesitate to reach out.
              </p>

              <!-- CTA Button -->
              <table cellpadding="0" cellspacing="0" border="0" style="margin:28px 0;">
                <tr>
                  <td style="background-color:#1a1a1a;border-radius:3px;padding:14px 32px;">
                    <a href="mailto:projects@everwoodinteriors.com" style="color:#c9a96e;font-size:13px;text-decoration:none;letter-spacing:1px;text-transform:uppercase;">Contact Your Project Team</a>
                  </td>
                </tr>
              </table>

              <p style="margin:0;font-size:15px;color:#333;line-height:1.7;">
                Thank you for choosing Everwood Bespoke Interiors.<br>
                <span style="color:#999;">The Everwood Project Team</span>
              </p>
            </td>
          </tr>
        </table>

        <!-- Footer -->
        <table width="620" cellpadding="0" cellspacing="0" border="0" style="background-color:#f0ede8;border-radius:0 0 4px 4px;">
          <tr>
            <td align="center" style="padding:20px 40px;">
              <p style="margin:0;font-size:11px;color:#aaa;letter-spacing:1px;">
                EVERWOOD BESPOKE INTERIORS · LUXURY KITCHEN & BATH REMODELING<br>
                projects@everwoodinteriors.com · (555) 400-7700
              </p>
            </td>
          </tr>
        </table>

      </td>
    </tr>
  </table>

</body>
</html>
```

---

### VIP Concierge Email Template

Paste this full HTML string into the `email_html` field of **Node 10** (Set — Build VIP Concierge Email HTML).

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>VIP Project Update — Everwood Bespoke Interiors</title>
</head>
<body style="margin:0;padding:0;background-color:#0f0f0f;font-family:Georgia,serif;">

  <table width="100%" cellpadding="0" cellspacing="0" border="0">
    <tr>
      <td align="center" style="padding:40px 0;">
        <table width="640" cellpadding="0" cellspacing="0" border="0" style="background-color:#111111;border:1px solid #2a2a2a;border-radius:6px;">

          <!-- Gold Header Band -->
          <tr>
            <td style="background:linear-gradient(135deg,#c9a96e,#e8c88a,#c9a96e);padding:3px 0;border-radius:6px 6px 0 0;"></td>
          </tr>

          <!-- Header -->
          <tr>
            <td align="center" style="padding:36px 48px 24px;">
              <p style="margin:0 0 4px;font-size:10px;color:#c9a96e;letter-spacing:4px;text-transform:uppercase;">✦ Private Client Services ✦</p>
              <p style="margin:0;font-family:Georgia,serif;font-size:26px;color:#e8e0d0;letter-spacing:4px;text-transform:uppercase;">EVERWOOD</p>
              <p style="margin:2px 0 0;font-size:11px;color:#666;letter-spacing:3px;text-transform:uppercase;">Bespoke Interiors</p>
            </td>
          </tr>

          <!-- Divider -->
          <tr>
            <td style="padding:0 48px;">
              <table width="100%" cellpadding="0" cellspacing="0" border="0">
                <tr><td style="border-top:1px solid #2a2a2a;"></td></tr>
              </table>
            </td>
          </tr>

          <!-- Body -->
          <tr>
            <td style="padding:36px 48px;">
              <p style="margin:0 0 6px;font-size:10px;color:#c9a96e;letter-spacing:3px;text-transform:uppercase;">Priority Status Update</p>
              <h1 style="margin:0 0 32px;font-size:28px;color:#e8e0d0;font-weight:normal;letter-spacing:1px;">{{$json["current_status"]}}</h1>

              <p style="margin:0 0 22px;font-size:16px;color:#c0b8a8;line-height:1.8;">Dear {{$json["customer_name"]}},</p>

              <p style="margin:0 0 22px;font-size:16px;color:#c0b8a8;line-height:1.8;">
                Your concierge team at Everwood Bespoke Interiors is pleased to bring you a personal update on your {{$json["project_type"]}} project. As one of our esteemed private clients, your project receives our highest level of attention and care at every stage of the process.
              </p>

              <!-- Status Panel -->
              <table width="100%" cellpadding="0" cellspacing="0" border="0" style="margin:28px 0;border:1px solid #2a2a2a;border-radius:4px;">
                <tr>
                  <td style="padding:28px 32px;">
                    <table width="100%" cellpadding="0" cellspacing="0" border="0">
                      <tr>
                        <td width="50%" style="padding-bottom:20px;vertical-align:top;">
                          <p style="margin:0 0 6px;font-size:10px;color:#c9a96e;letter-spacing:2px;text-transform:uppercase;">Project Status</p>
                          <p style="margin:0;font-size:17px;color:#e8e0d0;font-weight:bold;">{{$json["current_status"]}}</p>
                        </td>
                        <td width="50%" style="padding-bottom:20px;vertical-align:top;">
                          <p style="margin:0 0 6px;font-size:10px;color:#c9a96e;letter-spacing:2px;text-transform:uppercase;">Project Reference</p>
                          <p style="margin:0;font-size:17px;color:#e8e0d0;">#{{$json["order_id"]}}</p>
                        </td>
                      </tr>
                      <tr>
                        <td width="50%" style="vertical-align:top;">
                          <p style="margin:0 0 6px;font-size:10px;color:#c9a96e;letter-spacing:2px;text-transform:uppercase;">Delivery Window</p>
                          <p style="margin:0;font-size:17px;color:#e8e0d0;">{{$json["delivery_window_display"]}}</p>
                        </td>
                        <td width="50%" style="vertical-align:top;">
                          <p style="margin:0 0 6px;font-size:10px;color:#c9a96e;letter-spacing:2px;text-transform:uppercase;">Project Type</p>
                          <p style="margin:0;font-size:17px;color:#e8e0d0;">{{$json["project_type"]}} Renovation</p>
                        </td>
                      </tr>
                    </table>
                  </td>
                </tr>
              </table>

              <p style="margin:0 0 22px;font-size:16px;color:#c0b8a8;line-height:1.8;">
                Our concierge team is personally overseeing every detail of your installation and will reach out directly to confirm all next steps before they are taken. Your peace of mind is our commitment.
              </p>

              <p style="margin:0 0 22px;font-size:16px;color:#c0b8a8;line-height:1.8;">
                Should you have any questions or wish to speak with a team member, our dedicated concierge line is available to you personally.
              </p>

              <!-- CTA -->
              <table cellpadding="0" cellspacing="0" border="0" style="margin:28px 0;">
                <tr>
                  <td style="background-color:#c9a96e;border-radius:3px;padding:15px 36px;">
                    <a href="mailto:concierge@everwoodinteriors.com" style="color:#111;font-size:12px;font-weight:bold;text-decoration:none;letter-spacing:2px;text-transform:uppercase;">Contact Your Concierge Team</a>
                  </td>
                </tr>
              </table>

              <p style="margin:0;font-size:15px;color:#888;line-height:1.8;">
                With warmest regards,<br>
                <span style="color:#c9a96e;">Your Dedicated Everwood Concierge Team</span><br>
                <span style="font-size:13px;">concierge@everwoodinteriors.com · (555) 400-7700 ext. 1</span>
              </p>
            </td>
          </tr>

          <!-- Gold Footer Band -->
          <tr>
            <td style="background:linear-gradient(135deg,#c9a96e,#e8c88a,#c9a96e);padding:2px 0;border-radius:0 0 6px 6px;"></td>
          </tr>

        </table>
      </td>
    </tr>
  </table>

</body>
</html>
```

---

## 13. Slack & Discord Alert Templates

### Crew Alert (Node 14 → Node 15)

Displayed in the `#install-crew` channel. Formatted for Slack's markdown (mrkdwn).

```
🔧 *INSTALL CREW UPDATE — EVERWOOD PROJECT*
━━━━━━━━━━━━━━━━━━━━━━━━━━

*Order:*           #{{$json["order_id"]}}
*Client:*          {{$json["customer_name"]}}
*Project Type:*    {{$json["project_type"]}}
*Status:*          {{$json["current_status"]}}
*Lead Installer:*  {{$json["lead_installer"]}}
*Delivery Window:* {{$json["delivery_window_display"]}}
*Client Tier:*     {{$json["client_tier"]}}

━━━━━━━━━━━━━━━━━━━━━━━━━━
_Please confirm receipt by reacting with ✅_
_Sent automatically by Everwood Project Pulse_
```

---

### Fragile Management Alert (Node 17 → Node 19)

Displayed in the `#management-alerts` channel. High visual priority.

```
🚨 *SPECIALIZED HANDLING REQUIRED*
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

⚠️ *HIGH FRAGILITY* — This project requires specialized transport protocols.

*Order:*           #{{$json["order_id"]}}
*Client:*          {{$json["customer_name"]}} _({{$json["client_tier"]}})_
*Project Type:*    {{$json["project_type"]}} Renovation
*Status:*          {{$json["current_status"]}}
*Lead Installer:*  {{$json["lead_installer"]}}
*Delivery Window:* {{$json["delivery_window_display"]}}

*Required Actions Before Dispatch:*
• ☐ Confirm padded/climate transport vehicle booked
• ☐ Assign minimum 2-person delivery team
• ☐ Notify warehouse manager 48hrs in advance
• ☐ Confirm white-glove setup team on site

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
_@channel — management approval required before dispatch_
_Auto-generated by Everwood Project Pulse · {{$now.toISO()}}_
```

---

### Discord Alternative (Optional)

If the client uses Discord instead of Slack, replace the Slack nodes with **Discord** nodes and use this format (Discord uses slightly different markdown):

```
🔧 **INSTALL CREW UPDATE — EVERWOOD PROJECT**
Order: #{{$json["order_id"]}}
Client: {{$json["customer_name"]}}
Status: {{$json["current_status"]}}
Lead Installer: {{$json["lead_installer"]}}
Delivery Window: {{$json["delivery_window_display"]}}
```

In n8n: Replace the `Slack` node with a `Discord` node → Set Webhook URL → Channel name not required (the webhook URL is channel-specific).

---

## 14. Error Handling Architecture

### Philosophy

Every automation breaks eventually. Gmail OAuth tokens expire. Slack webhooks get revoked. A row gets a typo. The question is not whether errors will happen — it is whether the team knows about them immediately and has a permanent record.

### Error Trigger Setup

In n8n:
1. Build the Error Trigger workflow (Nodes 26–29) as a **separate workflow**
2. Go back to the main Everwood workflow
3. Click the **Settings** (gear icon) at the top
4. Find **Error Workflow**
5. Select the Error Trigger workflow from the dropdown

Now, whenever any node in the main workflow fails, n8n automatically calls the Error Trigger workflow.

### What Gets Caught

| Failure Type | Where It Breaks | Error Trigger Response |
|---|---|---|
| Gmail OAuth expired | Node 11 or 12 | Logged + Admin Slack alert |
| Missing `contact_email` | Node 5 (IF) | Logged to Technical Errors tab (not full error trigger) |
| Slack webhook revoked | Node 15 or 19 | Logged + Admin Slack alert |
| Invalid status_date format | Node 7 (Code) | Full error trigger fires |
| Google Sheets write fails | Node 20 or 23 | Logged + Admin Slack alert |
| Invalid JSON from Airtable | Node 3 (Merge) | Logged + Admin Slack alert |
| Network timeout | Any node | Full error trigger fires |

### Technical Errors Tab Structure

Create this sheet manually before going live:

| Column | Purpose |
|---|---|
| `Timestamp` | ISO timestamp of when the error occurred |
| `Workflow_Name` | Always "Everwood Project Pulse" |
| `Failed_Node` | Which node threw the error |
| `Error_Message` | The raw error text from n8n |
| `Execution_ID` | The n8n execution ID — click it to see the full failed run |
| `Execution_URL` | Direct link to the failed execution in n8n |
| `Raw_JSON` | The full data packet present at time of failure (for debugging) |

---

## 15. Logging & Tracking Enhancements

### Activity Log Tab Structure

This tab grows automatically as the workflow runs. One row per successful workflow execution.

| Column | What Goes Here |
|---|---|
| `Timestamp` | When the notification was sent |
| `Order_ID` | The project order number |
| `Customer` | Client name |
| `Status` | Status that triggered the run |
| `Client_Tier` | VIP or Standard |
| `Fragility` | High or Low |
| `Delivery_Window` | Calculated delivery date |
| `Email_Sent` | TRUE/FALSE |
| `Crew_Alerted` | TRUE/FALSE |
| `Fragile_Alert_Sent` | TRUE/FALSE |
| `Source` | Google Sheets or Airtable |

### What To Track Over Time

Once the Activity Log has 2–4 weeks of data, Everwood can answer questions like:

- How many VIP projects were processed this month?
- What percentage of projects involved high-fragility items?
- Which project types move fastest through statuses?
- Are there any projects that never got a notification? (cross-reference with the main sheet)

### Priority Level Logic (Written to Main Sheet)

The `Priority_Level` field (1–3) is calculated in Node 20:

| Condition | Priority Level |
|---|---|
| VIP + High Fragility | 1 (Highest — sort to top of dashboard) |
| VIP + Low Fragility | 2 |
| Standard (any fragility) | 3 |

---

## 16. Testing Procedure

### Before You Test

1. Make sure all credentials are connected and tested in n8n's Credential panel
2. Make sure the Google Sheet has data from the CSV import
3. Make sure both Slack webhook URLs work (test them directly from the n8n credential panel)
4. Make sure the Error Trigger workflow is activated and linked

---

### Test Case 1: VIP + High Fragility (Full Path)

**Setup:** Find or create a row in the sheet with:

```
Client_Tier: VIP
Item_Fragility: High
Current_Status: Ready for Delivery
Contact_Email: [your own email for testing]
Lead_Installer: Marcus Webb
Status_Date: [today's date]
```

**Expected Result:**

| Check | Expected |
|---|---|
| Node 5 (IF) | TRUE — passes validation |
| Node 7 (Code) | Delivery window calculated correctly |
| Node 8 (Switch) | Routes to VIP branch |
| Node 10 (Set) | VIP HTML built with subject containing 🌟 |
| Node 12 (Gmail) | VIP email received in your test inbox |
| Node 14 (Set) | Crew Slack message built |
| Node 15 (Slack) | Message appears in #install-crew |
| Node 16 (IF) | TRUE — routes to fragile alert |
| Node 19 (Slack) | 🚨 alert appears in #management-alerts |
| Node 20 (GSheets) | `Last_Notified` timestamp updated in the sheet |
| Node 23 (GSheets) | Row added to Activity Log tab |

---

### Test Case 2: Standard + Low Fragility (Abbreviated Path)

**Setup:** Find or create a row with:

```
Client_Tier: Standard
Item_Fragility: Low
Current_Status: In Production
Contact_Email: [your own email]
Lead_Installer: Jordan Lee
```

**Expected Result:**

| Check | Expected |
|---|---|
| Node 8 (Switch) | Routes to Standard branch |
| Node 9 (Set) | Standard HTML built with plain subject |
| Node 11 (Gmail) | Standard email received |
| Node 16 (IF) | FALSE — routes to NoOp skip node |
| Node 19 (Slack) | NOT fired — no fragile alert |
| Node 23 (GSheets) | Row added to Activity Log with Fragile_Alert_Sent = FALSE |

---

### Test Case 3: Invalid Row — Missing Email (Error Path)

**Setup:** Create a row with `Contact_Email` deliberately left blank.

**Expected Result:**

| Check | Expected |
|---|---|
| Node 5 (IF) | FALSE — fails validation |
| Node 6 (GSheets) | Row appended to Technical Errors tab |
| Slack admin | NO error Slack (this is caught before Email send, not a crash) |

---

## 17. Deliverables Checklist

Use this checklist when wrapping up the build for the client.

### Build Deliverables

- [ ] Main workflow JSON exported from n8n (download via **...** → **Download**)
- [ ] Error Trigger workflow JSON exported separately
- [ ] Google Sheet shared with client (Editor access)
- [ ] Activity Log tab set up with correct headers
- [ ] Technical Errors tab set up with correct headers
- [ ] All credentials documented (not the secrets — just what type and what name)

### Proof of Work Screenshots

- [ ] Full workflow canvas screenshot (zoom out to show all 25+ nodes)
- [ ] Node 7 (Code) execution screenshot showing correct delivery window output
- [ ] Gmail inbox screenshot showing both Standard and VIP email received
- [ ] Slack #install-crew screenshot showing crew alert
- [ ] Slack #management-alerts screenshot showing 🚨 fragile alert
- [ ] Activity Log tab screenshot showing at least 2 rows logged
- [ ] Technical Errors tab screenshot (even if empty — shows it's set up)
- [ ] n8n Execution history showing green (success) runs

### Video Deliverable

- [ ] 30–60 second Loom walkthrough:
  - Start with Google Sheet — change a row status manually
  - Switch to n8n and watch the execution light up live
  - Cut to Gmail inbox showing the email that just arrived
  - Cut to Slack showing the crew alert
  - End with Activity Log showing the new row

---

## 18. Troubleshooting

### Gmail OAuth Expired

**Symptom:** Node 11 or 12 fails with `401 Unauthorized` or `Token expired`  
**Fix:**
1. Go to n8n → Credentials → `Everwood Gmail`
2. Click **Reconnect** → Re-authorize with Google
3. Go to Google Cloud Console → Check that the OAuth consent screen is not in "Testing" mode (if Testing, only approved test users can authorize)
4. After reconnecting, re-open the Gmail node and re-select the credential

---

### Trigger Not Firing

**Symptom:** You update the sheet but the workflow never starts  
**Fix:**
1. Check the workflow is **Active** (toggle in top-right of n8n)
2. Verify the `Columns to Watch` setting in Node 1 is set to `Current_Status`
3. Make sure the poll interval hasn't been set too high (1 minute max for testing)
4. Check the sheet name in Node 1 exactly matches the tab name (case-sensitive)
5. Try manually clicking **Test Trigger** in n8n and updating a row within 60 seconds

---

### Duplicate Notifications

**Symptom:** The same project triggers two emails  
**Fix:**
1. Add a check in Node 5 (IF) for `Last_Notified` — if it was within the last 60 minutes, skip
2. Expression to add: `{{$json["last_notified"] === "" || $now.diff($DateTime.fromISO($json["last_notified"]), "minutes").minutes > 60}}`
3. This prevents re-triggers if the sheet auto-saves and re-triggers the watch

---

### HTML Email Broken Formatting

**Symptom:** Email arrives with raw HTML or broken layout  
**Fix:**
1. In the Gmail node, make sure **Email Type** is set to `HTML` (not Plain Text)
2. Make sure the HTML template in the Set node is stored as a plain string (not wrapped in extra quotes)
3. Test the HTML in a browser first — paste into an `.html` file and open it

---

### Date Math Wrong (Weekend Not Skipped)

**Symptom:** Delivery window falls on a Saturday or Sunday  
**Fix:**
1. Check `status_date` in the sheet — make sure it's in `YYYY-MM-DD` format
2. Open Node 7 in n8n → Run a test → Check `$json["status_date"]` in the input panel
3. The JavaScript uses `getUTCDay()` — if the date parsing is off by a timezone, the day number will be wrong
4. The fix is already baked in: `new Date(rawDate + 'T12:00:00Z')` forces noon UTC

---

### Slack Webhook Invalid

**Symptom:** Node 15 or 19 fails with `invalid_payload` or `no_service`  
**Fix:**
1. The webhook URL may have been deleted in Slack's app settings
2. Go to api.slack.com/apps → Your App → Incoming Webhooks → Add new webhook
3. Update the credential in n8n with the new URL

---

### Missing Row Values After Normalization

**Symptom:** Downstream nodes get `undefined` or empty values  
**Fix:**
1. Open Node 4 in n8n → Click **Test Step** → Check the output panel
2. Look for any field that shows `null` or is missing entirely
3. The original column names in the sheet must match exactly what Node 4 expects (e.g., `Customer_Name` not `customer name`)
4. Case matters — fix the column header in the sheet if needed

---

## 19. Pro Freelancer Tips

### Canvas Organization That Impresses Clients

When you record the Loom or take the full-canvas screenshot, the client will judge the quality of the work by how the workflow *looks* before they even test it. Take 20 minutes to do this right.

**Sticky Note Labels to Add (use the n8n Sticky Note node):**

| Sticky Note | Position |
|---|---|
| `🔗 TRIGGER LAYER — Watches for status changes every 60s` | Above Nodes 1–3 |
| `🧹 DATA PREP — Validates + normalizes every row before processing` | Above Nodes 4–7 |
| `📧 EMAIL ROUTING — VIP gets luxury template, Standard gets pro template` | Above Nodes 8–13 |
| `📢 CREW ALERTS — Install team + fragile item management escalation` | Above Nodes 14–19 |
| `📋 LOGGING — Every action is permanently recorded` | Above Nodes 20–25 |
| `🚨 ERROR BRANCH — Runs in a separate workflow automatically on any failure` | Near error workflow reference |

**Color Coding (right-click any node → Change Color):**

| Color | Use For |
|---|---|
| Gray | Trigger nodes |
| Blue | Set / normalize nodes |
| Purple | IF / Switch routing nodes |
| Green | Gmail send nodes |
| Yellow | Slack send nodes |
| Red | Error handling nodes |
| Teal | Google Sheets read/write nodes |

---

### Naming Convention

Every node follows this format: `[Type] — [What It Does]`

Examples:
- `Set — Normalize Fields` ✅
- `normalize` ❌
- `Gmail — Send VIP Email` ✅
- `Gmail node` ❌

Consistent naming makes the n8n execution logs readable at a glance — which matters when something goes wrong at 2am and the client needs to find the problem fast.

---

### Add Retry Logic to Critical Nodes

For Gmail and Slack nodes specifically:

1. Right-click the node → **Settings**
2. Enable **Retry on Fail**: ON
3. Max Tries: `3`
4. Wait Between Tries: `5000` ms (5 seconds)

This handles transient network errors without triggering the full error workflow.

---

### Make the Activity Log a Live Dashboard

Once the Activity Log has data, help the client build a simple Google Sheets dashboard:

1. On a new tab called `Dashboard`, use `COUNTIF` to count how many VIP vs Standard projects ran this month
2. Use a `SPARKLINE` to show notification volume over time
3. Use conditional formatting to highlight any rows where `Fragile_Alert_Sent = TRUE`
4. This transforms a simple log into a business intelligence tool the client will actually look at

---

### Realistic Test Data to Use

When demoing, use project names and statuses that sound real:

```
EW-2024-001 | Victoria Ashworth  | Kitchen  | VIP      | High | Ready for Delivery
EW-2024-002 | James Caldwell     | Bath     | Standard | Low  | In Production
EW-2024-003 | Margaux Delacroix  | Kitchen  | VIP      | High | Install Scheduled
EW-2024-004 | Robert Finley      | Bath     | Standard | Low  | Completed
EW-2024-005 | Eleanor Harrington | Kitchen  | VIP      | Low  | Delayed
```

Real-sounding names, realistic statuses, and a mix of tiers and fragility levels will make your demo feel like a production system rather than a test.

---

### Scope Expansion Ideas to Pitch After Delivery

Once Everwood is happy with this workflow, propose these as Phase 2 add-ons:

1. **Client Portal Link** — Add a URL to each email pointing to a Notion or Google Sites project tracker page personalized to the order
2. **Automated Follow-Up Sequence** — If `Last_Notified` is older than 7 days and the status hasn't changed to `Completed`, send a "still on track" check-in email
3. **Supplier Notification Branch** — If status changes to `In Production`, automatically email the cabinetry supplier with the project spec sheet (from Google Drive)
4. **SMS Alerts via Twilio** — Add a Twilio node after the Gmail node to send VIP clients a text confirmation in addition to the email
5. **Airtable Dashboard Integration** — Auto-update an Airtable view that the client's production manager uses as a Kanban board

---

*End of Implementation Guide*

---

> **Document Version:** 1.0  
> **Created For:** Everwood Bespoke Interiors  
> **Workflow Node Count:** 32 nodes (28 main + 4 error branch)  
> **Estimated Setup Time:** 4–6 hours for a freelancer familiar with n8n  
> **n8n Minimum Version:** 1.0.0  
> **Last Updated:** May 2026
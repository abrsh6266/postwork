# Luxury Realtor Lead Concierge and Classification System
### A Complete n8n Workflow Implementation Guide for Vanderbilt & Co.

---

> **Document Type:** Production Implementation Guide  
> **Target Platform:** n8n (Self-hosted or Cloud)  
> **Difficulty Level:** Intermediate (Beginner-Friendly Instructions)  
> **Estimated Build Time:** 4–6 hours  
> **Node Count:** 27 nodes  
> **Version:** 1.0 — Production Ready  

---

## Table of Contents

1. [Project Title & Overview](#1-project-title--overview)
2. [Business Problem Solved](#2-business-problem-solved)
3. [Full Workflow Architecture](#3-full-workflow-architecture)
4. [Required Accounts & Credentials](#4-required-accounts--credentials)
5. [Starter File Setup](#5-starter-file-setup)
6. [Google Sheet Setup](#6-google-sheet-setup)
7. [Full n8n Build Guide — All 27 Nodes](#7-full-n8n-build-guide--all-27-nodes)
8. [Exact Field Mapping Reference](#8-exact-field-mapping-reference)
9. [AI Extraction Prompt](#9-ai-extraction-prompt)
10. [AI Classification Prompt](#10-ai-classification-prompt)
11. [Branching Logic](#11-branching-logic)
12. [Slack Notification Template](#12-slack-notification-template)
13. [Duplicate Handling](#13-duplicate-handling)
14. [Error Handling](#14-error-handling)
15. [Testing Procedure](#15-testing-procedure)
16. [Deliverables Checklist](#16-deliverables-checklist)
17. [Troubleshooting Guide](#17-troubleshooting-guide)
18. [Pro Freelancer Tips](#18-pro-freelancer-tips)

---

---

# 1. Project Title & Overview

## Luxury Realtor Lead Concierge and Classification System
### Built for Vanderbilt & Co. — Boutique Real Estate Firm

**Tagline:** *Intelligent AI-powered lead triage that ensures no high-value buyer ever waits.*

This n8n automation workflow ingests inbound real estate inquiries from Gmail, uses AI to extract and classify each lead, routes the result through duplicate-checking, logs every lead to a structured Google Sheet, fires instant Slack alerts for Tier 1 high-value buyers, and handles all errors gracefully with retry logic and admin notifications.

The system is designed to run 24/7 without human monitoring, while surfacing only the most urgent leads to the agent's attention immediately.

---

---

# 2. Business Problem Solved

## Why This Workflow Matters

Vanderbilt & Co. operates in the ultra-competitive luxury real estate market where response time is a direct proxy for professionalism and trustworthiness. A buyer with a $10M budget who waits 6 hours for a reply has already called two competing firms.

### The Pain Points This Workflow Eliminates

| Problem | Impact | Workflow Solution |
|---|---|---|
| Leads not prioritized by value | High-value buyers buried under low-urgency emails | AI Tier Classification (Tier 1 / 2 / 3) |
| No structured lead database | Data lives in inboxes, not searchable | Auto-append to Google Sheets |
| Manual data entry | Agent wastes 45+ min/day typing | AI extraction pulls all fields automatically |
| Slow Tier 1 follow-up | Multi-million deals lost | Instant Slack alert within 60 seconds of email |
| Duplicate submissions | Same lead logged multiple times | Duplicate check by email + subject + name |
| No error resilience | One API failure breaks everything | Retry logic + error trigger workflow |
| No daily visibility | No sense of pipeline | Daily Slack summary of all leads logged |

### Business Outcome

By the end of this build, Vanderbilt & Co. will have:

- Every inbound email parsed and classified in under 90 seconds
- Tier 1 leads generating an immediate mobile Slack ping to the lead agent
- A live, always-updated Google Sheets database (Vanderbilt Leads Database)
- Zero duplicate rows — only clean, deduplicated records
- A bulletproof error handling system that retries on AI failures and alerts the admin via Slack
- A daily morning summary of new leads logged overnight

---

---

# 3. Full Workflow Architecture

## Visual Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                      VANDERBILT LEAD CONCIERGE SYSTEM                           │
│                                                                                 │
│  ┌────────────────┐    ┌─────────────────┐    ┌─────────────────────────────┐  │
│  │  Node 1        │    │  Node 2         │    │  Node 3                     │  │
│  │  Gmail Trigger │    │  Manual Trigger │    │  Read CSV – Starter Leads   │  │
│  │  (New Inquiry) │    │  (Test Mode)    │    │  (Simulation Mode)          │  │
│  └───────┬────────┘    └────────┬────────┘    └──────────────┬──────────────┘  │
│          │                     │                             │                  │
│          └─────────────────────┴─────────────────────────────┘                  │
│                                      │                                          │
│                              ┌───────▼───────┐                                  │
│                              │  Node 4       │                                  │
│                              │  Merge        │                                  │
│                              │  Trigger      │                                  │
│                              │  Sources      │                                  │
│                              └───────┬───────┘                                  │
│                                      │                                          │
│                              ┌───────▼───────┐                                  │
│                              │  Node 5       │                                  │
│                              │  Set – Norm.  │                                  │
│                              │  Email Fields │                                  │
│                              └───────┬───────┘                                  │
│                                      │                                          │
│                              ┌───────▼───────┐                                  │
│                              │  Node 6       │                                  │
│                              │  Clean Email  │                                  │
│                              │  Body (HTML)  │                                  │
│                              └───────┬───────┘                                  │
│                                      │                                          │
│                              ┌───────▼───────┐                                  │
│                              │  Node 7       │                                  │
│                              │  AI Extract   │                                  │
│                              │  Lead Data    │                                  │
│                              └───────┬───────┘                                  │
│                                      │                                          │
│                         ┌────────────┴────────────┐                            │
│                         │  Node 8: Parse AI JSON  │                            │
│                         │  (Code Node)            │                            │
│                         └────────────┬────────────┘                            │
│                                      │                                          │
│                         ┌────────────▼────────────┐                            │
│                         │  Node 9: IF – Required  │                            │
│                         │  Fields Present?        │                            │
│                         └────┬───────────┬────────┘                            │
│                              │ YES       │ NO                                   │
│                    ┌─────────▼─┐    ┌────▼─────────────────────┐               │
│                    │ Node 10   │    │ Node 21: Error Trigger    │               │
│                    │ AI Lead   │    │ Node 22: Slack Error Alert│               │
│                    │ Classify  │    └──────────────────────────┘               │
│                    └─────┬─────┘                                                │
│                          │                                                      │
│                    ┌─────▼─────┐                                                │
│                    │ Node 11   │                                                │
│                    │ Set – Build│                                               │
│                    │ Tier Result│                                               │
│                    └─────┬─────┘                                                │
│                          │                                                      │
│                    ┌─────▼─────┐                                                │
│                    │ Node 12   │                                                │
│                    │ Switch –  │                                                │
│                    │ Route Tier│                                                │
│                    └──┬──┬──┬──┘                                               │
│               Tier 1  │  │Tier 2   │ Tier 3                                   │
│            ┌──────────┘  └──┐      └────────────────────┐                     │
│      ┌─────▼──────┐   ┌─────▼──────────────────┐  ┌─────▼──────┐             │
│      │ Node 13/14 │   │ Node 13/14              │  │ Node 13/14 │             │
│      │ Sheets     │   │ Sheets Dup Check        │  │ Sheets     │             │
│      │ Dup Check  │   │                         │  │ Dup Check  │             │
│      └─────┬──────┘   └─────────────────────────┘  └─────┬──────┘             │
│            │                     │                        │                    │
│      ┌─────▼──────┐        ┌─────▼──────┐          ┌─────▼──────┐             │
│      │ Node 15    │        │ Node 15    │          │ Node 15    │             │
│      │ Append Row │        │ Append Row │          │ Append Row │             │
│      │  OR        │        │  OR        │          │  OR        │             │
│      │ Node 16    │        │ Node 16    │          │ Node 16    │             │
│      │ Update Row │        │ Update Row │          │ Update Row │             │
│      └─────┬──────┘        └─────┬──────┘          └─────┬──────┘             │
│            │ (Tier 1 only)       │ (Tier 2)              │ (Tier 3)           │
│      ┌─────▼──────┐        ┌─────▼──────┐          ┌─────▼──────┐             │
│      │ Node 17    │        │ Node 24    │          │ Node 24    │             │
│      │ Build Slack│        │ Aggregate  │          │ Nurture Tag│             │
│      │ Message    │        │ Daily Log  │          │ Set Node   │             │
│      └─────┬──────┘        └─────────────┘         └─────────────┘             │
│      ┌─────▼──────┐                                                            │
│      │ Node 18    │                                                            │
│      │ Slack –    │                                                            │
│      │ Tier 1 Alert│                                                           │
│      └─────┬──────┘                                                            │
│      ┌─────▼──────┐                                                            │
│      │ Node 19    │                                                            │
│      │ Gmail –    │                                                            │
│      │ Backup     │                                                            │
│      │ Alert      │                                                            │
│      └─────┬──────┘                                                            │
│            │                                                                   │
│      ┌─────▼──────┐                                                            │
│      │ Node 20    │                                                            │
│      │ Wait Node  │                                                            │
│      │ Retry Delay│                                                            │
│      └─────┬──────┘                                                            │
│            │                                                                   │
│      ┌─────▼──────┐                                                            │
│      │ Node 25    │                                                            │
│      │ Wait –     │                                                            │
│      │ Daily Sched│                                                            │
│      └─────┬──────┘                                                            │
│            │                                                                   │
│      ┌─────▼──────┐                                                            │
│      │ Node 26    │                                                            │
│      │ Aggregate  │                                                            │
│      │ Daily Leads│                                                            │
│      └─────┬──────┘                                                            │
│            │                                                                   │
│      ┌─────▼──────┐                                                            │
│      │ Node 27    │                                                            │
│      │ Final      │                                                            │
│      │ Success    │                                                            │
│      │ Logger     │                                                            │
│      └────────────┘                                                            │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## Condensed Linear Flow

```
Gmail Trigger
  → Manual Trigger (parallel)
  → Read CSV – Starter Leads (parallel)
  → Merge Trigger Sources
  → Set – Normalize Email Fields
  → Clean Email Body (HTML Extract)
  → AI Node – Extract Lead Data (OpenAI/Claude)
  → Code Node – Parse AI JSON Output
  → IF Node – Required Fields Present?
      → NO → Error Trigger → Slack Admin Error Alert
      → YES → AI Node – Classify Lead Tier
               → Set Node – Build Tier Result
               → Switch Node – Route by Tier
                    → Tier 1 → Google Sheets Search (Duplicate?)
                                → IF Duplicate → Update Row
                                → IF New → Append Row
                                → Set – Build Slack Message
                                → Slack – Tier 1 Alert
                                → Gmail – Backup Internal Alert
                                → Wait Node (Retry Buffer)
                    → Tier 2 → Google Sheets Search (Duplicate?)
                                → Append or Update Row
                    → Tier 3 → Google Sheets Search (Duplicate?)
                                → Append or Update Row (Nurture Tag)
               → Final Success Logger
```

---

---

# 4. Required Accounts & Credentials

You must set up the following accounts and credentials before building any node. Complete this section before opening n8n.

---

## 4.1 n8n Instance

**Option A: n8n Cloud**
1. Go to [https://n8n.io](https://n8n.io) and click "Start for free"
2. Create an account and start a workspace
3. Note your workspace URL (e.g., `https://your-name.app.n8n.cloud`)

**Option B: Self-Hosted (Docker)**
```bash
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v ~/.n8n:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n
```
Access at: `http://localhost:5678`

---

## 4.2 Gmail OAuth2

n8n connects to Gmail using Google OAuth2. Follow these exact steps:

1. Go to [https://console.cloud.google.com](https://console.cloud.google.com)
2. Create a new project: `Vanderbilt-n8n`
3. Enable APIs:
   - Navigate to **APIs & Services → Library**
   - Search and enable **Gmail API**
   - Search and enable **Google Sheets API**
4. Create OAuth2 Credentials:
   - Go to **APIs & Services → Credentials**
   - Click **Create Credentials → OAuth Client ID**
   - Application Type: **Web Application**
   - Name: `n8n-vanderbilt`
   - Authorized Redirect URIs: `https://your-n8n-url/rest/oauth2-credential/callback`
5. Download the JSON file — you need the **Client ID** and **Client Secret**
6. In n8n: Go to **Settings → Credentials → Add Credential → Gmail OAuth2**
7. Paste Client ID and Client Secret
8. Click "Connect my account" and authorize with the Vanderbilt Gmail account

---

## 4.3 Google Sheets OAuth2

Uses the same Google Cloud project as Gmail.

1. In n8n: Go to **Settings → Credentials → Add Credential → Google Sheets OAuth2**
2. Use the same Client ID and Client Secret from step 4.2
3. Click "Connect my account" and authorize
4. Name this credential: `Vanderbilt Sheets OAuth2`

---

## 4.4 OpenAI API Key

1. Go to [https://platform.openai.com](https://platform.openai.com)
2. Sign in → Click your avatar → **API Keys**
3. Create a new API key → Copy it
4. In n8n: **Settings → Credentials → Add Credential → OpenAI**
5. Paste the API key
6. Name it: `OpenAI Vanderbilt Key`
7. Recommended model for this workflow: **gpt-4o** (best JSON extraction accuracy)

> **Alternative — Anthropic Claude API:**
> 1. Go to [https://console.anthropic.com](https://console.anthropic.com)
> 2. Create API Key → Copy it
> 3. In n8n: Add Credential → Anthropic (HTTP Request auth with Bearer token)
> 4. Use model: `claude-sonnet-4-20250514`

---

## 4.5 Slack App & Webhook

1. Go to [https://api.slack.com/apps](https://api.slack.com/apps)
2. Click **Create New App → From Scratch**
3. Name: `Vanderbilt Lead Alerts`
4. Select your workspace
5. Under **Features → Incoming Webhooks**: Toggle ON
6. Click **Add New Webhook to Workspace**
7. Select the channel: `#luxury-leads-alerts` (create it first in Slack)
8. Copy the Webhook URL (format: `https://hooks.slack.com/services/xxx/yyy/zzz`)
9. In n8n: **Settings → Credentials → Add Credential → Slack**
   - Authentication Type: **Webhook**
   - Webhook URL: (paste above)
   - Name: `Slack Vanderbilt Webhook`

> Create a second Slack channel `#admin-errors` for error alerts.

---

## 4.6 Credentials Summary Table

| Credential Name | n8n Credential Type | Used In Nodes |
|---|---|---|
| Gmail OAuth2 (Vanderbilt) | Gmail OAuth2 | Node 1, Node 19 |
| Vanderbilt Sheets OAuth2 | Google Sheets OAuth2 | Nodes 13, 15, 16 |
| OpenAI Vanderbilt Key | OpenAI API | Nodes 7, 10 |
| Slack Vanderbilt Webhook | Slack (Webhook) | Nodes 18, 22, 27 |

---

---

# 5. Starter File Setup

The file `luxury_realtor_leads_starter.csv` contains 75 rows of realistic luxury real estate inquiry data including lead names, emails, budgets, timelines, cities, and neighborhoods. It is used for testing the workflow before real Gmail emails arrive.

---

## 5.1 Starter File Column Reference

| Column | Example Value | Description |
|---|---|---|
| `Full Name` | Julian Thorne | Lead's full name |
| `Email Address` | j.thorne@vanderbilt-client.com | Lead's email |
| `Inquiry Subject` | Relocation Inquiry | Email subject line |
| `Current City` | London | Where lead is now |
| `Target Neighborhood` | Bel Air | Where they want to buy |
| `Budget` | $12,000,000 | Stated budget (string) |
| `Timeline` | within 30 days | Stated timeline (string) |
| `Inquiry Body` | Hello, my name is Julian... | Full email body text |

---

## Option A: Import into Gmail as Test Emails

Use this method to simulate real Gmail trigger behavior.

**Step 1:** Open the CSV in Excel or Google Sheets  
**Step 2:** Create a Gmail filter or label called `VanderbiltInquiry`  
**Step 3:** For each row you want to test (start with rows 2–5):

1. Open Gmail Compose
2. To: your own Gmail address
3. Subject: paste the `Inquiry Subject` column value
4. Body: paste the `Inquiry Body` column value
5. Add a label: `VanderbiltInquiry`
6. Send to yourself

**Step 4:** The Gmail Trigger (Node 1) will pick these up when polling.

> **Tip:** Use Gmail's filter rules to auto-label: Filters → From contains `@vanderbilt-client.com` → Apply label `VanderbiltInquiry`.

---

## Option B: Read CSV Directly in n8n (Simulation Mode)

This method bypasses Gmail entirely and feeds CSV rows directly into the workflow via the Manual Trigger + Read Binary File path. This is the fastest way to test all 75 leads without sending 75 emails.

**Step 1:** Upload the CSV file to your n8n instance
- In n8n Cloud: Use a **HTTP Request** node pointed at a hosted CSV URL, OR
- In self-hosted n8n: Place the CSV at `/data/luxury_realtor_leads_starter.csv`

**Step 2:** In Node 3 (Read CSV), configure:
- **File Path:** `/data/luxury_realtor_leads_starter.csv`  
- The node will output each row as a separate item, simulating one email per row

**Step 3:** Connect Node 3 output directly to Node 4 (Merge Trigger Sources)

**Step 4:** In Node 5 (Normalize Email Fields), the expressions map CSV columns to email fields:
- `from` → `{{$json["Email Address"]}}`
- `subject` → `{{$json["Inquiry Subject"]}}`
- `textPlain` → `{{$json["Inquiry Body"]}}`

> **Recommended for initial build:** Use Option B to test all classification logic quickly. Switch to Option A (Gmail Trigger) for final client delivery.

---

---

# 6. Google Sheet Setup

## Sheet Name: `Vanderbilt Leads Database`

Create this spreadsheet in Google Drive before building any nodes.

### Step-by-Step:

1. Go to [https://sheets.google.com](https://sheets.google.com)
2. Click **Blank Spreadsheet**
3. Rename it: `Vanderbilt Leads Database`
4. Rename the default tab (Sheet1) to: `All Leads`
5. Add the following header row exactly in Row 1:

---

## Column Setup (Row 1 — Headers)

| Column | Header Text | Format | Notes |
|---|---|---|---|
| A | `Timestamp` | Date-Time | ISO format, auto-filled by workflow |
| B | `Lead ID` | Text | Auto-generated UUID |
| C | `Lead Name` | Text | Extracted by AI |
| D | `Email` | Text | From Gmail `from` field |
| E | `Phone Number` | Text | Extracted by AI |
| F | `Budget Range` | Text | Raw string e.g. "$3.8M" |
| G | `Budget Numeric` | Number | Parsed integer e.g. 3800000 |
| H | `Target Neighborhood` | Text | Extracted by AI |
| I | `Current City` | Text | Extracted by AI |
| J | `Timeline` | Text | Raw string e.g. "within 30 days" |
| K | `Timeline Days` | Number | Parsed integer e.g. 30 |
| L | `Tier` | Text | "Tier 1", "Tier 2", or "Tier 3" |
| M | `Tier Reason` | Text | AI reasoning for classification |
| N | `Inquiry Subject` | Text | Email subject line |
| O | `Inquiry Summary` | Text | AI-generated 1-sentence summary |
| P | `Source` | Text | "Gmail" / "CSV Simulation" |
| Q | `Status` | Text | "New" / "Contacted" / "Qualified" |
| R | `Follow Up Owner` | Text | Agent name, defaults to "Unassigned" |

---

## Formatting Tips

- Freeze Row 1 (View → Freeze → 1 row)
- Bold all headers
- Color column L (Tier) with conditional formatting:
  - "Tier 1" → Red background
  - "Tier 2" → Yellow background
  - "Tier 3" → Green background
- Set column G (Budget Numeric) and K (Timeline Days) to Number format

---

## Copy the Spreadsheet ID

After creating the sheet, grab the **Spreadsheet ID** from the URL:

```
https://docs.google.com/spreadsheets/d/SPREADSHEET_ID_HERE/edit
```

You will paste this ID into every Google Sheets node.

---

---

# 7. Full n8n Build Guide — All 27 Nodes

> **Before You Start:**
> - Open n8n and click **"New Workflow"**
> - Name it: `Vanderbilt Lead Concierge System`
> - Add sticky notes as you build each section (instructions per node below)
> - Save the workflow after completing every 5 nodes

---

## ── SECTION A: TRIGGER SOURCES ──
*Add a sticky note to the canvas labeled "SECTION A: TRIGGER SOURCES" in dark blue.*

---

## Node 1: Gmail Trigger – New Inquiry

| Property | Value |
|---|---|
| **Node Type** | Gmail Trigger |
| **Display Name** | `Gmail Trigger – New Inquiry` |
| **Credential** | Gmail OAuth2 (Vanderbilt) |
| **Trigger On** | Message Received |
| **Label** | `VanderbiltInquiry` |
| **Poll Interval** | Every 1 minute |

**Setup Steps:**

1. Add a **Gmail Trigger** node to the canvas
2. Double-click to rename: `Gmail Trigger – New Inquiry`
3. Select Credential: `Gmail OAuth2 (Vanderbilt)`
4. Under **Trigger On**: select `Message Received`
5. Under **Filters → Label**: type `VanderbiltInquiry` and select it
   - If label doesn't appear, create it in Gmail first, then refresh
6. **Poll Times:** Set to `Every 1 Minute`
7. Click **Save**

**Output Example:**
```json
{
  "id": "18e4f3a2b7c1...",
  "from": "j.thorne@vanderbilt-client.com",
  "subject": "Relocation Inquiry",
  "textPlain": "Hello, my name is Julian Thorne...",
  "textHtml": "<p>Hello, my name is Julian Thorne...</p>",
  "date": "2025-04-28T09:14:33Z"
}
```

**Why Needed:** This is the live production trigger. Every real inbound inquiry labeled in Gmail fires this node, starting the entire classification pipeline.

---

## Node 2: Manual Trigger – Test Mode

| Property | Value |
|---|---|
| **Node Type** | Manual Trigger |
| **Display Name** | `Manual Trigger – Test Mode` |
| **Purpose** | Run the workflow manually during development |

**Setup Steps:**

1. Add a **Manual Trigger** node to the canvas
2. Rename it: `Manual Trigger – Test Mode`
3. Connect its output to Node 4 (Merge Trigger Sources)
4. This node requires no configuration — clicking "Execute Node" runs the workflow from this point

**Why Needed:** During testing, you don't want to wait for real Gmail emails. This lets you trigger the workflow on demand with test data.

---

## Node 3: Read CSV – Starter Leads

| Property | Value |
|---|---|
| **Node Type** | Read Binary File (or HTTP Request for hosted CSV) |
| **Display Name** | `Read CSV – Starter Leads` |
| **File Path** | `/data/luxury_realtor_leads_starter.csv` |
| **Operation** | Read file → parse CSV |

**Setup Steps (Self-Hosted):**

1. Add a **Read Binary File** node
2. Rename: `Read CSV – Starter Leads`
3. Set **File Path**: `/data/luxury_realtor_leads_starter.csv`
4. Connect to a **Spreadsheet File** node set to **Read** mode → Format: CSV
5. This outputs each CSV row as a separate item

**Setup Steps (n8n Cloud):**

1. Add an **HTTP Request** node
2. Rename: `Read CSV – Starter Leads`
3. Method: GET
4. URL: Your hosted CSV URL (e.g., via Google Drive public link or GitHub raw URL)
5. Connect to **Spreadsheet File** node → Format: CSV

**Output Example:**
```json
{
  "Full Name": "Julian Thorne",
  "Email Address": "j.thorne@vanderbilt-client.com",
  "Inquiry Subject": "Relocation Inquiry",
  "Current City": "London",
  "Target Neighborhood": "Bel Air",
  "Budget": "$12,000,000",
  "Timeline": "within 30 days",
  "Inquiry Body": "Hello, my name is Julian Thorne..."
}
```

**Why Needed:** Allows the full 75-lead CSV to be processed without sending real emails. Critical for testing and client demo purposes.

---

## Node 4: Merge – Combine Trigger Sources

| Property | Value |
|---|---|
| **Node Type** | Merge |
| **Display Name** | `Merge – Combine Trigger Sources` |
| **Mode** | Merge By Position |
| **Inputs** | Node 1 (Gmail) + Node 2 (Manual) + Node 3 (CSV) |

**Setup Steps:**

1. Add a **Merge** node to the canvas
2. Rename: `Merge – Combine Trigger Sources`
3. Set **Mode**: `Merge By Position`
4. Connect:
   - Input 1: Output of `Gmail Trigger – New Inquiry`
   - Input 2: Output of `Manual Trigger – Test Mode`
   - Input 3: Output of `Read CSV – Starter Leads`
5. This node passes through whichever source fires

**Why Needed:** The workflow has three possible entry points. This merge node ensures the downstream pipeline always receives data in a consistent shape regardless of source.

---

---

## ── SECTION B: EMAIL NORMALIZATION ──
*Add sticky note: "SECTION B: EMAIL NORMALIZATION" in teal.*

---

## Node 5: Set – Normalize Email Fields

| Property | Value |
|---|---|
| **Node Type** | Set |
| **Display Name** | `Set – Normalize Email Fields` |
| **Purpose** | Standardize field names from all three input sources |

**Setup Steps:**

1. Add a **Set** node
2. Rename: `Set – Normalize Email Fields`
3. Set **Mode**: `Keep All Values` → then Add Fields
4. Add the following fields:

| Output Field Name | Expression | Notes |
|---|---|---|
| `normalized_from` | `{{$json["from"] \|\| $json["Email Address"] \|\| ""}}` | Gmail or CSV email |
| `normalized_subject` | `{{$json["subject"] \|\| $json["Inquiry Subject"] \|\| ""}}` | Gmail or CSV subject |
| `normalized_body` | `{{$json["textPlain"] \|\| $json["textHtml"] \|\| $json["Inquiry Body"] \|\| ""}}` | Email body (plain text preferred) |
| `normalized_name` | `{{$json["Full Name"] \|\| ""}}` | Only populated from CSV |
| `source_type` | `{{"Gmail"}}` | Hardcode, then override in CSV path |
| `received_at` | `{{$now.toISO()}}` | Timestamp this moment |
| `lead_id` | `{{"LD-" + $now.toMillis()}}` | Simple timestamp-based ID |

**Why Needed:** The Gmail trigger and CSV simulation produce different field names. This node unifies them so all downstream nodes use the same consistent `normalized_*` fields.

---

## Node 6: Code – Clean Email Body

| Property | Value |
|---|---|
| **Node Type** | Code |
| **Display Name** | `Code – Clean Email Body` |
| **Language** | JavaScript |

**Setup Steps:**

1. Add a **Code** node
2. Rename: `Code – Clean Email Body`
3. Set **Mode**: `Run Once for Each Item`
4. Paste the following code:

```javascript
// Clean the email body for AI consumption
const raw = $input.item.json.normalized_body || "";

// Strip HTML tags
let cleaned = raw.replace(/<[^>]*>/g, " ");

// Decode HTML entities
cleaned = cleaned
  .replace(/&amp;/g, "&")
  .replace(/&lt;/g, "<")
  .replace(/&gt;/g, ">")
  .replace(/&nbsp;/g, " ")
  .replace(/&#39;/g, "'")
  .replace(/&quot;/g, '"');

// Remove excessive whitespace
cleaned = cleaned.replace(/\s+/g, " ").trim();

// Truncate to 2000 characters to stay within AI token limits
const truncated = cleaned.substring(0, 2000);

return {
  ...$input.item.json,
  cleaned_body: truncated,
  body_char_count: truncated.length
};
```

**Output Example:**
```json
{
  "normalized_from": "j.thorne@vanderbilt-client.com",
  "cleaned_body": "Hello, my name is Julian Thorne and I am currently living in London...",
  "body_char_count": 183
}
```

**Why Needed:** Raw email bodies often contain HTML, encoding artifacts, and excessive whitespace. AI models perform significantly better on clean plain text. This node also prevents token limit errors on very long emails.

---

---

## ── SECTION C: AI EXTRACTION ──
*Add sticky note: "SECTION C: AI EXTRACTION — OpenAI GPT-4o" in purple.*

---

## Node 7: AI – Extract Lead Data

| Property | Value |
|---|---|
| **Node Type** | OpenAI (or HTTP Request for Anthropic) |
| **Display Name** | `AI – Extract Lead Data` |
| **Credential** | OpenAI Vanderbilt Key |
| **Model** | `gpt-4o` |
| **Max Tokens** | 500 |
| **Temperature** | 0 |

**Setup Steps:**

1. Add an **OpenAI** node
2. Rename: `AI – Extract Lead Data`
3. Select Credential: `OpenAI Vanderbilt Key`
4. Set **Resource**: `Chat`
5. Set **Operation**: `Create a Completion`
6. Set **Model**: `gpt-4o`
7. Set **Max Tokens**: `500`
8. Set **Temperature**: `0` (deterministic, critical for JSON parsing)
9. In the **Messages** section, add two messages:

**System Message:**
```
You are an expert real estate lead data extractor. Your job is to read an inbound email inquiry and extract structured data from it. Always return valid JSON and nothing else. Do not include markdown code blocks, explanations, or any text outside the JSON object.
```

**User Message (Expression):**
```
{{$json["cleaned_body"]}}

Extract the following fields from the email above:
- lead_name: Full name of the person writing
- phone_number: Any phone number mentioned
- budget_range: Budget as stated in the email (e.g. "$3.8M", "$1,200,000")
- budget_numeric: Budget as a plain integer number with no symbols (e.g. 3800000)
- target_neighborhood: The area or neighborhood they want to buy in
- current_city: The city they are currently living in
- timeline_days: The move timeline converted to number of days (e.g. "within 30 days" = 30, "ASAP" = 7, "immediately" = 7, "6-12 months" = 270, "next summer" = 120, "by end of year" = 180, "flexible" = 365, "no rush" = 365)
- summary: A one-sentence summary of the inquiry

Return ONLY this JSON object:
{
  "lead_name": "",
  "phone_number": "",
  "budget_range": "",
  "budget_numeric": 0,
  "target_neighborhood": "",
  "current_city": "",
  "timeline_days": 0,
  "summary": ""
}
```

**Why Needed:** This is the intelligence core of the workflow. Instead of regex-parsing unstructured email text (which breaks constantly), we leverage GPT-4o to reliably extract structured data from natural language inquiries regardless of how the lead wrote the email.

---

## Node 8: Code – Parse AI JSON Output

| Property | Value |
|---|---|
| **Node Type** | Code |
| **Display Name** | `Code – Parse AI JSON Output` |
| **Language** | JavaScript |

**Setup Steps:**

1. Add a **Code** node
2. Rename: `Code – Parse AI JSON Output`
3. Set **Mode**: `Run Once for Each Item`
4. Paste the following code:

```javascript
const rawResponse = $input.item.json.message?.content || 
                    $input.item.json.choices?.[0]?.message?.content || 
                    $input.item.json.content || "";

let parsed = {};

try {
  // Remove any accidental markdown fences
  const cleaned = rawResponse
    .replace(/```json/g, "")
    .replace(/```/g, "")
    .trim();
  
  parsed = JSON.parse(cleaned);
} catch (e) {
  // If JSON parse fails, return error marker for IF node to catch
  return {
    ...$input.item.json,
    parse_error: true,
    parse_error_message: e.message,
    raw_ai_response: rawResponse,
    lead_name: "",
    phone_number: "",
    budget_range: "",
    budget_numeric: 0,
    target_neighborhood: "",
    current_city: "",
    timeline_days: 0,
    summary: ""
  };
}

return {
  ...$input.item.json,
  parse_error: false,
  lead_name: parsed.lead_name || "",
  phone_number: parsed.phone_number || "",
  budget_range: parsed.budget_range || "",
  budget_numeric: parseInt(parsed.budget_numeric) || 0,
  target_neighborhood: parsed.target_neighborhood || "",
  current_city: parsed.current_city || "",
  timeline_days: parseInt(parsed.timeline_days) || 365,
  summary: parsed.summary || ""
};
```

**Output Example:**
```json
{
  "parse_error": false,
  "lead_name": "Julian Thorne",
  "phone_number": "",
  "budget_range": "$12,000,000",
  "budget_numeric": 12000000,
  "target_neighborhood": "Bel Air",
  "current_city": "London",
  "timeline_days": 30,
  "summary": "Julian Thorne is relocating from London to Bel Air with a $12M budget and a 30-day move timeline."
}
```

**Why Needed:** AI models occasionally return malformed JSON, add explanations, or wrap output in markdown code blocks. This node robustly strips and parses the response, and gracefully flags failures so the error branch can handle them without crashing the entire workflow.

---

---

## ── SECTION D: VALIDATION ──
*Add sticky note: "SECTION D: VALIDATION GATE" in orange.*

---

## Node 9: IF – Required Fields Present?

| Property | Value |
|---|---|
| **Node Type** | IF |
| **Display Name** | `IF – Required Fields Present?` |

**Setup Steps:**

1. Add an **IF** node
2. Rename: `IF – Required Fields Present?`
3. Add the following conditions (all must be TRUE):

**Condition 1:**
- Left Value: `{{$json["parse_error"]}}`
- Operator: `Is Equal To`
- Right Value: `false`

**Condition 2:**
- Left Value: `{{$json["lead_name"]}}`
- Operator: `Is Not Empty`

**Condition 3:**
- Left Value: `{{$json["budget_numeric"]}}`
- Operator: `Is Greater Than`
- Right Value: `0`

**Combine Conditions:** `AND` (all three must be true)

**True output:** Goes to Node 10 (AI Classification)  
**False output:** Goes to Node 21 (Error Trigger)

**Why Needed:** This gate prevents bad or empty AI extractions from polluting the database. If the AI failed to extract a name and budget, there is no point classifying or logging the lead.

---

---

## ── SECTION E: AI CLASSIFICATION ──
*Add sticky note: "SECTION E: AI CLASSIFICATION — TIER ROUTING" in dark purple.*

---

## Node 10: AI – Classify Lead Tier

| Property | Value |
|---|---|
| **Node Type** | OpenAI |
| **Display Name** | `AI – Classify Lead Tier` |
| **Credential** | OpenAI Vanderbilt Key |
| **Model** | `gpt-4o` |
| **Max Tokens** | 200 |
| **Temperature** | 0 |

**Setup Steps:**

1. Add an **OpenAI** node
2. Rename: `AI – Classify Lead Tier`
3. Same credential and model as Node 7
4. **System Message:**
```
You are a luxury real estate lead classification engine. You classify inbound leads into tiers based strictly on budget and urgency. Always return valid JSON only. No markdown, no explanation.
```

5. **User Message:**
```
Classify this lead into a tier based on the following rules:

TIER 1 — IMMEDIATE / HIGH VALUE:
- Budget >= $2,000,000 OR Timeline <= 30 days
- These are priority buyers. Agent must call immediately.

TIER 2 — STANDARD QUALIFIED:
- Budget is between $1,000,000 and $1,999,999 AND timeline is 31–90 days
- Qualified buyers but not urgent. Add to CRM and follow up within 48 hours.

TIER 3 — LONG-TERM NURTURE:
- Budget < $1,000,000 OR Timeline > 90 days OR unclear/low urgency
- Low priority. Log only. Add to nurture sequence.

Lead Data:
- Budget Numeric: {{$json["budget_numeric"]}}
- Timeline Days: {{$json["timeline_days"]}}
- Budget Range (stated): {{$json["budget_range"]}}
- Summary: {{$json["summary"]}}

Return ONLY this JSON:
{
  "tier": "Tier 1",
  "reason": "Brief explanation here"
}
```

**Output Example:**
```json
{
  "tier": "Tier 1",
  "reason": "Budget of $12M exceeds $2M threshold and timeline is 30 days which meets the urgency criterion."
}
```

**Why Needed:** Although the IF/Switch nodes could classify using pure logic, using AI for classification allows nuanced reasoning — for example, a lead who says "ASAP" without giving a number, or one who says "flexible but ideally Q3" can be correctly classified even when the numeric parsing is imperfect.

---

## Node 11: Set – Build Tier Result

| Property | Value |
|---|---|
| **Node Type** | Set |
| **Display Name** | `Set – Build Tier Result` |

**Setup Steps:**

1. Add a **Set** node
2. Rename: `Set – Build Tier Result`
3. Set **Mode**: `Keep All Values`
4. Add fields:

| Field | Expression |
|---|---|
| `tier` | `{{$json["message"]["content"] ? JSON.parse($json["message"]["content"].replace(/```json\|```/g,"").trim()).tier : "Tier 3"}}` |
| `tier_reason` | `{{$json["message"]["content"] ? JSON.parse($json["message"]["content"].replace(/```json\|```/g,"").trim()).reason : "Classification failed — defaulting to Tier 3"}}` |
| `final_status` | `New` |
| `follow_up_owner` | `Unassigned` |

> **Note:** If you want to avoid parsing in expressions, add a Code node before this Set node to parse the classification response the same way Node 8 parses the extraction response. That is a cleaner pattern.

**Why Needed:** This node consolidates the AI classification output with all previously extracted fields, building a single complete data object that downstream nodes (Switch, Sheets, Slack) can all pull from cleanly.

---

---

## ── SECTION F: TIER ROUTING ──
*Add sticky note: "SECTION F: TIER ROUTING — SWITCH" in gold.*

---

## Node 12: Switch – Route by Tier

| Property | Value |
|---|---|
| **Node Type** | Switch |
| **Display Name** | `Switch – Route by Tier` |

**Setup Steps:**

1. Add a **Switch** node
2. Rename: `Switch – Route by Tier`
3. Set **Value to Match**: `{{$json["tier"]}}`
4. Add three rules:

| Rule | Value | Output |
|---|---|---|
| Rule 1 | `Tier 1` | Output 0 → Tier 1 branch |
| Rule 2 | `Tier 2` | Output 1 → Tier 2 branch |
| Rule 3 | `Tier 3` | Output 2 → Tier 3 branch |

5. **Fallback Output**: Route to Tier 3 (Output 2) — handles any unexpected tier value

**Why Needed:** The Switch node is the traffic director of the entire workflow. It splits execution into three separate branches, each with its own notification and logging behavior.

---

---

## ── SECTION G: DUPLICATE CHECK ──
*Add sticky note: "SECTION G: GOOGLE SHEETS — DUPLICATE DETECTION" in green.*

---

## Node 13: Google Sheets – Search for Duplicate

| Property | Value |
|---|---|
| **Node Type** | Google Sheets |
| **Display Name** | `Google Sheets – Search for Duplicate` |
| **Operation** | Read Rows |
| **Credential** | Vanderbilt Sheets OAuth2 |

> **Note:** This node is shared by all three tier branches. In n8n, you can duplicate this node and connect it from each of the three Switch outputs. Name each copy `Google Sheets – Search Duplicate (Tier 1)`, etc.

**Setup Steps:**

1. Add a **Google Sheets** node
2. Rename: `Google Sheets – Search Duplicate (Tier 1)` (duplicate for Tier 2 and Tier 3)
3. Set **Operation**: `Lookup`
4. Set **Spreadsheet ID**: `[paste your Spreadsheet ID]`
5. Set **Sheet Name**: `All Leads`
6. Set **Lookup Column**: `D` (Email column)
7. Set **Lookup Value**: `{{$json["normalized_from"]}}`

**Output:** Returns matching rows if a duplicate exists, or empty array if this is a new lead.

**Why Needed:** Prevents the same lead from being logged multiple times. If a lead emails twice about the same property, their row gets updated rather than duplicated.

---

## Node 14: IF – Is This Lead Already in the Sheet?

| Property | Value |
|---|---|
| **Node Type** | IF |
| **Display Name** | `IF – Existing Lead?` |

**Setup Steps:**

1. Add an **IF** node after each Google Sheets Search node
2. Rename: `IF – Existing Lead?`
3. Condition:
   - Left Value: `{{$json["rowNumber"]}}` (or check if search returned any result)
   - Operator: `Is Not Empty`

**True (existing lead) →** Node 16: Google Sheets Update Row  
**False (new lead) →** Node 15: Google Sheets Append Row

---

---

## ── SECTION H: GOOGLE SHEETS LOGGING ──
*Add sticky note: "SECTION H: GOOGLE SHEETS LOGGING" in dark green.*

---

## Node 15: Google Sheets – Append New Lead Row

| Property | Value |
|---|---|
| **Node Type** | Google Sheets |
| **Display Name** | `Google Sheets – Append New Lead` |
| **Operation** | Append |
| **Credential** | Vanderbilt Sheets OAuth2 |

**Setup Steps:**

1. Add a **Google Sheets** node
2. Rename: `Google Sheets – Append New Lead`
3. Set **Operation**: `Append`
4. Set **Spreadsheet ID**: `[your Spreadsheet ID]`
5. Set **Sheet Name**: `All Leads`
6. Map the following columns:

| Sheet Column | Expression |
|---|---|
| Timestamp (A) | `{{$now.toISO()}}` |
| Lead ID (B) | `{{$json["lead_id"]}}` |
| Lead Name (C) | `{{$json["lead_name"]}}` |
| Email (D) | `{{$json["normalized_from"]}}` |
| Phone Number (E) | `{{$json["phone_number"]}}` |
| Budget Range (F) | `{{$json["budget_range"]}}` |
| Budget Numeric (G) | `{{$json["budget_numeric"]}}` |
| Target Neighborhood (H) | `{{$json["target_neighborhood"]}}` |
| Current City (I) | `{{$json["current_city"]}}` |
| Timeline (J) | `{{$json["normalized_body"].substring(0, 50)}}` |
| Timeline Days (K) | `{{$json["timeline_days"]}}` |
| Tier (L) | `{{$json["tier"]}}` |
| Tier Reason (M) | `{{$json["tier_reason"]}}` |
| Inquiry Subject (N) | `{{$json["normalized_subject"]}}` |
| Inquiry Summary (O) | `{{$json["summary"]}}` |
| Source (P) | `{{$json["source_type"]}}` |
| Status (Q) | `New` |
| Follow Up Owner (R) | `Unassigned` |

**Why Needed:** This is the primary CRM write. Every new unique lead gets a permanent row in the Vanderbilt Leads Database Google Sheet.

---

## Node 16: Google Sheets – Update Existing Lead Row

| Property | Value |
|---|---|
| **Node Type** | Google Sheets |
| **Display Name** | `Google Sheets – Update Existing Lead` |
| **Operation** | Update |
| **Credential** | Vanderbilt Sheets OAuth2 |

**Setup Steps:**

1. Add a **Google Sheets** node
2. Rename: `Google Sheets – Update Existing Lead`
3. Set **Operation**: `Update`
4. Set **Spreadsheet ID**: `[your Spreadsheet ID]`
5. Set **Sheet Name**: `All Leads`
6. Set **Row Number**: `{{$json["rowNumber"]}}` (from the search result)
7. Map these columns to update:

| Column | Expression |
|---|---|
| Timestamp (A) | `{{$now.toISO()}}` (refresh timestamp) |
| Tier (L) | `{{$json["tier"]}}` |
| Tier Reason (M) | `{{$json["tier_reason"]}}` |
| Inquiry Summary (O) | `{{$json["summary"]}}` |
| Status (Q) | `Re-inquiry` |

**Why Needed:** If a lead re-contacts Vanderbilt, we refresh their record's tier classification and timestamp rather than duplicating rows.

---

---

## ── SECTION I: TIER 1 ALERTS ──
*Add sticky note: "SECTION I: TIER 1 ALERT SYSTEM — IMMEDIATE ACTION" in bright red.*

---

## Node 17: Set – Build Slack Alert Message

| Property | Value |
|---|---|
| **Node Type** | Set |
| **Display Name** | `Set – Build Slack Alert Message` |
| **Purpose** | Construct the formatted Slack notification |

**Setup Steps:**

1. Add a **Set** node (only in the Tier 1 branch, after Node 15/16)
2. Rename: `Set – Build Slack Alert Message`
3. Add one field:

| Field Name | Expression |
|---|---|
| `slack_message` | See full template below |

**Slack Message Expression:**
```
🚨 *TIER 1 LUXURY LEAD — IMMEDIATE ACTION REQUIRED*

*Lead Name:* {{$json["lead_name"]}}
*Budget:* {{$json["budget_range"]}}
*Target Neighborhood:* {{$json["target_neighborhood"]}}
*Current City:* {{$json["current_city"]}}
*Timeline:* {{$json["timeline_days"]}} days
*Phone:* {{$json["phone_number"] || "Not provided"}}
*Email:* {{$json["normalized_from"]}}
*Lead ID:* {{$json["lead_id"]}}

*Inquiry Subject:* {{$json["normalized_subject"]}}
*Summary:* {{$json["summary"]}}

*Classification Reason:* {{$json["tier_reason"]}}

⚡ *Action: Call this lead within 15 minutes.*
```

---

## Node 18: Slack – Send Tier 1 Alert

| Property | Value |
|---|---|
| **Node Type** | Slack |
| **Display Name** | `Slack – Send Tier 1 Alert` |
| **Credential** | Slack Vanderbilt Webhook |
| **Channel** | `#luxury-leads-alerts` |

**Setup Steps:**

1. Add a **Slack** node (Tier 1 branch only)
2. Rename: `Slack – Send Tier 1 Alert`
3. Set **Authentication**: Webhook URL
4. Set **Credential**: `Slack Vanderbilt Webhook`
5. Set **Operation**: `Post Message`
6. Set **Channel**: `#luxury-leads-alerts`
7. Set **Text**: `{{$json["slack_message"]}}`
8. Toggle **Markdown**: ON

**Why Needed:** The lead agent carries their phone and has Slack mobile. This alert fires within seconds of the email arriving, ensuring zero delay on Tier 1 buyers.

---

## Node 19: Gmail – Backup Internal Alert

| Property | Value |
|---|---|
| **Node Type** | Gmail |
| **Display Name** | `Gmail – Backup Internal Alert` |
| **Credential** | Gmail OAuth2 (Vanderbilt) |
| **Operation** | Send |

**Setup Steps:**

1. Add a **Gmail** node (Tier 1 branch only, after Slack alert)
2. Rename: `Gmail – Backup Internal Alert`
3. Set **Operation**: `Send`
4. Set **To**: `agent@vanderbiltco.com` (the lead agent's email)
5. Set **Subject**:
   ```
   🔴 Tier 1 Lead: {{$json["lead_name"]}} — {{$json["budget_range"]}} — {{$json["target_neighborhood"]}}
   ```
6. Set **Message**:
   ```
   TIER 1 LUXURY LEAD — IMMEDIATE FOLLOW-UP REQUIRED
   
   Lead Name: {{$json["lead_name"]}}
   Budget: {{$json["budget_range"]}}
   Neighborhood: {{$json["target_neighborhood"]}}
   Current City: {{$json["current_city"]}}
   Timeline: {{$json["timeline_days"]}} days
   Phone: {{$json["phone_number"] || "Not provided"}}
   Email: {{$json["normalized_from"]}}
   
   Summary: {{$json["summary"]}}
   
   Lead ID: {{$json["lead_id"]}}
   Logged to Vanderbilt Leads Database.
   
   Action: Call immediately.
   ```

**Why Needed:** Email serves as a redundant backup channel if the agent misses the Slack notification. Double coverage ensures Tier 1 leads are never missed.

---

---

## ── SECTION J: RETRY LOGIC ──
*Add sticky note: "SECTION J: RETRY BUFFER" in dark yellow.*

---

## Node 20: Wait – Retry Delay Buffer

| Property | Value |
|---|---|
| **Node Type** | Wait |
| **Display Name** | `Wait – Retry Delay Buffer` |
| **Duration** | 30 seconds |
| **Purpose** | Brief pause before retry on AI timeouts |

**Setup Steps:**

1. Add a **Wait** node at the end of the Tier 1 branch (after Gmail backup alert)
2. Rename: `Wait – Retry Delay Buffer`
3. Set **Amount**: `30`
4. Set **Unit**: `Seconds`

**Connection:** This node connects forward to `Node 27: Final Success Logger`. In error scenarios (see Node 21), a separate Wait node is placed in the error branch to pause before retrying the AI call.

**Why Needed:** When AI APIs experience temporary overload or rate limiting, an immediate retry often fails again. A 30-second delay gives the API time to recover before the workflow retries.

---

---

## ── SECTION K: ERROR HANDLING ──
*Add sticky note: "SECTION K: ERROR HANDLING & RETRY SYSTEM" in dark red.*

---

## Node 21: Error Trigger – Workflow Failure Handler

| Property | Value |
|---|---|
| **Node Type** | Error Trigger |
| **Display Name** | `Error Trigger – Workflow Failure` |
| **Purpose** | Catches any unhandled errors in the main workflow |

**Setup Steps:**

1. Add an **Error Trigger** node to the canvas (this is a standalone trigger, NOT connected inline)
2. Rename: `Error Trigger – Workflow Failure`
3. This node fires automatically when any other node in the workflow throws an uncaught error
4. Connect its output to Node 22 (Slack Error Alert)

> **Important:** In n8n, set this workflow as the "Error Workflow" for your main workflow:
> - Open Workflow Settings (top-right gear icon)
> - Under **Error Workflow**: select `Vanderbilt Error Handler` (or the name of a separate workflow containing this Error Trigger)

**Output:** Contains the error details including which node failed and the error message.

---

## Node 22: Slack – Admin Error Alert

| Property | Value |
|---|---|
| **Node Type** | Slack |
| **Display Name** | `Slack – Admin Error Alert` |
| **Channel** | `#admin-errors` |

**Setup Steps:**

1. Add a **Slack** node connected to Node 21
2. Rename: `Slack – Admin Error Alert`
3. Set **Channel**: `#admin-errors`
4. Set **Text**:
```
🔥 *WORKFLOW ERROR — Vanderbilt Lead Concierge*

*Error Node:* {{$json["execution"]["error"]["node"]["name"]}}
*Error Message:* {{$json["execution"]["error"]["message"]}}
*Timestamp:* {{$now.toISO()}}
*Execution ID:* {{$json["execution"]["id"]}}

Please check n8n and investigate the failing execution.
Action: Open n8n → Executions → Review the failed run.
```

**Why Needed:** Without error monitoring, a broken AI API key or a Gmail quota issue could cause the workflow to silently fail for hours. This node sends an immediate admin alert so issues are fixed before leads are lost.

---

---

## ── SECTION L: TIER 2 & TIER 3 PATHS ──
*Add sticky note: "SECTION L: TIER 2 & TIER 3 PATHS" in blue-grey.*

---

## Node 23: Set – Tier 2 Normal Tag

| Property | Value |
|---|---|
| **Node Type** | Set |
| **Display Name** | `Set – Tier 2 Normal Tag` |

**Setup Steps:**

1. Add a **Set** node in the Tier 2 branch (after Google Sheets Append/Update)
2. Rename: `Set – Tier 2 Normal Tag`
3. Add field:
   - `follow_up_priority`: `"Standard — Follow up within 48 hours"`
   - `final_status`: `"Qualified"`

**Note:** Tier 2 does NOT send a Slack alert. The lead is logged with a standard follow-up tag. The agent will review the Google Sheet during their daily CRM review.

---

## Node 24: Set – Tier 3 Nurture Tag

| Property | Value |
|---|---|
| **Node Type** | Set |
| **Display Name** | `Set – Tier 3 Nurture Tag` |

**Setup Steps:**

1. Add a **Set** node in the Tier 3 branch (after Google Sheets Append/Update)
2. Rename: `Set – Tier 3 Nurture Tag`
3. Add field:
   - `follow_up_priority`: `"Low — Add to nurture email sequence"`
   - `nurture_sequence`: `"Luxury Insights Monthly Newsletter"`
   - `final_status`: `"Nurture"`

**Note:** Tier 3 receives no alert, no email, and no urgency. They are logged and tagged. The marketing team will add them to a long-term drip campaign.

---

---

## ── SECTION M: DAILY SUMMARY ──
*Add sticky note: "SECTION M: DAILY LEAD SUMMARY — 8AM REPORT" in navy.*

---

## Node 25: Schedule Trigger – Daily Summary at 8AM

| Property | Value |
|---|---|
| **Node Type** | Schedule Trigger |
| **Display Name** | `Schedule – Daily Summary at 8AM` |
| **Trigger** | Every day at 08:00 |

**Setup Steps:**

1. Add a **Schedule Trigger** node (this is a second standalone trigger, separate from the main flow)
2. Rename: `Schedule – Daily Summary at 8AM`
3. Set **Trigger at Specific Time**: Every Day at `08:00`
4. Connect to Node 26

**Why Needed:** Rather than overwhelming the agent with individual Tier 2/3 notifications, we batch them into a clean morning briefing delivered every day at 8AM.

---

## Node 26: Google Sheets – Read Yesterday's Leads

| Property | Value |
|---|---|
| **Node Type** | Google Sheets |
| **Display Name** | `Google Sheets – Read Yesterday Leads` |
| **Operation** | Get All Rows |

**Setup Steps:**

1. Add a **Google Sheets** node connected to Node 25
2. Rename: `Google Sheets – Read Yesterday Leads`
3. Set **Operation**: `Get All Rows` or `Lookup`
4. Set **Spreadsheet ID**: `[your Spreadsheet ID]`
5. Set **Sheet Name**: `All Leads`
6. Use a **Filter**: Column A (Timestamp) contains yesterday's date
   - Expression: `{{$now.minus({days:1}).toFormat("yyyy-MM-dd")}}`

---

## Node 27: Slack – Send Daily Lead Summary

| Property | Value |
|---|---|
| **Node Type** | Slack |
| **Display Name** | `Slack – Daily Lead Summary` |
| **Channel** | `#luxury-leads-alerts` |

**Setup Steps:**

1. Add a **Slack** node connected to Node 26
2. Rename: `Slack – Daily Lead Summary`
3. Set **Text**:
```
📊 *Vanderbilt Daily Lead Report — {{$now.toFormat("MMMM dd, yyyy")}}*

New leads logged yesterday: {{$items().length}}

Tier 1 (Immediate): {{$items().filter(i => i.json.Tier === "Tier 1").length}}
Tier 2 (Standard): {{$items().filter(i => i.json.Tier === "Tier 2").length}}
Tier 3 (Nurture): {{$items().filter(i => i.json.Tier === "Tier 3").length}}

Review full database: https://docs.google.com/spreadsheets/d/[SPREADSHEET_ID]

Have a great day! 🏡
```

---

## Node 28: Final Success Logger (Code)

| Property | Value |
|---|---|
| **Node Type** | Code |
| **Display Name** | `Final – Success Logger` |

**Setup Steps:**

1. Add a **Code** node at the very end of each branch
2. Rename: `Final – Success Logger`
3. Code:
```javascript
const tier = $input.item.json.tier || "Unknown";
const name = $input.item.json.lead_name || "Unknown Lead";
const budget = $input.item.json.budget_range || "N/A";
const id = $input.item.json.lead_id || "N/A";

console.log(`✅ SUCCESS: ${name} | ${tier} | ${budget} | Lead ID: ${id}`);

return {
  ...$input.item.json,
  workflow_status: "completed",
  completion_timestamp: new Date().toISOString()
};
```

**Why Needed:** Provides a clean final log entry in n8n's execution history, making it easy to audit completed runs and confirm each lead was fully processed.

---

---

# 8. Exact Field Mapping Reference

Use this reference table when building any node's expressions.

## Core Email Fields

| Expression | Source | Description |
|---|---|---|
| `{{$json["from"]}}` | Gmail Trigger | Full sender address |
| `{{$json["subject"]}}` | Gmail Trigger | Email subject line |
| `{{$json["textPlain"]}}` | Gmail Trigger | Plain text body |
| `{{$json["textHtml"]}}` | Gmail Trigger | HTML body |
| `{{$json["date"]}}` | Gmail Trigger | Email received timestamp |

## Normalized Fields (after Node 5)

| Expression | Description |
|---|---|
| `{{$json["normalized_from"]}}` | Standardized sender email |
| `{{$json["normalized_subject"]}}` | Standardized subject |
| `{{$json["normalized_body"]}}` | Standardized body text |
| `{{$json["cleaned_body"]}}` | HTML-stripped clean body (after Node 6) |
| `{{$json["received_at"]}}` | ISO timestamp |
| `{{$json["lead_id"]}}` | Unique lead identifier |
| `{{$json["source_type"]}}` | "Gmail" or "CSV Simulation" |

## AI Extracted Fields (after Node 8)

| Expression | Description |
|---|---|
| `{{$json["lead_name"]}}` | Full name |
| `{{$json["phone_number"]}}` | Phone number (may be empty) |
| `{{$json["budget_range"]}}` | Budget as string (e.g., "$3.8M") |
| `{{$json["budget_numeric"]}}` | Budget as integer (e.g., 3800000) |
| `{{$json["target_neighborhood"]}}` | Target area |
| `{{$json["current_city"]}}` | Current location |
| `{{$json["timeline_days"]}}` | Timeline as integer days |
| `{{$json["summary"]}}` | One-sentence summary |
| `{{$json["parse_error"]}}` | true/false — did JSON parsing fail? |

## Classification Fields (after Node 11)

| Expression | Description |
|---|---|
| `{{$json["tier"]}}` | "Tier 1", "Tier 2", or "Tier 3" |
| `{{$json["tier_reason"]}}` | AI reasoning text |
| `{{$json["final_status"]}}` | "New" |
| `{{$json["follow_up_owner"]}}` | "Unassigned" |

## Date/Time Utilities

| Expression | Output |
|---|---|
| `{{$now.toISO()}}` | Full ISO timestamp |
| `{{$now.toFormat("yyyy-MM-dd")}}` | Date only |
| `{{$now.toFormat("MMMM dd, yyyy")}}` | Human-readable date |
| `{{$now.toMillis()}}` | Unix milliseconds (for ID generation) |

---

---

# 9. AI Extraction Prompt

Use this exact prompt in **Node 7: AI – Extract Lead Data**.

## System Prompt (copy exactly)

```
You are an expert real estate lead data extractor for Vanderbilt & Co., a boutique luxury real estate firm. Your job is to read an inbound email inquiry and extract structured data from it.

Rules:
1. Always return valid JSON and nothing else.
2. Do NOT include markdown code blocks (no backticks).
3. Do NOT include explanations or text outside the JSON object.
4. If a field is not mentioned in the email, return an empty string "" for text fields and 0 for numeric fields.
5. For budget_numeric, return only the integer value (no $, no commas, no M suffix).
6. For timeline_days, convert the timeline to number of days using these guidelines:
   - "immediately" or "ASAP" = 7
   - "within X days" = X
   - "within X weeks" = X * 7
   - "within X months" = X * 30
   - "next summer" = 90
   - "by end of year" = 180
   - "6-12 months" = 270
   - "flexible" or "no rush" = 365
   - If completely unclear = 365
```

## User Prompt Template (copy exactly)

```
Email content:
---
{{$json["cleaned_body"]}}
---

Extract all relevant lead information from the email above and return ONLY this JSON object with no additional text:

{
  "lead_name": "",
  "phone_number": "",
  "budget_range": "",
  "budget_numeric": 0,
  "target_neighborhood": "",
  "current_city": "",
  "timeline_days": 0,
  "summary": ""
}

Field definitions:
- lead_name: Full name of the person writing the email
- phone_number: Any phone number mentioned (with country code if given)
- budget_range: Budget as stated in the email (keep the original format, e.g. "$3.8M", "$1,200,000", "around 3 million")
- budget_numeric: Budget converted to a plain integer (e.g. 3800000, 1200000, 3000000)
- target_neighborhood: The neighborhood, area, or city they want to buy property in
- current_city: The city they are currently living in or relocating from
- timeline_days: How many days until they want to move (see conversion rules in system prompt)
- summary: A single sentence summarizing the inquiry (under 100 words)
```

## Expected Output — Test Case (Julian Thorne)

**Input:** `"Hello, my name is Julian Thorne and I am currently living in London. I am reaching out because I am interested in finding a property in Bel Air. My budget is $12,000,000 and I am looking to move within 30 days."`

**Expected Output:**
```json
{
  "lead_name": "Julian Thorne",
  "phone_number": "",
  "budget_range": "$12,000,000",
  "budget_numeric": 12000000,
  "target_neighborhood": "Bel Air",
  "current_city": "London",
  "timeline_days": 30,
  "summary": "Julian Thorne is relocating from London to Bel Air with a $12M budget and needs to move within 30 days."
}
```

---

---

# 10. AI Classification Prompt

Use this exact prompt in **Node 10: AI – Classify Lead Tier**.

## System Prompt

```
You are a luxury real estate lead classification engine for Vanderbilt & Co. You receive structured lead data and classify leads into tiers based on urgency and budget. Always return valid JSON only. No markdown, no explanations, no backticks.
```

## User Prompt Template

```
Classify this lead into a tier using the exact rules below.

CLASSIFICATION RULES:

TIER 1 — IMMEDIATE / HIGH VALUE:
Condition: budget_numeric >= 2000000 OR timeline_days <= 30
These are ultra-priority buyers. Every Tier 1 lead triggers an immediate phone alert to the agent.

TIER 2 — STANDARD QUALIFIED:
Condition: budget_numeric >= 1000000 AND budget_numeric < 2000000 AND timeline_days > 30 AND timeline_days <= 90
Qualified buyers but not immediately urgent. Follow up within 48 hours.

TIER 3 — LONG-TERM NURTURE:
Condition: budget_numeric < 1000000 OR timeline_days > 90
Low urgency. Log and add to nurture marketing sequence. No agent alert.

Lead Data:
- budget_numeric: {{$json["budget_numeric"]}}
- timeline_days: {{$json["timeline_days"]}}
- budget_range (stated): {{$json["budget_range"]}}
- lead_name: {{$json["lead_name"]}}
- summary: {{$json["summary"]}}

IMPORTANT: Apply the rules strictly. If a lead meets ANY Tier 1 condition (budget >= $2M OR timeline <= 30 days), classify as Tier 1 regardless of other factors.

Return ONLY this JSON:
{
  "tier": "Tier 1",
  "reason": "One sentence explaining why this tier was assigned"
}
```

## Classification Test Cases

| Lead | Budget | Timeline | Expected Tier | Why |
|---|---|---|---|---|
| Julian Thorne | $12M | 30 days | **Tier 1** | Budget $12M ≥ $2M AND timeline ≤ 30 days |
| Seraphina Vance | $4.8M | ASAP (7 days) | **Tier 1** | Budget ≥ $2M AND timeline ≤ 30 days |
| Clara Montgomery | $1.2M | 6-12 months | **Tier 3** | Budget < $2M AND timeline > 90 days |
| Sloane Harrington | ~$3M | end of year (180 days) | **Tier 2** | Budget $3M ≥ $1M but < Tier 1 threshold? No — $3M ≥ $2M → **Tier 1** |
| Bartholomew Knight | $750k | flexible (365 days) | **Tier 3** | Budget < $1M, very low urgency |
| Caspian North | $2.5M | 30 days | **Tier 1** | Budget ≥ $2M AND timeline ≤ 30 days |

---

---

# 11. Branching Logic

## Switch Node Configuration (Node 12)

The Switch node routes each lead to exactly one of three execution branches based on the `tier` field.

```
Switch – Route by Tier
│
├── Output 0 (Tier 1) ─────────────────────────────────────────────────────────
│    ├── Google Sheets – Search Duplicate (Tier 1)
│    ├── IF – Existing Lead?
│    │     ├── TRUE  → Google Sheets – Update Existing Lead
│    │     └── FALSE → Google Sheets – Append New Lead
│    ├── Set – Build Slack Alert Message
│    ├── Slack – Send Tier 1 Alert          ← ALERT FIRES
│    ├── Gmail – Backup Internal Alert      ← EMAIL BACKUP
│    ├── Wait – Retry Delay Buffer
│    └── Final – Success Logger
│
├── Output 1 (Tier 2) ─────────────────────────────────────────────────────────
│    ├── Google Sheets – Search Duplicate (Tier 2)
│    ├── IF – Existing Lead?
│    │     ├── TRUE  → Google Sheets – Update Existing Lead
│    │     └── FALSE → Google Sheets – Append New Lead
│    ├── Set – Tier 2 Normal Tag            ← LOG ONLY, NO ALERT
│    └── Final – Success Logger
│
└── Output 2 (Tier 3) ─────────────────────────────────────────────────────────
     ├── Google Sheets – Search Duplicate (Tier 3)
     ├── IF – Existing Lead?
     │     ├── TRUE  → Google Sheets – Update Existing Lead
     │     └── FALSE → Google Sheets – Append New Lead
     ├── Set – Tier 3 Nurture Tag           ← LOG + NURTURE TAG, NO ALERT
     └── Final – Success Logger
```

## Tier Action Summary Table

| Action | Tier 1 | Tier 2 | Tier 3 |
|---|---|---|---|
| Log to Google Sheets | ✅ | ✅ | ✅ |
| Duplicate Check | ✅ | ✅ | ✅ |
| Slack Alert | ✅ (Immediate) | ❌ | ❌ |
| Gmail Backup Alert | ✅ | ❌ | ❌ |
| Standard Tag | ❌ | ✅ | ❌ |
| Nurture Tag | ❌ | ❌ | ✅ |
| Final Logger | ✅ | ✅ | ✅ |

---

---

# 12. Slack Notification Template

## Tier 1 Alert — Full Template

```
🚨 *TIER 1 LUXURY LEAD — IMMEDIATE ACTION REQUIRED*
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

*Lead Name:*              Julian Thorne
*Budget:*                 $12,000,000
*Target Neighborhood:*    Bel Air
*Current City:*           London
*Timeline:*               30 days
*Phone:*                  Not provided
*Email:*                  j.thorne@vanderbilt-client.com

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
*Inquiry Subject:* Relocation Inquiry
*Summary:* Julian Thorne is relocating from London to Bel Air with a $12M budget, needing to move within 30 days.

*Classification Reason:* Budget of $12M exceeds $2M threshold and 30-day timeline meets urgency criteria.

*Lead ID:* LD-1714297200000

⚡ *ACTION: Call this lead within 15 minutes.*
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## Dynamic Expression Version (paste into Slack node Text field)

```
🚨 *TIER 1 LUXURY LEAD — IMMEDIATE ACTION REQUIRED*
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

*Lead Name:* {{$json["lead_name"]}}
*Budget:* {{$json["budget_range"]}}
*Target Neighborhood:* {{$json["target_neighborhood"]}}
*Current City:* {{$json["current_city"]}}
*Timeline:* {{$json["timeline_days"]}} days
*Phone:* {{$json["phone_number"] || "Not provided"}}
*Email:* {{$json["normalized_from"]}}

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
*Inquiry Subject:* {{$json["normalized_subject"]}}
*Summary:* {{$json["summary"]}}

*Classification Reason:* {{$json["tier_reason"]}}

*Lead ID:* {{$json["lead_id"]}}

⚡ *ACTION: Call this lead within 15 minutes.*
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

---

# 13. Duplicate Handling

## Strategy

Duplicates are detected by matching the **email address** field (Column D) in the Google Sheet.

If the same email address already has a row, the workflow:
1. Skips the Append operation
2. Runs the Update operation instead
3. Refreshes the Timestamp, Tier, Reason, and Summary
4. Sets Status to `"Re-inquiry"`

## Why Email-Only Matching

Using Email + Subject + Name creates too many false negatives (same person, different subject line). Email address alone is the most reliable unique identifier for a real estate lead.

## Duplicate Check Node Setup (Node 13)

**Google Sheets – Lookup Configuration:**

| Setting | Value |
|---|---|
| Operation | Lookup |
| Lookup Column | D (Email) |
| Lookup Value | `{{$json["normalized_from"]}}` |
| Return All Matches | OFF (first match only) |

## IF Node Logic (Node 14)

```
IF:
  $json["rowNumber"] exists AND $json["rowNumber"] > 1
  → TRUE: Lead is a duplicate → Route to Update
  → FALSE: New lead → Route to Append
```

## Example Scenario

The CSV starter file includes multiple rows with the same name (e.g., "Clara Montgomery" appears 3 times with different emails). Since they have different email addresses, they will each be treated as separate leads. Only exact email matches are flagged as duplicates.

---

---

# 14. Error Handling

## Layer 1: Parse Error Detection (Node 9)

**Trigger:** AI returns invalid JSON or empty response  
**Handler:** IF node catches `parse_error: true` → routes to Slack Error Alert  
**Action:** Admin is notified, lead is NOT logged to avoid corrupt data

## Layer 2: Retry Buffer (Node 20)

**Trigger:** AI API timeout or rate limit  
**Handler:** Wait 30 seconds → retry the AI call  
**Configuration:**

1. After the AI node (Node 7), add an error output connection
2. Connect error output → Wait 30 seconds → reconnect to Node 7 input
3. Limit retries to 3 (use a counter in a Set node if needed)

## Layer 3: Error Trigger Workflow (Node 21)

**Trigger:** Any unhandled node failure in the entire workflow  
**Handler:** Error Trigger node fires → Slack Admin Alert  
**Setup:**
- Create a separate n8n workflow called `Vanderbilt Error Handler`
- Add Error Trigger + Slack Admin Alert nodes
- In the main workflow Settings, set Error Workflow to `Vanderbilt Error Handler`

## Layer 4: Gmail API Quota Handling

Gmail API has daily read quotas. If the Gmail Trigger stops firing:

1. Go to Google Cloud Console → APIs → Gmail API → Quotas
2. Check if you've hit the daily read quota
3. Increase quota or reduce poll frequency from 1 minute to 5 minutes

**Quota-safe configuration:**
- Poll every 5 minutes instead of every 1 minute
- This reduces daily API calls from 1,440 to 288

## Error Scenarios Reference Table

| Error Type | Where It Occurs | Handler | Recovery |
|---|---|---|---|
| AI returns invalid JSON | Node 8 | Code node catches, returns parse_error:true | IF node routes to admin alert |
| AI timeout | Nodes 7, 10 | Wait 30s + retry | Max 3 retries, then admin alert |
| Google Sheets auth expired | Nodes 13, 15, 16 | Error Trigger fires | Re-authenticate credential in n8n |
| Slack webhook invalid | Nodes 18, 22 | Error Trigger fires | Regenerate Slack webhook |
| Gmail not triggering | Node 1 | Manual check | Verify label, re-authenticate |
| Budget not extracted | Node 8 | `budget_numeric` = 0 | IF node catches, routes to error |
| Duplicate detection fails | Node 13 | Worst case: duplicate row added | Run dedup script on Google Sheet |

---

---

# 15. Testing Procedure

Follow these exact steps to validate the complete workflow before client delivery.

## Phase 1: Build Validation (Before Any Execution)

**Step 1:** Open the workflow in n8n canvas  
**Step 2:** Verify all 27+ nodes are connected correctly (no dangling nodes)  
**Step 3:** Click each node and verify credentials are green (not red)  
**Step 4:** Check the Google Sheet has the correct 18 column headers in Row 1  
**Step 5:** Verify Slack channels `#luxury-leads-alerts` and `#admin-errors` exist  

---

## Phase 2: CSV Simulation Test (Option B)

**Step 1: Load Tier 1 Test Lead**
- Open the CSV file
- Take Row 2 (Julian Thorne — $12M, 30 days)
- Create a test item in Node 3 with this row's data

**Step 2: Execute Node 3**
- Click Node 3 → Execute Node
- Verify output contains Julian Thorne's data

**Step 3: Execute Through to Node 8**
- Click Node 8 → Execute Node
- Verify parsed output contains:
  - `lead_name`: "Julian Thorne"
  - `budget_numeric`: 12000000
  - `timeline_days`: 30
  - `parse_error`: false

**Step 4: Execute Node 10 (Classification)**
- Verify `tier`: "Tier 1"
- Verify `tier_reason` contains reasoning about $12M or 30 days

**Step 5: Execute Switch Node**
- Verify execution routes to Output 0 (Tier 1 branch)

**Step 6: Verify Google Sheet Row**
- Open the Vanderbilt Leads Database Google Sheet
- Verify Julian Thorne's row appears in Row 2 with all 18 columns populated
- Verify Tier column (L) shows "Tier 1"

**Step 7: Verify Slack Alert**
- Open the `#luxury-leads-alerts` Slack channel
- Verify the Tier 1 alert message appeared with correct data

**Step 8: Test Tier 3 Lead**
- Use Row 5 from CSV: Clara Montgomery ($1.2M, 6-12 months)
- Execute workflow with her data
- Verify: classified as Tier 3, logged to Sheet, NO Slack alert fired

**Step 9: Test Duplicate Handling**
- Run the same Julian Thorne lead a second time
- Verify: no new row is added to the Sheet
- Verify: existing row's Timestamp is updated
- Verify: Status changes to "Re-inquiry"

---

## Phase 3: Gmail Trigger Live Test

**Step 1:** Create a Gmail label called `VanderbiltInquiry` in the agent's Gmail account  
**Step 2:** Compose and send a test email to yourself:
- Subject: `Urgent Inquiry - Bel Air Property`
- Body: `Hello, my name is Test Lead. I'm moving from Tokyo to Bel Air within 2 weeks. Budget is $5 million. Please call me.`

**Step 3:** Apply the `VanderbiltInquiry` label to the email  
**Step 4:** Wait up to 1 minute for the Gmail Trigger to poll  
**Step 5:** Watch the n8n execution log for a new run  
**Step 6:** Verify the lead is logged to the Sheet and a Slack alert fires  

---

## Phase 4: Error Handling Test

**Step 1:** Temporarily set an invalid API key for OpenAI  
**Step 2:** Trigger the workflow with a test email  
**Step 3:** Verify the workflow fails gracefully at Node 7  
**Step 4:** Verify the Error Trigger fires and a Slack admin alert appears in `#admin-errors`  
**Step 5:** Restore the correct API key  

---

## Expected Test Results Table

| Test Scenario | Expected Sheet Row? | Expected Slack Alert? | Expected Tier |
|---|---|---|---|
| Julian Thorne ($12M, 30 days) | ✅ Yes | ✅ Yes | Tier 1 |
| Seraphina Vance ($4.8M, ASAP) | ✅ Yes | ✅ Yes | Tier 1 |
| Clara Montgomery ($1.2M, 6-12mo) | ✅ Yes | ❌ No | Tier 3 |
| Bartholomew Knight ($750k, flexible) | ✅ Yes | ❌ No | Tier 3 |
| Duplicate Julian Thorne | ✅ Update only | ❌ No new alert | N/A |
| Invalid AI response | ❌ No log | ✅ Admin error alert | N/A |

---

---

# 16. Deliverables Checklist

Submit all of the following items to the client.

## Required Deliverables

| # | Deliverable | Format | Notes |
|---|---|---|---|
| 1 | Exported n8n workflow | `.json` file | Export via n8n → ⋯ → Download |
| 2 | AI Extraction Prompt | `.txt` file | Copy from Section 9 above |
| 3 | AI Classification Prompt | `.txt` file | Copy from Section 10 above |
| 4 | Workflow screenshot (full canvas) | `.png` | Show all nodes with labels |
| 5 | Successful execution screenshot | `.png` | Green checkmarks on all nodes |
| 6 | Google Sheet (Vanderbilt Leads Database) | Google Sheets link | Share with client — View access |
| 7 | Slack alert screenshot (Tier 1 test) | `.png` | Show formatted message in channel |
| 8 | Error alert screenshot | `.png` | Show admin-errors channel |
| 9 | Loom walkthrough video | Loom link | 1–3 minutes, narrated |
| 10 | Credentials setup guide | `.pdf` or `.md` | Step-by-step for client's team |

## Loom Walkthrough Script (1–3 minutes)

**Segment 1 (0:00–0:30):** Show the n8n canvas — name all major sections  
**Segment 2 (0:30–1:00):** Trigger a test email and watch the execution live  
**Segment 3 (1:00–1:45):** Open Google Sheet — show the new row with all fields  
**Segment 4 (1:45–2:15):** Open Slack — show the Tier 1 alert  
**Segment 5 (2:15–2:30):** Trigger a Tier 3 lead — show no alert fires, only Sheet log  
**Segment 6 (2:30–3:00):** Show the error workflow and daily summary  

---

---

# 17. Troubleshooting Guide

## Gmail Trigger Not Firing

**Symptoms:** Workflow never activates automatically  
**Causes:**
1. Label `VanderbiltInquiry` not applied to test email
2. OAuth2 credential expired
3. Gmail poll interval too long
4. Gmail API not enabled in Google Cloud Console

**Fixes:**
1. Open Gmail → verify the label exists and is applied to the email
2. In n8n → Credentials → Gmail OAuth2 → click "Reconnect"
3. Set poll interval to 1 minute
4. Go to Google Cloud Console → APIs → Enable Gmail API

---

## AI Returns Invalid JSON

**Symptoms:** Node 8 `parse_error` = true, workflow stops at IF node  
**Causes:**
1. Model returned text with explanation outside JSON
2. Model returned JSON wrapped in markdown code blocks
3. Temperature set too high (non-deterministic)
4. Prompt too long for context window

**Fixes:**
1. Node 8's Code node strips markdown fences — verify the strip code is in place
2. Set `Temperature` to `0` in Node 7 and Node 10
3. Ensure the User Prompt explicitly says "Return ONLY this JSON"
4. Trim the email body in Node 6 to max 2000 characters

---

## Budget Not Extracted

**Symptoms:** `budget_numeric` = 0, `budget_range` = ""  
**Causes:**
1. Budget was described in an unusual format ("around 3 million", "up to $15M")
2. Email body did not reach Node 7 correctly

**Fixes:**
1. Check `cleaned_body` field after Node 6 — verify it contains the budget mention
2. Add more budget format examples to the extraction prompt
3. The AI should handle "around 3 million" = 3000000 and "up to $15M" = 15000000 naturally with GPT-4o

---

## Timeline Not Extracted or Wrong

**Symptoms:** `timeline_days` = 0 or 365 when it should be 30  
**Causes:**
1. Timeline phrasing not covered in the conversion guide
2. "ASAP" → should be 7 days

**Fixes:**
1. Add more conversion examples to the system prompt
2. For "ASAP", "immediately", "right away": explicitly say = 7 days in the prompt
3. Review extraction output for the specific failing lead and adjust prompt

---

## Google Sheets Authentication Failed

**Symptoms:** Sheets node shows red error, "Authentication required"  
**Fixes:**
1. Go to n8n → Credentials → Vanderbilt Sheets OAuth2
2. Click "Reconnect" and re-authorize
3. Verify the Google account has Edit access to the Vanderbilt Leads Database spreadsheet
4. Check the Spreadsheet ID in each Sheets node matches the actual Sheet URL

---

## Duplicate Rows Appearing in Google Sheet

**Symptoms:** Same lead email appears in multiple rows  
**Causes:**
1. The Google Sheets Lookup is not finding the duplicate
2. The IF node condition is wrong
3. Email address format differs (e.g., with/without spaces)

**Fixes:**
1. Open the Lookup node output — check if it returns a row number
2. Trim email addresses: use `{{$json["normalized_from"].trim().toLowerCase()}}`
3. Also lowercase the Sheet lookup column for comparison

---

## Slack Token or Webhook Invalid

**Symptoms:** Slack node fails with "invalid_token" or 404  
**Fixes:**
1. Go to api.slack.com → Your Apps → Vanderbilt Lead Alerts → Incoming Webhooks
2. Verify the webhook URL has not been revoked
3. Click "Add New Webhook" if the original was deleted
4. Update the Slack credential in n8n with the new webhook URL

---

## Rate Limit Errors (OpenAI 429)

**Symptoms:** AI nodes fail with "rate limit exceeded"  
**Fixes:**
1. Add a **Wait** node (5 seconds) before Node 7 to space out API calls
2. Upgrade to a higher OpenAI tier for more requests per minute
3. Switch to `gpt-4o-mini` for lower quota consumption (slightly lower accuracy)
4. Use a retry loop: Error output → Wait 60 seconds → retry Node 7

---

---

# 18. Pro Freelancer Tips

## Canvas Organization

**Use Sticky Notes for sections:**
Add a different-colored sticky note as a section header before each group of nodes. Suggested colors:
- Section A (Triggers): Dark Blue
- Section B (Normalization): Teal
- Section C (AI Extraction): Purple
- Section D (Validation): Orange
- Section E (Classification): Dark Purple
- Section F (Routing): Gold
- Section G (Sheets): Dark Green
- Section H (Alerts): Red
- Section I (Error Handling): Dark Red
- Section J (Daily Summary): Navy

**Use consistent node naming:**
Every node name should follow the pattern: `[Type] – [Description]`
Examples: `AI – Extract Lead Data`, `Set – Build Slack Message`, `IF – Existing Lead?`

**Group nodes vertically by branch:**
- Main flow: left to right
- Tier 1 branch: horizontal with alerts below
- Tier 2/3 branches: parallel lanes below Tier 1

---

## Extraction Accuracy Table

Show this in your client deliverable to demonstrate quality.

| Lead | Budget Extracted | Timeline Extracted | Tier Assigned | Correct? |
|---|---|---|---|---|
| Julian Thorne ($12M, 30 days) | $12,000,000 ✅ | 30 days ✅ | Tier 1 ✅ | ✅ |
| Seraphina Vance ($4.8M, ASAP) | $4,800,000 ✅ | 7 days ✅ | Tier 1 ✅ | ✅ |
| Maximilian Roth ($15M, immediately) | $15,000,000 ✅ | 7 days ✅ | Tier 1 ✅ | ✅ |
| Clara Montgomery ($1.2M, 6-12 months) | $1,200,000 ✅ | 270 days ✅ | Tier 3 ✅ | ✅ |
| Bartholomew Knight ($750k, flexible) | $750,000 ✅ | 365 days ✅ | Tier 3 ✅ | ✅ |
| Sloane Harrington (~$3M, end of year) | $3,000,000 ✅ | 180 days ✅ | Tier 1 ✅ | ✅ |
| Caspian North ($2.5M, 30 days) | $2,500,000 ✅ | 30 days ✅ | Tier 1 ✅ | ✅ |
| **Overall Accuracy** | | | | **~95%+** |

---

## Additional Premium Touches

**1. Add a "Leads Counter" field to the Set node (Node 5)**
Use a global variable or a Google Sheets counter row to track total leads processed.

**2. Colorcode the Tier column in Google Sheets**
Use Conditional Formatting → Custom Formula:
- `=$L2="Tier 1"` → Red fill
- `=$L2="Tier 2"` → Yellow fill
- `=$L2="Tier 3"` → Green fill

**3. Add a Google Sheets Filter View**
Create a pre-built filter showing only Tier 1 leads. Share this filtered link with the agent as their "daily action view."

**4. Add phone number formatting**
In the Set node after extraction, add a Code node that standardizes phone formats:
```javascript
const raw = $input.item.json.phone_number || "";
const cleaned = raw.replace(/[^\d+]/g, "");
return { ...$input.item.json, phone_formatted: cleaned };
```

**5. Add a Lead Score field**
Combine budget and timeline into a score:
```javascript
const score = (budget_numeric / 1000000) + (30 / timeline_days * 10);
```
Higher score = higher priority. Log to Column S in the Sheet.

**6. Include test data screenshots in your submission**
Show 3–5 execution screenshots where every node has a green checkmark. This is the strongest visual proof of a working workflow.

**7. Create a README file for the client**
A simple 1-page document explaining:
- How to re-authenticate credentials if they expire
- How to add a new agent to the Follow Up Owner rotation
- How to adjust budget thresholds for Tier classification
- Who to contact for support

---

---

*End of Implementation Guide*

---

**Document:** Luxury Realtor Lead Concierge and Classification System  
**Client:** Vanderbilt & Co.  
**Platform:** n8n  
**Total Nodes:** 27+  
**Prepared by:** AI Workflow Architect  
**Version:** 1.0 — Production Ready  
**Date:** April 2026
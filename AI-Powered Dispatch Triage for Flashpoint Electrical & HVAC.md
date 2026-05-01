# AI-Powered Dispatch Triage for Flashpoint Electrical & HVAC

This guide is written so you can build the workflow manually in n8n yourself, step by step, node by node, without depending on a JSON import.

It follows the same build-it-yourself style as other workflow guides in this series.

It assumes:

- You are using a recent n8n version with the current `Webhook`, `Code`, `Set`, `IF`, `Switch`, `AI`, `Google Sheets`, `Slack`, `Wait`, `Schedule Trigger`, and `Error Trigger` nodes.
- You already have these credentials available inside n8n:
  - `Google Sheets OAuth2`
  - `OpenAI API` (or `Anthropic API` / `Google Gemini API`)
  - `Slack OAuth2` or a `Slack Incoming Webhook URL`

Use these exact workflow names:

- `Flashpoint Dispatch Triage – Main`
- `Flashpoint Dispatch Triage – Error Logger`

---

## What this workflow does

1. Accepts incoming service request submissions through a live webhook or a manual test trigger.
2. Validates that every required field is present before any processing begins.
3. Generates a unique Ticket ID for every new request.
4. Sends the customer's service description to an AI node that returns a trade classification, urgency level, confidence score, and reason.
5. Checks the confidence score. If the AI is uncertain, it routes to a manual review queue and sends an admin alert.
6. Routes confident results through two switch nodes — one for urgency level and one for trade type.
7. Checks Google Sheets for a duplicate ticket from the same phone number, same address, on the same calendar day.
8. If a duplicate exists, it updates the existing row instead of appending a new one.
9. If no duplicate exists, it appends a new row to the Flashpoint Dispatch Log sheet.
10. Sends a formatted Slack alert for Emergency and Urgent jobs. Routine jobs are silently logged.
11. Runs a separate schedule-triggered daily summary that reads the sheet and posts a morning ops report to Slack.
12. Uses a second error workflow to catch execution-level failures and log them centrally.

---

## Business problem solved

Flashpoint Electrical & HVAC receives service requests from multiple sources at all hours. Before this workflow, a Team Lead had to read every submission manually and decide which trade handles it, how urgent it is, and who gets dispatched first. During high-volume periods that creates a bottleneck. Emergency jobs sit unread. Urgent jobs get delayed. The morning standup wastes time discussing jobs that could have been pre-sorted overnight.

This workflow eliminates that bottleneck. The moment a form is submitted, the AI reads the description, classifies the trade and urgency, logs the job, and fires an alert if action is needed right now. The Team Lead wakes up to a morning summary with everything pre-sorted instead of an inbox full of raw submissions.

---

## Before you build

Create all external assets first. You do not want to stop halfway through the canvas to create a spreadsheet.

### 1. Google Sheets spreadsheet

Create one spreadsheet named:

```
Flashpoint Dispatch Log
```

Create these exact tabs:

1. `Dispatch Log`
2. `Manual Review`
3. `Error Logs`

#### Dispatch Log headers

Create this exact header row in row 1:

```
Timestamp | Ticket ID | Customer Name | Contact Phone | Property Address | Service Description | Trade | Urgency | Confidence Score | Status | Assigned Queue | Notes
```

Each column explained:

| Column | Type | Notes |
|---|---|---|
| Timestamp | Text | ISO string from submission |
| Ticket ID | Text | Auto-generated, format FP-YYYYMMDD-XXXX |
| Customer Name | Text | From form payload |
| Contact Phone | Text | Formatted as submitted |
| Property Address | Text | Full address string |
| Service Description | Text | Raw customer description |
| Trade | Text | Electrical, HVAC, or Plumbing |
| Urgency | Text | Emergency, Urgent, or Routine |
| Confidence Score | Number | 0.00 to 1.00 |
| Status | Text | New, Updated, Manual Review |
| Assigned Queue | Text | Emergency Dispatch, Ops Queue, Morning Schedule |
| Notes | Text | AI reason or admin notes |

#### Manual Review headers

```
Timestamp | Ticket ID | Customer Name | Contact Phone | Service Description | AI Trade | AI Urgency | Confidence Score | Reason | Reviewed
```

#### Error Logs headers

```
Timestamp | Workflow Name | Node Name | Error Message | Execution ID
```

Keep the spreadsheet ID ready. You will use it in every Google Sheets node. The spreadsheet ID is the long string between `/d/` and `/edit` in the URL.

### 2. Slack channels

Create or designate these three Slack channels:

- `#flashpoint-emergency` — receives immediate emergency dispatch alerts
- `#flashpoint-ops` — receives urgent job alerts and the daily ops summary
- `#flashpoint-admin` — receives manual review alerts and error notifications

Keep the OAuth credential ready and note the channel IDs for all three channels. Channel IDs look like `C0123456789`. Find them by right-clicking the channel name in Slack and choosing Copy Link, then extracting the ID from the URL.

### 3. AI API key

This guide uses OpenAI. The node type is `OpenAI` with model `gpt-4o-mini`.

If you prefer Anthropic Claude, use model `claude-3-5-haiku-20241022`. The prompt in Step 9 works for either.

If you prefer Google Gemini, use model `gemini-1.5-flash`. Same prompt applies.

Whichever you choose, add the credential to n8n before building the AI node.

### 4. Starter file

The starter file is:

```
flashpoint_service_leads_starter.csv
```

It contains 30 sample service requests including Emergency, Urgent, and Routine jobs across all three trades. You will use it to simulate webhook submissions during testing.

The file columns match the webhook payload exactly:

```
SubmissionTimestamp, CustomerName, ContactPhone, PropertyAddress, ServiceDescription, Trade, Urgency
```

You do not need the `Trade` and `Urgency` columns during workflow testing — the AI will determine those. They are included in the file as reference labels so you can verify the AI output is correct.

### 5. Test payload

This is the payload structure the webhook expects:

```json
{
  "SubmissionTimestamp": "2024-10-20T09:15:00",
  "CustomerName": "Robert Miller",
  "ContactPhone": "555-001-9102",
  "PropertyAddress": "001 Oak Lane, Metropolis",
  "ServiceDescription": "My kitchen outlet is sparking and smells like burning plastic."
}
```

---

## STEP 0: Create the two workflows

### 0A. Create the main workflow

1. Open n8n.
2. Click `New Workflow`.
3. Rename it to:

```
Flashpoint Dispatch Triage – Main
```

4. Save once before adding any nodes.

### 0B. Create the error workflow

1. Create a second new workflow.
2. Rename it to:

```
Flashpoint Dispatch Triage – Error Logger
```

3. Save once.

---

## STEP 1: Build the main workflow skeleton

You will build nodes in this order:

1. `Webhook – New Service Lead`
2. `Respond to Webhook – Immediate Success`
3. `Manual Trigger – Test Mode`
4. `Code – Read CSV Starter Leads`
5. `Merge – Combine Trigger Sources`
6. `Set – Normalize Lead Fields`
7. `IF – Required Fields Present?`
8. `Set – Flag Validation Failure`
9. `Code – Generate Ticket ID`
10. `AI Node – Categorize Trade and Urgency`
11. `Wait – AI Retry Delay`
12. `Code – Parse AI JSON Response`
13. `IF – Confidence Acceptable?`
14. `Set – Build Low Confidence Alert`
15. `Slack – Send Admin Review Alert`
16. `Google Sheets – Log Manual Review`
17. `Switch – Route by Urgency`
18. `Switch – Route by Trade`
19. `Google Sheets – Search Duplicate`
20. `IF – Existing Ticket?`
21. `Google Sheets – Append New Row`
22. `Google Sheets – Update Existing Row`
23. `Set – Build Emergency Alert`
24. `Slack – Send Emergency Alert`
25. `Set – Build Urgent Alert`
26. `Slack – Send Urgent Alert`
27. `Set – Routine Queue Logger`
28. `Set – Final Success Logger`
29. `Schedule Trigger – Daily Summary`
30. `Google Sheets – Read All Rows`
31. `Code – Build Summary Message`
32. `Slack – Send Daily Ops Summary`

The canvas has four clear horizontal lanes:

- Top lane: Emergency path
- Second lane: Urgent path
- Third lane: Routine path
- Bottom lane: Low confidence / Manual review path

The error workflow is built separately and linked to the main workflow in settings.

---

## STEP 2: Create the webhook trigger

### Node 1: Webhook – New Service Lead

Add a `Webhook` node. Rename it to:

```
Webhook – New Service Lead
```

Configure it:

- `HTTP Method`: `POST`
- `Path`: `flashpoint-service-lead`
- `Authentication`: `None`
- `Respond`: `Using Respond to Webhook Node`

The path creates this URL pattern:

```
https://your-n8n-instance.com/webhook/flashpoint-service-lead
```

Set `Respond` to `Using Respond to Webhook Node` because you want to immediately send back a 200 status code while processing continues in the background. This prevents form submissions from hanging while the AI call resolves.

Do not activate this workflow yet. Use `Test URL` during building.

---

## STEP 3: Send the immediate webhook response

### Node 2: Respond to Webhook – Immediate Success

Add a `Respond to Webhook` node immediately after `Webhook – New Service Lead`. Rename it to:

```
Respond to Webhook – Immediate Success
```

Configure it:

- `Respond With`: `JSON`
- `Response Code`: `200`
- `Response Body`:

```json
{
  "status": "received",
  "message": "Your service request has been received. A dispatcher will contact you shortly."
}
```

Connect the `Webhook – New Service Lead` output directly to this node first.

Then also connect the `Webhook – New Service Lead` output to `Merge – Combine Trigger Sources` (Node 5). In n8n, one node can connect to multiple downstream nodes.

The reason this response goes out first is so the customer's browser or form platform gets a confirmation immediately. The rest of the workflow continues in parallel.

---

## STEP 4: Create the manual test trigger

### Node 3: Manual Trigger – Test Mode

Add a `Manual Trigger` node. Rename it to:

```
Manual Trigger – Test Mode
```

No configuration is needed. This node starts the workflow manually from inside the n8n editor.

Connect it to `Code – Read CSV Starter Leads` (Node 4).

---

## STEP 5: Load the CSV starter data

### Node 4: Code – Read CSV Starter Leads

Add a `Code` node after `Manual Trigger – Test Mode`. Rename it to:

```
Code – Read CSV Starter Leads
```

Set:

- `Mode`: `Run Once for All Items`
- `Language`: `JavaScript`

Paste this exact code:

```javascript
// This node simulates the CSV rows as webhook payloads.
// Replace the rows array with any subset of the starter file for focused testing.

const rows = [
  {
    SubmissionTimestamp: "2024-10-20T09:15:00",
    CustomerName: "Robert Miller",
    ContactPhone: "555-001-9102",
    PropertyAddress: "001 Oak Lane, Metropolis",
    ServiceDescription: "My kitchen outlet is sparking and smells like burning plastic."
  },
  {
    SubmissionTimestamp: "2024-10-20T10:30:00",
    CustomerName: "Sarah Jenkins",
    ContactPhone: "555-002-9102",
    PropertyAddress: "002 Oak Lane, Metropolis",
    ServiceDescription: "I need a quote for installing a new smart thermostat."
  },
  {
    SubmissionTimestamp: "2024-10-20T11:45:00",
    CustomerName: "Michael Chen",
    ContactPhone: "555-003-9102",
    PropertyAddress: "003 Oak Lane, Metropolis",
    ServiceDescription: "The water heater is leaking all over the basement floor."
  },
  {
    SubmissionTimestamp: "2024-10-20T13:10:00",
    CustomerName: "Linda Thompson",
    ContactPhone: "555-004-9102",
    PropertyAddress: "004 Oak Lane, Metropolis",
    ServiceDescription: "My AC unit is making a loud grinding noise but still cooling."
  },
  {
    SubmissionTimestamp: "2024-10-21T08:12:00",
    CustomerName: "Patricia Moore",
    ContactPhone: "555-008-9102",
    PropertyAddress: "008 Oak Lane, Metropolis",
    ServiceDescription: "A tree limb hit the power line and now half my house has no power."
  },
  {
    SubmissionTimestamp: "2024-10-22T10:45:00",
    CustomerName: "Thomas Lewis",
    ContactPhone: "555-017-9102",
    PropertyAddress: "017 Oak Lane, Metropolis",
    ServiceDescription: "Basement is flooding because the sump pump failed during the storm."
  }
];

return rows.map(row => ({ json: row }));
```

This node outputs multiple items, one per row. Each item has the same shape as the live webhook payload, so the rest of the workflow behaves identically whether triggered by a real form submission or by this test node.

To test with a different row, add or remove entries from the `rows` array.

---

## STEP 6: Merge the two trigger paths

### Node 5: Merge – Combine Trigger Sources

Add a `Merge` node. Rename it to:

```
Merge – Combine Trigger Sources
```

Configure it:

- `Mode`: `Pass-Through`

Connect:

- `Webhook – New Service Lead` output → `Merge – Combine Trigger Sources` input 1
- `Code – Read CSV Starter Leads` output → `Merge – Combine Trigger Sources` input 2

This node ensures a single downstream pipeline regardless of how the workflow was triggered. Every item that exits this node has the same field structure.

---

## STEP 7: Normalize incoming fields

### Node 6: Set – Normalize Lead Fields

Add a `Set` node after `Merge – Combine Trigger Sources`. Rename it to:

```
Set – Normalize Lead Fields
```

Set `Mode` to `Keep All Fields`. Then add these assignments:

| Field Name | Value |
|---|---|
| `SubmissionTimestamp` | `{{ $json["body"]["SubmissionTimestamp"] \|\| $json["SubmissionTimestamp"] \|\| $now.toISO() }}` |
| `CustomerName` | `{{ ($json["body"]["CustomerName"] \|\| $json["CustomerName"] \|\| "").toString().trim() }}` |
| `ContactPhone` | `{{ ($json["body"]["ContactPhone"] \|\| $json["ContactPhone"] \|\| "").toString().trim() }}` |
| `PropertyAddress` | `{{ ($json["body"]["PropertyAddress"] \|\| $json["PropertyAddress"] \|\| "").toString().trim() }}` |
| `ServiceDescription` | `{{ ($json["body"]["ServiceDescription"] \|\| $json["ServiceDescription"] \|\| "").toString().trim() }}` |

The double-path expressions (`body.X || X`) handle both live webhook payloads (which nest fields inside `body`) and the manual CSV items (which have fields at the top level). After this node all downstream references use clean, consistent field names.

---

## STEP 8: Validate required fields

### Node 7: IF – Required Fields Present?

Add an `IF` node after `Set – Normalize Lead Fields`. Rename it to:

```
IF – Required Fields Present?
```

Add four conditions connected with `AND`:

| Field | Condition | Value |
|---|---|---|
| `{{ $json["CustomerName"] }}` | `is not empty` | |
| `{{ $json["ContactPhone"] }}` | `is not empty` | |
| `{{ $json["PropertyAddress"] }}` | `is not empty` | |
| `{{ $json["ServiceDescription"] }}` | `is not empty` | |

- `TRUE` output → continues to `Code – Generate Ticket ID` (Node 9)
- `FALSE` output → goes to `Set – Flag Validation Failure` (Node 8)

### Node 8: Set – Flag Validation Failure

Add a `Set` node on the `FALSE` branch. Rename it to:

```
Set – Flag Validation Failure
```

Add these assignments:

| Field | Value |
|---|---|
| `Status` | `Validation Failed` |
| `Notes` | `Required fields missing. CustomerName, ContactPhone, PropertyAddress, and ServiceDescription are all required.` |

In the node settings panel, turn on `Retry On Fail`: `Off`. This is a terminal node for invalid submissions. No further processing should occur.

---

## STEP 9: Generate the ticket ID

### Node 9: Code – Generate Ticket ID

Add a `Code` node on the `TRUE` branch from `IF – Required Fields Present?`. Rename it to:

```
Code – Generate Ticket ID
```

Set:

- `Mode`: `Run Once for Each Item`
- `Language`: `JavaScript`

Paste this exact code:

```javascript
const ts = $json["SubmissionTimestamp"] || new Date().toISOString();
const date = new Date(ts);

const year = date.getFullYear();
const month = String(date.getMonth() + 1).padStart(2, '0');
const day = String(date.getDate()).padStart(2, '0');
const dateStr = `${year}${month}${day}`;

const rand = Math.floor(1000 + Math.random() * 9000);
const ticketId = `FP-${dateStr}-${rand}`;

const submissionDate = `${year}-${month}-${day}`;

return [{
  json: {
    ...$json,
    TicketID: ticketId,
    SubmissionDate: submissionDate,
  }
}];
```

After this node, every item has a `TicketID` like `FP-20241020-4823` and a `SubmissionDate` like `2024-10-20`. The `SubmissionDate` field is used later for duplicate checking.

---

## STEP 10: Send the description to AI

### Node 10: AI Node – Categorize Trade and Urgency

Add an `OpenAI` node (or your chosen AI provider node) after `Code – Generate Ticket ID`. Rename it to:

```
AI Node – Categorize Trade and Urgency
```

Configure it:

- `Resource`: `Chat`
- `Operation`: `Create a Chat Completion`
- `Model`: `gpt-4o-mini`
- `Credential`: your OpenAI API credential

Set `Messages` to `Define Below`.

Add one message:

- `Role`: `User`
- `Content`: use this exact prompt. Paste it as a single expression:

```
You are a dispatch triage assistant for Flashpoint Electrical & HVAC, a field service company.

A customer has submitted this service request:

"{{ $json["ServiceDescription"] }}"

Your job is to classify this request and return ONLY a valid JSON object. Do not write any explanation, preamble, markdown, or backticks. Return only raw JSON.

The JSON must have exactly these four fields:

{
  "trade": "",
  "urgency": "",
  "confidence": 0.0,
  "reason": ""
}

Rules for "trade":
- Use exactly one of these three values: Electrical, HVAC, Plumbing
- Electrical: outlets, breakers, wiring, panels, lights, switches, power outages, ceiling fans, circuit issues
- HVAC: air conditioning, heating, furnace, thermostat, vents, gas smell near HVAC unit, ductwork
- Plumbing: water heater, pipes, drains, toilets, faucets, sump pump, flooding from water, sewer

Rules for "urgency":
- Use exactly one of these three values: Emergency, Urgent, Routine
- Emergency: sparking, burning smell, fire risk, no power affecting safety, flooding, gas smell, frozen pipes in winter, no heat below 50°F, sewage backup, overflowing toilet
- Urgent: equipment still working but failing, loud noises from HVAC, slow drains, intermittent outages, no hot water, tripping breakers under specific load
- Routine: quotes, estimates, installations, annual maintenance, cosmetic upgrades, minor drips, general inquiries

Rules for "confidence":
- A decimal number between 0.00 and 1.00
- Use 0.95 or higher when the description clearly matches one trade and one urgency level
- Use 0.70 to 0.94 when the description is reasonable but slightly ambiguous
- Use below 0.70 when the description is very vague, could fit multiple trades, or contradicts itself

Rules for "reason":
- One sentence explaining why you chose this trade and urgency
- Do not repeat the customer description verbatim

Examples:
- "My kitchen outlet is sparking and smells like burning plastic" → Electrical, Emergency, 0.98
- "I need a quote for installing a new smart thermostat" → HVAC, Routine, 0.95
- "The water heater is leaking all over the basement floor" → Plumbing, Emergency, 0.97
- "My AC unit is making a loud grinding noise but still cooling" → HVAC, Urgent, 0.93
- "I want to add some outdoor lighting to my backyard patio" → Electrical, Routine, 0.94
- "There is a strong smell of gas near the HVAC unit in the attic" → HVAC, Emergency, 0.99
- "Basement is flooding because the sump pump failed during the storm" → Plumbing, Emergency, 0.98
- "Light flickering in the hallway whenever the fridge kicks on" → Electrical, Urgent, 0.91
```

In the node `Settings` tab, turn on:

- `Retry On Fail`: `On`
- `Max Tries`: `3`
- `Wait Between Tries (ms)`: `5000`

This protects against transient OpenAI timeouts without manual intervention.

---

## STEP 11: Add the AI retry wait node

### Node 11: Wait – AI Retry Delay

Add a `Wait` node immediately after `AI Node – Categorize Trade and Urgency`. Rename it to:

```
Wait – AI Retry Delay
```

Configure it:

- `Amount`: `5`
- `Unit`: `Seconds`

This node exists as a manual delay fallback for cases where you want to throttle AI calls if you are batch-processing the full CSV during testing. It is safe to keep even in production since the delay is short and the retry logic in the AI node itself handles most failure scenarios. You can disable this node in production if latency is a concern.

---

## STEP 12: Parse the AI JSON response

### Node 12: Code – Parse AI JSON Response

Add a `Code` node after `Wait – AI Retry Delay`. Rename it to:

```
Code – Parse AI JSON Response
```

Set:

- `Mode`: `Run Once for Each Item`
- `Language`: `JavaScript`

Paste this exact code:

```javascript
// Extract the raw text from the AI response
const aiMessage = $json?.choices?.[0]?.message?.content
  || $json?.message?.content
  || $json?.text
  || "";

const rawText = String(aiMessage).trim();

// Attempt to parse. Strip markdown fences if present.
let parsed = null;
let parseError = "";

try {
  const cleaned = rawText
    .replace(/^```json\s*/i, "")
    .replace(/^```\s*/i, "")
    .replace(/```\s*$/i, "")
    .trim();
  parsed = JSON.parse(cleaned);
} catch (e) {
  parseError = e.message;
}

// Validate the expected fields exist
const validTrades = ["Electrical", "HVAC", "Plumbing"];
const validUrgencies = ["Emergency", "Urgent", "Routine"];

const trade = parsed?.trade || "";
const urgency = parsed?.urgency || "";
const confidence = parseFloat(parsed?.confidence ?? 0);
const reason = parsed?.reason || "";

const tradeValid = validTrades.includes(trade);
const urgencyValid = validUrgencies.includes(urgency);
const parseSuccess = parsed !== null && tradeValid && urgencyValid;

return [{
  json: {
    ...$json,
    AI_Trade: tradeValid ? trade : "Unknown",
    AI_Urgency: urgencyValid ? urgency : "Unknown",
    AI_Confidence: isNaN(confidence) ? 0 : confidence,
    AI_Reason: reason,
    AI_ParseSuccess: parseSuccess,
    AI_RawResponse: rawText,
    AI_ParseError: parseError,
  }
}];
```

After this node, every item has these new fields:

```json
{
  "AI_Trade": "Electrical",
  "AI_Urgency": "Emergency",
  "AI_Confidence": 0.98,
  "AI_Reason": "Sparking outlet with burning smell is an electrical emergency and fire risk.",
  "AI_ParseSuccess": true,
  "AI_RawResponse": "{\"trade\":\"Electrical\",\"urgency\":\"Emergency\",\"confidence\":0.98,\"reason\":\"...\"}",
  "AI_ParseError": ""
}
```

---

## STEP 13: Check the confidence score

### Node 13: IF – Confidence Acceptable?

Add an `IF` node after `Code – Parse AI JSON Response`. Rename it to:

```
IF – Confidence Acceptable?
```

Add two conditions connected with `AND`:

| Field | Condition | Value |
|---|---|---|
| `{{ $json["AI_ParseSuccess"] }}` | `is true` | |
| `{{ $json["AI_Confidence"] }}` | `is greater than or equal to` | `0.7` |

- `TRUE` output → continues to `Switch – Route by Urgency` (Node 17)
- `FALSE` output → goes to `Set – Build Low Confidence Alert` (Node 14)

The threshold of 0.70 means the AI must be at least 70% confident before automatic routing. Anything below that goes to a human reviewer.

---

## STEP 14: Build the low-confidence alert

### Node 14: Set – Build Low Confidence Alert

Add a `Set` node on the `FALSE` branch. Rename it to:

```
Set – Build Low Confidence Alert
```

Add these assignments:

| Field | Value |
|---|---|
| `AlertMessage` | `⚠️ *Manual Review Required*\n\n*Ticket:* {{ $json["TicketID"] }}\n*Customer:* {{ $json["CustomerName"] }}\n*Phone:* {{ $json["ContactPhone"] }}\n*Issue:* {{ $json["ServiceDescription"] }}\n\n*AI Result:* {{ $json["AI_Trade"] }} / {{ $json["AI_Urgency"] }}\n*Confidence:* {{ $json["AI_Confidence"] }}\n*Reason:* {{ $json["AI_Reason"] }}\n\nPlease review and assign manually.` |
| `Status` | `Manual Review` |
| `AssignedQueue` | `Manual Review Queue` |

---

## STEP 15: Send the admin review alert

### Node 15: Slack – Send Admin Review Alert

Add a `Slack` node after `Set – Build Low Confidence Alert`. Rename it to:

```
Slack – Send Admin Review Alert
```

Configure it:

- `Resource`: `Message`
- `Operation`: `Post`
- `Channel`: your `#flashpoint-admin` channel ID
- `Text`: `{{ $json["AlertMessage"] }}`
- `Credential`: your Slack OAuth credential

In settings, turn on:

- `Retry On Fail`: `On`
- `Max Tries`: `3`
- `Wait Between Tries (ms)`: `3000`

---

## STEP 16: Log to manual review sheet

### Node 16: Google Sheets – Log Manual Review

Add a `Google Sheets` node after `Slack – Send Admin Review Alert`. Rename it to:

```
Google Sheets – Log Manual Review
```

Configure it:

- `Resource`: `Sheet Within Document`
- `Operation`: `Append Row`
- `Document ID`: your spreadsheet ID
- `Sheet Name`: `Manual Review`

Map these columns:

| Sheet Column | Expression |
|---|---|
| `Timestamp` | `{{ $json["SubmissionTimestamp"] }}` |
| `Ticket ID` | `{{ $json["TicketID"] }}` |
| `Customer Name` | `{{ $json["CustomerName"] }}` |
| `Contact Phone` | `{{ $json["ContactPhone"] }}` |
| `Service Description` | `{{ $json["ServiceDescription"] }}` |
| `AI Trade` | `{{ $json["AI_Trade"] }}` |
| `AI Urgency` | `{{ $json["AI_Urgency"] }}` |
| `Confidence Score` | `{{ $json["AI_Confidence"] }}` |
| `Reason` | `{{ $json["AI_Reason"] }}` |
| `Reviewed` | `No` |

In settings, turn on `Retry On Fail`, max 3 tries, 2000ms wait.

---

## STEP 17: Route by urgency level

### Node 17: Switch – Route by Urgency

Add a `Switch` node on the `TRUE` branch from `IF – Confidence Acceptable?`. Rename it to:

```
Switch – Route by Urgency
```

Set `Mode` to `Rules`.

Add three rules in this order:

**Rule 1 — Emergency:**
- `Value 1`: `{{ $json["AI_Urgency"] }}`
- `Operation`: `equals`
- `Value 2`: `Emergency`
- Output label: `Emergency`

**Rule 2 — Urgent:**
- `Value 1`: `{{ $json["AI_Urgency"] }}`
- `Operation`: `equals`
- `Value 2`: `Urgent`
- Output label: `Urgent`

**Rule 3 — Routine:**
- `Value 1`: `{{ $json["AI_Urgency"] }}`
- `Operation`: `equals`
- `Value 2`: `Routine`
- Output label: `Routine`

Set `Fallback Output` to `None`. The confidence check before this node ensures only valid urgency values reach it.

---

## STEP 18: Route by trade type

### Node 18: Switch – Route by Trade

Add a `Switch` node after `Switch – Route by Urgency`. Rename it to:

```
Switch – Route by Trade
```

All three urgency outputs from Node 17 converge on this single node. Connect the Emergency output, Urgent output, and Routine output all to the same `Switch – Route by Trade` input.

Set `Mode` to `Rules`.

Add three rules:

**Rule 1 — Electrical:**
- `Value 1`: `{{ $json["AI_Trade"] }}`
- `Operation`: `equals`
- `Value 2`: `Electrical`
- Output label: `Electrical`

**Rule 2 — HVAC:**
- `Value 1`: `{{ $json["AI_Trade"] }}`
- `Operation`: `equals`
- `Value 2`: `HVAC`
- Output label: `HVAC`

**Rule 3 — Plumbing:**
- `Value 1`: `{{ $json["AI_Trade"] }}`
- `Operation`: `equals`
- `Value 2`: `Plumbing`
- Output label: `Plumbing`

The urgency level is still on the item as `AI_Urgency`. All three trade branches reconnect to the duplicate check node (Node 19). The urgency context travels with the item throughout.

---

## STEP 19: Check for duplicate submissions

### Node 19: Google Sheets – Search Duplicate

Add a `Google Sheets` node after `Switch – Route by Trade`. Connect all three trade outputs to the same node. Rename it to:

```
Google Sheets – Search Duplicate
```

Configure it:

- `Resource`: `Sheet Within Document`
- `Operation`: `Read Rows`
- `Document ID`: your spreadsheet ID
- `Sheet Name`: `Dispatch Log`
- `Filters`:

Add filter — Column `Contact Phone` equals `{{ $json["ContactPhone"] }}`

Return all matching rows. You will check the address and date in the next code step.

In settings, turn on `Retry On Fail`, max 3 tries, 2000ms wait.

---

## STEP 20: Check if the duplicate is real

### Node 20: IF – Existing Ticket?

Add an `IF` node after `Google Sheets – Search Duplicate`. Rename it to:

```
IF – Existing Ticket?
```

The Google Sheets read returns the matched rows as items. Use a Code node approach inside the IF expression.

Set the condition:

- `Value 1`:

```
{{ $json["Contact Phone"] !== undefined && $json["Property Address"] === $("Set – Normalize Lead Fields").item.json["PropertyAddress"] && $json["Timestamp"].startsWith($("Code – Generate Ticket ID").item.json["SubmissionDate"]) }}
```

- `Operation`: `is true`

If n8n does not allow this complex expression in the IF node directly, replace this with a `Code` node before it:

Add a `Code – Check Duplicate Match` node between the Google Sheets read and the IF:

```javascript
const incoming = $("Code – Generate Ticket ID").first().json;
const sheetRows = $input.all();

const duplicate = sheetRows.find(row => {
  const samePhone = row.json["Contact Phone"] === incoming.ContactPhone;
  const sameAddress = row.json["Property Address"] === incoming.PropertyAddress;
  const sameDay = (row.json["Timestamp"] || "").startsWith(incoming.SubmissionDate);
  return samePhone && sameAddress && sameDay;
});

return [{
  json: {
    ...incoming,
    DuplicateFound: duplicate ? true : false,
    ExistingRowNumber: duplicate ? duplicate.json["_rowNumber"] : null,
  }
}];
```

Then the IF node is simply:

- `Value 1`: `{{ $json["DuplicateFound"] }}`
- `Operation`: `is true`

- `TRUE` output → `Google Sheets – Update Existing Row` (Node 22)
- `FALSE` output → `Google Sheets – Append New Row` (Node 21)

---

## STEP 21: Append a new row

### Node 21: Google Sheets – Append New Row

Add a `Google Sheets` node on the `FALSE` branch. Rename it to:

```
Google Sheets – Append New Row
```

Configure it:

- `Resource`: `Sheet Within Document`
- `Operation`: `Append Row`
- `Document ID`: your spreadsheet ID
- `Sheet Name`: `Dispatch Log`

Map these columns:

| Sheet Column | Expression |
|---|---|
| `Timestamp` | `{{ $json["SubmissionTimestamp"] }}` |
| `Ticket ID` | `{{ $json["TicketID"] }}` |
| `Customer Name` | `{{ $json["CustomerName"] }}` |
| `Contact Phone` | `{{ $json["ContactPhone"] }}` |
| `Property Address` | `{{ $json["PropertyAddress"] }}` |
| `Service Description` | `{{ $json["ServiceDescription"] }}` |
| `Trade` | `{{ $json["AI_Trade"] }}` |
| `Urgency` | `{{ $json["AI_Urgency"] }}` |
| `Confidence Score` | `{{ $json["AI_Confidence"] }}` |
| `Status` | `New` |
| `Assigned Queue` | `{{ $json["AI_Urgency"] === "Emergency" ? "Emergency Dispatch" : $json["AI_Urgency"] === "Urgent" ? "Ops Queue" : "Morning Schedule" }}` |
| `Notes` | `{{ $json["AI_Reason"] }}` |

In settings, turn on `Retry On Fail`, max 3 tries, 2000ms wait.

After this node, connect the output to `Set – Build Emergency Alert` (Node 23) for items where `AI_Urgency` is `Emergency`, and similarly for the other urgency branches. The routing back to urgency-specific alert nodes is handled through a second IF or Switch after the sheet operations.

Add a `Switch – Route Alert by Urgency` node after both `Google Sheets – Append New Row` and `Google Sheets – Update Existing Row`. Use the same urgency routing rules as Node 17.

---

## STEP 22: Update an existing duplicate row

### Node 22: Google Sheets – Update Existing Row

Add a `Google Sheets` node on the `TRUE` branch from `IF – Existing Ticket?`. Rename it to:

```
Google Sheets – Update Existing Row
```

Configure it:

- `Resource`: `Sheet Within Document`
- `Operation`: `Update Row`
- `Document ID`: your spreadsheet ID
- `Sheet Name`: `Dispatch Log`
- `Row Number`: `{{ $json["ExistingRowNumber"] }}`

Map these columns to update:

| Sheet Column | Expression |
|---|---|
| `Status` | `Updated` |
| `Notes` | `Duplicate submission received. Original ticket retained. Re-submitted at {{ $now.toISO() }}` |

In settings, turn on `Retry On Fail`, max 3 tries, 2000ms wait.

Connect the output of this node to the same `Switch – Route Alert by Urgency` node used after Node 21.

---

## STEP 23: Build the emergency alert

### Node 23: Set – Build Emergency Alert

Add a `Set` node on the `Emergency` branch from your post-sheet urgency switch. Rename it to:

```
Set – Build Emergency Alert
```

Add this assignment:

| Field | Value |
|---|---|
| `AlertMessage` | `🚨 *EMERGENCY SERVICE REQUEST*\n\n*Ticket:* {{ $json["TicketID"] }}\n*Customer:* {{ $json["CustomerName"] }}\n*Phone:* {{ $json["ContactPhone"] }}\n*Trade:* {{ $json["AI_Trade"] }}\n*Issue:* {{ $json["ServiceDescription"] }}\n*Address:* {{ $json["PropertyAddress"] }}\n*Confidence:* {{ ($json["AI_Confidence"] * 100).toFixed(0) }}%\n\n⚡ *DISPATCH IMMEDIATELY*` |

---

## STEP 24: Send the emergency alert

### Node 24: Slack – Send Emergency Alert

Add a `Slack` node after `Set – Build Emergency Alert`. Rename it to:

```
Slack – Send Emergency Alert
```

Configure it:

- `Resource`: `Message`
- `Operation`: `Post`
- `Channel`: your `#flashpoint-emergency` channel ID
- `Text`: `{{ $json["AlertMessage"] }}`

In settings:

- `Retry On Fail`: `On`
- `Max Tries`: `5`
- `Wait Between Tries (ms)`: `3000`

Emergency alerts get 5 retry attempts instead of 3 because these are life-safety situations.

---

## STEP 25: Build the urgent alert

### Node 25: Set – Build Urgent Alert

Add a `Set` node on the `Urgent` branch. Rename it to:

```
Set – Build Urgent Alert
```

Add this assignment:

| Field | Value |
|---|---|
| `AlertMessage` | `⚠️ *Urgent Service Request*\n\n*Ticket:* {{ $json["TicketID"] }}\n*Customer:* {{ $json["CustomerName"] }}\n*Phone:* {{ $json["ContactPhone"] }}\n*Trade:* {{ $json["AI_Trade"] }}\n*Issue:* {{ $json["ServiceDescription"] }}\n*Address:* {{ $json["PropertyAddress"] }}\n*Confidence:* {{ ($json["AI_Confidence"] * 100).toFixed(0) }}%\n\nSchedule within 2–4 hours.` |

---

## STEP 26: Send the urgent alert

### Node 26: Slack – Send Urgent Alert

Add a `Slack` node after `Set – Build Urgent Alert`. Rename it to:

```
Slack – Send Urgent Alert
```

Configure it:

- `Resource`: `Message`
- `Operation`: `Post`
- `Channel`: your `#flashpoint-ops` channel ID
- `Text`: `{{ $json["AlertMessage"] }}`

In settings, turn on `Retry On Fail`, max 3 tries, 3000ms wait.

---

## STEP 27: Log routine jobs silently

### Node 27: Set – Routine Queue Logger

Add a `Set` node on the `Routine` branch. Rename it to:

```
Set – Routine Queue Logger
```

Add these assignments:

| Field | Value |
|---|---|
| `Status` | `Logged` |
| `AssignedQueue` | `Morning Schedule` |
| `Notes` | `{{ $json["AI_Reason"] }}` |
| `LogMessage` | `📋 Routine job logged: {{ $json["TicketID"] }} – {{ $json["AI_Trade"] }} – {{ $json["CustomerName"] }}` |

Routine jobs do not trigger Slack alerts. They appear in the morning daily summary instead. No further action nodes are needed on this branch.

---

## STEP 28: Mark every path as complete

### Node 28: Set – Final Success Logger

Add a `Set` node after the Emergency alert, Urgent alert, and Routine logger branches. Connect all three branches to this same node. Rename it to:

```
Set – Final Success Logger
```

Add these assignments:

| Field | Value |
|---|---|
| `WorkflowStatus` | `Complete` |
| `CompletedAt` | `{{ $now.toISO() }}` |
| `ProcessedTicket` | `{{ $json["TicketID"] }}` |

This node serves as the terminus for every successful execution. In the n8n execution log, you will always see this node as the last node for healthy runs, which makes debugging easy.

---

## STEP 29: Daily summary trigger

### Node 29: Schedule Trigger – Daily Summary

Add a `Schedule Trigger` node. Rename it to:

```
Schedule Trigger – Daily Summary
```

Configure it:

- `Trigger Rule`: `Cron`
- `Expression`: `0 7 * * 1-5`

This runs every weekday at 7:00 AM. Adjust the hour to match the Team Lead's morning standup time.

This node is the start of a separate mini-flow within the same workflow canvas. It does not connect to the main webhook path.

---

## STEP 30: Read all rows for the summary

### Node 30: Google Sheets – Read All Rows

Add a `Google Sheets` node after `Schedule Trigger – Daily Summary`. Rename it to:

```
Google Sheets – Read All Rows
```

Configure it:

- `Resource`: `Sheet Within Document`
- `Operation`: `Read Rows`
- `Document ID`: your spreadsheet ID
- `Sheet Name`: `Dispatch Log`

Add a filter: Column `Timestamp` contains today's date.

To build today's date string for the filter: use `{{ $now.toFormat('yyyy-MM-dd') }}` in the filter value field.

In settings, turn on `Retry On Fail`, max 3 tries, 2000ms wait.

---

## STEP 31: Build the daily summary message

### Node 31: Code – Build Summary Message

Add a `Code` node after `Google Sheets – Read All Rows`. Rename it to:

```
Code – Build Summary Message
```

Set:

- `Mode`: `Run Once for All Items`
- `Language`: `JavaScript`

Paste this code:

```javascript
const rows = $input.all().map(i => i.json);

const today = new Date().toLocaleDateString('en-US', {
  weekday: 'long', year: 'numeric', month: 'long', day: 'numeric'
});

const emergencies = rows.filter(r => r["Urgency"] === "Emergency");
const urgents = rows.filter(r => r["Urgency"] === "Urgent");
const routines = rows.filter(r => r["Urgency"] === "Routine");

const electrical = rows.filter(r => r["Trade"] === "Electrical").length;
const hvac = rows.filter(r => r["Trade"] === "HVAC").length;
const plumbing = rows.filter(r => r["Trade"] === "Plumbing").length;

const emergencyLines = emergencies.map(r =>
  `  • ${r["Ticket ID"]} — ${r["Customer Name"]} — ${r["Property Address"]}`
).join("\n") || "  None";

const urgentLines = urgents.map(r =>
  `  • ${r["Ticket ID"]} — ${r["Customer Name"]} — ${r["Trade"]}`
).join("\n") || "  None";

const summary = `📋 *Flashpoint Daily Ops Summary — ${today}*

*Total Jobs Logged Today:* ${rows.length}

🚨 *Emergency (${emergencies.length}):*
${emergencyLines}

⚠️ *Urgent (${urgents.length}):*
${urgentLines}

📅 *Routine Scheduled (${routines.length})*

*By Trade:*
  ⚡ Electrical: ${electrical}
  🌡 HVAC: ${hvac}
  🔧 Plumbing: ${plumbing}

Review the full log: https://docs.google.com/spreadsheets/d/YOUR_SHEET_ID/edit`;

return [{ json: { SummaryMessage: summary } }];
```

Replace `YOUR_SHEET_ID` with your actual spreadsheet ID.

---

## STEP 32: Send the daily ops summary

### Node 32: Slack – Send Daily Ops Summary

Add a `Slack` node after `Code – Build Summary Message`. Rename it to:

```
Slack – Send Daily Ops Summary
```

Configure it:

- `Resource`: `Message`
- `Operation`: `Post`
- `Channel`: your `#flashpoint-ops` channel ID
- `Text`: `{{ $json["SummaryMessage"] }}`

In settings, turn on `Retry On Fail`, max 3 tries, 3000ms wait.

---

## STEP 33: Build the error workflow

Switch to the second workflow: `Flashpoint Dispatch Triage – Error Logger`.

### Node E1: Error Trigger – Global

Add an `Error Trigger` node. Rename it to:

```
Error Trigger – Global
```

No configuration is needed. n8n automatically passes the error payload into this node when the main workflow crashes.

### Node E2: Set – Prepare Error Payload

Add a `Set` node after `Error Trigger – Global`. Rename it to:

```
Set – Prepare Error Payload
```

Add these assignments:

| Field | Value |
|---|---|
| `ErrorTimestamp` | `{{ new Date().toISOString() }}` |
| `WorkflowName` | `{{ $json.workflow?.name \|\| "Flashpoint Dispatch Triage – Main" }}` |
| `NodeName` | `{{ $json.execution?.lastNodeExecuted \|\| "Unknown" }}` |
| `ErrorMessage` | `{{ $json.execution?.error?.message \|\| $json.error?.message \|\| "Unknown error" }}` |
| `ExecutionID` | `{{ $json.execution?.id \|\| "" }}` |
| `SlackAlert` | `🔴 *Workflow Error – Flashpoint Dispatch Triage*\n\n*Node:* {{ $json.execution?.lastNodeExecuted \|\| "Unknown" }}\n*Error:* {{ $json.execution?.error?.message \|\| "Unknown error" }}\n*Time:* {{ new Date().toISOString() }}\n\nCheck n8n execution logs for details.` |

### Node E3: Google Sheets – Log Error Row

Add a `Google Sheets` node after `Set – Prepare Error Payload`. Rename it to:

```
Google Sheets – Log Error Row
```

Configure it:

- `Resource`: `Sheet Within Document`
- `Operation`: `Append Row`
- `Document ID`: your spreadsheet ID
- `Sheet Name`: `Error Logs`

Map:

| Sheet Column | Expression |
|---|---|
| `Timestamp` | `{{ $json["ErrorTimestamp"] }}` |
| `Workflow Name` | `{{ $json["WorkflowName"] }}` |
| `Node Name` | `{{ $json["NodeName"] }}` |
| `Error Message` | `{{ $json["ErrorMessage"] }}` |
| `Execution ID` | `{{ $json["ExecutionID"] }}` |

In settings, turn on `Retry On Fail`, max 3 tries, 2000ms wait.

### Node E4: Slack – Admin Error Alert

Add a `Slack` node after `Google Sheets – Log Error Row`. Rename it to:

```
Slack – Admin Error Alert
```

Configure it:

- `Resource`: `Message`
- `Operation`: `Post`
- `Channel`: your `#flashpoint-admin` channel ID
- `Text`: `{{ $json["SlackAlert"] }}`

In settings, turn on `Retry On Fail`, max 3 tries, 3000ms wait.

### Link the error workflow to the main workflow

1. Open the main workflow `Flashpoint Dispatch Triage – Main`.
2. Click the three-dot menu in the top right → `Settings`.
3. Find `Error Workflow`.
4. Select `Flashpoint Dispatch Triage – Error Logger`.
5. Save the main workflow.

---

## STEP 34: Full connection map

Use this to verify your canvas connections are correct.

```
Webhook – New Service Lead
  -> Respond to Webhook – Immediate Success
  -> Merge – Combine Trigger Sources

Manual Trigger – Test Mode
  -> Code – Read CSV Starter Leads
  -> Merge – Combine Trigger Sources

Merge – Combine Trigger Sources
  -> Set – Normalize Lead Fields
  -> IF – Required Fields Present?

IF – Required Fields Present? FALSE
  -> Set – Flag Validation Failure (terminal)

IF – Required Fields Present? TRUE
  -> Code – Generate Ticket ID
  -> AI Node – Categorize Trade and Urgency
  -> Wait – AI Retry Delay
  -> Code – Parse AI JSON Response
  -> IF – Confidence Acceptable?

IF – Confidence Acceptable? FALSE
  -> Set – Build Low Confidence Alert
  -> Slack – Send Admin Review Alert
  -> Google Sheets – Log Manual Review (terminal)

IF – Confidence Acceptable? TRUE
  -> Switch – Route by Urgency

Switch – Route by Urgency (all three outputs)
  -> Switch – Route by Trade

Switch – Route by Trade (all three outputs)
  -> Google Sheets – Search Duplicate
  -> Code – Check Duplicate Match
  -> IF – Existing Ticket?

IF – Existing Ticket? FALSE
  -> Google Sheets – Append New Row

IF – Existing Ticket? TRUE
  -> Google Sheets – Update Existing Row

Google Sheets – Append New Row
  -> Switch – Route Alert by Urgency

Google Sheets – Update Existing Row
  -> Switch – Route Alert by Urgency

Switch – Route Alert by Urgency Emergency
  -> Set – Build Emergency Alert
  -> Slack – Send Emergency Alert
  -> Set – Final Success Logger

Switch – Route Alert by Urgency Urgent
  -> Set – Build Urgent Alert
  -> Slack – Send Urgent Alert
  -> Set – Final Success Logger

Switch – Route Alert by Urgency Routine
  -> Set – Routine Queue Logger
  -> Set – Final Success Logger

Schedule Trigger – Daily Summary
  -> Google Sheets – Read All Rows
  -> Code – Build Summary Message
  -> Slack – Send Daily Ops Summary
```

Error workflow:

```
Error Trigger – Global
  -> Set – Prepare Error Payload
  -> Google Sheets – Log Error Row
  -> Slack – Admin Error Alert
```

---

## STEP 35: Recommended canvas layout

Lay the canvas out left to right with four horizontal lanes:

- Far left column: trigger nodes (Webhook, Manual, Schedule)
- Second column: normalization (Merge, Set, IF validation)
- Third column: AI processing (Code Ticket, AI Node, Wait, Code Parse, IF Confidence)
- Fourth column: routing switches
- Fifth column: duplicate check
- Sixth column: sheet operations
- Far right: alert nodes and Final Success Logger

Lane arrangement top to bottom:

- Top lane (red): Emergency path — `Set – Build Emergency Alert` → `Slack – Send Emergency Alert`
- Second lane (orange): Urgent path — `Set – Build Urgent Alert` → `Slack – Send Urgent Alert`
- Third lane (green): Routine path — `Set – Routine Queue Logger`
- Bottom lane (gray): Low confidence / Manual Review path

Keep the daily summary mini-flow completely separate on the canvas, placed below the main flow with a visible gap.

Add Sticky Notes to label each section:

- `"⚡ ENTRY POINT — Webhook + Test Trigger"`
- `"🧠 AI CLASSIFICATION ENGINE"`
- `"🔀 URGENCY + TRADE ROUTING"`
- `"📊 GOOGLE SHEETS LOGGING"`
- `"🚨 EMERGENCY ALERTS"`
- `"📋 DAILY OPS SUMMARY"`
- `"❌ LOW CONFIDENCE → MANUAL REVIEW"`

---

## STEP 36: Testing procedure

Build one branch at a time and test it before moving on.

### Test 1: Emergency Electrical (Robert Miller)

Send this payload to your test webhook URL:

```json
{
  "SubmissionTimestamp": "2024-10-20T09:15:00",
  "CustomerName": "Robert Miller",
  "ContactPhone": "555-001-9102",
  "PropertyAddress": "001 Oak Lane, Metropolis",
  "ServiceDescription": "My kitchen outlet is sparking and smells like burning plastic."
}
```

Expected result:

1. Webhook receives payload.
2. Normalize node cleans fields.
3. Validation passes — all four fields are present.
4. Ticket ID is generated — format `FP-20241020-XXXX`.
5. AI returns `Electrical`, `Emergency`, confidence above 0.90.
6. Parse node extracts all four AI fields cleanly.
7. Confidence check passes.
8. Urgency switch routes to Emergency output.
9. Trade switch routes to Electrical output.
10. Google Sheets search finds no duplicate.
11. New row appended to Dispatch Log.
12. Emergency alert built and sent to `#flashpoint-emergency`.
13. Final Success Logger marks complete.

### Test 2: HVAC Routine (Sarah Jenkins)

```json
{
  "SubmissionTimestamp": "2024-10-20T10:30:00",
  "CustomerName": "Sarah Jenkins",
  "ContactPhone": "555-002-9102",
  "PropertyAddress": "002 Oak Lane, Metropolis",
  "ServiceDescription": "I need a quote for installing a new smart thermostat."
}
```

Expected result:

1. AI returns `HVAC`, `Routine`, confidence above 0.90.
2. Routes to Routine output.
3. No Slack alert sent.
4. Row appended with `Assigned Queue = Morning Schedule`.

### Test 3: Plumbing Emergency (Thomas Lewis)

```json
{
  "SubmissionTimestamp": "2024-10-22T10:45:00",
  "CustomerName": "Thomas Lewis",
  "ContactPhone": "555-017-9102",
  "PropertyAddress": "017 Oak Lane, Metropolis",
  "ServiceDescription": "Basement is flooding because the sump pump failed during the storm."
}
```

Expected result:

1. AI returns `Plumbing`, `Emergency`, confidence above 0.95.
2. Emergency alert sent to `#flashpoint-emergency`.

### Test 4: Duplicate submission

Send the same Robert Miller payload a second time on the same day.

Expected result:

1. Duplicate check finds the existing row by phone + address + date.
2. Workflow routes to `Google Sheets – Update Existing Row` instead of Append.
3. The existing row Status field changes to `Updated`.
4. No second row created.

### Test 5: Missing field (validation failure)

```json
{
  "SubmissionTimestamp": "2024-10-20T09:15:00",
  "CustomerName": "John Doe",
  "ContactPhone": "555-999-0000"
}
```

Expected result:

1. `IF – Required Fields Present?` evaluates FALSE — PropertyAddress and ServiceDescription are missing.
2. `Set – Flag Validation Failure` runs.
3. Workflow stops there with Status = `Validation Failed`.

### Test 6: Low confidence scenario

Temporarily change the confidence threshold in the AI prompt to force a low-confidence result, or submit a very vague description:

```json
{
  "SubmissionTimestamp": "2024-10-20T09:00:00",
  "CustomerName": "Test User",
  "ContactPhone": "555-000-0000",
  "PropertyAddress": "000 Test Lane, Metropolis",
  "ServiceDescription": "Something is wrong with my house."
}
```

Expected result:

1. AI returns low confidence (below 0.70).
2. `IF – Confidence Acceptable?` routes to FALSE.
3. Manual review alert sent to `#flashpoint-admin`.
4. Row logged in Manual Review sheet with `Reviewed = No`.

### Test 7: Daily summary

Trigger the `Schedule Trigger – Daily Summary` manually by clicking Execute in the editor.

Expected result:

1. Google Sheets reads today's rows.
2. Summary message is built with counts by urgency and trade.
3. Message posted to `#flashpoint-ops`.

---

## STEP 37: Troubleshooting

### Webhook not receiving data

- Confirm the workflow is in test mode and the `Test URL` is active. Live URL only works when the workflow is activated.
- Check the HTTP method. The webhook expects `POST`. Sending `GET` will not work.
- Confirm the Content-Type header is `application/json`.

### AI node returns invalid JSON

- Check the `AI_ParseSuccess` field. If it is `false`, the `AI_RawResponse` field shows you exactly what the AI returned.
- Common cause: the AI returned markdown code fences around the JSON. The parse node strips these but if the format is unusual, check `AI_ParseError`.
- Fix: add `Return only raw JSON without markdown or backticks.` as a final line in the system prompt.

### Confidence always below threshold

- Check that the prompt instructs the AI to use `0.95` for clear-cut cases. Some AI models default to conservative confidence scores.
- Lower the threshold temporarily to `0.60` during testing, then restore to `0.70` for production.

### Google Sheets returns auth error

- Re-authorize the Google Sheets credential inside n8n. OAuth tokens expire.
- Make sure the Google account that authorized n8n has edit access to the spreadsheet.
- Confirm the spreadsheet ID is the correct one. Copy it directly from the spreadsheet URL.

### Duplicate rows appearing in the sheet

- Confirm the `Code – Check Duplicate Match` node is referencing the correct field names. Google Sheets returns column headers with spaces exactly as written. `Contact Phone` is different from `ContactPhone`.
- Add a console log inside the duplicate check code to inspect what `sheetRows` contains.

### Slack node fails

- Verify the channel ID is correct. Channel IDs start with `C`. Channel names do not work in the API.
- Confirm the Slack OAuth credential has `chat:write` scope.
- If using incoming webhooks instead of OAuth, replace the Slack node with an `HTTP Request` node posting JSON to the webhook URL.

### Expressions return undefined

- Use the n8n expression editor (click the lightning bolt icon) and inspect the item shape at each node.
- If a field shows `undefined`, check the upstream Set or Code node to confirm the field was actually assigned.
- Webhook payloads nest fields under `body` in n8n. The Normalize node (Step 7) handles this but if you add nodes before the Normalize step, use `$json.body.FieldName` not `$json.FieldName`.

### Wrong trade or urgency from AI

- The AI makes mistakes on ambiguous descriptions. This is expected. The confidence check catches low-certainty results.
- For persistent misclassifications, add more examples to the prompt in STEP 10 using real descriptions from the starter CSV.
- The label columns in `flashpoint_service_leads_starter.csv` show the correct expected classification for each row. Use them to spot-check AI accuracy during testing.

---

## STEP 38: Using the starter CSV for batch testing

You have two methods to test with the full 30-row CSV.

### Option A: Code node batch replay

The `Code – Read CSV Starter Leads` node (Step 5) already has six sample rows hardcoded. To test all 30 rows:

1. Open the code node.
2. Replace the `rows` array with all 30 rows from the CSV file.
3. Click `Execute Node`.
4. The workflow processes each row as a separate item.

Note: running all 30 rows will make 30 OpenAI API calls. This takes approximately 30–90 seconds and costs a small amount. During development, test with 3–5 rows at a time.

### Option B: Google Sheets row replay

1. Import the CSV into a new tab in your spreadsheet called `Test Leads`.
2. Add a `Google Sheets – Read Test Leads` node triggered by `Manual Trigger – Test Mode` that reads from this tab.
3. Connect it to the `Merge – Combine Trigger Sources` node.
4. When you click Execute, the workflow pulls the rows from the sheet and processes them one at a time.

This option is useful for re-running specific rows or adding test cases without editing code.

---

## STEP 39: Deliverables checklist

When delivering this workflow to a client, include:

- Exported workflow JSON (click the three-dot menu → Download)
- Screenshot of the full canvas showing all four urgency lanes and the daily summary path
- Screenshot of a successful Emergency execution with green nodes
- Screenshot of the Dispatch Log Google Sheet with at least 5 rows populated
- Screenshot of the Emergency alert in `#flashpoint-emergency` Slack channel
- Screenshot of the Daily Ops Summary in `#flashpoint-ops`
- Screenshot of the Manual Review sheet with at least one low-confidence row

---

## STEP 40: Activation checklist

Before activating the workflow for live use:

1. Google Sheets headers exactly match the field names in this guide. One typo breaks every append.
2. All three Slack channel IDs are valid and the bot is a member of each channel.
3. The AI API credential is valid and has remaining quota.
4. The error workflow is assigned in the main workflow settings.
5. At least one of each urgency type (Emergency, Urgent, Routine) has been successfully tested in the editor.
6. The daily summary trigger time matches the Team Lead's morning schedule.
7. Activate the error workflow first.
8. Then activate the main workflow.

---

## Final result

Once built, this system gives Flashpoint Electrical & HVAC a production-ready dispatch triage layer:

- Every incoming service request is validated, given a unique ticket ID, and sent to AI within seconds of form submission.
- Emergency jobs trigger an immediate Slack alert to `#flashpoint-emergency` so dispatchers act without delay.
- Urgent jobs land in `#flashpoint-ops` for same-day scheduling.
- Routine jobs are silently logged and appear in the morning summary.
- Duplicate submissions from the same customer on the same day update the existing ticket instead of creating noise.
- Ambiguous AI responses go to a manual review queue so the Team Lead never acts on uncertain data.
- Execution failures are caught by the error workflow and logged before anyone has to notice they happened.
- The Team Lead starts every morning with a pre-sorted ops summary instead of a raw inbox.

# Architectural Lead Triage AI Assistant for Foundations & Frame
## Complete n8n Workflow Implementation Guide

> **Version:** 1.0  
> **Target Platform:** n8n (self-hosted or cloud)  
> **Skill Level:** Beginner-friendly (every step explained)  
> **Total Nodes:** ~30+ nodes across all layers

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Prerequisites & Setup](#2-prerequisites--setup)
3. [Workflow Architecture Summary](#3-workflow-architecture-summary)
4. [Layer 1 — Webhook Trigger](#4-layer-1--webhook-trigger)
5. [Layer 2 — Preprocessing](#5-layer-2--preprocessing)
6. [Layer 3 — AI Parsing](#6-layer-3--ai-parsing)
7. [Layer 4 — Parsing Validation](#7-layer-4--parsing-validation)
8. [Layer 5 — Google Sheets Integration](#8-layer-5--google-sheets-integration)
9. [Layer 6 — Style Matching Logic](#9-layer-6--style-matching-logic)
10. [Layer 7 — Budget Evaluation](#10-layer-7--budget-evaluation)
11. [Layer 8 — Lead Classification](#11-layer-8--lead-classification)
12. [Layer 9 — AI Summary Generator](#12-layer-9--ai-summary-generator)
13. [Layer 10 — Slack Integration](#13-layer-10--slack-integration)
14. [Layer 11 — Airtable Integration](#14-layer-11--airtable-integration)
15. [Layer 12 — Error Handling](#15-layer-12--error-handling)
16. [Layer 13 — Logging & Observability](#16-layer-13--logging--observability)
17. [Node Connection Map](#17-node-connection-map)
18. [Testing Guide](#18-testing-guide)
19. [Deliverables](#19-deliverables)
20. [Proof of Execution Guide](#20-proof-of-execution-guide)
21. [Acceptance Validation Checklist](#21-acceptance-validation-checklist)

---

## 1. Project Overview

### What This Workflow Does

The **Architectural Lead Triage AI Assistant** is a fully automated pipeline that:

1. Receives a raw text inquiry from a potential architecture client via a website form
2. Cleans and normalizes the raw message
3. Uses AI (OpenAI/Anthropic) to extract structured lead data (name, budget, architectural style)
4. Validates that each extracted field is present and properly typed
5. Cross-references the requested architectural style against a Google Sheet of "priority styles"
6. Evaluates the lead's budget threshold
7. Classifies the lead as **Hot** or **General**
8. Sends a formatted Slack alert for hot leads and logs general leads to Airtable
9. Handles all errors gracefully with manual review routing
10. Logs every execution with timestamps and metadata

### Business Context

**Company:** Foundations & Frame (architectural design firm)  
**Use Case:** High-volume lead qualification without manual effort  
**Outcome:** Sales team only sees pre-qualified, AI-summarized leads

### Lead Classification Rules

| Lead Type | Budget | Priority Style | Action |
|-----------|--------|---------------|--------|
| 🔥 Hot Lead | > $1,500,000 | Yes | Slack alert to sales team |
| 📋 General Lead | Any other combination | Any | Logged to Airtable nurture base |
| ⚠️ Error / Incomplete | Missing fields | N/A | Slack alert to manual review channel |

---

## 2. Prerequisites & Setup

### Required Accounts & Services

Before building this workflow, ensure you have:

- [ ] **n8n account** — [n8n.io](https://n8n.io) (cloud) or self-hosted instance
- [ ] **OpenAI API key** — [platform.openai.com](https://platform.openai.com) (for AI parsing)
- [ ] **Google account** — with a Google Sheet created (details in Layer 5)
- [ ] **Slack workspace** — with two channels created: `#hot-leads` and `#manual-review`
- [ ] **Airtable account** — with a base named `Nurture` created

### n8n Credential Setup

#### OpenAI Credentials
1. In n8n, go to **Settings → Credentials → Add Credential**
2. Search for **OpenAI**
3. Paste your API key in the `API Key` field
4. Name it: `OpenAI - Foundations & Frame`
5. Click **Save**

#### Google Sheets Credentials
1. Go to **Settings → Credentials → Add Credential**
2. Search for **Google Sheets OAuth2**
3. Follow the OAuth flow to connect your Google account
4. Name it: `Google Sheets - F&F`

#### Slack Credentials
1. Go to **Settings → Credentials → Add Credential**
2. Search for **Slack**
3. Use the OAuth2 flow or a Bot Token
4. Required scopes: `chat:write`, `channels:read`
5. Name it: `Slack - F&F Bot`

#### Airtable Credentials
1. Go to **Settings → Credentials → Add Credential**
2. Search for **Airtable**
3. Paste your Personal Access Token from [airtable.com/create/tokens](https://airtable.com/create/tokens)
4. Name it: `Airtable - Nurture Base`

---

## 3. Workflow Architecture Summary

```
[Webhook] → [Normalize Payload] → [Clean Text] → [Validate Non-Empty]
     ↓
[AI Parsing - Extract Fields]
     ↓
[Check Name] → [Check Budget] → [Check Style] → [Merge Validations]
     ↓                                                    ↓
[Missing Fields?]                           [Google Sheets Read]
     ↓ YES                                               ↓
[Slack: Manual Review]              [Normalize Styles - Function]
                                                         ↓
                                          [Style Exists in Sheet?]
                                                         ↓
                                          [Budget > 1.5M? IF Node]
                                                         ↓
                                       [Merge Conditions - Hot Check]
                                                         ↓
                               ┌────────────────────────┤
                               ↓                        ↓
                         [Hot Lead]               [General Lead]
                               ↓                        ↓
                    [AI Summary Generator]    [Airtable: Create Record]
                               ↓
                    [Format Slack Message]
                               ↓
                    [Slack: Hot Leads Channel]
                               ↓
                    [Add Timestamp] → [Attach Metadata] → [Log to Sheets]
```

---

## 4. Layer 1 — Webhook Trigger

### Node 1: Webhook

**Purpose:** This is the entry point of the entire workflow. It listens for incoming HTTP POST requests from your website contact form and passes the raw data into n8n.

**Node Type:** `Webhook`

#### Step-by-Step Configuration

1. In your n8n canvas, click the **+** button to add a new node
2. Search for **"Webhook"** and select it
3. Configure the following fields:

| Field | Value |
|-------|-------|
| **HTTP Method** | `POST` |
| **Path** | `lead-intake` |
| **Response Mode** | `Respond to Webhook` |
| **Response Code** | `200` |
| **Response Data** | `First Entry JSON` |

4. Click **Execute Node** to activate test mode
5. Copy the **Test URL** shown (e.g., `https://your-n8n.app/webhook-test/lead-intake`)

#### Example Webhook URL
```
https://your-n8n.app/webhook/lead-intake
```

#### Example Incoming Payload (from website form)
```json
{
  "message": "Hi, my name is Sarah Chen. I'm interested in a modern minimalist home build in Austin. My budget is around 2.2 million dollars. I'd love to discuss timelines.",
  "source": "website_contact_form",
  "submitted_at": "2024-11-15T14:32:00Z"
}
```

#### Example Output (what passes to next node)
```json
{
  "body": {
    "message": "Hi, my name is Sarah Chen. I'm interested in a modern minimalist home build in Austin. My budget is around 2.2 million dollars. I'd love to discuss timelines.",
    "source": "website_contact_form",
    "submitted_at": "2024-11-15T14:32:00Z"
  },
  "headers": {
    "content-type": "application/json"
  }
}
```

> **Note:** The webhook will produce a "Waiting for test event" state. Use a tool like Postman or curl to send a test POST request.

---

## 5. Layer 2 — Preprocessing

### Node 2: Set Node — Normalize Incoming Payload

**Purpose:** Extracts and renames the fields from the raw webhook payload into clean, predictable variable names used throughout the rest of the workflow.

**Node Type:** `Set`

#### Step-by-Step Configuration

1. Add a **Set** node after the Webhook node
2. Name it: `Normalize Payload`
3. Set **Keep Only Set** to `ON` (this removes all other fields from output)
4. Add the following fields:

| Name | Type | Value (Expression) |
|------|------|--------------------|
| `raw_message` | String | `{{ $json.body.message }}` |
| `source` | String | `{{ $json.body.source \|\| "unknown" }}` |
| `submitted_at` | String | `{{ $json.body.submitted_at \|\| $now.toISO() }}` |
| `workflow_id` | String | `{{ $workflow.id }}` |

5. Connect: **Webhook → Normalize Payload**

#### Example Output
```json
{
  "raw_message": "Hi, my name is Sarah Chen. I'm interested in a modern minimalist home build in Austin. My budget is around 2.2 million dollars. I'd love to discuss timelines.",
  "source": "website_contact_form",
  "submitted_at": "2024-11-15T14:32:00Z",
  "workflow_id": "wf_abc123"
}
```

---

### Node 3: Function Node — Clean Text

**Purpose:** Removes noise from the raw message — extra whitespace, special characters, HTML tags, and excessive punctuation — so the AI receives clean, parseable text.

**Node Type:** `Code` (Function)

#### Step-by-Step Configuration

1. Add a **Code** node after `Normalize Payload`
2. Name it: `Clean Text`
3. Set **Language** to `JavaScript`
4. Paste the following code:

```javascript
// Get the raw message from the previous node
const rawMessage = $input.first().json.raw_message || "";

// Step 1: Remove any HTML tags (in case form submitted HTML)
let cleaned = rawMessage.replace(/<[^>]*>/g, "");

// Step 2: Remove excessive whitespace (tabs, multiple spaces, newlines)
cleaned = cleaned.replace(/\s+/g, " ");

// Step 3: Trim leading and trailing whitespace
cleaned = cleaned.trim();

// Step 4: Remove special noise characters but keep punctuation useful for parsing
cleaned = cleaned.replace(/[^\w\s.,!?$'-]/g, "");

// Step 5: Normalize currency expressions
cleaned = cleaned.replace(/\$\s+/g, "$");

// Step 6: Calculate a character count for downstream validation
const charCount = cleaned.length;
const wordCount = cleaned.split(" ").filter(w => w.length > 0).length;

// Return all previous fields PLUS the cleaned message
return [{
  json: {
    ...$input.first().json,
    cleaned_message: cleaned,
    char_count: charCount,
    word_count: wordCount
  }
}];
```

5. Connect: **Normalize Payload → Clean Text**

#### Example Output
```json
{
  "raw_message": "Hi, my name is Sarah Chen...",
  "cleaned_message": "Hi, my name is Sarah Chen. I'm interested in a modern minimalist home build in Austin. My budget is around 2.2 million dollars. I'd love to discuss timelines.",
  "char_count": 168,
  "word_count": 31,
  "source": "website_contact_form",
  "submitted_at": "2024-11-15T14:32:00Z"
}
```

---

### Node 4: IF Node — Validate Non-Empty Input

**Purpose:** Prevents empty or extremely short messages from being processed by AI (which wastes API credits and generates useless outputs).

**Node Type:** `IF`

#### Step-by-Step Configuration

1. Add an **IF** node after `Clean Text`
2. Name it: `Validate Non-Empty`
3. Configure conditions:

**Condition 1:**
| Field | Value |
|-------|-------|
| Value 1 | `{{ $json.char_count }}` |
| Operation | `Larger Than` |
| Value 2 | `10` |

**Condition 2:**
| Field | Value |
|-------|-------|
| Value 1 | `{{ $json.cleaned_message }}` |
| Operation | `Is Not Empty` |

4. Set **Combine Conditions** to `AND`
5. Connect: **Clean Text → Validate Non-Empty**

#### Routing:
- **True branch** → Continue to AI Parsing
- **False branch** → Route to Error Handler (Node 25)

---

## 6. Layer 3 — AI Parsing

### Node 5: OpenAI Node — Extract Lead Fields

**Purpose:** Sends the cleaned message to GPT-4 with a carefully engineered prompt that extracts the three key fields (name, budget, style) as structured JSON output.

**Node Type:** `OpenAI` (or `HTTP Request` to OpenAI API)

#### Step-by-Step Configuration

1. Add an **OpenAI** node after `Validate Non-Empty` (True branch)
2. Name it: `AI Parse Lead Fields`
3. Configure:

| Field | Value |
|-------|-------|
| **Credential** | `OpenAI - Foundations & Frame` |
| **Resource** | `Text` |
| **Operation** | `Message a Model` |
| **Model** | `gpt-4o` |
| **Max Tokens** | `500` |
| **Temperature** | `0` (deterministic output) |

#### System Prompt
```
You are a lead data extraction assistant for an architectural design firm called Foundations & Frame. 

Your job is to read a raw client inquiry message and extract exactly three pieces of structured information.

RULES:
1. Always respond with ONLY valid JSON — no markdown, no explanation, no preamble
2. If you cannot find a field, set it to null
3. Budget must ALWAYS be a plain integer (e.g., 2200000), never a string or formatted number
4. Architectural style must be a short, clean label (e.g., "Modern Minimalist", "Mid-Century Modern", "Industrial")
5. Name should be the full name of the person who sent the inquiry
6. Include a confidence score (0.0 to 1.0) representing how confident you are in the extraction

RESPOND ONLY WITH THIS JSON STRUCTURE:
{
  "name": "<string or null>",
  "budget": <integer or null>,
  "architectural_style": "<string or null>",
  "confidence": <float 0.0-1.0>,
  "intent_summary": "<one sentence describing what the client wants>"
}
```

#### User Prompt Expression
```
Analyze the following client inquiry and extract the structured data:

---
{{ $json.cleaned_message }}
---

Remember: respond only with JSON. No markdown code blocks.
```

4. In the node configuration, add the **System Prompt** in the "System Message" field
5. Add the **User Prompt** expression in the "User Message" field
6. Connect: **Validate Non-Empty (True) → AI Parse Lead Fields**

#### Example AI Response (raw)
```json
{
  "name": "Sarah Chen",
  "budget": 2200000,
  "architectural_style": "Modern Minimalist",
  "confidence": 0.95,
  "intent_summary": "Client seeks a modern minimalist custom home build in Austin with a $2.2M budget."
}
```

---

### Node 6: Code Node — Parse AI JSON Response

**Purpose:** Safely parses the AI's JSON string response into a usable JavaScript object, with error catching if the JSON is malformed.

**Node Type:** `Code`

#### Step-by-Step Configuration

1. Add a **Code** node after `AI Parse Lead Fields`
2. Name it: `Parse AI JSON Response`
3. Paste the following code:

```javascript
const aiResponse = $input.first().json.message?.content || 
                   $input.first().json.choices?.[0]?.message?.content || 
                   $input.first().json.text || "";

let parsed = {};
let parseError = null;

try {
  // Remove any accidental markdown code fences
  const cleanJson = aiResponse
    .replace(/```json/gi, "")
    .replace(/```/g, "")
    .trim();
  
  parsed = JSON.parse(cleanJson);
} catch (e) {
  parseError = e.message;
  parsed = {
    name: null,
    budget: null,
    architectural_style: null,
    confidence: 0,
    intent_summary: null
  };
}

return [{
  json: {
    ...$input.first().json,
    parsed_name: parsed.name,
    parsed_budget: parsed.budget,
    parsed_style: parsed.architectural_style,
    parsed_confidence: parsed.confidence || 0,
    parsed_intent: parsed.intent_summary,
    parse_error: parseError,
    ai_raw_response: aiResponse
  }
}];
```

4. Connect: **AI Parse Lead Fields → Parse AI JSON Response**

#### Example Output
```json
{
  "cleaned_message": "Hi, my name is Sarah Chen...",
  "parsed_name": "Sarah Chen",
  "parsed_budget": 2200000,
  "parsed_style": "Modern Minimalist",
  "parsed_confidence": 0.95,
  "parsed_intent": "Client seeks a modern minimalist custom home build in Austin with a $2.2M budget.",
  "parse_error": null
}
```

---

## 7. Layer 4 — Parsing Validation

### Node 7: IF Node — Check if Name Exists

**Purpose:** Verifies that the AI successfully extracted a client name. A lead without a name cannot be properly routed or contacted.

**Node Type:** `IF`

#### Step-by-Step Configuration

1. Add an **IF** node after `Parse AI JSON Response`
2. Name it: `Check Name Exists`
3. Configure:

| Field | Value |
|-------|-------|
| Value 1 | `{{ $json.parsed_name }}` |
| Operation | `Is Not Empty` |

4. Connect: **Parse AI JSON Response → Check Name Exists**

#### Routing:
- **True** → `Check Budget Is Numeric`
- **False** → Set a flag (handled in Merge node)

---

### Node 8: Set Node — Flag Missing Name

**Purpose:** When the name is missing, set a structured error flag that can be caught downstream.

**Node Type:** `Set`

1. Add a **Set** node on the **False** branch of `Check Name Exists`
2. Name it: `Flag Missing Name`
3. Add field:

| Name | Type | Value |
|------|------|-------|
| `validation_error` | String | `Missing client name` |
| `has_name` | Boolean | `false` |

4. Connect: **Check Name Exists (False) → Flag Missing Name**

---

### Node 9: Set Node — Flag Name Valid

**Purpose:** When name is found, mark it as valid so the merge node can proceed.

**Node Type:** `Set`

1. Add a **Set** node on the **True** branch of `Check Name Exists`
2. Name it: `Flag Name Valid`
3. Add field:

| Name | Type | Value |
|------|------|-------|
| `has_name` | Boolean | `true` |

4. Connect: **Check Name Exists (True) → Flag Name Valid**

---

### Node 10: IF Node — Check Budget Is Numeric

**Purpose:** Ensures the extracted budget is an actual number and not null or a string. Budget must be parseable as an integer for financial routing logic.

**Node Type:** `IF`

#### Step-by-Step Configuration

1. Add an **IF** node
2. Name it: `Check Budget Is Numeric`
3. Configure TWO conditions with `AND`:

**Condition 1:**
| Field | Value |
|-------|-------|
| Value 1 | `{{ $json.parsed_budget }}` |
| Operation | `Is Not Empty` |

**Condition 2:**
| Field | Value |
|-------|-------|
| Value 1 | `{{ typeof $json.parsed_budget }}` |
| Operation | `Equal` |
| Value 2 | `number` |

4. Connect both: **Flag Name Valid → Check Budget Is Numeric**  
   (Also connect **Flag Missing Name → Check Budget Is Numeric** so all items pass through)

#### Routing:
- **True** → `Flag Budget Valid`
- **False** → `Flag Missing Budget`

---

### Node 11: Set Node — Flag Missing Budget

**Node Type:** `Set`  
**Name:** `Flag Missing Budget`

| Name | Type | Value |
|------|------|-------|
| `validation_error` | String | `{{ $json.validation_error + " | Missing or invalid budget" }}` |
| `has_budget` | Boolean | `false` |

---

### Node 12: Set Node — Flag Budget Valid

**Node Type:** `Set`  
**Name:** `Flag Budget Valid`

| Name | Type | Value |
|------|------|-------|
| `has_budget` | Boolean | `true` |

---

### Node 13: IF Node — Check Style Exists

**Purpose:** Validates that an architectural style was identified. Without a style, we cannot match against the priority styles list.

**Node Type:** `IF`  
**Name:** `Check Style Exists`

**Condition:**
| Field | Value |
|-------|-------|
| Value 1 | `{{ $json.parsed_style }}` |
| Operation | `Is Not Empty` |

Connect: Both budget flag nodes → `Check Style Exists`

#### Routing:
- **True** → `Flag Style Valid`
- **False** → `Flag Missing Style`

---

### Node 14: Set Node — Flag Missing Style

**Node Type:** `Set`  
**Name:** `Flag Missing Style`

| Name | Type | Value |
|------|------|-------|
| `validation_error` | String | `{{ ($json.validation_error || "") + " | Missing architectural style" }}` |
| `has_style` | Boolean | `false` |

---

### Node 15: Set Node — Flag Style Valid

**Node Type:** `Set`  
**Name:** `Flag Style Valid`

| Name | Type | Value |
|------|------|-------|
| `has_style` | Boolean | `true` |

---

### Node 16: Merge Node — Combine Validation Results

**Purpose:** Rejoins both the valid and invalid branches into a single flow after validation so the workflow can make a unified decision about whether to proceed or route to error handling.

**Node Type:** `Merge`  
**Name:** `Merge Validation Results`

#### Configuration:
| Field | Value |
|-------|-------|
| **Mode** | `Append` |
| **Number of Inputs** | `2` |

1. Connect: **Flag Style Valid → Merge Validation Results (Input 1)**
2. Connect: **Flag Missing Style → Merge Validation Results (Input 2)**

---

### Node 17: IF Node — Check All Fields Valid

**Purpose:** Final validation check. Only proceeds to lead classification if ALL three fields (name, budget, style) were successfully extracted.

**Node Type:** `IF`  
**Name:** `All Fields Valid?`

**Conditions (AND):**

| Condition | Value 1 | Op | Value 2 |
|-----------|---------|-----|---------|
| Has name | `{{ $json.has_name }}` | Equal | `true` |
| Has budget | `{{ $json.has_budget }}` | Equal | `true` |
| Has style | `{{ $json.has_style }}` | Equal | `true` |

#### Routing:
- **True** → Continue to Google Sheets
- **False** → Route to Slack Manual Review (Node 30)

---

## 8. Layer 5 — Google Sheets Integration

### Google Sheet Setup (Do This First)

Before configuring this node, create the following Google Sheet:

**Sheet Name:** `Priority Styles`  
**Spreadsheet Name:** `Foundations & Frame - Lead Config`

| Column A | Column B | Column C |
|----------|----------|----------|
| **Style Name** | **Priority Level** | **Notes** |
| Modern Minimalist | High | Flagship style |
| Japandi | High | High-margin projects |
| Mid-Century Modern | High | Premium clients |
| Biophilic Design | High | Emerging trend |
| Scandinavian | Medium | Growing segment |
| Industrial | Medium | Commercial crossover |
| Traditional | Low | Standard tier |
| Craftsman | Low | Standard tier |

### Node 18: Google Sheets Node — Read Priority Styles

**Purpose:** Reads the current list of priority architectural styles from your Google Sheet so the workflow can check whether the client's requested style is a business priority.

**Node Type:** `Google Sheets`  
**Name:** `Read Priority Styles`

#### Step-by-Step Configuration

1. Add a **Google Sheets** node
2. Name it: `Read Priority Styles`
3. Configure:

| Field | Value |
|-------|-------|
| **Credential** | `Google Sheets - F&F` |
| **Resource** | `Sheet` |
| **Operation** | `Read Rows` |
| **Document ID** | `<your spreadsheet ID from URL>` |
| **Sheet Name** | `Priority Styles` |
| **Data Start Row** | `2` (skip header row) |
| **Return All** | `ON` |

> **Finding your Spreadsheet ID:** In your Google Sheets URL, it's the long string between `/d/` and `/edit`. Example: `https://docs.google.com/spreadsheets/d/1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OgVE2upms/edit` → ID is `1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OgVE2upms`

4. Connect: **All Fields Valid? (True) → Read Priority Styles**

#### Example Output (array of rows)
```json
[
  { "Style Name": "Modern Minimalist", "Priority Level": "High", "Notes": "Flagship style" },
  { "Style Name": "Japandi", "Priority Level": "High", "Notes": "High-margin projects" },
  { "Style Name": "Mid-Century Modern", "Priority Level": "High", "Notes": "Premium clients" }
]
```

---

## 9. Layer 6 — Style Matching Logic

### Node 19: Code Node — Normalize Styles for Comparison

**Purpose:** Takes the list of styles from Google Sheets and the client's requested style, then normalizes both to lowercase with no extra spaces for a reliable string comparison.

**Node Type:** `Code`  
**Name:** `Normalize Styles`

```javascript
// Get all items (Google Sheets returns multiple items, one per row)
const allItems = $input.all();

// The client's parsed style comes from earlier in the workflow
// We need to get it from the first item that has it (it was set upstream)
// In n8n, when a node receives multiple items from Sheets, the JSON context
// from previous nodes can be accessed differently.
// We will use the first item's full JSON which should contain all fields.

const clientStyle = allItems[0]?.json?.parsed_style || "";
const normalizedClientStyle = clientStyle.toLowerCase().trim();

// Build a set of all priority style names from the sheet
const priorityStyles = allItems
  .map(item => (item.json["Style Name"] || "").toLowerCase().trim())
  .filter(s => s.length > 0);

// Check if client style matches any priority style
const isPriorityStyle = priorityStyles.some(style => {
  // Check for exact match OR if one contains the other
  return style === normalizedClientStyle || 
         style.includes(normalizedClientStyle) || 
         normalizedClientStyle.includes(style);
});

// Pass through only a single consolidated item
return [{
  json: {
    ...allItems[0].json,
    normalized_client_style: normalizedClientStyle,
    priority_styles_list: priorityStyles,
    isPriorityStyle: isPriorityStyle
  }
}];
```

4. Connect: **Read Priority Styles → Normalize Styles**

#### Example Output
```json
{
  "parsed_name": "Sarah Chen",
  "parsed_budget": 2200000,
  "parsed_style": "Modern Minimalist",
  "normalized_client_style": "modern minimalist",
  "priority_styles_list": ["modern minimalist", "japandi", "mid-century modern"],
  "isPriorityStyle": true
}
```

---

### Node 20: IF Node — Is Priority Style?

**Purpose:** Creates a boolean branch based on whether the client's style is in the priority list.

**Node Type:** `IF`  
**Name:** `Is Priority Style?`

**Condition:**
| Field | Value |
|-------|-------|
| Value 1 | `{{ $json.isPriorityStyle }}` |
| Operation | `Equal` |
| Value 2 | `true` |

Connect: **Normalize Styles → Is Priority Style?**

---

### Node 21: Set Node — Mark Priority True

**Node Type:** `Set`  
**Name:** `Mark Priority: True`

| Name | Type | Value |
|------|------|-------|
| `style_priority_flag` | Boolean | `true` |

Connect: **Is Priority Style? (True) → Mark Priority: True**

---

### Node 22: Set Node — Mark Priority False

**Node Type:** `Set`  
**Name:** `Mark Priority: False`

| Name | Type | Value |
|------|------|-------|
| `style_priority_flag` | Boolean | `false` |

Connect: **Is Priority Style? (False) → Mark Priority: False**

---

## 10. Layer 7 — Budget Evaluation

### Node 23: Merge Node — Rejoin After Style Check

**Purpose:** Merge the priority-true and priority-false branches back together before budget evaluation.

**Node Type:** `Merge`  
**Name:** `Merge After Style Check`

| Field | Value |
|-------|-------|
| **Mode** | `Append` |
| **Number of Inputs** | `2` |

1. Connect: **Mark Priority: True → Merge After Style Check (Input 1)**
2. Connect: **Mark Priority: False → Merge After Style Check (Input 2)**

---

### Node 24: IF Node — Budget > 1,500,000

**Purpose:** Evaluates whether the client's extracted budget meets the minimum threshold for hot lead classification.

**Node Type:** `IF`  
**Name:** `Budget > 1.5M?`

**Condition:**
| Field | Value |
|-------|-------|
| Value 1 | `{{ $json.parsed_budget }}` |
| Operation | `Larger Than` |
| Value 2 | `1500000` |

Connect: **Merge After Style Check → Budget > 1.5M?**

---

### Node 25: Set Node — Flag High Budget

**Node Type:** `Set`  
**Name:** `Flag: High Budget`

| Name | Type | Value |
|------|------|-------|
| `is_high_budget` | Boolean | `true` |

Connect: **Budget > 1.5M? (True) → Flag: High Budget**

---

### Node 26: Set Node — Flag Standard Budget

**Node Type:** `Set`  
**Name:** `Flag: Standard Budget`

| Name | Type | Value |
|------|------|-------|
| `is_high_budget` | Boolean | `false` |

Connect: **Budget > 1.5M? (False) → Flag: Standard Budget**

---

## 11. Layer 8 — Lead Classification

### Node 27: Merge Node — Combine Budget Flags

**Purpose:** Rejoin high/standard budget branches before final classification.

**Node Type:** `Merge`  
**Name:** `Merge Budget Flags`

| Field | Value |
|-------|-------|
| **Mode** | `Append` |

1. Connect: **Flag: High Budget → Merge Budget Flags (Input 1)**
2. Connect: **Flag: Standard Budget → Merge Budget Flags (Input 2)**

---

### Node 28: IF Node — Classify Hot Lead

**Purpose:** The final classification gate. A lead is HOT only if BOTH conditions are true: budget exceeds $1.5M AND the style is a priority style.

**Node Type:** `IF`  
**Name:** `Is Hot Lead?`

**Conditions (AND):**

| Condition | Value 1 | Op | Value 2 |
|-----------|---------|-----|---------|
| High budget | `{{ $json.is_high_budget }}` | Equal | `true` |
| Priority style | `{{ $json.style_priority_flag }}` | Equal | `true` |

Connect: **Merge Budget Flags → Is Hot Lead?**

#### Routing:
- **True** → AI Summary Generator
- **False** → Airtable (General Lead)

---

## 12. Layer 9 — AI Summary Generator

### Node 29: OpenAI Node — Generate Professional Summary

**Purpose:** Generates a polished, human-readable summary of the lead that the sales team will see in Slack. This summary is professional, friendly, and action-oriented.

**Node Type:** `OpenAI`  
**Name:** `AI Generate Summary`

#### Configuration:

| Field | Value |
|-------|-------|
| **Model** | `gpt-4o` |
| **Temperature** | `0.7` (slightly creative) |
| **Max Tokens** | `300` |

#### System Prompt:
```
You are a professional client relations assistant for Foundations & Frame, a high-end architectural design firm.

Write a short, polished summary of an incoming lead for the sales team. The tone should be:
- Professional but warm
- Concise (3-4 sentences maximum)
- Action-oriented
- Use the client's first name

Format:
- Start with a greeting about the lead
- Mention their architectural vision
- State their budget
- Close with a recommended action

Do NOT use markdown headers or bullet points. Write as flowing prose.
```

#### User Prompt Expression:
```
Generate a lead summary for:

Client Name: {{ $json.parsed_name }}
Budget: ${{ $json.parsed_budget.toLocaleString() }}
Architectural Style: {{ $json.parsed_style }}
Client Intent: {{ $json.parsed_intent }}
Original Message: {{ $json.cleaned_message }}
```

Connect: **Is Hot Lead? (True) → AI Generate Summary**

---

### Node 30: Code Node — Extract Summary Text

**Purpose:** Safely extracts the summary text from the OpenAI response object.

**Node Type:** `Code`  
**Name:** `Extract Summary Text`

```javascript
const response = $input.first().json;
const summaryText = response.message?.content || 
                    response.choices?.[0]?.message?.content || 
                    response.text || 
                    "Summary unavailable.";

return [{
  json: {
    ...response,
    ai_summary: summaryText.trim()
  }
}];
```

Connect: **AI Generate Summary → Extract Summary Text**

---

## 13. Layer 10 — Slack Integration

### Node 31: Code Node — Format Slack Message

**Purpose:** Constructs a beautifully formatted Slack Block Kit message that is visually clear and actionable for the sales team.

**Node Type:** `Code`  
**Name:** `Format Hot Lead Slack Message`

```javascript
const data = $input.first().json;

// Format budget with commas
const formattedBudget = new Intl.NumberFormat('en-US', {
  style: 'currency',
  currency: 'USD',
  maximumFractionDigits: 0
}).format(data.parsed_budget);

const blocks = [
  {
    type: "header",
    text: {
      type: "plain_text",
      text: "🔥 Hot Lead Alert — Foundations & Frame",
      emoji: true
    }
  },
  {
    type: "section",
    text: {
      type: "mrkdwn",
      text: data.ai_summary
    }
  },
  {
    type: "divider"
  },
  {
    type: "section",
    fields: [
      {
        type: "mrkdwn",
        text: `*Client Name:*\n${data.parsed_name}`
      },
      {
        type: "mrkdwn",
        text: `*Budget:*\n${formattedBudget}`
      },
      {
        type: "mrkdwn",
        text: `*Style:*\n${data.parsed_style}`
      },
      {
        type: "mrkdwn",
        text: `*Lead Source:*\n${data.source || "Website Form"}`
      }
    ]
  },
  {
    type: "divider"
  },
  {
    type: "section",
    text: {
      type: "mrkdwn",
      text: `*Client Message:*\n_"${data.cleaned_message}"_`
    }
  },
  {
    type: "actions",
    elements: [
      {
        type: "button",
        text: {
          type: "plain_text",
          text: "📅 Schedule Discovery Call",
          emoji: true
        },
        style: "primary",
        url: "https://calendly.com/foundationsandframe/discovery-call"
      },
      {
        type: "button",
        text: {
          type: "plain_text",
          text: "📧 View Full Lead",
          emoji: true
        },
        url: `https://airtable.com/your-base-link`
      }
    ]
  },
  {
    type: "context",
    elements: [
      {
        type: "mrkdwn",
        text: `Submitted: ${data.submitted_at} | AI Confidence: ${Math.round((data.parsed_confidence || 0) * 100)}%`
      }
    ]
  }
];

return [{
  json: {
    ...data,
    slack_blocks: JSON.stringify(blocks),
    slack_channel: "#hot-leads"
  }
}];
```

Connect: **Extract Summary Text → Format Hot Lead Slack Message**

---

### Node 32: Slack Node — Send Hot Lead Alert

**Purpose:** Delivers the formatted message to the `#hot-leads` Slack channel.

**Node Type:** `Slack`  
**Name:** `Slack: Hot Lead Alert`

#### Configuration:

| Field | Value |
|-------|-------|
| **Credential** | `Slack - F&F Bot` |
| **Resource** | `Message` |
| **Operation** | `Send` |
| **Channel** | `#hot-leads` |
| **Message Type** | `Block Kit` |
| **Blocks** | `{{ $json.slack_blocks }}` |

Connect: **Format Hot Lead Slack Message → Slack: Hot Lead Alert**

---

### Node 33: Slack Node — Send Manual Review Alert

**Purpose:** When a lead fails validation or has missing fields, alert the `#manual-review` channel so a human can process it.

**Node Type:** `Slack`  
**Name:** `Slack: Manual Review Alert`

#### Configuration:

| Field | Value |
|-------|-------|
| **Credential** | `Slack - F&F Bot` |
| **Resource** | `Message` |
| **Operation** | `Send` |
| **Channel** | `#manual-review` |
| **Message Text** | See below |

**Message Text Expression:**
```
⚠️ *Manual Review Required — Incomplete Lead*

*Validation Error:* {{ $json.validation_error || "Unknown error" }}
*Raw Message:* 
_"{{ $json.cleaned_message || $json.raw_message }}"_

*AI Confidence:* {{ Math.round(($json.parsed_confidence || 0) * 100) }}%
*Submitted At:* {{ $json.submitted_at }}
*Source:* {{ $json.source }}

Please review this lead manually and update the CRM.
```

Connect the following nodes → `Slack: Manual Review Alert`:
- **All Fields Valid? (False)** 
- **Validate Non-Empty (False)**

---

## 14. Layer 11 — Airtable Integration

### Airtable Setup (Do This First)

Create a base in Airtable with these specs:

**Base Name:** `Nurture`  
**Table Name:** `Leads`

| Field Name | Airtable Type | Notes |
|------------|---------------|-------|
| Name | Single Line Text | Client full name |
| Budget | Number | Integer, no decimals |
| Architectural Style | Single Line Text | |
| Raw Message | Long Text | Original inquiry |
| Cleaned Message | Long Text | After preprocessing |
| Source | Single Line Text | e.g., website_contact_form |
| Timestamp | Date and Time | ISO format |
| AI Confidence | Percent | 0-100% |
| AI Intent Summary | Long Text | |
| Lead Status | Single Select | Options: New, Contacted, Qualified, Closed |
| Is Priority Style | Checkbox | |

### Node 34: Airtable Node — Create General Lead Record

**Purpose:** Logs all non-hot leads to the Airtable Nurture base for future follow-up by the marketing team.

**Node Type:** `Airtable`  
**Name:** `Airtable: Create Lead Record`

#### Configuration:

| Field | Value |
|-------|-------|
| **Credential** | `Airtable - Nurture Base` |
| **Resource** | `Record` |
| **Operation** | `Create` |
| **Base** | `Nurture` |
| **Table** | `Leads` |

**Fields to Map:**

| Airtable Field | n8n Expression |
|----------------|----------------|
| Name | `{{ $json.parsed_name }}` |
| Budget | `{{ $json.parsed_budget }}` |
| Architectural Style | `{{ $json.parsed_style }}` |
| Raw Message | `{{ $json.raw_message }}` |
| Cleaned Message | `{{ $json.cleaned_message }}` |
| Source | `{{ $json.source }}` |
| Timestamp | `{{ $json.submitted_at }}` |
| AI Confidence | `{{ $json.parsed_confidence }}` |
| AI Intent Summary | `{{ $json.parsed_intent }}` |
| Lead Status | `New` |
| Is Priority Style | `{{ $json.style_priority_flag }}` |

Connect: **Is Hot Lead? (False) → Airtable: Create Lead Record**

---

## 15. Layer 12 — Error Handling

### Node 35: IF Node — Check AI Confidence Score

**Purpose:** Even if fields are present, a very low AI confidence score may indicate a hallucinated extraction that needs human review.

**Node Type:** `IF`  
**Name:** `Confidence Sufficient?`

**Condition:**
| Field | Value |
|-------|-------|
| Value 1 | `{{ $json.parsed_confidence }}` |
| Operation | `Larger Than` |
| Value 2 | `0.6` |

> Insert this node AFTER `Parse AI JSON Response` and BEFORE `Check Name Exists`.

#### Routing:
- **True** → Continue to validation chain
- **False** → Route to `Set Low Confidence Error`

---

### Node 36: Set Node — Set Low Confidence Error

**Node Type:** `Set`  
**Name:** `Set Low Confidence Error`

| Name | Type | Value |
|------|------|-------|
| `validation_error` | String | `AI confidence too low ({{ $json.parsed_confidence }}) — requires manual review` |
| `has_name` | Boolean | `false` |
| `has_budget` | Boolean | `false` |
| `has_style` | Boolean | `false` |

Connect: **Confidence Sufficient? (False) → Set Low Confidence Error → Slack: Manual Review Alert**

---

### Node 37: IF Node — Check Parse Error

**Purpose:** Catches cases where the AI returned malformed JSON that couldn't be parsed at all.

**Node Type:** `IF`  
**Name:** `Has Parse Error?`

> Insert AFTER `Parse AI JSON Response`, BEFORE `Confidence Sufficient?`

**Condition:**
| Field | Value |
|-------|-------|
| Value 1 | `{{ $json.parse_error }}` |
| Operation | `Is Not Empty` |

#### Routing:
- **True** → `Set Parse Error Flag` → `Slack: Manual Review Alert`
- **False** → `Confidence Sufficient?`

---

### Node 38: Set Node — Set Parse Error Flag

**Node Type:** `Set`  
**Name:** `Set Parse Error Flag`

| Name | Type | Value |
|------|------|-------|
| `validation_error` | String | `AI JSON parse failed: {{ $json.parse_error }}` |

---

## 16. Layer 13 — Logging & Observability

### Node 39: Set Node — Add Execution Timestamp

**Purpose:** Records exactly when this lead was processed by the workflow, separate from when it was submitted.

**Node Type:** `Set`  
**Name:** `Add Execution Timestamp`

| Name | Type | Value |
|------|------|-------|
| `processed_at` | String | `{{ $now.toISO() }}` |
| `execution_id` | String | `{{ $execution.id }}` |

> Place this node at the very end of both the Hot Lead and General Lead paths (after Slack send and Airtable create).

---

### Node 40: Code Node — Attach Metadata

**Purpose:** Assembles a final metadata object summarizing the entire execution for logging purposes.

**Node Type:** `Code`  
**Name:** `Attach Metadata`

```javascript
const data = $input.first().json;

const metadata = {
  execution_id: data.execution_id,
  workflow_version: "1.0",
  processed_at: data.processed_at,
  submitted_at: data.submitted_at,
  source: data.source,
  lead_classification: (data.is_high_budget && data.style_priority_flag) ? "Hot" : "General",
  parsed_confidence: data.parsed_confidence,
  validation_passed: data.has_name && data.has_budget && data.has_style,
  budget_tier: data.is_high_budget ? "High (>$1.5M)" : "Standard",
  style_priority: data.style_priority_flag ? "Priority" : "Standard",
  fields_extracted: {
    name: !!data.parsed_name,
    budget: !!data.parsed_budget,
    style: !!data.parsed_style
  }
};

return [{
  json: {
    ...data,
    execution_metadata: metadata
  }
}];
```

Connect: **Add Execution Timestamp → Attach Metadata**

---

### Node 41: Google Sheets Node — Log to Execution Log Sheet

**Purpose:** Writes a row to a separate "Execution Log" tab in your Google Sheet for observability and auditing.

**Node Type:** `Google Sheets`  
**Name:** `Log Execution to Sheets`

#### Create a second tab in your Google Sheet named `Execution Log` with these columns:

| Column | Content |
|--------|---------|
| A - Execution ID | Workflow execution ID |
| B - Processed At | ISO timestamp |
| C - Lead Classification | Hot / General / Error |
| D - Client Name | Extracted name |
| E - Budget | Extracted budget |
| F - Style | Extracted style |
| G - Confidence | AI confidence score |
| H - Source | Form source |
| I - Validation Passed | TRUE / FALSE |

#### Node Configuration:

| Field | Value |
|-------|-------|
| **Resource** | `Sheet` |
| **Operation** | `Append Row` |
| **Sheet Name** | `Execution Log` |

**Column Mappings:**

| Sheet Column | Expression |
|-------------|-----------|
| Execution ID | `{{ $json.execution_id }}` |
| Processed At | `{{ $json.processed_at }}` |
| Lead Classification | `{{ $json.execution_metadata.lead_classification }}` |
| Client Name | `{{ $json.parsed_name }}` |
| Budget | `{{ $json.parsed_budget }}` |
| Style | `{{ $json.parsed_style }}` |
| Confidence | `{{ $json.parsed_confidence }}` |
| Source | `{{ $json.source }}` |
| Validation Passed | `{{ $json.execution_metadata.validation_passed }}` |

---

## 17. Node Connection Map

This section provides a complete, ordered list of every connection in the workflow.

```
Node 1  (Webhook)                       → Node 2  (Normalize Payload)
Node 2  (Normalize Payload)             → Node 3  (Clean Text)
Node 3  (Clean Text)                    → Node 4  (Validate Non-Empty)
Node 4  (Validate Non-Empty - TRUE)     → Node 5  (AI Parse Lead Fields)
Node 4  (Validate Non-Empty - FALSE)    → Node 33 (Slack: Manual Review Alert)
Node 5  (AI Parse Lead Fields)          → Node 6  (Parse AI JSON Response)
Node 6  (Parse AI JSON Response)        → Node 37 (Has Parse Error?)
Node 37 (Has Parse Error? - TRUE)       → Node 38 (Set Parse Error Flag)
Node 37 (Has Parse Error? - FALSE)      → Node 35 (Confidence Sufficient?)
Node 38 (Set Parse Error Flag)          → Node 33 (Slack: Manual Review Alert)
Node 35 (Confidence Sufficient? - TRUE) → Node 7  (Check Name Exists)
Node 35 (Confidence Sufficient? -FALSE) → Node 36 (Set Low Confidence Error)
Node 36 (Set Low Confidence Error)      → Node 33 (Slack: Manual Review Alert)
Node 7  (Check Name Exists - TRUE)      → Node 9  (Flag Name Valid)
Node 7  (Check Name Exists - FALSE)     → Node 8  (Flag Missing Name)
Node 8  (Flag Missing Name)             → Node 10 (Check Budget Is Numeric)
Node 9  (Flag Name Valid)               → Node 10 (Check Budget Is Numeric)
Node 10 (Check Budget - TRUE)           → Node 12 (Flag Budget Valid)
Node 10 (Check Budget - FALSE)          → Node 11 (Flag Missing Budget)
Node 11 (Flag Missing Budget)           → Node 13 (Check Style Exists)
Node 12 (Flag Budget Valid)             → Node 13 (Check Style Exists)
Node 13 (Check Style - TRUE)            → Node 15 (Flag Style Valid)
Node 13 (Check Style - FALSE)           → Node 14 (Flag Missing Style)
Node 14 (Flag Missing Style)            → Node 16 (Merge Validation Results - Input 2)
Node 15 (Flag Style Valid)              → Node 16 (Merge Validation Results - Input 1)
Node 16 (Merge Validation Results)      → Node 17 (All Fields Valid?)
Node 17 (All Fields Valid? - TRUE)      → Node 18 (Read Priority Styles)
Node 17 (All Fields Valid? - FALSE)     → Node 33 (Slack: Manual Review Alert)
Node 18 (Read Priority Styles)          → Node 19 (Normalize Styles)
Node 19 (Normalize Styles)              → Node 20 (Is Priority Style?)
Node 20 (Is Priority Style? - TRUE)     → Node 21 (Mark Priority: True)
Node 20 (Is Priority Style? - FALSE)    → Node 22 (Mark Priority: False)
Node 21 (Mark Priority: True)           → Node 23 (Merge After Style Check - Input 1)
Node 22 (Mark Priority: False)          → Node 23 (Merge After Style Check - Input 2)
Node 23 (Merge After Style Check)       → Node 24 (Budget > 1.5M?)
Node 24 (Budget > 1.5M? - TRUE)         → Node 25 (Flag: High Budget)
Node 24 (Budget > 1.5M? - FALSE)        → Node 26 (Flag: Standard Budget)
Node 25 (Flag: High Budget)             → Node 27 (Merge Budget Flags - Input 1)
Node 26 (Flag: Standard Budget)         → Node 27 (Merge Budget Flags - Input 2)
Node 27 (Merge Budget Flags)            → Node 28 (Is Hot Lead?)
Node 28 (Is Hot Lead? - TRUE)           → Node 29 (AI Generate Summary)
Node 28 (Is Hot Lead? - FALSE)          → Node 34 (Airtable: Create Lead Record)
Node 29 (AI Generate Summary)           → Node 30 (Extract Summary Text)
Node 30 (Extract Summary Text)          → Node 31 (Format Hot Lead Slack Message)
Node 31 (Format Slack Message)          → Node 32 (Slack: Hot Lead Alert)
Node 32 (Slack: Hot Lead Alert)         → Node 39 (Add Execution Timestamp)
Node 34 (Airtable: Create Lead Record)  → Node 39 (Add Execution Timestamp)
Node 39 (Add Execution Timestamp)       → Node 40 (Attach Metadata)
Node 40 (Attach Metadata)               → Node 41 (Log Execution to Sheets)
```

---

## 18. Testing Guide

### How to Send Test Requests

Use the following `curl` commands to test each scenario. Replace `YOUR_WEBHOOK_URL` with your n8n test webhook URL.

---

### Test Case 1: Hot Lead (Expected → Slack Alert)

**Input:**
```bash
curl -X POST YOUR_WEBHOOK_URL \
  -H "Content-Type: application/json" \
  -d '{
    "message": "Hello, my name is James Thornton. I am looking to build a Japandi-inspired private residence on a lakeside property in Colorado. My budget for this project is approximately 3.2 million dollars. I would love to discuss next steps with your design team.",
    "source": "website_contact_form",
    "submitted_at": "2024-11-15T10:00:00Z"
  }'
```

**Expected AI Extraction:**
```json
{
  "parsed_name": "James Thornton",
  "parsed_budget": 3200000,
  "parsed_style": "Japandi",
  "parsed_confidence": 0.97
}
```

**Expected Path:**
1. ✅ Validation passes (name, budget, style all found)
2. ✅ Japandi is in priority styles list → `isPriorityStyle: true`
3. ✅ $3.2M > $1.5M → `is_high_budget: true`
4. ✅ Both conditions met → Hot Lead
5. ✅ AI generates summary
6. ✅ Slack message sent to `#hot-leads`
7. ✅ Execution logged to Sheets

**Expected Slack Output:**
```
🔥 Hot Lead Alert — Foundations & Frame
[AI-generated professional summary about James Thornton...]

Client Name: James Thornton    | Budget: $3,200,000
Style: Japandi                 | Lead Source: Website Form

Client Message:
"Hello, my name is James Thornton..."

[📅 Schedule Discovery Call] [📧 View Full Lead]
Submitted: 2024-11-15T10:00:00Z | AI Confidence: 97%
```

---

### Test Case 2: General Lead (Expected → Airtable Record)

**Input:**
```bash
curl -X POST YOUR_WEBHOOK_URL \
  -H "Content-Type: application/json" \
  -d '{
    "message": "Hi there! I am Maria Santos. I am interested in a Craftsman-style kitchen and bathroom renovation. My budget is around $450,000. Can we set up a consultation?",
    "source": "website_contact_form",
    "submitted_at": "2024-11-15T11:30:00Z"
  }'
```

**Expected AI Extraction:**
```json
{
  "parsed_name": "Maria Santos",
  "parsed_budget": 450000,
  "parsed_style": "Craftsman",
  "parsed_confidence": 0.92
}
```

**Expected Path:**
1. ✅ Validation passes
2. ✅ Craftsman is in sheet but Priority Level = "Low"
   - Depending on your matching logic, this may or may not be `isPriorityStyle`
   - With the current inclusive matching: Craftsman IS in the sheet → `isPriorityStyle: true`
   - But budget $450,000 < $1.5M → `is_high_budget: false`
3. ✅ Both conditions NOT met → General Lead
4. ✅ Airtable record created in Nurture base
5. ✅ Execution logged

**Expected Airtable Record:**
```
Name:               Maria Santos
Budget:             450000
Architectural Style: Craftsman
Raw Message:        "Hi there! I am Maria Santos..."
Lead Status:        New
Timestamp:          2024-11-15T11:30:00Z
AI Confidence:      0.92
```

---

### Test Case 3: Invalid / Incomplete Input (Expected → Manual Review)

**Input:**
```bash
curl -X POST YOUR_WEBHOOK_URL \
  -H "Content-Type: application/json" \
  -d '{
    "message": "Hello, please call me",
    "source": "website_contact_form",
    "submitted_at": "2024-11-15T12:00:00Z"
  }'
```

**Expected AI Extraction:**
```json
{
  "parsed_name": null,
  "parsed_budget": null,
  "parsed_style": null,
  "parsed_confidence": 0.15
}
```

**Expected Path:**
1. ✅ Message passes non-empty check (it has content)
2. ✅ AI attempts extraction but confidence is very low
3. ❌ Confidence < 0.6 → Low Confidence Error
4. ✅ Routed to `Slack: Manual Review Alert`

**Expected Slack Output (in `#manual-review`):**
```
⚠️ Manual Review Required — Incomplete Lead

Validation Error: AI confidence too low (0.15) — requires manual review
Raw Message: 
"Hello, please call me"

AI Confidence: 15%
Submitted At: 2024-11-15T12:00:00Z
Source: website_contact_form

Please review this lead manually and update the CRM.
```

---

### Test Case 4: Empty Input (Expected → Manual Review at Preprocessing)

**Input:**
```bash
curl -X POST YOUR_WEBHOOK_URL \
  -H "Content-Type: application/json" \
  -d '{
    "message": "",
    "source": "website_contact_form"
  }'
```

**Expected Path:**
1. ✅ Webhook receives request
2. ✅ Normalize Payload runs
3. ✅ Clean Text runs (char_count = 0)
4. ❌ `Validate Non-Empty` IF → False branch
5. ✅ Routed directly to `Slack: Manual Review Alert`

---

## 19. Deliverables

### 1. n8n Workflow JSON (Partial Structure)

> This is a representative JSON structure. Import into n8n via **File → Import from JSON**.

```json
{
  "name": "Architectural Lead Triage AI Assistant - Foundations & Frame",
  "nodes": [
    {
      "parameters": {
        "httpMethod": "POST",
        "path": "lead-intake",
        "responseMode": "responseNode",
        "options": {}
      },
      "id": "node-webhook-001",
      "name": "Lead Intake Webhook",
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 1,
      "position": [240, 300]
    },
    {
      "parameters": {
        "keepOnlySet": true,
        "values": {
          "string": [
            {
              "name": "raw_message",
              "value": "={{ $json.body.message }}"
            },
            {
              "name": "source",
              "value": "={{ $json.body.source || 'unknown' }}"
            },
            {
              "name": "submitted_at",
              "value": "={{ $json.body.submitted_at || $now.toISO() }}"
            }
          ]
        }
      },
      "id": "node-set-001",
      "name": "Normalize Payload",
      "type": "n8n-nodes-base.set",
      "typeVersion": 1,
      "position": [460, 300]
    },
    {
      "parameters": {
        "jsCode": "const rawMessage = $input.first().json.raw_message || '';\nlet cleaned = rawMessage.replace(/<[^>]*>/g, '');\ncleaned = cleaned.replace(/\\s+/g, ' ').trim();\nconst charCount = cleaned.length;\nreturn [{ json: { ...$input.first().json, cleaned_message: cleaned, char_count: charCount } }];"
      },
      "id": "node-code-001",
      "name": "Clean Text",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [680, 300]
    },
    {
      "parameters": {
        "conditions": {
          "number": [
            {
              "value1": "={{ $json.char_count }}",
              "operation": "larger",
              "value2": 10
            }
          ]
        }
      },
      "id": "node-if-001",
      "name": "Validate Non-Empty",
      "type": "n8n-nodes-base.if",
      "typeVersion": 1,
      "position": [900, 300]
    }
  ],
  "connections": {
    "Lead Intake Webhook": {
      "main": [[{ "node": "Normalize Payload", "type": "main", "index": 0 }]]
    },
    "Normalize Payload": {
      "main": [[{ "node": "Clean Text", "type": "main", "index": 0 }]]
    },
    "Clean Text": {
      "main": [[{ "node": "Validate Non-Empty", "type": "main", "index": 0 }]]
    },
    "Validate Non-Empty": {
      "main": [
        [{ "node": "AI Parse Lead Fields", "type": "main", "index": 0 }],
        [{ "node": "Slack: Manual Review Alert", "type": "main", "index": 0 }]
      ]
    }
  },
  "settings": {
    "executionOrder": "v1",
    "saveManualExecutions": true,
    "callerPolicy": "workflowsFromSameOwner",
    "errorWorkflow": ""
  }
}
```

---

### 2. Google Sheet Schema

**Sheet 1: Priority Styles**

| Column Name | Type | Description | Example Value |
|-------------|------|-------------|---------------|
| Style Name | Text | Exact architectural style label | Modern Minimalist |
| Priority Level | Text | High / Medium / Low | High |
| Notes | Text | Internal notes on this style | Flagship style, highest margins |

**Sheet 2: Execution Log**

| Column Name | Type | Description | Example Value |
|-------------|------|-------------|---------------|
| Execution ID | Text | n8n execution ID | exec_abc123 |
| Processed At | DateTime | When workflow ran | 2024-11-15T10:05:00Z |
| Lead Classification | Text | Hot / General / Error | Hot |
| Client Name | Text | Extracted name | James Thornton |
| Budget | Number | Extracted budget as integer | 3200000 |
| Style | Text | Extracted architectural style | Japandi |
| Confidence | Number | AI confidence (0-1) | 0.97 |
| Source | Text | Form source | website_contact_form |
| Validation Passed | Boolean | TRUE or FALSE | TRUE |

---

### 3. Airtable Schema

**Base:** `Nurture`  
**Table:** `Leads`

| Field Name | Airtable Type | Required | Description |
|------------|---------------|----------|-------------|
| Name | Single Line Text | Yes | Client full name |
| Budget | Number | Yes | Budget as integer (e.g., 450000) |
| Architectural Style | Single Line Text | Yes | Extracted style |
| Raw Message | Long Text | No | Original unmodified inquiry |
| Cleaned Message | Long Text | No | After preprocessing |
| Source | Single Line Text | No | Where lead came from |
| Timestamp | Date and Time | Yes | When submitted |
| AI Confidence | Percent | No | 0-100% |
| AI Intent Summary | Long Text | No | One-line AI summary |
| Lead Status | Single Select | Yes | New, Contacted, Qualified, Closed |
| Is Priority Style | Checkbox | No | Checked if style is priority |
| Notes | Long Text | No | Manual notes from sales team |

---

## 20. Proof of Execution Guide

### Capturing Workflow Screenshots

1. Open your n8n workflow in full-screen mode
2. Use your browser's zoom to fit the entire workflow on screen (`Ctrl/Cmd + -`)
3. Press `Ctrl/Cmd + Shift + S` (or use your OS screenshot tool)
4. Alternatively, use n8n's built-in **Export as Image** option (available in some versions)

### Recording a Test Execution

1. Click **Execute Workflow** in n8n (the ▶ button)
2. In a separate terminal window, send a test curl request (see Testing section)
3. Watch nodes light up green (success) or red (error) in the canvas
4. Click any node to see its **Input** and **Output** data panels
5. For recording: use Loom, OBS, or macOS QuickTime screen recording

### Validating Outputs

After each test execution, verify:

**For Hot Leads:**
- [ ] Check `#hot-leads` Slack channel for the formatted Block Kit message
- [ ] Verify all fields are correct (name, budget, style)
- [ ] Verify the "Schedule Discovery Call" button link is correct
- [ ] Check the Execution Log in Google Sheets for a new row

**For General Leads:**
- [ ] Open Airtable → Nurture base → Leads table
- [ ] Verify a new record exists with all mapped fields
- [ ] Check that Lead Status = "New"
- [ ] Check the Execution Log in Google Sheets

**For Error Cases:**
- [ ] Check `#manual-review` Slack channel for the alert
- [ ] Verify the error message contains the correct validation error
- [ ] Verify the raw message is included for manual review

### n8n Execution History

1. In n8n, go to **Executions** in the left sidebar
2. See all past executions with status (Success/Error)
3. Click any execution to see the full data flow
4. Use the **Execution ID** to correlate with your Google Sheets log

---

## 21. Acceptance Validation Checklist

Use this checklist to confirm the workflow is working correctly before going live.

### Data Extraction
- [ ] ✅ Budget is parsed as a **plain integer** (no "$" signs, no commas, no "million" text)
- [ ] ✅ Architectural style is extracted as a clean string (e.g., "Modern Minimalist" not "modern minimalist style architecture")
- [ ] ✅ Client name is extracted correctly including first and last name
- [ ] ✅ AI confidence score is between 0 and 1
- [ ] ✅ Null values are returned when fields cannot be found (not empty strings)

### Validation Logic
- [ ] ✅ Empty messages are rejected before reaching AI
- [ ] ✅ Low-confidence extractions are routed to manual review
- [ ] ✅ JSON parse errors are caught and routed to manual review
- [ ] ✅ Missing name routes to manual review channel
- [ ] ✅ Missing budget routes to manual review channel
- [ ] ✅ Missing style routes to manual review channel

### Style Matching
- [ ] ✅ Styles from Google Sheet are compared case-insensitively
- [ ] ✅ `isPriorityStyle: true` when client style matches sheet
- [ ] ✅ `isPriorityStyle: false` when client style is not in sheet
- [ ] ✅ Google Sheets reads correctly with valid credentials

### Budget Evaluation
- [ ] ✅ Budget = $1,500,001 → `is_high_budget: true`
- [ ] ✅ Budget = $1,500,000 → `is_high_budget: false` (not strictly greater)
- [ ] ✅ Budget = $500,000 → `is_high_budget: false`

### Routing Logic
- [ ] ✅ Hot Lead (budget > 1.5M AND priority style) → Slack `#hot-leads`
- [ ] ✅ General Lead (any other valid combination) → Airtable Nurture
- [ ] ✅ Errors / missing fields → Slack `#manual-review`

### Slack Integration
- [ ] ✅ Hot lead Slack message uses Block Kit formatting (not plain text)
- [ ] ✅ Budget is formatted with dollar sign and commas (e.g., $2,200,000)
- [ ] ✅ "Schedule Discovery Call" button links to Calendly (or your preferred scheduler)
- [ ] ✅ AI confidence percentage is shown in the message footer
- [ ] ✅ Manual review message includes the original raw message

### Airtable Integration
- [ ] ✅ All fields are correctly mapped to Airtable columns
- [ ] ✅ Budget is stored as a number type (not text)
- [ ] ✅ Timestamp is in ISO format
- [ ] ✅ Lead Status defaults to "New"
- [ ] ✅ Record appears in correct table within Nurture base

### Logging & Observability
- [ ] ✅ Every execution (success and failure) writes to the Execution Log sheet
- [ ] ✅ Execution timestamp is in ISO format
- [ ] ✅ Lead classification is correctly recorded (Hot / General / Error)
- [ ] ✅ n8n Execution History shows correct status

### Error Handling
- [ ] ✅ All error paths lead to Slack `#manual-review` — nothing is silently dropped
- [ ] ✅ Error messages include enough context for manual processing
- [ ] ✅ Workflow does not crash on empty or malformed input

---

*Document prepared for: Foundations & Frame*  
*Workflow: Architectural Lead Triage AI Assistant*  
*Platform: n8n*  
*Version: 1.0*

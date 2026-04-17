# Autonomous Intelligence Agent for Wholesale Lead Enrichment

This guide is written so you can build the system yourself in n8n, step by step, without importing a prebuilt workflow.

It implements a production-style modular design for Heirloom & Hide:

- 1 main workflow: orchestration, cache logic, ICP reasoning, tier routing, HITL, and logging
- 1 sub-workflow: deep research engine

It uses:

- `Webhook` nodes for lead intake and Slack callback handling
- `Execute Sub-workflow` for orchestration
- `Google Sheets` as both log store and lightweight memory
- `Wait` plus a Slack callback webhook for real human-in-the-loop approvals
- `OpenAI` nodes for enrichment extraction, ICP reasoning, hook extraction, and pitch generation

## Important implementation choice

Your prompt contains one conflict:

- it asks for exactly `2` Google Sheets tabs
- it also asks for Tier 3 leads to go to a `Nurture` tab

To keep the architecture consistent and still satisfy the two-tab requirement, this build uses:

- `Raw Leads` tab
- `Deep Intelligence` tab

Tier 3 leads are written to `Deep Intelligence` with:

```text
Action Taken = Nurture
```

If you want a third physical `Nurture` tab later, you can duplicate that same final row into a third sheet, but this guide keeps the core system at exactly two tabs.

## Workflow names

Create these two workflows with these exact names:

1. `Heirloom & Hide - Wholesale Intelligence Orchestrator`
2. `Heirloom & Hide - Deep Research Engine`

## Credentials and variables you need before building

### 1. Google Sheets

Create one spreadsheet named:

`Heirloom & Hide Lead Intelligence`

Create two tabs:

1. `Raw Leads`
2. `Deep Intelligence`

### 2. OpenAI

Use a current n8n `OpenAI` credential.

Recommended model:

`gpt-4o-mini`

### 3. Slack app

Create a Slack app with:

- `chat:write` bot scope
- Interactivity enabled
- One channel where approvals should appear

You need:

- a bot token like `xoxb-...`
- the channel ID where HITL messages will be posted

### 4. Optional n8n variables

Recommended n8n variables:

- `SLACK_BOT_TOKEN`
- `SLACK_CHANNEL_ID`

If you don’t use n8n variables, replace the expressions in this guide with literal values.

---

# STEP 0: Create workflows (Main + Sub)

## 0A. Create the main workflow

1. In n8n, click `Create Workflow`.
2. Rename it to:

```text
Heirloom & Hide - Wholesale Intelligence Orchestrator
```

3. Save it once.

## 0B. Create the sub-workflow

1. Click `Create Workflow`.
2. Rename it to:

```text
Heirloom & Hide - Deep Research Engine
```

3. Save it once.

## 0C. Sub-workflow settings

Inside `Heirloom & Hide - Deep Research Engine`:

1. Open workflow `Settings`.
2. Set:
   - `This workflow can be called by`: `Any workflow` or your main workflow policy
   - `Save successful production executions`: `Save`

This matters because the `Execute Sub-workflow` docs require the sub-workflow trigger and saved production executions for a reliable parent/child workflow build.

---

# STEP 1: Lead intake

We will use a `Webhook` node as the public entry point.

The expected inbound JSON body is:

```json
{
  "email": "[email protected]",
  "domain": "marriott.com",
  "company_name": "Marriott"
}
```

`domain` and `company_name` are optional. `email` is required.

## 1A. Add the intake webhook

In the main workflow:

1. Add a `Webhook` node.
2. Rename it to:

```text
Webhook - Lead Intake
```

3. Configure these fields exactly:

- `HTTP Method`: `POST`
- `Path`:

```text
heirloom-hide-wholesale-lead
```

- `Authentication`: `None`
- `Respond`: `Immediately`
- `Response Code`: `200`

In `Options`, set:

- `No Response Body`: `Off`
- `Response Data`:

```text
Lead received
```

This gives you an always-on lead intake endpoint while allowing the workflow to keep running in the background.

## 1B. Add the lead normalization node

1. Click `+` after `Webhook - Lead Intake`.
2. Add a `Code` node.
3. Rename it to:

```text
Code - Normalize Lead
```

4. Set:
   - `Mode`: `Run Once for Each Item`

5. Paste this exact JavaScript into `JavaScript Code`:

```javascript
const body = $json.body ?? $json;

const email = String(body.email || '').trim().toLowerCase();
const providedDomain = String(body.domain || '').trim().toLowerCase();
const companyName = String(body.company_name || body.companyName || '').trim();

const nowIso = new Date().toISOString();
const random = Math.random().toString(36).slice(2, 8);
const leadId = `lead_${Date.now()}_${random}`;

const emailDomain = email.includes('@') ? email.split('@')[1].toLowerCase() : '';
const cleanedProvidedDomain = providedDomain
  .replace(/^https?:\/\//, '')
  .replace(/^www\./, '')
  .split('/')[0]
  .trim();

const personalDomains = new Set([
  'gmail.com',
  'yahoo.com',
  'hotmail.com',
  'outlook.com',
  'icloud.com',
  'aol.com',
  'protonmail.com',
  'pm.me'
]);

const extractedDomain = cleanedProvidedDomain || emailDomain;
const usableBusinessDomain =
  extractedDomain && !personalDomains.has(extractedDomain) ? extractedDomain : '';

const validationErrors = [];

if (!email) validationErrors.push('missing_email');
if (!usableBusinessDomain) validationErrors.push('missing_or_personal_domain');

return [
  {
    json: {
      lead_id: leadId,
      email,
      domain: usableBusinessDomain,
      domain_status: usableBusinessDomain ? 'usable' : 'missing_or_personal',
      company_name: companyName,
      website: usableBusinessDomain ? `https://${usableBusinessDomain}` : '',
      created_at: nowIso,
      source: 'webhook',
      validation_errors: validationErrors
    }
  }
];
```

This node handles:

- unique lead ID generation
- company domain extraction from email if the user didn’t provide one
- personal email detection
- safe fallback values

---

# STEP 2: Domain extraction

The domain extraction already happens in `Code - Normalize Lead`, but this step is where we make it operational and log the raw intake.

## 2A. Create the Raw Leads row

1. Add an `Edit Fields` node after `Code - Normalize Lead`.
2. Rename it to:

```text
Edit Fields - Raw Lead Row
```

3. Set:
   - `Mode`: `JSON Output`

4. Paste this exact JSON into `JSON Output`:

```json
{
  "Lead ID": "={{ $json.lead_id }}",
  "Email": "={{ $json.email }}",
  "Domain": "={{ $json.domain || 'UNKNOWN' }}",
  "Company Name": "={{ $json.company_name || '' }}",
  "Timestamp": "={{ $json.created_at }}",
  "Domain Status": "={{ $json.domain_status }}",
  "Source": "={{ $json.source }}"
}
```

## 2B. Log the raw lead

1. Add a `Google Sheets` node after `Edit Fields - Raw Lead Row`.
2. Rename it to:

```text
Google Sheets - Append Raw Lead
```

3. Configure these fields:

- `Credential to connect with`: your Google Sheets credential
- `Resource`: `Sheet Within Document`
- `Operation`: `Append Row`
- `Document`: `Heirloom & Hide Lead Intelligence`
- `Sheet`: `Raw Leads`
- `Mapping Column Mode`: `Map Automatically`

4. In `Options`, set:

- `Use Append`: `On`
- `Handling extra fields in input`: `Ignore Them`

## Raw Leads schema

Create this exact header row in the `Raw Leads` tab:

```text
Lead ID | Email | Domain | Company Name | Timestamp | Domain Status | Source
```

This tab gives you the immutable intake log.

---

# STEP 3: State engine (30-day check)

This is the cache layer that prevents redundant enrichment work.

We will:

1. look up existing intelligence rows by `Domain`
2. select the newest row
3. compare `Research Timestamp` to `now - 30 days`
4. if fresh, reuse intelligence
5. if stale or missing, run the research sub-workflow

## 3A. Look up prior intelligence by domain

1. Add a `Google Sheets` node after `Google Sheets - Append Raw Lead`.
2. Rename it to:

```text
Google Sheets - Lookup Intelligence by Domain
```

3. Configure these fields:

- `Credential to connect with`: your Google Sheets credential
- `Resource`: `Sheet Within Document`
- `Operation`: `Get Row(s)`
- `Document`: `Heirloom & Hide Lead Intelligence`
- `Sheet`: `Deep Intelligence`

In `Filters`, set:

- `Column`: `Domain`
- `Value`:

```javascript
={{ $('Code - Normalize Lead').item.json.domain || '__NO_DOMAIN__' }}
```

4. Under `Options`, add:

- `When Filter Has Multiple Matches`: `Return All Matches`

5. In the node `Settings` tab, set:

- `Always Output Data`: `On`

## 3B. Evaluate the 30-day cache

1. Add a `Code` node after `Google Sheets - Lookup Intelligence by Domain`.
2. Rename it to:

```text
Code - Evaluate 30 Day Cache
```

3. Set:
   - `Mode`: `Run Once for All Items`

4. Paste this exact JavaScript into `JavaScript Code`:

```javascript
const lead = $('Code - Normalize Lead').first().json;

function parseJsonField(value, fallback = {}) {
  if (!value) return fallback;
  if (typeof value === 'object' && !Array.isArray(value)) return value;
  try {
    return JSON.parse(String(value));
  } catch {
    return fallback;
  }
}

function toDate(value) {
  const d = new Date(value || 0);
  return Number.isNaN(d.getTime()) ? null : d;
}

const rows = items
  .map((item) => item.json || {})
  .filter((row) => Object.keys(row).length > 0);

rows.sort((a, b) => {
  const aTime =
    toDate(a['Research Timestamp'] || a.research_timestamp || a.Timestamp)?.getTime() || 0;
  const bTime =
    toDate(b['Research Timestamp'] || b.research_timestamp || b.Timestamp)?.getTime() || 0;
  return bTime - aTime;
});

const latest = rows[0] || null;
const latestResearchDate = latest
  ? toDate(latest['Research Timestamp'] || latest.research_timestamp || latest.Timestamp)
  : null;

const thirtyDaysMs = 30 * 24 * 60 * 60 * 1000;
const cacheHit = Boolean(
  lead.domain &&
  latest &&
  latestResearchDate &&
  Date.now() - latestResearchDate.getTime() <= thirtyDaysMs
);

return [
  {
    json: {
      ...lead,
      cache_hit: cacheHit,
      cache_checked_at: new Date().toISOString(),
      cached_row_found: Boolean(latest),
      cached_enriched_data: parseJsonField(latest?.['Enriched Data'] || latest?.enriched_data, {}),
      cached_tier: String(latest?.Tier || ''),
      cached_score: Number(latest?.Score || 0),
      cached_reasoning: String(latest?.Reasoning || ''),
      cached_pitch: String(latest?.Pitch || ''),
      cached_action_taken: String(latest?.['Action Taken'] || ''),
      cached_research_timestamp: String(
        latest?.['Research Timestamp'] || latest?.research_timestamp || ''
      ),
      cached_research_source: String(latest?.['Research Source'] || ''),
      cached_hitl_status: String(latest?.['HITL Status'] || '')
    }
  }
];
```

## 3C. Add the cache IF node

1. Add an `IF` node after `Code - Evaluate 30 Day Cache`.
2. Rename it to:

```text
IF - Use Cached Intelligence?
```

3. Configure:

- `Left Value`:

```javascript
={{ $json.cache_hit }}
```

- `Operator`: `is true`

## 3D. Normalize the cached branch

1. Connect the `TRUE` output to a `Code` node.
2. Rename it to:

```text
Code - Prepare Cached Intelligence
```

3. Set:
   - `Mode`: `Run Once for Each Item`

4. Paste this exact JavaScript:

```javascript
const cached = $json.cached_enriched_data || {};

return [
  {
    json: {
      ...$json,
      company_size: cached.company_size || 'Unknown',
      industry: cached.industry || 'Unknown',
      location: cached.location || 'Unknown',
      recent_news: cached.recent_news || 'Unknown',
      website: cached.website || $json.website || ($json.domain ? `https://${$json.domain}` : ''),
      tier: $json.cached_tier || 'Tier 3',
      score: Number($json.cached_score || 0),
      reasoning:
        $json.cached_reasoning ||
        'Used cached intelligence from the last 30 days because the domain was already researched.',
      research_source: 'cache'
    }
  }
];
```

At this point:

- `TRUE` branch skips fresh enrichment
- `FALSE` branch will call the deep research workflow

---

# STEP 4: Execute sub-workflow

Only the `FALSE` branch of `IF - Use Cached Intelligence?` should connect here.

## 4A. Add the Execute Sub-workflow node

1. Connect the `FALSE` output of `IF - Use Cached Intelligence?` to an `Execute Sub-workflow` node.
2. Rename it to:

```text
Execute Workflow - Deep Research Engine
```

3. Configure these fields:

- `Source`: `Database`
- `Workflow`: select `Heirloom & Hide - Deep Research Engine`
- `Mode`: `Run once for each item`

If your sub-workflow uses `Accept all data`, n8n will pass the current item directly into the sub-workflow.

The current item at this stage already contains:

- `lead_id`
- `email`
- `domain`
- `company_name`
- `website`
- `created_at`

The sub-workflow should return:

```json
{
  "lead_id": "...",
  "email": "...",
  "domain": "...",
  "company_name": "...",
  "website": "...",
  "company_size": "...",
  "industry": "...",
  "location": "...",
  "recent_news": "..."
}
```

---

# STEP 5: Deep research workflow

Now switch to the sub-workflow:

`Heirloom & Hide - Deep Research Engine`

This workflow does the enrichment work only.

It does not:

- assign tier
- decide routing
- generate sales pitch
- send Slack

That separation keeps the design modular and clean.

## 5A. Add the sub-workflow trigger

1. Add the `Execute Sub-workflow Trigger` node.
2. Search terms may show it as `When Executed by Another Workflow`.
3. Rename it to:

```text
Subworkflow Input
```

4. Set:

- `Input data mode`: `Accept all data`

## 5B. Normalize research input

1. Add a `Code` node after `Subworkflow Input`.
2. Rename it to:

```text
Code - Normalize Research Input
```

3. Set:
   - `Mode`: `Run Once for Each Item`

4. Paste this JavaScript:

```javascript
const domain = String($json.domain || '').trim().toLowerCase();
const website = String($json.website || (domain ? `https://${domain}` : '')).trim();

return [
  {
    json: {
      ...$json,
      domain,
      website,
      company_name: String($json.company_name || '').trim(),
      research_started_at: new Date().toISOString()
    }
  }
];
```

## 5C. Add domain availability IF

1. Add an `IF` node after `Code - Normalize Research Input`.
2. Rename it to:

```text
IF - Domain Available?
```

3. Configure:

- `Left Value`:

```javascript
={{ !!$json.domain }}
```

- `Operator`: `is true`

## 5D. Fallback output if no domain exists

1. Connect the `FALSE` output to an `Edit Fields` node.
2. Rename it to:

```text
Edit Fields - Empty Research Output
```

3. Set:
   - `Mode`: `JSON Output`

4. Paste this JSON:

```json
{
  "lead_id": "={{ $json.lead_id }}",
  "email": "={{ $json.email }}",
  "domain": "",
  "company_name": "={{ $json.company_name || '' }}",
  "website": "",
  "company_size": "Unknown",
  "industry": "Unknown",
  "location": "Unknown",
  "recent_news": "No company domain was available, so no external research was performed."
}
```

## 5E. First pass website research

1. Connect the `TRUE` output of `IF - Domain Available?` to an `HTTP Request` node.
2. Rename it to:

```text
HTTP Request - Homepage HTML
```

3. Configure these fields:

- `Method`: `GET`
- `URL`:

```javascript
={{ $json.website }}
```

- `Authentication`: `None`
- `Response Format`: `String`

Under `Options`, set:

- `Timeout`: `15000`
- `Never Error`: `On`

In the `Settings` tab, set:

- `Retry On Fail`: `On`
- `Max Tries`: `3`
- `Wait Between Tries (ms)`: `2000`
- `On Error`: `Continue`

## 5F. Strip website HTML into research text

1. Add a `Code` node after `HTTP Request - Homepage HTML`.
2. Rename it to:

```text
Code - Strip Homepage Text
```

3. Set:
   - `Mode`: `Run Once for Each Item`

4. Paste this JavaScript:

```javascript
const upstream = $('Code - Normalize Research Input').item.json;
const html =
  typeof $json === 'string'
    ? $json
    : typeof $json.body === 'string'
      ? $json.body
      : typeof $json.data === 'string'
        ? $json.data
        : JSON.stringify($json);

const text = String(html || '')
  .replace(/<script[\s\S]*?<\/script>/gi, ' ')
  .replace(/<style[\s\S]*?<\/style>/gi, ' ')
  .replace(/<[^>]+>/g, ' ')
  .replace(/&nbsp;/g, ' ')
  .replace(/\s+/g, ' ')
  .trim()
  .slice(0, 7000);

return [
  {
    json: {
      ...upstream,
      homepage_text: text || 'No homepage text extracted.'
    }
  }
];
```

## 5G. First pass extractor prompt

1. Add an `OpenAI` node after `Code - Strip Homepage Text`.
2. Rename it to:

```text
OpenAI - Research Pass 1
```

3. Configure:

- `Credential to connect with`: your OpenAI credential
- `Resource`: `Text`
- `Operation`: `Generate a Model Response`
- `Model`: `gpt-4o-mini`

Add two messages.

### System message

```text
You are a B2B company research analyst.

Your task is to extract structured company intelligence from a company website text dump.

Rules:
- Return JSON only.
- Do not use markdown.
- If a field is not supported by evidence, return "Unknown".
- For company_size, prefer broad employee ranges such as:
  - 1-10
  - 11-50
  - 51-200
  - 201-1000
  - 1000+
- For industry, use concise B2B wording.
- For location, prefer headquarters or primary region if available.
- For recent_news, summarize any public signal such as expansion, opening, partnership, award, renovation, funding, or launch. If there is none, return "Unknown".
- Set needs_second_pass to true if company_size, industry, or location are still Unknown, or if the company type is still unclear.

Return exactly:
{
  "company_size": "...",
  "industry": "...",
  "location": "...",
  "recent_news": "...",
  "website": "...",
  "needs_second_pass": true,
  "gap_reason": "..."
}
```

### User message

```javascript
=Company Name: {{ $('Code - Normalize Research Input').item.json.company_name || 'Unknown' }}

Domain: {{ $('Code - Normalize Research Input').item.json.domain }}

Website: {{ $('Code - Normalize Research Input').item.json.website }}

Homepage Text:
{{ $('Code - Strip Homepage Text').item.json.homepage_text }}
```

In `Options`, set:

- `Maximum Number of Tokens`: `500`
- `Output Randomness (Temperature)`: `0.1`
- `Output Content as JSON`: `On`

In `Settings`, set:

- `Retry On Fail`: `On`
- `Max Tries`: `3`
- `Wait Between Tries (ms)`: `2000`
- `On Error`: `Continue`

## 5H. Parse research pass 1

1. Add a `Code` node after `OpenAI - Research Pass 1`.
2. Rename it to:

```text
Code - Parse Research Pass 1
```

3. Set:
   - `Mode`: `Run Once for Each Item`

4. Paste this JavaScript:

```javascript
const source = $('Code - Strip Homepage Text').item.json;

function cleanJsonString(value) {
  return String(value || '')
    .trim()
    .replace(/^```json/i, '')
    .replace(/^```/i, '')
    .replace(/```$/i, '')
    .trim();
}

function safeParse(value) {
  if (!value) return null;
  if (typeof value === 'object' && !Array.isArray(value)) return value;
  try {
    return JSON.parse(cleanJsonString(value));
  } catch {
    return null;
  }
}

const rawModelOutput =
  $json.output ||
  $json.response ||
  $json.text ||
  $json.content ||
  $json.data ||
  $json.message ||
  $json;

const parsed =
  safeParse(rawModelOutput?.json) ||
  safeParse(rawModelOutput?.output_text) ||
  safeParse(rawModelOutput?.text) ||
  safeParse(rawModelOutput?.response) ||
  safeParse(rawModelOutput?.content) ||
  safeParse(rawModelOutput);

return [
  {
    json: {
      ...source,
      company_size: String(parsed?.company_size || 'Unknown'),
      industry: String(parsed?.industry || 'Unknown'),
      location: String(parsed?.location || 'Unknown'),
      recent_news: String(parsed?.recent_news || 'Unknown'),
      website: String(parsed?.website || source.website || ''),
      needs_second_pass: Boolean(parsed?.needs_second_pass ?? true),
      gap_reason: String(parsed?.gap_reason || 'No structured pass 1 output.')
    }
  }
];
```

## 5I. Second pass IF node

1. Add an `IF` node after `Code - Parse Research Pass 1`.
2. Rename it to:

```text
IF - Needs Second Pass?
```

3. Configure:

- `Left Value`:

```javascript
={{ $json.needs_second_pass }}
```

- `Operator`: `is true`

## 5J. Add a short politeness wait

1. Connect the `TRUE` output to a `Wait` node.
2. Rename it to:

```text
Wait - Research Buffer
```

3. Set:

- `Resume`: `After Time Interval`
- `Wait Amount`: `1`
- `Wait Unit`: `Seconds`

## 5K. Search results fetch for second pass

1. Add an `HTTP Request` node after `Wait - Research Buffer`.
2. Rename it to:

```text
HTTP Request - Bing Search HTML
```

3. Configure:

- `Method`: `GET`
- `URL`:

```javascript
={{ 'https://www.bing.com/search?q=' + encodeURIComponent((($json.company_name || $json.domain) + ' hospitality hotel resort corporate client leather goods')) }}
```

- `Authentication`: `None`
- `Response Format`: `String`

Under `Options`, set:

- `Timeout`: `15000`
- `Never Error`: `On`

In `Settings`, set:

- `Retry On Fail`: `On`
- `Max Tries`: `3`
- `Wait Between Tries (ms)`: `2000`
- `On Error`: `Continue`

## 5L. Strip search results text

1. Add a `Code` node after `HTTP Request - Bing Search HTML`.
2. Rename it to:

```text
Code - Strip Search Text
```

3. Set:
   - `Mode`: `Run Once for Each Item`

4. Paste this JavaScript:

```javascript
const pass1 = $('Code - Parse Research Pass 1').item.json;
const html =
  typeof $json === 'string'
    ? $json
    : typeof $json.body === 'string'
      ? $json.body
      : typeof $json.data === 'string'
        ? $json.data
        : JSON.stringify($json);

const text = String(html || '')
  .replace(/<script[\s\S]*?<\/script>/gi, ' ')
  .replace(/<style[\s\S]*?<\/style>/gi, ' ')
  .replace(/<[^>]+>/g, ' ')
  .replace(/&nbsp;/g, ' ')
  .replace(/\s+/g, ' ')
  .trim()
  .slice(0, 7000);

return [
  {
    json: {
      ...pass1,
      search_text: text || 'No search text extracted.'
    }
  }
];
```

## 5M. Second pass research prompt

1. Add an `OpenAI` node after `Code - Strip Search Text`.
2. Rename it to:

```text
OpenAI - Research Pass 2
```

2. Configure:

- `Credential to connect with`: your OpenAI credential
- `Resource`: `Text`
- `Operation`: `Generate a Model Response`
- `Model`: `gpt-4o-mini`

### System message

```text
You are a B2B company research analyst performing a second-pass research step.

You will receive:
- an incomplete first-pass company profile
- search result text

Your job is to improve the profile only where evidence supports it.

Rules:
- Return JSON only.
- Do not use markdown.
- Preserve "Unknown" when evidence is weak.
- Use broad employee ranges for company_size.
- Keep recent_news to one short sentence.

Return exactly:
{
  "company_size": "...",
  "industry": "...",
  "location": "...",
  "recent_news": "...",
  "website": "..."
}
```

### User message

```javascript
=Current Pass 1 Output:
{
  "company_size": "{{ $('Code - Parse Research Pass 1').item.json.company_size }}",
  "industry": "{{ $('Code - Parse Research Pass 1').item.json.industry }}",
  "location": "{{ $('Code - Parse Research Pass 1').item.json.location }}",
  "recent_news": "{{ $('Code - Parse Research Pass 1').item.json.recent_news }}",
  "website": "{{ $('Code - Parse Research Pass 1').item.json.website }}"
}

Search Results Text:
{{ $('Code - Strip Search Text').item.json.search_text }}
```

In `Options`, set:

- `Maximum Number of Tokens`: `400`
- `Output Randomness (Temperature)`: `0.1`
- `Output Content as JSON`: `On`

In `Settings`, set:

- `Retry On Fail`: `On`
- `Max Tries`: `3`
- `Wait Between Tries (ms)`: `2000`
- `On Error`: `Continue`

## 5N. Parse research pass 2

1. Add a `Code` node after `OpenAI - Research Pass 2`.
2. Rename it to:

```text
Code - Parse Research Pass 2
```

3. Set:
   - `Mode`: `Run Once for Each Item`

4. Paste this JavaScript:

```javascript
const pass1 = $('Code - Parse Research Pass 1').item.json;

function cleanJsonString(value) {
  return String(value || '')
    .trim()
    .replace(/^```json/i, '')
    .replace(/^```/i, '')
    .replace(/```$/i, '')
    .trim();
}

function safeParse(value) {
  if (!value) return null;
  if (typeof value === 'object' && !Array.isArray(value)) return value;
  try {
    return JSON.parse(cleanJsonString(value));
  } catch {
    return null;
  }
}

const rawModelOutput =
  $json.output ||
  $json.response ||
  $json.text ||
  $json.content ||
  $json.data ||
  $json.message ||
  $json;

const parsed =
  safeParse(rawModelOutput?.json) ||
  safeParse(rawModelOutput?.output_text) ||
  safeParse(rawModelOutput?.text) ||
  safeParse(rawModelOutput?.response) ||
  safeParse(rawModelOutput?.content) ||
  safeParse(rawModelOutput);

return [
  {
    json: {
      ...pass1,
      pass2_company_size: String(parsed?.company_size || 'Unknown'),
      pass2_industry: String(parsed?.industry || 'Unknown'),
      pass2_location: String(parsed?.location || 'Unknown'),
      pass2_recent_news: String(parsed?.recent_news || 'Unknown'),
      pass2_website: String(parsed?.website || pass1.website || '')
    }
  }
];
```

## 5O. Merge first-pass and second-pass research

1. Add a `Code` node after `Code - Parse Research Pass 2`.
2. Rename it to:

```text
Code - Merge Research Output
```

3. Set:
   - `Mode`: `Run Once for Each Item`

4. Paste this JavaScript:

```javascript
function bestValue(a, b) {
  return a && a !== 'Unknown' ? a : b && b !== 'Unknown' ? b : 'Unknown';
}

return [
  {
    json: {
      lead_id: $json.lead_id,
      email: $json.email,
      domain: $json.domain,
      company_name: $json.company_name,
      website: bestValue($json.pass2_website, $json.website),
      company_size: bestValue($json.pass2_company_size, $json.company_size),
      industry: bestValue($json.pass2_industry, $json.industry),
      location: bestValue($json.pass2_location, $json.location),
      recent_news: bestValue($json.pass2_recent_news, $json.recent_news)
    }
  }
];
```

## 5P. Finalize the research output

1. Add an `Edit Fields` node.
2. Rename it to:

```text
Edit Fields - Final Research Output
```

3. Set:
   - `Mode`: `JSON Output`

4. Paste this JSON:

```json
{
  "lead_id": "={{ $json.lead_id }}",
  "email": "={{ $json.email }}",
  "domain": "={{ $json.domain }}",
  "company_name": "={{ $json.company_name || '' }}",
  "website": "={{ $json.website || ($json.domain ? 'https://' + $json.domain : '') }}",
  "company_size": "={{ $json.company_size || 'Unknown' }}",
  "industry": "={{ $json.industry || 'Unknown' }}",
  "location": "={{ $json.location || 'Unknown' }}",
  "recent_news": "={{ $json.recent_news || 'Unknown' }}"
}
```

### Connection rule for the sub-workflow

Connect the nodes like this:

```text
Subworkflow Input
  -> Code - Normalize Research Input
  -> IF - Domain Available?

IF - Domain Available? FALSE
  -> Edit Fields - Empty Research Output

IF - Domain Available? TRUE
  -> HTTP Request - Homepage HTML
  -> Code - Strip Homepage Text
  -> OpenAI - Research Pass 1
  -> Code - Parse Research Pass 1
  -> IF - Needs Second Pass?

IF - Needs Second Pass? FALSE
  -> Edit Fields - Final Research Output

IF - Needs Second Pass? TRUE
  -> Wait - Research Buffer
  -> HTTP Request - Bing Search HTML
  -> Code - Strip Search Text
  -> OpenAI - Research Pass 2
  -> Code - Parse Research Pass 2
  -> Code - Merge Research Output
  -> Edit Fields - Final Research Output
```

The last node in the sub-workflow must be `Edit Fields - Final Research Output`, because the last node sends data back to the parent workflow.

---

# STEP 6: AI reasoning (ICP matching)

Now go back to the main workflow.

We will only run ICP reasoning on the fresh-research branch. Cached leads already have stored tiering and should reuse it.

## 6A. ICP JSON definition

Use this exact ICP JSON in the reasoning prompt:

```json
{
  "brand_name": "Heirloom & Hide",
  "offer": "Handmade premium leather goods for hospitality brands and corporate clients, including custom leather amenities, desk accessories, gifting, and branded guest-experience pieces.",
  "ideal_segments": [
    "hotel groups",
    "boutique hotels",
    "luxury resorts",
    "hospitality management companies",
    "premium corporate gifting programs"
  ],
  "must_have_signals": {
    "industry_alignment": [
      "hospitality",
      "hotels",
      "resorts",
      "travel",
      "corporate gifting",
      "premium B2B guest experience"
    ],
    "minimum_company_size_employees": 50,
    "buying_context": [
      "multiple locations or properties",
      "premium brand positioning",
      "guest experience investment",
      "customization or gifting budget"
    ]
  },
  "positive_signals": [
    "luxury or premium positioning",
    "multiple properties or locations",
    "award-winning brand language",
    "recent openings, renovations, expansions, or partnerships",
    "clear focus on guest experience, design, service, or executive gifting"
  ],
  "negative_signals": [
    "consumer-only small shop",
    "personal email with no company domain",
    "price-first or discount positioning",
    "unrelated industry with no hospitality or gifting use case",
    "very small local business with no scale signals"
  ],
  "tier_definition": {
    "Tier 1": "Strong ICP fit and likely capable of meaningful premium B2B purchasing.",
    "Tier 2": "Partial fit; worth human review before outreach.",
    "Tier 3": "Weak fit, weak buying potential, or insufficient business credibility."
  }
}
```

## 6B. Add the reasoner node

1. Connect `Execute Workflow - Deep Research Engine` to an `OpenAI` node.
2. Rename it to:

```text
OpenAI - ICP Reasoner
```

3. Configure:

- `Credential to connect with`: your OpenAI credential
- `Resource`: `Text`
- `Operation`: `Generate a Model Response`
- `Model`: `gpt-4o-mini`

### System message

```text
You are a senior B2B lead qualification analyst.

Your job is to evaluate a researched company against the ICP for Heirloom & Hide.

You must reason from evidence, not simple keyword matching.

You will consider:
- whether the company is actually in a relevant buying context
- whether the company likely has the scale and budget for premium custom leather goods
- whether brand positioning suggests premium or luxury alignment
- whether the lead is a strong direct prospect, a borderline prospect, or a low-value lead

ICP JSON:
{
  "brand_name": "Heirloom & Hide",
  "offer": "Handmade premium leather goods for hospitality brands and corporate clients, including custom leather amenities, desk accessories, gifting, and branded guest-experience pieces.",
  "ideal_segments": [
    "hotel groups",
    "boutique hotels",
    "luxury resorts",
    "hospitality management companies",
    "premium corporate gifting programs"
  ],
  "must_have_signals": {
    "industry_alignment": [
      "hospitality",
      "hotels",
      "resorts",
      "travel",
      "corporate gifting",
      "premium B2B guest experience"
    ],
    "minimum_company_size_employees": 50,
    "buying_context": [
      "multiple locations or properties",
      "premium brand positioning",
      "guest experience investment",
      "customization or gifting budget"
    ]
  },
  "positive_signals": [
    "luxury or premium positioning",
    "multiple properties or locations",
    "award-winning brand language",
    "recent openings, renovations, expansions, or partnerships",
    "clear focus on guest experience, design, service, or executive gifting"
  ],
  "negative_signals": [
    "consumer-only small shop",
    "personal email with no company domain",
    "price-first or discount positioning",
    "unrelated industry with no hospitality or gifting use case",
    "very small local business with no scale signals"
  ],
  "tier_definition": {
    "Tier 1": "Strong ICP fit and likely capable of meaningful premium B2B purchasing.",
    "Tier 2": "Partial fit; worth human review before outreach.",
    "Tier 3": "Weak fit, weak buying potential, or insufficient business credibility."
  }
}

Scoring rules:
- 85 to 100: Tier 1
- 50 to 84: Tier 2
- 0 to 49: Tier 3

Rules:
- Return JSON only.
- Do not use markdown.
- reasoning must be clear, evidence-based, and 2 to 4 sentences.
- If data quality is weak, reduce confidence and score accordingly.
- Do not simply match on the word hotel. Explain why the account is commercially attractive or not.

Return exactly:
{
  "tier": "Tier 1 | Tier 2 | Tier 3",
  "score": 0,
  "reasoning": "clear explanation"
}
```

### User message

```javascript
=Lead ID: {{ $('Execute Workflow - Deep Research Engine').item.json.lead_id }}

Email: {{ $('Execute Workflow - Deep Research Engine').item.json.email }}

Domain: {{ $('Execute Workflow - Deep Research Engine').item.json.domain }}

Company Name: {{ $('Execute Workflow - Deep Research Engine').item.json.company_name || 'Unknown' }}

Website: {{ $('Execute Workflow - Deep Research Engine').item.json.website || 'Unknown' }}

Company Size: {{ $('Execute Workflow - Deep Research Engine').item.json.company_size || 'Unknown' }}

Industry: {{ $('Execute Workflow - Deep Research Engine').item.json.industry || 'Unknown' }}

Location: {{ $('Execute Workflow - Deep Research Engine').item.json.location || 'Unknown' }}

Recent News: {{ $('Execute Workflow - Deep Research Engine').item.json.recent_news || 'Unknown' }}
```

In `Options`, set:

- `Maximum Number of Tokens`: `450`
- `Output Randomness (Temperature)`: `0.2`
- `Output Content as JSON`: `On`

In `Settings`, set:

- `Retry On Fail`: `On`
- `Max Tries`: `3`
- `Wait Between Tries (ms)`: `2000`
- `On Error`: `Continue`

## 6C. Parse the reasoner output

1. Add a `Code` node after `OpenAI - ICP Reasoner`.
2. Rename it to:

```text
Code - Parse ICP Result
```

3. Set:
   - `Mode`: `Run Once for Each Item`

4. Paste this JavaScript:

```javascript
const research = $('Execute Workflow - Deep Research Engine').item.json;

function cleanJsonString(value) {
  return String(value || '')
    .trim()
    .replace(/^```json/i, '')
    .replace(/^```/i, '')
    .replace(/```$/i, '')
    .trim();
}

function safeParse(value) {
  if (!value) return null;
  if (typeof value === 'object' && !Array.isArray(value)) return value;
  try {
    return JSON.parse(cleanJsonString(value));
  } catch {
    return null;
  }
}

function fallbackReasoning() {
  const domain = String(research.domain || '');
  const industry = String(research.industry || '').toLowerCase();
  const size = String(research.company_size || '');
  const news = String(research.recent_news || '').toLowerCase();
  const email = String(research.email || '').toLowerCase();
  const personalDomains = ['gmail.com', 'yahoo.com', 'hotmail.com', 'outlook.com', 'icloud.com'];

  if (!domain || personalDomains.some((d) => email.endsWith(`@${d}`))) {
    return {
      tier: 'Tier 3',
      score: 15,
      reasoning:
        'The lead lacks a credible business domain or appears to use a personal mailbox, which makes it a weak wholesale prospect.'
    };
  }

  if (
    /hotel|resort|hospitality|lodging|travel/i.test(industry) &&
    /51-200|201-1000|1000\+/i.test(size)
  ) {
    return {
      tier: 'Tier 1',
      score: /award|opening|expansion|renovation|partnership/i.test(news) ? 92 : 87,
      reasoning:
        'The account appears to be a scaled hospitality business with a plausible need for premium guest-experience products and sufficient buying potential.'
    };
  }

  if (/retail|shop|boutique|design|lifestyle|gift/i.test(industry) || /11-50|1-10/i.test(size)) {
    return {
      tier: 'Tier 2',
      score: 61,
      reasoning:
        'The lead shows some brand or gifting relevance, but scale, hospitality alignment, or procurement potential remain uncertain and should be reviewed by a human.'
    };
  }

  return {
    tier: 'Tier 3',
    score: 30,
    reasoning:
      'The lead does not show strong evidence of hospitality scale, premium B2B buying context, or clear alignment with Heirloom & Hide’s ICP.'
  };
}

const rawModelOutput =
  $json.output ||
  $json.response ||
  $json.text ||
  $json.content ||
  $json.data ||
  $json.message ||
  $json;

const parsed =
  safeParse(rawModelOutput?.json) ||
  safeParse(rawModelOutput?.output_text) ||
  safeParse(rawModelOutput?.text) ||
  safeParse(rawModelOutput?.response) ||
  safeParse(rawModelOutput?.content) ||
  safeParse(rawModelOutput);

const fallback = fallbackReasoning();
const validTier = ['Tier 1', 'Tier 2', 'Tier 3'].includes(parsed?.tier)
  ? parsed.tier
  : fallback.tier;
const score = Number.isFinite(Number(parsed?.score)) ? Number(parsed.score) : fallback.score;
const reasoning = String(parsed?.reasoning || fallback.reasoning).trim();

return [
  {
    json: {
      ...research,
      tier: validTier,
      score,
      reasoning,
      research_source: 'fresh'
    }
  }
];
```

## 6D. Draft the Deep Intelligence row

Both of these branches should eventually connect into the same node:

- `Code - Prepare Cached Intelligence`
- `Code - Parse ICP Result`

1. Add an `Edit Fields` node.
2. Rename it to:

```text
Edit Fields - Intelligence Draft Row
```

3. Set:
   - `Mode`: `JSON Output`

4. Paste this JSON:

```json
{
  "Lead ID": "={{ $json.lead_id }}",
  "Domain": "={{ $json.domain || 'UNKNOWN' }}",
  "Research Timestamp": "={{ $now.toISO() }}",
  "Enriched Data": "={{ JSON.stringify({ company_size: $json.company_size || 'Unknown', industry: $json.industry || 'Unknown', location: $json.location || 'Unknown', recent_news: $json.recent_news || 'Unknown', website: $json.website || '' }) }}",
  "Company Size": "={{ $json.company_size || 'Unknown' }}",
  "Industry": "={{ $json.industry || 'Unknown' }}",
  "Location": "={{ $json.location || 'Unknown' }}",
  "Recent News": "={{ $json.recent_news || 'Unknown' }}",
  "Website": "={{ $json.website || '' }}",
  "Tier": "={{ $json.tier }}",
  "Score": "={{ $json.score }}",
  "Reasoning": "={{ $json.reasoning }}",
  "Research Source": "={{ $json.research_source || 'fresh' }}",
  "HITL Status": "Not Required",
  "HITL Resume URL": "",
  "Action Taken": "Pending branch routing",
  "Pitch": ""
}
```

## 6E. Upsert the draft intelligence row

1. Add a `Google Sheets` node after `Edit Fields - Intelligence Draft Row`.
2. Rename it to:

```text
Google Sheets - Upsert Deep Intelligence Draft
```

3. Configure these fields:

- `Credential to connect with`: your Google Sheets credential
- `Resource`: `Sheet Within Document`
- `Operation`: `Append or Update Row`
- `Document`: `Heirloom & Hide Lead Intelligence`
- `Sheet`: `Deep Intelligence`
- `Mapping Column Mode`: `Map Automatically`
- `Column to Match On`: `Lead ID`

4. In `Options`, set:

- `Use Append`: `On`
- `Handling extra fields in input`: `Ignore Them`

This node gives you one canonical Deep Intelligence row per lead ID, which is important for:

- cache reuse
- HITL resume URL storage
- final pitch updates
- clean reporting

---

# STEP 7: Tier-based branching

## 7A. Add the Tier 1 IF node

1. Add an `IF` node after `Google Sheets - Upsert Deep Intelligence Draft`.
2. Rename it to:

```text
IF - Tier 1?
```

3. Configure:

- `Left Value`:

```javascript
={{ $json['Tier'] || $json.tier }}
```

- `Operator`: `is equal to`
- `Right Value`:

```text
Tier 1
```

## 7B. Add the Tier 2 IF node

1. Connect the `FALSE` output of `IF - Tier 1?` to another `IF` node.
2. Rename it to:

```text
IF - Tier 2?
```

3. Configure:

- `Left Value`:

```javascript
={{ $json['Tier'] || $json.tier }}
```

- `Operator`: `is equal to`
- `Right Value`:

```text
Tier 2
```

## Branch meaning

- `IF - Tier 1?` TRUE:
  - do website scraping
  - extract hook
  - generate AI sales pitch

- `IF - Tier 1?` FALSE and `IF - Tier 2?` TRUE:
  - send Slack HITL message
  - pause until button click
  - resume workflow

- `IF - Tier 1?` FALSE and `IF - Tier 2?` FALSE:
  - log as nurture
  - end

---

# STEP 8: Tier 1 -> scraping + pitch generation

This path is also reused when a Tier 2 lead gets approved by a human reviewer.

## 8A. Fetch website content

1. Connect the `TRUE` output of `IF - Tier 1?` to an `HTTP Request` node.
2. Rename it to:

```text
HTTP Request - Tier 1 Website HTML
```

3. Configure:

- `Method`: `GET`
- `URL`:

```javascript
={{ $json['Website'] || $json.website || ($json.domain ? 'https://' + $json.domain : '') }}
```

- `Authentication`: `None`
- `Response Format`: `String`

Under `Options`, set:

- `Timeout`: `15000`
- `Never Error`: `On`

In `Settings`, set:

- `Retry On Fail`: `On`
- `Max Tries`: `3`
- `Wait Between Tries (ms)`: `2000`
- `On Error`: `Continue`

## 8B. Strip Tier 1 website text

1. Add a `Code` node after `HTTP Request - Tier 1 Website HTML`.
2. Rename it to:

```text
Code - Strip Tier 1 Website Text
```

3. Set:
   - `Mode`: `Run Once for Each Item`

4. Paste this JavaScript:

```javascript
const prior = $('Code - Parse HITL Decision').isExecuted
  ? $('Code - Parse HITL Decision').item.json
  : $('Google Sheets - Upsert Deep Intelligence Draft').item.json;
const html =
  typeof $json === 'string'
    ? $json
    : typeof $json.body === 'string'
      ? $json.body
      : typeof $json.data === 'string'
        ? $json.data
        : JSON.stringify($json);

const text = String(html || '')
  .replace(/<script[\s\S]*?<\/script>/gi, ' ')
  .replace(/<style[\s\S]*?<\/style>/gi, ' ')
  .replace(/<[^>]+>/g, ' ')
  .replace(/&nbsp;/g, ' ')
  .replace(/\s+/g, ' ')
  .trim()
  .slice(0, 9000);

return [
  {
    json: {
      ...prior,
      website_text: text || 'No website text extracted.'
    }
  }
];
```

## 8C. Extract a hook

1. Add an `OpenAI` node after `Code - Strip Tier 1 Website Text`.
2. Rename it to:

```text
OpenAI - Tier 1 Hook Extractor
```

3. Configure:

- `Credential to connect with`: your OpenAI credential
- `Resource`: `Text`
- `Operation`: `Generate a Model Response`
- `Model`: `gpt-4o-mini`

### System message

```text
You are a B2B outbound strategist.

Read the website text and identify the best outreach hook for Heirloom & Hide.

A good hook is one of:
- award or recognition
- premium brand positioning
- guest experience message
- craftsmanship or design message
- expansion, opening, renovation, or partnership signal

Rules:
- Return JSON only.
- Do not use markdown.
- Keep hook_summary to one sentence.
- Keep pitch_angle to one sentence explaining how Heirloom & Hide could be relevant.

Return exactly:
{
  "hook_type": "...",
  "hook_summary": "...",
  "pitch_angle": "..."
}
```

### User message

```javascript
=Company Name: {{ $('Google Sheets - Upsert Deep Intelligence Draft').item.json['Domain'] }}

Website Text:
{{ $('Code - Strip Tier 1 Website Text').item.json.website_text }}
```

In `Options`, set:

- `Maximum Number of Tokens`: `300`
- `Output Randomness (Temperature)`: `0.2`
- `Output Content as JSON`: `On`

In `Settings`, set:

- `Retry On Fail`: `On`
- `Max Tries`: `3`
- `Wait Between Tries (ms)`: `2000`
- `On Error`: `Continue`

## 8D. Parse the hook output

1. Add a `Code` node after `OpenAI - Tier 1 Hook Extractor`.
2. Rename it to:

```text
Code - Parse Hook JSON
```

3. Set:
   - `Mode`: `Run Once for Each Item`

4. Paste this JavaScript:

```javascript
const prior = $('Code - Strip Tier 1 Website Text').item.json;

function cleanJsonString(value) {
  return String(value || '')
    .trim()
    .replace(/^```json/i, '')
    .replace(/^```/i, '')
    .replace(/```$/i, '')
    .trim();
}

function safeParse(value) {
  if (!value) return null;
  if (typeof value === 'object' && !Array.isArray(value)) return value;
  try {
    return JSON.parse(cleanJsonString(value));
  } catch {
    return null;
  }
}

const rawModelOutput =
  $json.output ||
  $json.response ||
  $json.text ||
  $json.content ||
  $json.data ||
  $json.message ||
  $json;

const parsed =
  safeParse(rawModelOutput?.json) ||
  safeParse(rawModelOutput?.output_text) ||
  safeParse(rawModelOutput?.text) ||
  safeParse(rawModelOutput?.response) ||
  safeParse(rawModelOutput?.content) ||
  safeParse(rawModelOutput);

return [
  {
    json: {
      ...prior,
      hook_type: String(parsed?.hook_type || 'brand_positioning'),
      hook_summary: String(
        parsed?.hook_summary ||
          'The brand presents itself as a premium business with a strong guest-experience orientation.'
      ),
      pitch_angle: String(
        parsed?.pitch_angle ||
          'Heirloom & Hide could support the brand with premium custom leather goods aligned to guest and brand experience.'
      )
    }
  }
];
```

## 8E. Generate the sales pitch

1. Add an `OpenAI` node after `Code - Parse Hook JSON`.
2. Rename it to:

```text
OpenAI - Generate Sales Pitch
```

3. Configure:

- `Credential to connect with`: your OpenAI credential
- `Resource`: `Text`
- `Operation`: `Generate a Model Response`
- `Model`: `gpt-4o-mini`

### System message

```text
You write premium B2B outreach for Heirloom & Hide.

Your goal is to generate a short, intelligent sales pitch for a hospitality or premium-brand prospect.

Rules:
- Keep it between 90 and 140 words.
- Sound polished, specific, and commercially relevant.
- Use the hook naturally.
- Do not sound spammy.
- Do not overclaim.
- Do not invent news or product details beyond the provided context.
- Return JSON only.
- Do not use markdown.

Return exactly:
{
  "pitch": "..."
}
```

### User message

```javascript
=Lead Context:
{
  "domain": "{{ $('Google Sheets - Upsert Deep Intelligence Draft').item.json['Domain'] }}",
  "company_size": "{{ $('Google Sheets - Upsert Deep Intelligence Draft').item.json['Company Size'] }}",
  "industry": "{{ $('Google Sheets - Upsert Deep Intelligence Draft').item.json['Industry'] }}",
  "location": "{{ $('Google Sheets - Upsert Deep Intelligence Draft').item.json['Location'] }}",
  "recent_news": "{{ $('Google Sheets - Upsert Deep Intelligence Draft').item.json['Recent News'] }}",
  "tier": "{{ $('Google Sheets - Upsert Deep Intelligence Draft').item.json['Tier'] }}",
  "score": "{{ $('Google Sheets - Upsert Deep Intelligence Draft').item.json['Score'] }}",
  "reasoning": "{{ $('Google Sheets - Upsert Deep Intelligence Draft').item.json['Reasoning'] }}"
}

Hook:
{
  "hook_type": "{{ $('Code - Parse Hook JSON').item.json.hook_type }}",
  "hook_summary": "{{ $('Code - Parse Hook JSON').item.json.hook_summary }}",
  "pitch_angle": "{{ $('Code - Parse Hook JSON').item.json.pitch_angle }}"
}
```

In `Options`, set:

- `Maximum Number of Tokens`: `350`
- `Output Randomness (Temperature)`: `0.5`
- `Output Content as JSON`: `On`

In `Settings`, set:

- `Retry On Fail`: `On`
- `Max Tries`: `3`
- `Wait Between Tries (ms)`: `2000`
- `On Error`: `Continue`

## 8F. Parse the pitch JSON

1. Add a `Code` node after `OpenAI - Generate Sales Pitch`.
2. Rename it to:

```text
Code - Parse Pitch JSON
```

3. Set:
   - `Mode`: `Run Once for Each Item`

4. Paste this JavaScript:

```javascript
const prior = $('Code - Parse Hook JSON').item.json;

function cleanJsonString(value) {
  return String(value || '')
    .trim()
    .replace(/^```json/i, '')
    .replace(/^```/i, '')
    .replace(/```$/i, '')
    .trim();
}

function safeParse(value) {
  if (!value) return null;
  if (typeof value === 'object' && !Array.isArray(value)) return value;
  try {
    return JSON.parse(cleanJsonString(value));
  } catch {
    return null;
  }
}

const rawModelOutput =
  $json.output ||
  $json.response ||
  $json.text ||
  $json.content ||
  $json.data ||
  $json.message ||
  $json;

const parsed =
  safeParse(rawModelOutput?.json) ||
  safeParse(rawModelOutput?.output_text) ||
  safeParse(rawModelOutput?.text) ||
  safeParse(rawModelOutput?.response) ||
  safeParse(rawModelOutput?.content) ||
  safeParse(rawModelOutput);

const fallbackPitch =
  `Heirloom & Hide could be a strong fit for ${prior['Domain'] || 'this account'} given its premium brand positioning and guest-experience focus. ` +
  `We create handmade leather goods that help hospitality and corporate brands bring a more considered, elevated feel to gifting, in-room presentation, and branded experience moments. ` +
  `Based on the signals we found on the website, there may be a natural opportunity to align custom leather pieces with the company’s brand standards and service experience.`;

return [
  {
    json: {
      ...prior,
      pitch: String(parsed?.pitch || fallbackPitch)
    }
  }
];
```

## 8G. Final Tier 1 update

1. Add an `Edit Fields` node after `Code - Parse Pitch JSON`.
2. Rename it to:

```text
Edit Fields - Tier 1 Final Update
```

3. Set:
   - `Mode`: `JSON Output`

4. Paste this JSON:

```json
{
  "Lead ID": "={{ $('Google Sheets - Upsert Deep Intelligence Draft').item.json['Lead ID'] }}",
  "Domain": "={{ $('Google Sheets - Upsert Deep Intelligence Draft').item.json['Domain'] }}",
  "Research Timestamp": "={{ $now.toISO() }}",
  "Enriched Data": "={{ $('Google Sheets - Upsert Deep Intelligence Draft').item.json['Enriched Data'] }}",
  "Company Size": "={{ $('Google Sheets - Upsert Deep Intelligence Draft').item.json['Company Size'] }}",
  "Industry": "={{ $('Google Sheets - Upsert Deep Intelligence Draft').item.json['Industry'] }}",
  "Location": "={{ $('Google Sheets - Upsert Deep Intelligence Draft').item.json['Location'] }}",
  "Recent News": "={{ $('Google Sheets - Upsert Deep Intelligence Draft').item.json['Recent News'] }}",
  "Website": "={{ $('Google Sheets - Upsert Deep Intelligence Draft').item.json['Website'] }}",
  "Tier": "={{ $('Google Sheets - Upsert Deep Intelligence Draft').item.json['Tier'] }}",
  "Score": "={{ $('Google Sheets - Upsert Deep Intelligence Draft').item.json['Score'] }}",
  "Reasoning": "={{ $('Google Sheets - Upsert Deep Intelligence Draft').item.json['Reasoning'] }}",
  "Research Source": "={{ $('Google Sheets - Upsert Deep Intelligence Draft').item.json['Research Source'] }}",
  "HITL Status": "={{ $json.hitl_decision === 'Approve' ? 'Approved' : 'Not Required' }}",
  "HITL Resume URL": "",
  "Action Taken": "={{ $json.hitl_decision === 'Approve' ? 'Tier 2 approved - pitch generated' : 'Tier 1 - pitch generated' }}",
  "Pitch": "={{ $json.pitch }}"
}
```

5. Add a `Google Sheets` node after it.
6. Rename it to:

```text
Google Sheets - Upsert Deep Intelligence Final
```

7. Configure exactly like the draft upsert:

- `Credential to connect with`: your Google Sheets credential
- `Resource`: `Sheet Within Document`
- `Operation`: `Append or Update Row`
- `Document`: `Heirloom & Hide Lead Intelligence`
- `Sheet`: `Deep Intelligence`
- `Mapping Column Mode`: `Map Automatically`
- `Column to Match On`: `Lead ID`

Under `Options`, set:

- `Use Append`: `On`
- `Handling extra fields in input`: `Ignore Them`

---

# STEP 9: Tier 2 -> Slack HITL system

This step uses:

- Slack interactive buttons
- a static Slack callback webhook in the same main workflow
- a `Wait` node in the lead execution
- a stored `HITL Resume URL` in Google Sheets

## 9A. Add the Slack callback webhook in the main workflow

This is a second trigger node in the same main workflow.

1. Add another `Webhook` node anywhere on the canvas.
2. Rename it to:

```text
Webhook - Slack HITL Callback
```

3. Configure:

- `HTTP Method`: `POST`
- `Path`:

```text
heirloom-hide-slack-hitl
```

- `Authentication`: `None`
- `Respond`: `Immediately`
- `Response Code`: `200`

In `Options`, set:

- `No Response Body`: `On`

## 9B. Parse the Slack button payload

1. Add a `Code` node after `Webhook - Slack HITL Callback`.
2. Rename it to:

```text
Code - Parse Slack Action Payload
```

3. Set:
   - `Mode`: `Run Once for Each Item`

4. Paste this JavaScript:

```javascript
const body = $json.body ?? $json;
const rawPayload = body.payload || '';

let payload = {};
try {
  payload = typeof rawPayload === 'string' ? JSON.parse(rawPayload) : rawPayload;
} catch {
  payload = {};
}

const action = payload.actions?.[0] || {};

let value = {};
try {
  value = JSON.parse(action.value || '{}');
} catch {
  value = {};
}

return [
  {
    json: {
      lead_id: String(value.lead_id || ''),
      decision: String(
        value.decision ||
          (action.action_id === 'approve_lead' ? 'approve' : action.action_id === 'reject_lead' ? 'reject' : '')
      ),
      reviewer: String(payload.user?.username || payload.user?.name || payload.user?.id || 'unknown'),
      reviewer_id: String(payload.user?.id || ''),
      channel_id: String(payload.channel?.id || ''),
      message_ts: String(payload.message?.ts || ''),
      response_url: String(payload.response_url || '')
    }
  }
];
```

## 9C. Look up the waiting row

1. Add a `Google Sheets` node after `Code - Parse Slack Action Payload`.
2. Rename it to:

```text
Google Sheets - Lookup Pending HITL Row
```

3. Configure:

- `Credential to connect with`: your Google Sheets credential
- `Resource`: `Sheet Within Document`
- `Operation`: `Get Row(s)`
- `Document`: `Heirloom & Hide Lead Intelligence`
- `Sheet`: `Deep Intelligence`

In `Filters`, set:

- `Column`: `Lead ID`
- `Value`:

```javascript
={{ $('Code - Parse Slack Action Payload').item.json.lead_id }}
```

In `Settings`, set:

- `Always Output Data`: `On`

## 9D. Build the resume request

1. Add a `Code` node after `Google Sheets - Lookup Pending HITL Row`.
2. Rename it to:

```text
Code - Build Resume Request
```

3. Set:
   - `Mode`: `Run Once for Each Item`

4. Paste this JavaScript:

```javascript
const action = $('Code - Parse Slack Action Payload').item.json;

return [
  {
    json: {
      ...action,
      resume_url: String($json['HITL Resume URL'] || '').trim(),
      resume_body: {
        decision: action.decision,
        reviewer: action.reviewer,
        reviewer_id: action.reviewer_id,
        decided_at: new Date().toISOString()
      }
    }
  }
];
```

## 9E. Resume the waiting execution

1. Add an `HTTP Request` node after `Code - Build Resume Request`.
2. Rename it to:

```text
HTTP Request - Resume Wait Execution
```

3. Configure:

- `Method`: `POST`
- `URL`:

```javascript
={{ $json.resume_url }}
```

- `Send Body`: `On`
- `Specify Body`: `Using JSON`
- `JSON Body`:

```javascript
={{ $json.resume_body }}
```

Under `Options`, set:

- `Never Error`: `On`
- `Timeout`: `10000`

In `Settings`, set:

- `Retry On Fail`: `On`
- `Max Tries`: `3`
- `Wait Between Tries (ms)`: `1000`
- `On Error`: `Continue`

## 9F. Tier 2 branch setup in the lead execution path

Now return to the lead path:

1. Connect the `TRUE` output of `IF - Tier 2?` to a `Code` node.
2. Rename it to:

```text
Code - Prepare HITL Context
```

3. Set:
   - `Mode`: `Run Once for Each Item`

4. Paste this JavaScript:

```javascript
return [
  {
    json: {
      ...$json,
      hitl_resume_url: $execution.resumeUrl,
      hitl_status: 'Pending',
      lead_summary:
        `${$json['Domain'] || $json.domain || 'Unknown domain'} | ` +
        `${$json['Industry'] || $json.industry || 'Unknown industry'} | ` +
        `${$json['Company Size'] || $json.company_size || 'Unknown size'} | ` +
        `${$json['Location'] || $json.location || 'Unknown location'}`
    }
  }
];
```

Important:

- `$execution.resumeUrl` is provided by n8n for workflows containing a `Wait` node configured to resume on webhook.
- The docs note that the node exposing this URL must run in the same execution as the `Wait` node.

## 9G. Upsert the pending HITL row

1. Add an `Edit Fields` node after `Code - Prepare HITL Context`.
2. Rename it to:

```text
Edit Fields - Tier 2 Pending Update
```

3. Set:
   - `Mode`: `JSON Output`

4. Paste this JSON:

```json
{
  "Lead ID": "={{ $json['Lead ID'] || $json.lead_id }}",
  "Domain": "={{ $json['Domain'] || $json.domain }}",
  "Research Timestamp": "={{ $now.toISO() }}",
  "Enriched Data": "={{ $json['Enriched Data'] || JSON.stringify({ company_size: $json['Company Size'] || $json.company_size || 'Unknown', industry: $json['Industry'] || $json.industry || 'Unknown', location: $json['Location'] || $json.location || 'Unknown', recent_news: $json['Recent News'] || $json.recent_news || 'Unknown', website: $json['Website'] || $json.website || '' }) }}",
  "Company Size": "={{ $json['Company Size'] || $json.company_size || 'Unknown' }}",
  "Industry": "={{ $json['Industry'] || $json.industry || 'Unknown' }}",
  "Location": "={{ $json['Location'] || $json.location || 'Unknown' }}",
  "Recent News": "={{ $json['Recent News'] || $json.recent_news || 'Unknown' }}",
  "Website": "={{ $json['Website'] || $json.website || '' }}",
  "Tier": "={{ $json['Tier'] || $json.tier }}",
  "Score": "={{ $json['Score'] || $json.score }}",
  "Reasoning": "={{ $json['Reasoning'] || $json.reasoning }}",
  "Research Source": "={{ $json['Research Source'] || $json.research_source || 'fresh' }}",
  "HITL Status": "Pending",
  "HITL Resume URL": "={{ $json.hitl_resume_url }}",
  "Action Taken": "Tier 2 - awaiting human review",
  "Pitch": ""
}
```

5. Add a `Google Sheets` node after it.
6. Rename it to:

```text
Google Sheets - Upsert Tier 2 Pending
```

7. Configure exactly like the other upsert nodes:

- `Credential to connect with`: your Google Sheets credential
- `Resource`: `Sheet Within Document`
- `Operation`: `Append or Update Row`
- `Document`: `Heirloom & Hide Lead Intelligence`
- `Sheet`: `Deep Intelligence`
- `Mapping Column Mode`: `Map Automatically`
- `Column to Match On`: `Lead ID`

## 9H. Send the Slack HITL message

1. Add an `HTTP Request` node after `Google Sheets - Upsert Tier 2 Pending`.
2. Rename it to:

```text
HTTP Request - Slack HITL Message
```

3. Configure:

- `Method`: `POST`
- `URL`:

```text
https://slack.com/api/chat.postMessage
```

- `Send Headers`: `On`

Add these headers:

- `Name`: `Authorization`
- `Value`:

```javascript
={{ 'Bearer ' + $vars.SLACK_BOT_TOKEN }}
```

- `Name`: `Content-Type`
- `Value`:

```text
application/json; charset=utf-8
```

- `Send Body`: `On`
- `Specify Body`: `Using JSON`

In `JSON Body`, switch to `Expression` and paste this exact object:

```javascript
={{
  {
    "channel": $vars.SLACK_CHANNEL_ID,
    "text": `Tier 2 lead review required for ${$json['Domain'] || $json.domain}`,
    "blocks": [
      {
        "type": "header",
        "text": {
          "type": "plain_text",
          "text": "Tier 2 Lead Review Required",
          "emoji": true
        }
      },
      {
        "type": "section",
        "fields": [
          {
            "type": "mrkdwn",
            "text": `*Lead ID*\n${$json['Lead ID'] || $json.lead_id}`
          },
          {
            "type": "mrkdwn",
            "text": `*Domain*\n${$json['Domain'] || $json.domain || 'Unknown'}`
          },
          {
            "type": "mrkdwn",
            "text": `*Tier*\n${$json['Tier'] || $json.tier}`
          },
          {
            "type": "mrkdwn",
            "text": `*Score*\n${$json['Score'] || $json.score}`
          }
        ]
      },
      {
        "type": "section",
        "text": {
          "type": "mrkdwn",
          "text": `*Lead summary*\n${$json.lead_summary}`
        }
      },
      {
        "type": "section",
        "text": {
          "type": "mrkdwn",
          "text": `*Reasoning*\n${$json['Reasoning'] || $json.reasoning}`
        }
      },
      {
        "type": "actions",
        "block_id": "hitl_review_actions",
        "elements": [
          {
            "type": "button",
            "text": {
              "type": "plain_text",
              "text": "Approve",
              "emoji": true
            },
            "style": "primary",
            "action_id": "approve_lead",
            "value": JSON.stringify({
              lead_id: $json['Lead ID'] || $json.lead_id,
              decision: 'approve'
            })
          },
          {
            "type": "button",
            "text": {
              "type": "plain_text",
              "text": "Reject",
              "emoji": true
            },
            "style": "danger",
            "action_id": "reject_lead",
            "value": JSON.stringify({
              lead_id: $json['Lead ID'] || $json.lead_id,
              decision: 'reject'
            })
          }
        ]
      }
    ]
  }
}}
```

This requires your Slack app `Interactivity & Shortcuts` request URL to point to:

```text
https://YOUR_N8N_BASE_URL/webhook/heirloom-hide-slack-hitl
```

## 9I. Add the waiting node

1. Add a `Wait` node after `HTTP Request - Slack HITL Message`.
2. Rename it to:

```text
Wait - Slack Decision
```

3. Configure:

- `Resume`: `On Webhook Call`
- `Authentication`: `None`

Turn on:

- `Limit Wait Time`: `On`

Then set:

- `Limit Type`: `After Time Interval`
- `Amount`: `48`
- `Unit`: `Hours`

Under `On Webhook Call options`, set:

- `Raw Body`: `On`
- `No Response Body`: `On`

## 9J. Parse the HITL decision after resume

1. Add a `Code` node after `Wait - Slack Decision`.
2. Rename it to:

```text
Code - Parse HITL Decision
```

3. Set:
   - `Mode`: `Run Once for Each Item`

4. Paste this JavaScript:

```javascript
const pending = $('Code - Prepare HITL Context').item.json;
const incoming = $json.body ?? $json;

const decision = String(incoming.decision || '').toLowerCase();

return [
  {
    json: {
      ...pending,
      hitl_decision:
        decision === 'approve' ? 'Approve' : decision === 'reject' ? 'Reject' : 'Timed Out',
      hitl_reviewer: String(incoming.reviewer || 'unknown'),
      hitl_reviewer_id: String(incoming.reviewer_id || ''),
      hitl_decision_at: String(incoming.decided_at || new Date().toISOString())
    }
  }
];
```

## 9K. Upsert the decision

1. Add an `Edit Fields` node after `Code - Parse HITL Decision`.
2. Rename it to:

```text
Edit Fields - HITL Decision Update
```

3. Set:
   - `Mode`: `JSON Output`

4. Paste this JSON:

```json
{
  "Lead ID": "={{ $json['Lead ID'] || $json.lead_id }}",
  "Domain": "={{ $json['Domain'] || $json.domain }}",
  "Research Timestamp": "={{ $now.toISO() }}",
  "Enriched Data": "={{ $json['Enriched Data'] || JSON.stringify({ company_size: $json['Company Size'] || $json.company_size || 'Unknown', industry: $json['Industry'] || $json.industry || 'Unknown', location: $json['Location'] || $json.location || 'Unknown', recent_news: $json['Recent News'] || $json.recent_news || 'Unknown', website: $json['Website'] || $json.website || '' }) }}",
  "Company Size": "={{ $json['Company Size'] || $json.company_size || 'Unknown' }}",
  "Industry": "={{ $json['Industry'] || $json.industry || 'Unknown' }}",
  "Location": "={{ $json['Location'] || $json.location || 'Unknown' }}",
  "Recent News": "={{ $json['Recent News'] || $json.recent_news || 'Unknown' }}",
  "Website": "={{ $json['Website'] || $json.website || '' }}",
  "Tier": "={{ $json['Tier'] || $json.tier }}",
  "Score": "={{ $json['Score'] || $json.score }}",
  "Reasoning": "={{ $json['Reasoning'] || $json.reasoning }}",
  "Research Source": "={{ $json['Research Source'] || $json.research_source || 'fresh' }}",
  "HITL Status": "={{ $json.hitl_decision }}",
  "HITL Resume URL": "",
  "Action Taken": "={{ 'Tier 2 - human review result: ' + $json.hitl_decision }}",
  "Pitch": ""
}
```

5. Add a `Google Sheets` node after it.
6. Rename it to:

```text
Google Sheets - Upsert HITL Decision
```

7. Configure exactly like the other upsert nodes.

## 9L. Add the approval IF node

1. Add an `IF` node after `Google Sheets - Upsert HITL Decision`.
2. Rename it to:

```text
IF - HITL Approved?
```

3. Configure:

- `Left Value`:

```javascript
={{ $json['HITL Status'] || $json.hitl_decision }}
```

- `Operator`: `is equal to`
- `Right Value`:

```text
Approve
```

Connection rule:

- `TRUE` output of `IF - HITL Approved?` should connect to `HTTP Request - Tier 1 Website HTML`
- `FALSE` output of `IF - HITL Approved?` should connect to the Tier 3 nurture update path

## Example Slack message JSON with buttons

```json
{
  "channel": "C0123456789",
  "text": "Tier 2 lead review required",
  "blocks": [
    {
      "type": "header",
      "text": {
        "type": "plain_text",
        "text": "Tier 2 Lead Review Required"
      }
    },
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "*Reasoning*\nPartial hospitality fit with some premium signals, but procurement scale is still uncertain."
      }
    },
    {
      "type": "actions",
      "block_id": "hitl_review_actions",
      "elements": [
        {
          "type": "button",
          "text": {
            "type": "plain_text",
            "text": "Approve"
          },
          "style": "primary",
          "action_id": "approve_lead",
          "value": "{\"lead_id\":\"lead_1713440000000_ab12cd\",\"decision\":\"approve\"}"
        },
        {
          "type": "button",
          "text": {
            "type": "plain_text",
            "text": "Reject"
          },
          "style": "danger",
          "action_id": "reject_lead",
          "value": "{\"lead_id\":\"lead_1713440000000_ab12cd\",\"decision\":\"reject\"}"
        }
      ]
    }
  ]
}
```

---

# STEP 10: Tier 3 -> nurture logging

This path handles:

- direct Tier 3 leads
- Tier 2 leads rejected by a human reviewer
- Tier 2 leads that time out without a decision

## 10A. Create the final nurture update node

1. Add an `Edit Fields` node.
2. Rename it to:

```text
Edit Fields - Tier 3 Final Update
```

3. Connect both of these into it:

- the `FALSE` output of `IF - Tier 2?`
- the `FALSE` output of `IF - HITL Approved?`

4. Set:
   - `Mode`: `JSON Output`

5. Paste this JSON:

```json
{
  "Lead ID": "={{ $json['Lead ID'] || $json.lead_id }}",
  "Domain": "={{ $json['Domain'] || $json.domain || 'UNKNOWN' }}",
  "Research Timestamp": "={{ $now.toISO() }}",
  "Enriched Data": "={{ $json['Enriched Data'] || JSON.stringify({ company_size: $json['Company Size'] || $json.company_size || 'Unknown', industry: $json['Industry'] || $json.industry || 'Unknown', location: $json['Location'] || $json.location || 'Unknown', recent_news: $json['Recent News'] || $json.recent_news || 'Unknown', website: $json['Website'] || $json.website || '' }) }}",
  "Company Size": "={{ $json['Company Size'] || $json.company_size || 'Unknown' }}",
  "Industry": "={{ $json['Industry'] || $json.industry || 'Unknown' }}",
  "Location": "={{ $json['Location'] || $json.location || 'Unknown' }}",
  "Recent News": "={{ $json['Recent News'] || $json.recent_news || 'Unknown' }}",
  "Website": "={{ $json['Website'] || $json.website || '' }}",
  "Tier": "={{ $json['Tier'] || $json.tier || 'Tier 3' }}",
  "Score": "={{ $json['Score'] || $json.score || 0 }}",
  "Reasoning": "={{ $json['Reasoning'] || $json.reasoning || 'Lead did not meet ICP requirements strongly enough.' }}",
  "Research Source": "={{ $json['Research Source'] || $json.research_source || 'fresh' }}",
  "HITL Status": "={{ $json['HITL Status'] || $json.hitl_decision || 'Not Required' }}",
  "HITL Resume URL": "",
  "Action Taken": "={{ ($json['HITL Status'] || $json.hitl_decision) === 'Reject' ? 'Nurture - rejected in HITL' : (($json['HITL Status'] || $json.hitl_decision) === 'Timed Out' ? 'Nurture - HITL timed out' : 'Nurture - Tier 3') }}",
  "Pitch": ""
}
```

## 10B. Use the same final upsert node

Connect:

```text
Edit Fields - Tier 3 Final Update
  -> Google Sheets - Upsert Deep Intelligence Final
```

This keeps all final outcomes in one canonical intelligence table.

---

# STEP 11: Logging system (2-tab Sheets)

## 11A. Raw Leads tab schema

Use this exact header row in `Raw Leads`:

```text
Lead ID | Email | Domain | Company Name | Timestamp | Domain Status | Source
```

## 11B. Deep Intelligence tab schema

Use this exact header row in `Deep Intelligence`:

```text
Lead ID | Domain | Research Timestamp | Enriched Data | Company Size | Industry | Location | Recent News | Website | Tier | Score | Reasoning | Research Source | HITL Status | HITL Resume URL | Action Taken | Pitch
```

These extra operational columns are necessary for:

- 30-day cache reuse
- HITL pause/resume state
- final pitch logging
- auditability

## 11C. Sample rows

### Raw Leads sample rows

| Lead ID | Email | Domain | Company Name | Timestamp | Domain Status | Source |
| --- | --- | --- | --- | --- | --- | --- |
| lead_1713440000000_ab12cd | [email protected] | marriott.com | Marriott | 2026-04-17T08:00:00.000Z | usable | webhook |
| lead_1713440300000_ef34gh | [email protected] | littleatelier.com | Little Atelier | 2026-04-17T08:05:00.000Z | usable | webhook |
| lead_1713440600000_ij56kl | [email protected] | UNKNOWN |  | 2026-04-17T08:10:00.000Z | missing_or_personal | webhook |

### Deep Intelligence sample rows

| Lead ID | Domain | Research Timestamp | Enriched Data | Company Size | Industry | Location | Recent News | Website | Tier | Score | Reasoning | Research Source | HITL Status | HITL Resume URL | Action Taken | Pitch |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| lead_1713440000000_ab12cd | marriott.com | 2026-04-17T08:03:00.000Z | {"company_size":"1000+","industry":"Hospitality","location":"Global","recent_news":"The brand continues to expand and refresh properties.","website":"https://marriott.com"} | 1000+ | Hospitality | Global | The brand continues to expand and refresh properties. | https://marriott.com | Tier 1 | 93 | Large hospitality group with clear premium guest-experience context and strong enterprise buying potential. | fresh | Not Required |  | Tier 1 - pitch generated | Heirloom & Hide could help elevate guest and gifting moments... |
| lead_1713440300000_ef34gh | littleatelier.com | 2026-04-17T08:06:30.000Z | {"company_size":"11-50","industry":"Lifestyle Retail","location":"Austin, Texas","recent_news":"Unknown","website":"https://littleatelier.com"} | 11-50 | Lifestyle Retail | Austin, Texas | Unknown | https://littleatelier.com | Tier 2 | 61 | Brand quality looks strong, but hospitality scale and procurement fit are still uncertain. | fresh | Pending | https://YOUR_N8N_INSTANCE/webhook-waiting/... | Tier 2 - awaiting human review |  |
| lead_1713440600000_ij56kl | UNKNOWN | 2026-04-17T08:10:30.000Z | {"company_size":"Unknown","industry":"Unknown","location":"Unknown","recent_news":"No company domain was available, so no external research was performed.","website":""} | Unknown | Unknown | Unknown | No company domain was available, so no external research was performed. |  | Tier 3 | 15 | The lead lacks a credible business domain or appears to use a personal mailbox, which makes it a weak wholesale prospect. | fresh | Not Required |  | Nurture - Tier 3 |  |

## 11D. Sample outputs for each Tier

### Tier 1 sample output

```json
{
  "tier": "Tier 1",
  "score": 93,
  "reasoning": "The company appears to be a large hospitality group with clear enterprise scale, premium brand positioning, and a strong guest-experience orientation. Those signals suggest a credible buying context for premium custom leather goods at meaningful volume."
}
```

### Tier 2 sample output

```json
{
  "tier": "Tier 2",
  "score": 61,
  "reasoning": "The account has some premium brand or gifting relevance, but hospitality alignment and procurement scale are still uncertain. It is worth human review before outreach, rather than immediate rejection or immediate pitch generation."
}
```

### Tier 3 sample output

```json
{
  "tier": "Tier 3",
  "score": 15,
  "reasoning": "The lead lacks a credible business domain and does not present strong evidence of enterprise hospitality or corporate buying potential. It should be retained only for nurture or discarded from active outbound focus."
}
```

---

# STEP 12: Error handling

This build is designed not to break on bad inputs or temporary upstream failures.

## 12A. Empty or missing domain

Handled in:

- `Code - Normalize Lead`
- `Edit Fields - Empty Research Output`
- `Code - Parse ICP Result`

Effect:

- no hard failure
- the lead still logs
- the lead naturally falls into `Tier 3`

## 12B. API failure or empty HTTP response

For all `HTTP Request` nodes in this guide, set:

- `Retry On Fail`: `On`
- `Max Tries`: `3`
- `Wait Between Tries (ms)`: `2000`
- `On Error`: `Continue`
- `Never Error`: `On` where supported in node options

This applies to:

- `HTTP Request - Homepage HTML`
- `HTTP Request - Bing Search HTML`
- `HTTP Request - Tier 1 Website HTML`
- `HTTP Request - Slack HITL Message`
- `HTTP Request - Resume Wait Execution`

## 12C. OpenAI malformed or empty JSON

Every OpenAI node in this guide has a paired parser `Code` node that:

- attempts safe JSON parsing
- strips markdown fences
- validates expected keys
- falls back to deterministic defaults

This applies to:

- `Code - Parse Research Pass 1`
- `Code - Parse Research Pass 2`
- `Code - Parse ICP Result`
- `Code - Parse Hook JSON`
- `Code - Parse Pitch JSON`

## 12D. Slack HITL issues

Possible failures:

- Slack button clicked after wait expired
- Slack callback arrives without a stored resume URL
- reviewer never responds

Protections:

- `Wait - Slack Decision` has a `48 hour` limit
- timed-out leads route to nurture
- callback resume uses `On Error: Continue`
- decision is logged separately in `Google Sheets - Upsert HITL Decision`

## 12E. Recommended node settings checklist

For every external node:

- turn on `Retry On Fail`
- set `Max Tries = 3`
- set `Wait Between Tries (ms) = 2000`
- use `On Error = Continue`

For lookup nodes:

- use `Always Output Data = On` where missing rows would otherwise stop execution

For stateful logic:

- never assume a field exists
- always default to `Unknown` or `Tier 3`

---

# STEP 13: Testing scenarios

Use these exact scenarios.

## Test 1: Marriott-like hotel -> Tier 1

Send this payload to `Webhook - Lead Intake`:

```json
{
  "email": "[email protected]",
  "domain": "marriott.com",
  "company_name": "Marriott"
}
```

Expected result:

- Raw lead logged
- no cache hit on first run
- sub-workflow runs
- `OpenAI - ICP Reasoner` returns `Tier 1`
- website scraping runs
- hook extracted
- pitch generated
- `Deep Intelligence` row updated with `Action Taken = Tier 1 - pitch generated`

## Test 2: Small boutique shop -> Tier 2

```json
{
  "email": "[email protected]",
  "domain": "littleatelier.com",
  "company_name": "Little Atelier"
}
```

Expected result:

- Raw lead logged
- sub-workflow runs
- `Tier 2`
- Slack interactive message is sent
- workflow pauses at `Wait - Slack Decision`
- click `Approve` in Slack
- callback webhook fires
- wait resumes
- website scrape + pitch generation runs
- final row becomes `Tier 2 approved - pitch generated`

If you click `Reject`:

- row becomes `Nurture - rejected in HITL`
- no pitch is generated

## Test 3: Random Gmail user -> Tier 3

```json
{
  "email": "[email protected]"
}
```

Expected result:

- Raw lead logged with `Domain Status = missing_or_personal`
- no meaningful external enrichment
- `Tier 3`
- no Slack HITL
- no pitch
- final row becomes `Nurture - Tier 3`

## Test 4: Cache hit within 30 days

Run the Marriott-like test twice within 30 days.

Expected result on second run:

- `Google Sheets - Lookup Intelligence by Domain` finds an existing row
- `Code - Evaluate 30 Day Cache` sets `cache_hit = true`
- `Execute Workflow - Deep Research Engine` does not run
- cached intelligence is reused
- routing still works

## Test checklist inside n8n

For each scenario:

1. Run the webhook once with production-style sample data.
2. Inspect:
   - `Code - Normalize Lead`
   - `Code - Evaluate 30 Day Cache`
   - `Execute Workflow - Deep Research Engine`
   - `Code - Parse ICP Result`
   - `Wait - Slack Decision` when Tier 2
3. Confirm exactly one `Raw Leads` row was appended.
4. Confirm exactly one current `Deep Intelligence` row exists for the lead ID.
5. Confirm the tier and score match expectation.
6. Confirm the Slack callback path resumes the wait execution.
7. Confirm the second run of the same domain skips fresh research for 30 days.

---

# STEP 14: Activation

Before activating, verify these items carefully.

## 14A. Slack app setup

In Slack app settings:

1. Enable `Interactivity & Shortcuts`.
2. Set `Request URL` to:

```text
https://YOUR_N8N_BASE_URL/webhook/heirloom-hide-slack-hitl
```

3. Install or reinstall the app after scope changes.
4. Confirm the bot has `chat:write`.

## 14B. Webhook URLs

Use production webhook URLs only after the workflow is active.

Lead intake production URL:

```text
https://YOUR_N8N_BASE_URL/webhook/heirloom-hide-wholesale-lead
```

Slack callback production URL:

```text
https://YOUR_N8N_BASE_URL/webhook/heirloom-hide-slack-hitl
```

## 14C. Turn on the workflows

1. Save `Heirloom & Hide - Deep Research Engine`.
2. Save `Heirloom & Hide - Wholesale Intelligence Orchestrator`.
3. Activate the sub-workflow first.
4. Activate the main workflow second.

## 14D. Live sanity check

Run these in order:

1. One high-fit hospitality lead
2. One partial-fit boutique lead
3. One personal-email low-fit lead
4. One repeat lead to test the 30-day cache

You should confirm:

- the sub-workflow executes only when needed
- the cache bypass works
- Slack HITL pauses and resumes correctly
- final tier routing is correct
- all logs are written

---

## Full connection map

### Main workflow

```text
Webhook - Lead Intake
  -> Code - Normalize Lead
  -> Edit Fields - Raw Lead Row
  -> Google Sheets - Append Raw Lead
  -> Google Sheets - Lookup Intelligence by Domain
  -> Code - Evaluate 30 Day Cache
  -> IF - Use Cached Intelligence?

IF - Use Cached Intelligence? TRUE
  -> Code - Prepare Cached Intelligence
  -> Edit Fields - Intelligence Draft Row

IF - Use Cached Intelligence? FALSE
  -> Execute Workflow - Deep Research Engine
  -> OpenAI - ICP Reasoner
  -> Code - Parse ICP Result
  -> Edit Fields - Intelligence Draft Row

Edit Fields - Intelligence Draft Row
  -> Google Sheets - Upsert Deep Intelligence Draft
  -> IF - Tier 1?

IF - Tier 1? TRUE
  -> HTTP Request - Tier 1 Website HTML

IF - Tier 1? FALSE
  -> IF - Tier 2?

IF - Tier 2? TRUE
  -> Code - Prepare HITL Context
  -> Edit Fields - Tier 2 Pending Update
  -> Google Sheets - Upsert Tier 2 Pending
  -> HTTP Request - Slack HITL Message
  -> Wait - Slack Decision
  -> Code - Parse HITL Decision
  -> Edit Fields - HITL Decision Update
  -> Google Sheets - Upsert HITL Decision
  -> IF - HITL Approved?

IF - HITL Approved? TRUE
  -> HTTP Request - Tier 1 Website HTML

IF - HITL Approved? FALSE
  -> Edit Fields - Tier 3 Final Update

IF - Tier 2? FALSE
  -> Edit Fields - Tier 3 Final Update

HTTP Request - Tier 1 Website HTML
  -> Code - Strip Tier 1 Website Text
  -> OpenAI - Tier 1 Hook Extractor
  -> Code - Parse Hook JSON
  -> OpenAI - Generate Sales Pitch
  -> Code - Parse Pitch JSON
  -> Edit Fields - Tier 1 Final Update
  -> Google Sheets - Upsert Deep Intelligence Final

Edit Fields - Tier 3 Final Update
  -> Google Sheets - Upsert Deep Intelligence Final

Webhook - Slack HITL Callback
  -> Code - Parse Slack Action Payload
  -> Google Sheets - Lookup Pending HITL Row
  -> Code - Build Resume Request
  -> HTTP Request - Resume Wait Execution
```

### Sub-workflow

```text
Subworkflow Input
  -> Code - Normalize Research Input
  -> IF - Domain Available?

IF - Domain Available? FALSE
  -> Edit Fields - Empty Research Output

IF - Domain Available? TRUE
  -> HTTP Request - Homepage HTML
  -> Code - Strip Homepage Text
  -> OpenAI - Research Pass 1
  -> Code - Parse Research Pass 1
  -> IF - Needs Second Pass?

IF - Needs Second Pass? FALSE
  -> Edit Fields - Final Research Output

IF - Needs Second Pass? TRUE
  -> Wait - Research Buffer
  -> HTTP Request - Bing Search HTML
  -> Code - Strip Search Text
  -> OpenAI - Research Pass 2
  -> Code - Parse Research Pass 2
  -> Code - Merge Research Output
  -> Edit Fields - Final Research Output
```

---

## Near-complete workflow JSON structure

This is not a direct import file, but it is a near-complete structural map that matches the build above.

```json
{
  "main_workflow": {
    "name": "Heirloom & Hide - Wholesale Intelligence Orchestrator",
    "triggers": [
      "Webhook - Lead Intake",
      "Webhook - Slack HITL Callback"
    ],
    "nodes": [
      "Code - Normalize Lead",
      "Edit Fields - Raw Lead Row",
      "Google Sheets - Append Raw Lead",
      "Google Sheets - Lookup Intelligence by Domain",
      "Code - Evaluate 30 Day Cache",
      "IF - Use Cached Intelligence?",
      "Code - Prepare Cached Intelligence",
      "Execute Workflow - Deep Research Engine",
      "OpenAI - ICP Reasoner",
      "Code - Parse ICP Result",
      "Edit Fields - Intelligence Draft Row",
      "Google Sheets - Upsert Deep Intelligence Draft",
      "IF - Tier 1?",
      "IF - Tier 2?",
      "HTTP Request - Tier 1 Website HTML",
      "Code - Strip Tier 1 Website Text",
      "OpenAI - Tier 1 Hook Extractor",
      "Code - Parse Hook JSON",
      "OpenAI - Generate Sales Pitch",
      "Code - Parse Pitch JSON",
      "Edit Fields - Tier 1 Final Update",
      "Google Sheets - Upsert Deep Intelligence Final",
      "Code - Prepare HITL Context",
      "Edit Fields - Tier 2 Pending Update",
      "Google Sheets - Upsert Tier 2 Pending",
      "HTTP Request - Slack HITL Message",
      "Wait - Slack Decision",
      "Code - Parse HITL Decision",
      "Edit Fields - HITL Decision Update",
      "Google Sheets - Upsert HITL Decision",
      "IF - HITL Approved?",
      "Edit Fields - Tier 3 Final Update",
      "Code - Parse Slack Action Payload",
      "Google Sheets - Lookup Pending HITL Row",
      "Code - Build Resume Request",
      "HTTP Request - Resume Wait Execution"
    ]
  },
  "sub_workflow": {
    "name": "Heirloom & Hide - Deep Research Engine",
    "nodes": [
      "Subworkflow Input",
      "Code - Normalize Research Input",
      "IF - Domain Available?",
      "Edit Fields - Empty Research Output",
      "HTTP Request - Homepage HTML",
      "Code - Strip Homepage Text",
      "OpenAI - Research Pass 1",
      "Code - Parse Research Pass 1",
      "IF - Needs Second Pass?",
      "Wait - Research Buffer",
      "HTTP Request - Bing Search HTML",
      "Code - Strip Search Text",
      "OpenAI - Research Pass 2",
      "Code - Parse Research Pass 2",
      "Code - Merge Research Output",
      "Edit Fields - Final Research Output"
    ]
  }
}
```

---

## Why this build is production-grade

- It uses a modular orchestrator + sub-workflow design.
- It avoids redundant enrichment with a real 30-day cache.
- It stores state in Google Sheets so humans can inspect and audit it.
- It separates research, reasoning, outreach, and HITL concerns cleanly.
- It uses a real pause/resume pattern for Slack approvals.
- It includes deterministic fallback logic for API and model failures.
- It preserves a single lead ID across every step.

---

## Reference links

These are the main docs this build aligns to:

- n8n Execute Sub-workflow: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.executeworkflow/
- n8n Execute Sub-workflow Trigger: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.executeworkflowtrigger/
- n8n Webhook: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook/
- n8n Wait: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.wait/
- n8n Google Sheets: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets/
- n8n Google Sheets sheet operations: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets/sheet-operations/
- n8n OpenAI text operations: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-langchain.openai/text-operations/
- Slack Block Kit buttons: https://api.slack.com/block-kit/block-elements
- Slack interaction payloads: https://api.slack.com/reference/interaction-payloads/block-actions
- Slack handling interactivity: https://api.slack.com/interactivity/handling

## Done

Once built, this system will:

- intake wholesale leads automatically
- extract and validate domains
- skip redundant research for 30 days
- run a two-pass deep research loop when needed
- score and tier leads against a realistic ICP
- generate premium pitches for strong leads
- require human approval for ambiguous leads
- log everything in a simple auditable spreadsheet

If you want, the next thing I can do is create a shorter companion checklist file for this workflow too, the same way I offered for the email-triage guide.

##  postwork Project
```

Autonomous Intelligence Agent for Wholesale Lead Enrichment
Overview Heirloom & Hide, a boutique manufacturer of handmade leather goods for the hospitality and corporate gifting sector, is seeking an n8n expert to build a proactive research agent. The Operations Manager currently spends hours manually researching wholesale inquiries to distinguish high-value hotel chains from small one-off buyers. This project involves building a multi-turn research loop in n8n that automatically enriches leads, reasons against an Ideal Customer Profile (ICP), and manages decision-branching through a Human-in-the-Loop (HITL) interface.

Key Requirements

Modular Orchestration: Build a main workflow that utilizes the "Execute Workflow" node to trigger a sub-workflow dedicated specifically to "Deep Research" (keeping the logic clean and modular).
Multi-Turn Enrichment Loop: Upon receiving a lead (email/domain), the agent must query a public API (such as Apollo, Clearbit, or a Google Search tool) to identify company size, location, and recent industry news.
AI Reasoning Node: Implement an n8n AI Agent or LLM node that compares enriched lead data against a JSON-formatted ICP (provided by the freelancer as a mock reference) to determine "Match Tiers."
Dynamic Decision Branching:
Tier 1 (High Match): The agent must scrape the lead's website (using a tool like Firecrawl or a simple HTTP request) to find a specific "hook" (e.g., a recent award or a specific brand value) and generate a draft pitch.
Tier 2 (Partial Match): Route to a Slack channel with interactive buttons (Approve/Reject) to pause the workflow until a human confirms the lead's quality.
Tier 3 (Low Match): Automatically log to a "Nurture" tab in a Google Sheet.
State Management: Use a Google Sheet or n8n internal memory to check if a domain has been researched in the last 30 days to prevent redundant API calls and credit waste.
Advanced Logging: Maintain a two-tab Google Sheet system: Tab 1 for "Raw Leads" and Tab 2 for "Deep Intelligence," connected by a unique Lead ID.
Deliverables

Workflow Export: The primary .json export of the n8n workflow(s).
Technical Documentation: A brief walkthrough (in Markdown) explaining how the "State Engine" prevents redundant lookups.
Screenshots: Visual confirmation of the n8n canvas, the Slack Interactive message interface, and the two-tab Google Sheet structure.
Screen Recording: A short video (2-3 minutes) showing a "High Match" lead flowing through the research loop and a "Partial Match" lead triggering the Slack HITL buttons.
Acceptance Criteria

The workflow successfully distinguishes between lead tiers based on reasoning, not just simple "if/else" keyword matching.
The Slack integration correctly pauses the execution and resumes based on the button click.
The "State Engine" correctly identifies a repeat domain and skips the research phase.
Error handling is implemented for failed API calls or empty search results.
Your work will be judged on the sophistication of your prompt engineering within the reasoning node and the modularity of the workflow design.

```
# Instructor Talent & Certification Screener for Shadow-Kick Studios

This guide is written so you can build the workflow manually in n8n yourself, step by step, without depending on a prebuilt import.

It implements:

- 1 main workflow for intake, AI classification, availability matching, branching, Slack alerting, and Google Sheets logging
- 1 error workflow for central error capture

It uses only real n8n nodes:

- `Webhook`
- `HTTP Request`
- `Code`
- `Merge`
- `Edit Fields (Set)`
- `IF`
- `Switch`
- `Google Sheets`
- `Slack`
- `Error Trigger`

This build uses `HTTP Request` for OpenAI instead of the OpenAI node so the AI step is fully explicit and easy to audit.

## What this workflow does

1. Accepts an inbound application from a webhook.
2. Normalizes freeform application data.
3. Sends the application text to OpenAI with a strict JSON-only screening prompt.
4. Extracts:
   - `category`
   - `expertise_score`
   - `key_skills`
   - `certifications`
   - `missing_data`
5. Reads current instructor demand from a Google Sheet.
6. Compares applicant availability against schedule needs.
7. Marks the candidate as:
   - `high_priority`
   - `standard`
   - `incomplete`
8. Sends high-priority candidates to Slack.
9. Writes standard candidates to `Talent Pool`.
10. Writes incomplete candidates to `Needs Review`.
11. Logs workflow errors into an `Error Logs` tab through a separate error workflow.

---

## Before you build

Create these assets first.

### 1. Google Sheet

Create one Google Sheet.

Recommended spreadsheet name:

`Shadow-Kick Studios Instructor Screening`

Create these tabs exactly:

1. `Scheduling Needs`
2. `Talent Pool`
3. `Needs Review`
4. `Error Logs`

Create this exact header row in `Scheduling Needs`:

```text
Day | TimeSlot | Priority
```

Example rows:

```text
Tuesday | Evening | High
Thursday | Evening | High
Saturday | Morning | Medium
```

Create this exact header row in `Talent Pool`:

```text
Application ID | Timestamp | Name | Category | Expertise Score | Availability Match | Matched Slots | Availability | Key Skills | Certifications | Score Rationale | Route | Application Text
```

Create this exact header row in `Needs Review`:

```text
Application ID | Timestamp | Name | Category | Expertise Score | Availability Match | Matched Slots | Availability | Missing Data | Key Skills | Certifications | Score Rationale | Route | Application Text
```

Create this exact header row in `Error Logs`:

```text
Timestamp | Workflow Name | Workflow ID | Execution ID | Failed Node | Error Message | Error Stack
```

### 2. OpenAI API key

Set this environment variable for your n8n instance:

```text
OPENAI_API_KEY
```

This workflow reads it with:

```text
{{$env.OPENAI_API_KEY}}
```

### 3. Slack

Create or choose one Slack channel for urgent instructor alerts.

You need:

- a working `Slack OAuth2` credential in n8n
- the target channel ID

### 4. Google Sheets credential

Create a `Google Sheets OAuth2` credential in n8n and make sure the Google account behind it can edit the spreadsheet.

---

## Workflow names

Create these two workflows with these exact names:

1. `Shadow-Kick Studios - Instructor Talent & Certification Screener`
2. `Shadow-Kick Studios - Instructor Screener Error Logger`

---

# STEP 0: Create both workflows

## 0A. Create the main workflow

1. Open n8n.
2. Click `Create Workflow`.
3. Rename it to:

```text
Shadow-Kick Studios - Instructor Talent & Certification Screener
```

4. Save once before adding nodes.

You will create these nodes in this exact order:

1. `Webhook - Instructor Application`
2. `Code - Normalize Application`
3. `HTTP Request - OpenAI Classify Instructor`
4. `Code - Parse AI Response`
5. `Merge - Applicant + AI`
6. `Set - Screening Record`
7. `Google Sheets - Read Scheduling Needs`
8. `Merge - Applicant + Needs`
9. `Code - Match Availability & Decide Route`
10. `IF - High Priority?`
11. `Slack - High Priority Alert`
12. `Switch - Standard or Incomplete`
13. `Google Sheets - Append Talent Pool`
14. `Google Sheets - Append Needs Review`

## 0B. Create the error workflow

1. Click `Create Workflow`.
2. Rename it to:

```text
Shadow-Kick Studios - Instructor Screener Error Logger
```

3. Save once.

You will create these nodes in this exact order:

1. `Error Trigger`
2. `Set - Prepare Error Log`
3. `Google Sheets - Log Error`

---

# STEP 1: Intake webhook

## Add the webhook

1. Add a `Webhook` node.
2. Rename it to:

```text
Webhook - Instructor Application
```

## Configure it

Set these fields exactly:

- `HTTP Method`: `POST`
- `Path`:

```text
shadow-kick-instructor-screener
```

Leave the rest at default values.

## Expected input payload

The webhook expects JSON like this:

```json
{
  "name": "John Doe",
  "application_text": "4th Dan Taekwondo instructor with youth teaching experience and first aid certification. Available Tuesday Evening and Thursday Morning.",
  "availability": ["Tuesday Evening", "Thursday Morning"]
}
```

---

# STEP 2: Normalize the application payload

## Add the Code node

1. Add a `Code` node after the webhook.
2. Rename it to:

```text
Code - Normalize Application
```

## Configure it

Set:

- `Mode`: `Run Once for All Items`
- `Language`: `JavaScript`

Paste this exact code:

```javascript
const source = $input.first().json.body ?? $input.first().json;
const name = String(source.name ?? '').trim();
const applicationText = String(source.application_text ?? '').trim();

let availability = source.availability ?? [];
if (typeof availability === 'string') {
  availability = [availability];
}
if (!Array.isArray(availability)) {
  availability = [];
}

availability = availability
  .map((slot) => String(slot).trim())
  .filter(Boolean);

return [
  {
    json: {
      name,
      application_text: applicationText,
      availability,
      received_at: new Date().toISOString()
    }
  }
];
```

## What this node does

It:

- accepts either raw webhook JSON or `body`-wrapped webhook JSON
- trims the candidate name and application text
- normalizes `availability` into a clean array
- adds `received_at`

---

# STEP 3: AI classification

## Add the HTTP Request node

1. Add an `HTTP Request` node after `Code - Normalize Application`.
2. Rename it to:

```text
HTTP Request - OpenAI Classify Instructor
```

## Configure it

Set these fields exactly:

- `Method`: `POST`
- `URL`:

```text
https://api.openai.com/v1/chat/completions
```

- `Send Headers`: `On`
- `Send Body`: `On`
- `Specify Body`: `JSON`

Under `Headers`, add:

- `Authorization`:

```text
={{ 'Bearer ' + $env.OPENAI_API_KEY }}
```

- `Content-Type`:

```text
application/json
```

Under `Options > Response`, set:

- `Response Format`: `JSON`

## JSON body

Paste this exact expression into `JSON Body`:

```javascript
={{ {
  model: 'gpt-4o-mini',
  temperature: 0,
  response_format: { type: 'json_object' },
  messages: [
    {
      role: 'system',
      content: 'You are a deterministic screening classifier for Shadow-Kick Studios. Classify only from explicit evidence in the provided application text. Never invent qualifications, certifications, experience, or schedule details. Use these category rules exactly: Dan rankings, black belt references, karate, taekwondo, kickboxing, judo, BJJ, Muay Thai, sparring, or martial arts instructor evidence => Martial Arts. RAD, ISTD, ballet, jazz, hip-hop, choreography, dance teacher, or dance certification evidence => Dance. If both martial arts and dance evidence are clearly present => Hybrid. Scoring logic for expertise_score: 1 = beginner or minimal evidence. 2 = some training evidence but limited teaching or credential evidence. 3 = solid practitioner or assistant-level teaching evidence. 4 = strong instructor or professional background, notable certifications, or advanced ranks. 5 = elite, master-level, or highly credentialed senior instructor evidence. Return JSON only with this exact schema: {"category":"Martial Arts|Dance|Hybrid","expertise_score":1,"key_skills":[],"certifications":[],"score_rationale":"","missing_data":[]}. Requirements: return only valid JSON; key_skills and certifications must be arrays of strings; score_rationale must briefly explain the score using only evidence from the application text; missing_data must list missing important details such as years_experience, teaching_history, certifications, age_groups, style_specialization, or schedule_constraints when absent.'
    },
    {
      role: 'user',
      content: `Candidate name: ${$json.name}
Availability: ${JSON.stringify($json.availability)}
Application text:
${$json.application_text}`
    }
  ]
} }}
```

## What this node returns

The response content should contain JSON with:

- `category`
- `expertise_score`
- `key_skills`
- `certifications`
- `score_rationale`
- `missing_data`

---

# STEP 4: Parse the AI JSON output

## Add the parser Code node

1. Add a `Code` node after the OpenAI request.
2. Rename it to:

```text
Code - Parse AI Response
```

## Configure it

Set:

- `Mode`: `Run Once for All Items`
- `Language`: `JavaScript`

Paste this exact code:

```javascript
const content = $input.first().json.choices?.[0]?.message?.content ?? '';
if (!content) {
  throw new Error('OpenAI returned an empty response');
}

const cleaned = String(content)
  .trim()
  .replace(/^```json/i, '')
  .replace(/^```/, '')
  .replace(/```$/, '')
  .trim();

let parsed;
try {
  parsed = JSON.parse(cleaned);
} catch (error) {
  throw new Error(`Unable to parse OpenAI JSON: ${cleaned}`);
}

const normalizeArray = (value) =>
  Array.isArray(value) ? value.map((item) => String(item).trim()).filter(Boolean) : [];

const categoryMap = {
  'martial arts': 'Martial Arts',
  dance: 'Dance',
  hybrid: 'Hybrid'
};

const rawCategory = String(parsed.category ?? '').trim().toLowerCase();
const category = categoryMap[rawCategory] ?? '';
const expertiseScore = Number(parsed.expertise_score);

const missingData = normalizeArray(parsed.missing_data);
if (!category) missingData.push('category');
if (!Number.isFinite(expertiseScore)) missingData.push('expertise_score');

return [
  {
    json: {
      category,
      expertise_score: Number.isFinite(expertiseScore)
        ? Math.max(1, Math.min(5, Math.round(expertiseScore)))
        : 1,
      key_skills: normalizeArray(parsed.key_skills),
      certifications: normalizeArray(parsed.certifications),
      score_rationale: String(parsed.score_rationale ?? '').trim(),
      missing_data: [...new Set(missingData)]
    }
  }
];
```

## Why this parser matters

It makes the workflow safer by:

- stripping markdown code fences if the model returns them
- parsing the JSON strictly
- normalizing arrays
- correcting category capitalization
- forcing `expertise_score` into the `1-5` range

---

# STEP 5: Merge the applicant input with the AI output

## Add the Merge node

1. Add a `Merge` node.
2. Rename it to:

```text
Merge - Applicant + AI
```

## Configure it

Set:

- `Mode`: `Combine`
- `Combine By`: `Position`
- `Number of Inputs`: `2`

## Connection rule

Connect:

- `Code - Normalize Application` to input `1`
- `Code - Parse AI Response` to input `2`

This combines the original application payload with the parsed AI classification into one item.

---

# STEP 6: Build the screening record

## Add the Edit Fields node

1. Add an `Edit Fields (Set)` node after `Merge - Applicant + AI`.
2. Rename it to:

```text
Set - Screening Record
```

## Configure it

Set:

- `Mode`: `Manual Mapping`
- `Include Other Input Fields`: `On`

Add these assignments exactly:

| Field Name | Type | Value |
| --- | --- | --- |
| `application_id` | `String` | `={{ 'app_' + Date.now() + '_' + ($json.name || 'candidate').toLowerCase().replace(/[^a-z0-9]+/g, '_').replace(/^_|_$/g, '') }}` |
| `record_type` | `String` | `applicant` |
| `name` | `String` | `={{ $json.name }}` |
| `category` | `String` | `={{ $json.category }}` |
| `expertise_score` | `Number` | `={{ $json.expertise_score }}` |
| `availability_display` | `String` | `={{ Array.isArray($json.availability) ? $json.availability.join(', ') : '' }}` |
| `key_skills_csv` | `String` | `={{ Array.isArray($json.key_skills) ? $json.key_skills.join(', ') : '' }}` |
| `certifications_csv` | `String` | `={{ Array.isArray($json.certifications) ? $json.certifications.join(', ') : '' }}` |
| `missing_data_csv` | `String` | `={{ Array.isArray($json.missing_data) ? $json.missing_data.join(', ') : '' }}` |
| `score_rationale` | `String` | `={{ $json.score_rationale }}` |

## Why this node exists

It turns the mixed webhook + AI payload into a consistent screening record that is easy to route and log.

---

# STEP 7: Read scheduling demand from Google Sheets

## Add the Google Sheets reader

1. Add a `Google Sheets` node after `Set - Screening Record`.
2. Rename it to:

```text
Google Sheets - Read Scheduling Needs
```

## Configure it

Set:

- `Credential`: your Google Sheets credential
- `Resource`: `Sheet Within Document`
- `Operation`: `Get Row(s)`
- `Document`: your sheet ID
- `Sheet`: `Scheduling Needs`

Do not add filters.

That makes the node read the whole `Scheduling Needs` tab so the next node can compare every demand row against the candidate's availability.

---

# STEP 8: Merge the applicant record with the schedule rows

## Add the second Merge node

1. Add a `Merge` node.
2. Rename it to:

```text
Merge - Applicant + Needs
```

## Configure it

Set:

- `Mode`: `Append`
- `Number of Inputs`: `2`

## Connection rule

Connect:

- `Set - Screening Record` to input `1`
- `Google Sheets - Read Scheduling Needs` to input `2`

This creates one input stream that contains:

- the applicant record
- all scheduling-needs rows

The next Code node will split these back apart and compare them.

---

# STEP 9: Availability matching and route decision

## Add the decision Code node

1. Add a `Code` node after `Merge - Applicant + Needs`.
2. Rename it to:

```text
Code - Match Availability & Decide Route
```

## Configure it

Set:

- `Mode`: `Run Once for All Items`
- `Language`: `JavaScript`

Paste this exact code:

```javascript
const items = $input.all();
const applicantItem = items.find((item) => item.json.record_type === 'applicant');

if (!applicantItem) {
  throw new Error('Applicant record was not found before availability matching');
}

const applicant = applicantItem.json;
const needsRows = items
  .filter((item) => item.json.record_type !== 'applicant')
  .map((item) => item.json);

const normalize = (value = '') =>
  String(value)
    .trim()
    .toLowerCase()
    .replace(/[_-]+/g, ' ')
    .replace(/\s+/g, ' ');

const availability = Array.isArray(applicant.availability) ? applicant.availability : [];
const availabilitySet = new Set(availability.map((slot) => normalize(slot)));

const matchedDetails = [];

for (const row of needsRows) {
  const day = String(row.Day ?? row.day ?? '').trim();
  const timeSlot = String(row.TimeSlot ?? row.timeSlot ?? row.timeslot ?? row['Time Slot'] ?? '').trim();
  const priority = String(row.Priority ?? row.priority ?? '').trim();

  if (!day && !timeSlot) {
    continue;
  }

  const label = `${day} ${timeSlot}`.trim();
  if (availabilitySet.has(normalize(label))) {
    matchedDetails.push({
      day,
      time_slot: timeSlot,
      priority,
      label
    });
  }
}

const expertise = Number(applicant.expertise_score ?? 0);
const aiMissing = Array.isArray(applicant.missing_data)
  ? applicant.missing_data.map((entry) => String(entry).trim()).filter(Boolean)
  : [];

const coreMissing = [];
if (!String(applicant.name ?? '').trim()) coreMissing.push('name');
if (!String(applicant.application_text ?? '').trim()) coreMissing.push('application_text');
if (!String(applicant.category ?? '').trim()) coreMissing.push('category');
if (!Array.isArray(applicant.availability) || applicant.availability.length === 0) coreMissing.push('availability');

const combinedMissing = [...new Set([...coreMissing, ...aiMissing])];
const availabilityMatch = matchedDetails.length > 0;
const incomplete = combinedMissing.length > 0 || expertise < 3;
const isHighPriority = !incomplete && expertise >= 4 && availabilityMatch;
const route = isHighPriority ? 'high_priority' : (!incomplete && expertise >= 3 ? 'standard' : 'incomplete');

return [
  {
    json: {
      ...applicant,
      expertise_score: Number.isFinite(expertise) ? expertise : 1,
      availability_match: availabilityMatch,
      matched_slots: matchedDetails.map((slot) => slot.label),
      matched_slot_details: matchedDetails,
      matched_slots_csv: matchedDetails
        .map((slot) => `${slot.label}${slot.priority ? ` (${slot.priority})` : ''}`)
        .join(', '),
      missing_data: combinedMissing,
      missing_data_csv: combinedMissing.join(', '),
      is_high_priority: isHighPriority,
      route,
      scheduling_needs_count: needsRows.length
    }
  }
];
```

## Decision logic

This node enforces these rules:

- `high_priority` if:
  - `expertise_score >= 4`
  - availability matches at least one schedule row
  - candidate is not incomplete
- `standard` if:
  - candidate is not incomplete
  - `expertise_score >= 3`
- `incomplete` if:
  - required data is missing
  - or `expertise_score < 3`

---

# STEP 10: High-priority branch

## 10A. Add the IF node

1. Add an `IF` node after `Code - Match Availability & Decide Route`.
2. Rename it to:

```text
IF - High Priority?
```

## Configure it

Add one condition:

- `Left Value`:

```text
={{$json.is_high_priority}}
```

- `Operator`: `Boolean -> Is True`

## TRUE and FALSE meaning

- `TRUE`: candidate is high priority
- `FALSE`: candidate is not high priority, so continue to the standard/incomplete split

## 10B. Add the Slack node on the TRUE branch

1. Add a `Slack` node to the `TRUE` output.
2. Rename it to:

```text
Slack - High Priority Alert
```

## Configure it

Set:

- `Authentication`: `OAuth2`
- `Send Message To`: `Channel`
- `Channel`: your target Slack channel ID
- `Message Type`: `Simple Text Message`
- `Message Text`:

```text
=🔥 High Priority Candidate: {{$json.name}} ({{$json.category}} - Score {{$json.expertise_score}})
```

Under `Other Options`, set:

- `Include Link to Workflow`: `Off`

## Important connection detail

Connect the same `TRUE` branch from `IF - High Priority?` to:

1. `Slack - High Priority Alert`
2. `Google Sheets - Append Talent Pool`

That ensures high-priority candidates are both alerted and written to the talent pool.

---

# STEP 11: Standard vs incomplete split

## Add the Switch node on the FALSE branch

1. Add a `Switch` node to the `FALSE` output of `IF - High Priority?`.
2. Rename it to:

```text
Switch - Standard or Incomplete
```

## Configure it

Use `Rules` mode with two outputs.

### Output 1

- `Output Name`: `Standard`
- Condition:
  - `Left Value`:

```text
={{ $json.route }}
```

  - `Operator`: `String -> Equals`
  - `Right Value`:

```text
standard
```

### Output 2

- `Output Name`: `Incomplete`
- Condition:
  - `Left Value`:

```text
={{ $json.route }}
```

  - `Operator`: `String -> Equals`
  - `Right Value`:

```text
incomplete
```

## Branch meaning

- `Standard` output goes to `Talent Pool`
- `Incomplete` output goes to `Needs Review`

---

# STEP 12: Talent Pool Google Sheets logger

## Add the Google Sheets node

1. Add a `Google Sheets` node.
2. Rename it to:

```text
Google Sheets - Append Talent Pool
```

## Configure it

Set:

- `Credential`: your Google Sheets credential
- `Resource`: `Sheet Within Document`
- `Operation`: `Append Row`
- `Document`: your Google Sheet ID
- `Sheet`: `Talent Pool`
- `Columns`: `Map Each Column Below`

Map these values exactly:

| Sheet Column | Value |
| --- | --- |
| `Application ID` | `={{ $json.application_id }}` |
| `Timestamp` | `={{ $json.received_at || $now }}` |
| `Name` | `={{ $json.name }}` |
| `Category` | `={{ $json.category }}` |
| `Expertise Score` | `={{ $json.expertise_score }}` |
| `Availability Match` | `={{ $json.availability_match }}` |
| `Matched Slots` | `={{ $json.matched_slots_csv }}` |
| `Availability` | `={{ $json.availability_display }}` |
| `Key Skills` | `={{ $json.key_skills_csv }}` |
| `Certifications` | `={{ $json.certifications_csv }}` |
| `Score Rationale` | `={{ $json.score_rationale }}` |
| `Route` | `={{ $json.route }}` |
| `Application Text` | `={{ $json.application_text }}` |

## Connection rule

Connect this node from:

- the `TRUE` output of `IF - High Priority?`
- the `Standard` output of `Switch - Standard or Incomplete`

That is intentional. Both high-priority and standard candidates belong in `Talent Pool`.

---

# STEP 13: Needs Review Google Sheets logger

## Add the Google Sheets node

1. Add another `Google Sheets` node.
2. Rename it to:

```text
Google Sheets - Append Needs Review
```

## Configure it

Set:

- `Credential`: your Google Sheets credential
- `Resource`: `Sheet Within Document`
- `Operation`: `Append Row`
- `Document`: your Google Sheet ID
- `Sheet`: `Needs Review`
- `Columns`: `Map Each Column Below`

Map these values exactly:

| Sheet Column | Value |
| --- | --- |
| `Application ID` | `={{ $json.application_id }}` |
| `Timestamp` | `={{ $json.received_at || $now }}` |
| `Name` | `={{ $json.name }}` |
| `Category` | `={{ $json.category }}` |
| `Expertise Score` | `={{ $json.expertise_score }}` |
| `Availability Match` | `={{ $json.availability_match }}` |
| `Matched Slots` | `={{ $json.matched_slots_csv }}` |
| `Availability` | `={{ $json.availability_display }}` |
| `Missing Data` | `={{ $json.missing_data_csv }}` |
| `Key Skills` | `={{ $json.key_skills_csv }}` |
| `Certifications` | `={{ $json.certifications_csv }}` |
| `Score Rationale` | `={{ $json.score_rationale }}` |
| `Route` | `={{ $json.route }}` |
| `Application Text` | `={{ $json.application_text }}` |

## Connection rule

Connect only the `Incomplete` output of `Switch - Standard or Incomplete` into this node.

---

# STEP 14: Error workflow

## 14A. Add the Error Trigger

In the second workflow:

1. Add an `Error Trigger` node.
2. Rename it to:

```text
Error Trigger
```

No extra configuration is needed.

## 14B. Add the error preparation node

1. Add an `Edit Fields (Set)` node after `Error Trigger`.
2. Rename it to:

```text
Set - Prepare Error Log
```

Add these assignments exactly:

| Field Name | Type | Value |
| --- | --- | --- |
| `Timestamp` | `String` | `={{ $now }}` |
| `Workflow Name` | `String` | `={{ $json.workflow?.name || '' }}` |
| `Workflow ID` | `String` | `={{ $json.workflow?.id || '' }}` |
| `Execution ID` | `String` | `={{ $json.execution?.id || '' }}` |
| `Failed Node` | `String` | `={{ $json.execution?.lastNodeExecuted || '' }}` |
| `Error Message` | `String` | `={{ $json.execution?.error?.message || '' }}` |
| `Error Stack` | `String` | `={{ $json.execution?.error?.stack || '' }}` |

## 14C. Add the error logger

1. Add a `Google Sheets` node after `Set - Prepare Error Log`.
2. Rename it to:

```text
Google Sheets - Log Error
```

Set:

- `Credential`: your Google Sheets credential
- `Resource`: `Sheet Within Document`
- `Operation`: `Append Row`
- `Document`: your Google Sheet ID
- `Sheet`: `Error Logs`

Map these columns:

| Sheet Column | Value |
| --- | --- |
| `Timestamp` | `={{ $json['Timestamp'] }}` |
| `Workflow Name` | `={{ $json['Workflow Name'] }}` |
| `Workflow ID` | `={{ $json['Workflow ID'] }}` |
| `Execution ID` | `={{ $json['Execution ID'] }}` |
| `Failed Node` | `={{ $json['Failed Node'] }}` |
| `Error Message` | `={{ $json['Error Message'] }}` |
| `Error Stack` | `={{ $json['Error Stack'] }}` |

## 14D. Attach the error workflow to the main workflow

In the main workflow:

1. Open workflow settings.
2. Set the workflow error handler to:

```text
Shadow-Kick Studios - Instructor Screener Error Logger
```

---

# STEP 15: Testing scenarios

## Test 1: High-priority martial arts candidate

Send:

```json
{
  "name": "Amanuel Bekele",
  "application_text": "4th Dan Taekwondo instructor with 8 years of youth teaching experience, first aid certification, and competition coaching background.",
  "availability": ["Tuesday Evening", "Thursday Evening"]
}
```

Expected result:

- category becomes `Martial Arts`
- score is likely `4` or `5`
- availability matches
- Slack alert is sent
- candidate is written to `Talent Pool`

## Test 2: Standard dance candidate

Send:

```json
{
  "name": "Selam Tadesse",
  "application_text": "Dance instructor with RAD ballet training, choreography experience, and assistant teaching history.",
  "availability": ["Monday Morning"]
}
```

Expected result:

- category becomes `Dance`
- score is likely `3` or `4`
- if no high-priority availability match exists, route becomes `standard`
- candidate is written to `Talent Pool`

## Test 3: Incomplete candidate

Send:

```json
{
  "name": "Candidate X",
  "application_text": "I like fitness and have helped friends train.",
  "availability": []
}
```

Expected result:

- low score or missing data
- route becomes `incomplete`
- candidate is written to `Needs Review`

---

## Full connection map

### Main workflow

```text
Webhook - Instructor Application
  -> Code - Normalize Application

Code - Normalize Application
  -> HTTP Request - OpenAI Classify Instructor
  -> Merge - Applicant + AI (Input 1)

HTTP Request - OpenAI Classify Instructor
  -> Code - Parse AI Response

Code - Parse AI Response
  -> Merge - Applicant + AI (Input 2)

Merge - Applicant + AI
  -> Set - Screening Record

Set - Screening Record
  -> Google Sheets - Read Scheduling Needs
  -> Merge - Applicant + Needs (Input 1)

Google Sheets - Read Scheduling Needs
  -> Merge - Applicant + Needs (Input 2)

Merge - Applicant + Needs
  -> Code - Match Availability & Decide Route

Code - Match Availability & Decide Route
  -> IF - High Priority?

IF - High Priority? TRUE
  -> Slack - High Priority Alert
  -> Google Sheets - Append Talent Pool

IF - High Priority? FALSE
  -> Switch - Standard or Incomplete

Switch - Standard or Incomplete: Standard
  -> Google Sheets - Append Talent Pool

Switch - Standard or Incomplete: Incomplete
  -> Google Sheets - Append Needs Review
```

### Error workflow

```text
Error Trigger
  -> Set - Prepare Error Log
  -> Google Sheets - Log Error
```

---

## Why this build is production-friendly

- The OpenAI prompt is deterministic and JSON-only.
- AI parsing is strict and rejects malformed output.
- Availability matching is normalized to reduce formatting mismatches.
- High-priority routing requires both strong expertise and real schedule fit.
- Standard and incomplete branches are separated cleanly with `IF` plus `Switch`.
- Errors are centralized into a dedicated logging workflow.

---

## Done

At this point you will have:

- a public webhook intake for instructor applications
- AI-based category and expertise classification
- availability matching against real scheduling demand
- Slack alerting for urgent candidates
- Google Sheets logging for talent pool and review queues
- a dedicated error logging workflow

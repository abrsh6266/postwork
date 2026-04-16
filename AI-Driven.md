# AI-Driven Triage & Response System for Wildwood & Wax

This guide is written so you can build the workflow manually in n8n yourself.

It assumes:

- You are using a recent n8n version with the current `Gmail Trigger`, `OpenAI`, `Gmail`, `Wait`, `IF`, `Edit Fields`, `Google Sheets`, and `HTTP Request` nodes.
- You have these credentials ready in n8n:
  - `Gmail OAuth2`
  - `OpenAI`
  - `Google Sheets OAuth2`
  - A Slack Incoming Webhook URL

Use this exact workflow name:

`Wildwood & Wax - Virtual Triage`

## What this workflow does

1. Watches a Gmail inbox for new incoming emails.
2. Pre-filters out out-of-office replies, auto-generated email, marketing spam, malformed senders, empty emails, and loop-risk emails.
3. Uses OpenAI to classify the email into:
   - `Shipping Issue`
   - `Subscription Change`
   - `Wholesale Inquiry`
   - `Spam`
4. Assigns urgency:
   - `Low`
   - `Medium`
   - `High`
5. Generates a short summary.
6. Stops if the email is spam.
7. Sends a Slack alert for high-urgency items.
8. Drafts a warm, premium, customer-safe reply.
9. Sends the reply through Gmail.
10. Logs every outcome to Google Sheets.

---

## Before you build

Create these assets first.

### 1. Gmail account

Use the same Gmail account for:

- `Gmail Trigger`
- `Gmail - Reply to Sender`

Recommended inbox aliases to account for in the pre-filter:

- `support@wildwoodandwax.com`
- `hello@wildwoodandwax.com`
- `orders@wildwoodandwax.com`

If your actual sender addresses are different, replace them everywhere in this guide.

### 2. Slack webhook

Create a Slack Incoming Webhook and keep the full URL ready.

Example placeholder:

```text
https://hooks.slack.com/services/REPLACE/THIS/URL
```

### 3. Google Sheet

Create one Google Sheet.

Spreadsheet name:

`Wildwood & Wax Email Triage`

Sheet tab name:

`Email Triage Log`

Create this exact header row in row 1:

```text
Timestamp | Sender Email | Subject | Intent | Urgency | AI Summary | Action Taken | AI Response
```

### 4. OpenAI model

Use:

`gpt-4o-mini`

This is a good balance of cost, speed, and reliability for triage + reply drafting.

---

# STEP 0: Create workflow

1. Open n8n.
2. Click `Create Workflow`.
3. In the top-left workflow title, rename it to:

```text
Wildwood & Wax - Virtual Triage
```

4. Save once before adding nodes.

You will create these nodes in this exact order:

1. `Gmail Trigger`
2. `Normalize Email`
3. `IF Ignore Pre-Filter?`
4. `Prepare Ignored Log`
5. `Google Sheets - Log Ignored`
6. `Wait - Rate Limit Buffer`
7. `OpenAI - Classify Email`
8. `Parse Classification JSON`
9. `IF Spam?`
10. `Prepare Spam Log`
11. `Google Sheets - Log Spam`
12. `IF High Urgency?`
13. `Slack Alert Webhook`
14. `OpenAI - Draft Reply`
15. `Prepare Final Reply`
16. `Gmail - Reply to Sender`
17. `Prepare Success Log`
18. `Google Sheets - Log Success`

---

# STEP 1: Gmail Trigger setup

## Add the trigger

1. Click `Add first step`.
2. Search for `Gmail Trigger`.
3. Add the node.
4. Rename the node to:

```text
Gmail Trigger
```

## Configure the node

Set these fields exactly:

- `Credential to connect with`: your Gmail credential
- `Poll Times`
  - `Mode`: `Every X`
  - `Value`: `2`
  - `Unit`: `Minutes`
- `Simplify`: `On`

In `Filters`, set:

- `Include Spam and Trash`: `Off`
- `Search`:

```text
-from:(support@wildwoodandwax.com OR hello@wildwoodandwax.com OR orders@wildwoodandwax.com) -label:SENT
```

- `Read Status`: `Unread emails only`

Notes:

- This search filter reduces self-trigger loops before the workflow even starts.
- We still add a deeper loop-prevention check later in Code, because the trigger filter alone is not enough for production.

## What this node gives us

From the Gmail Trigger output, we will use:

- `From`
- `Subject`
- `id`
- `threadId`
- Any available body/snippet fields such as `textPlain`, `text`, `snippet`, `body`, `textHtml`, or `html`

---

# STEP 2: Pre-filtering logic

This step handles:

- loop prevention
- self-email ignoring
- bot sender ignoring
- out-of-office detection
- auto-generated email detection
- marketing spam pre-filtering
- malformed sender validation
- empty body validation

## 2A. Add the normalization Code node

1. Click the `+` after `Gmail Trigger`.
2. Add a `Code` node.
3. Rename it to:

```text
Normalize Email
```

4. Set:
   - `Mode`: `Run Once for Each Item`

5. Paste this exact JavaScript into `JavaScript Code`:

```javascript
const rawFrom = ($json.From || $json.from || '').toString();
const rawSubject = ($json.Subject || $json.subject || '(No Subject)').toString().trim();
const rawText = (
  $json.textPlain ||
  $json.text ||
  $json.snippet ||
  $json.body ||
  ''
).toString();
const htmlText = ($json.textHtml || $json.html || '').toString();

const bodyText = (rawText || htmlText)
  .replace(/<[^>]*>/g, ' ')
  .replace(/&nbsp;/g, ' ')
  .replace(/\s+/g, ' ')
  .trim();

const senderMatch = rawFrom.match(/<([^>]+)>/);
const senderEmail = (senderMatch?.[1] || rawFrom)
  .replace(/"/g, '')
  .trim()
  .toLowerCase();
const senderName =
  rawFrom.replace(/<[^>]+>/g, '').replace(/"/g, '').trim() || senderEmail;

const subjectLower = rawSubject.toLowerCase();
const bodyLower = bodyText.toLowerCase();
const combined = `${subjectLower}\n${bodyLower}`;

const ownAddresses = [
  'support@wildwoodandwax.com',
  'hello@wildwoodandwax.com',
  'orders@wildwoodandwax.com'
];

const autoReplyPatterns = [
  /out of office/i,
  /auto[ -]?reply/i,
  /automatic reply/i,
  /vacation/i,
  /away from the office/i,
  /i am currently out/i,
  /i'm currently out/i
];

const autoGeneratedPatterns = [
  /delivery status notification/i,
  /undeliverable/i,
  /mail delivery subsystem/i,
  /failure notice/i,
  /auto-submitted:\s*auto-generated/i
];

const marketingPatterns = [
  /seo service/i,
  /lead generation/i,
  /buy our/i,
  /marketing service/i,
  /guest post/i,
  /backlink/i,
  /boost your traffic/i,
  /cold outreach/i,
  /appointment setter/i,
  /social media growth/i
];

const botSenderPatterns = [
  /mailer-daemon/i,
  /postmaster/i,
  /no-?reply/i,
  /do-?not-?reply/i,
  /notification/i,
  /bounce/i,
  /automated/i
];

const malformedEmail = !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(senderEmail);
const emptyContent = !rawSubject && !bodyText;
const isFromSelf = ownAddresses.includes(senderEmail);
const isBotSender = botSenderPatterns.some((pattern) => pattern.test(rawFrom));
const isAutoReply = autoReplyPatterns.some(
  (pattern) => pattern.test(subjectLower) || pattern.test(bodyLower)
);
const isAutoGenerated = autoGeneratedPatterns.some((pattern) =>
  pattern.test(combined)
);
const isMarketingSpam = marketingPatterns.some((pattern) =>
  pattern.test(combined)
);

let ignoreReason = '';

if (isFromSelf) ignoreReason = 'self_email';
else if (isBotSender) ignoreReason = 'bot_sender';
else if (isAutoReply) ignoreReason = 'out_of_office';
else if (isAutoGenerated) ignoreReason = 'auto_generated';
else if (isMarketingSpam) ignoreReason = 'marketing_spam';
else if (malformedEmail) ignoreReason = 'malformed_sender';
else if (emptyContent) ignoreReason = 'empty_subject_and_body';

return [
  {
    json: {
      ...$json,
      sender_email: senderEmail,
      sender_name: senderName,
      subject_clean: rawSubject || '(No Subject)',
      body_clean: bodyText || rawSubject || '',
      message_id: $json.id || $json.messageId || '',
      thread_id: $json.threadId || '',
      received_at: new Date().toISOString(),
      prefilter_ignore: Boolean(ignoreReason),
      ignore_reason: ignoreReason
    }
  }
];
```

Important:

- Replace the three email addresses in `ownAddresses` with your real support mailbox identities if needed.

## 2B. Add the pre-filter IF node

1. Click `+` after `Normalize Email`.
2. Add an `IF` node.
3. Rename it to:

```text
IF Ignore Pre-Filter?
```

4. Configure the condition:

- `Left Value`:

```javascript
={{ $json.prefilter_ignore }}
```

- `Operator`: `is true`

## TRUE and FALSE meaning

- `TRUE` branch:
  - ignore the email
  - log it
  - stop
- `FALSE` branch:
  - continue to AI classification

## 2C. Add the ignored-log preparation node

1. Connect the `TRUE` output of `IF Ignore Pre-Filter?` to a new `Edit Fields` node.
2. Rename it to:

```text
Prepare Ignored Log
```

3. Set:
   - `Mode`: `JSON Output`

4. Paste this exact JSON into `JSON Output`:

```json
{
  "Timestamp": "={{ $json.received_at }}",
  "Sender Email": "={{ $json.sender_email }}",
  "Subject": "={{ $json.subject_clean }}",
  "Intent": "Ignored",
  "Urgency": "Low",
  "AI Summary": "={{ 'Ignored during pre-filter: ' + $json.ignore_reason }}",
  "Action Taken": "={{ 'Ignored - ' + $json.ignore_reason }}",
  "AI Response": ""
}
```

This node prepares data with column names that match Google Sheets exactly.

## 2D. Add the first Google Sheets logger

1. Connect `Prepare Ignored Log` to a new `Google Sheets` node.
2. Rename it to:

```text
Google Sheets - Log Ignored
```

3. Set these fields:

- `Credential to connect with`: your Google Sheets credential
- `Resource`: `Sheet Within Document`
- `Operation`: `Append Row`
- `Document`: your spreadsheet `Wildwood & Wax Email Triage`
- `Sheet`: `Email Triage Log`
- `Mapping Column Mode`: `Map Automatically`

4. In `Options`, set:

- `Use Append`: `On`
- `Handling extra fields in input`: `Ignore Them`

The ignored branch ends here.

## 2E. Add the Wait node on the FALSE branch

1. Connect the `FALSE` output of `IF Ignore Pre-Filter?` to a new `Wait` node.
2. Rename it to:

```text
Wait - Rate Limit Buffer
```

3. Set:

- `Resume`: `After Time Interval`
- `Wait Amount`: `1`
- `Wait Unit`: `Seconds`

This reduces burst pressure on OpenAI and Gmail when multiple messages arrive close together.

---

# STEP 3: OpenAI classification

## Add the classification node

1. Click `+` after `Wait - Rate Limit Buffer`.
2. Add an `OpenAI` node.
3. Rename it to:

```text
OpenAI - Classify Email
```

## Configure the node

Set these fields:

- `Credential to connect with`: your OpenAI credential
- `Resource`: `Text`
- `Operation`: `Generate a Model Response`
- `Model`: `gpt-4o-mini`

In `Messages`, add two message rows.

### Message 1

- `Message Type`: `Text`
- `Role`: `System`
- `Text`:

```text
You are an email triage model for Wildwood & Wax, a premium candle and home fragrance ecommerce brand.

Classify the incoming email into exactly one intent:
- Shipping Issue
- Subscription Change
- Wholesale Inquiry
- Spam

Also assign exactly one urgency:
- Low
- Medium
- High

Rules:
- Shipping damage, missing orders, broken items, wrong item, or upset customers = Shipping Issue.
- Skip, pause, cancel, change frequency, change address for a subscription = Subscription Change.
- Reseller, stockist, retailer, distributor, bulk order, partnership, wholesale pricing = Wholesale Inquiry.
- Cold outreach, SEO, backlink, marketing pitch, irrelevant sales message, or obvious junk = Spam.
- Use High urgency for broken/damaged orders, charge disputes, angry customers, or time-sensitive delivery issues.
- Use Medium urgency for subscription requests or non-angry customer problems.
- Use Low urgency for wholesale inquiries and low-risk questions.
- Summary must be one sentence, 10 to 25 words, plain English.
- Return JSON only.
- Do not wrap the JSON in markdown.
- Do not add extra keys.

Required JSON shape:
{
  "intent": "Shipping Issue | Subscription Change | Wholesale Inquiry | Spam",
  "urgency": "Low | Medium | High",
  "summary": "short summary"
}
```

### Message 2

- `Message Type`: `Text`
- `Role`: `User`
- `Text`:

```javascript
=Subject: {{ $('Normalize Email').item.json.subject_clean }}

Sender: {{ $('Normalize Email').item.json.sender_email }}

Body:
{{ $('Normalize Email').item.json.body_clean }}
```

## Options

Open `Options` and set:

- `Maximum Number of Tokens`: `350`
- `Output Randomness (Temperature)`: `0.1`
- `Output Format`: `JSON Object`

## Reliability settings

Open the `Settings` tab and set:

- `Retry On Fail`: `On`
- `Max Tries`: `3`
- `Wait Between Tries (ms)`: `2000`
- `On Error`: `Continue`

This is important. If OpenAI fails or returns malformed output, the next Code node will apply a safe heuristic fallback.

---

# STEP 4: Parse JSON output

This step validates the model output and adds a fallback classifier if the model fails or returns bad JSON.

## Add the parser Code node

1. Click `+` after `OpenAI - Classify Email`.
2. Add a `Code` node.
3. Rename it to:

```text
Parse Classification JSON
```

4. Set:
   - `Mode`: `Run Once for Each Item`

5. Paste this exact JavaScript into `JavaScript Code`:

```javascript
const source = $('Normalize Email').item.json;

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

function heuristicClassify(text) {
  const lower = text.toLowerCase();

  if (
    /broken|damaged|arrived broken|wrong item|missing order|where is my order|late delivery|shipping|refund|urgent/i.test(lower)
  ) {
    return {
      intent: 'Shipping Issue',
      urgency: /broken|damaged|refund|urgent|angry|upset|immediately/i.test(lower)
        ? 'High'
        : 'Medium',
      summary: 'Customer reported a shipping or order problem and needs support.'
    };
  }

  if (
    /subscription|cancel my subscription|pause my subscription|skip this month|change frequency|update subscription|change address/i.test(lower)
  ) {
    return {
      intent: 'Subscription Change',
      urgency: 'Medium',
      summary: 'Customer wants to update, pause, skip, or cancel their subscription.'
    };
  }

  if (
    /wholesale|stockist|retailer|retail partner|bulk order|distributor|reseller|partnership/i.test(lower)
  ) {
    return {
      intent: 'Wholesale Inquiry',
      urgency: 'Low',
      summary: 'Sender is asking about wholesale, stockist, or business partnership opportunities.'
    };
  }

  if (
    /seo|marketing service|lead generation|guest post|backlink|boost your traffic|appointment setter|cold outreach/i.test(lower)
  ) {
    return {
      intent: 'Spam',
      urgency: 'Low',
      summary: 'Message appears to be promotional spam or irrelevant outreach.'
    };
  }

  return {
    intent: 'Wholesale Inquiry',
    urgency: 'Low',
    summary: 'Message needs manual review because classification confidence was low.'
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

const fallback = heuristicClassify(`${source.subject_clean}\n${source.body_clean}`);

const validIntents = [
  'Shipping Issue',
  'Subscription Change',
  'Wholesale Inquiry',
  'Spam'
];
const validUrgencies = ['Low', 'Medium', 'High'];

const intent = validIntents.includes(parsed?.intent) ? parsed.intent : fallback.intent;
const urgency = validUrgencies.includes(parsed?.urgency)
  ? parsed.urgency
  : fallback.urgency;
const summary = String(parsed?.summary || fallback.summary || 'No summary available.')
  .replace(/\s+/g, ' ')
  .trim()
  .slice(0, 220);

return [
  {
    json: {
      ...source,
      intent,
      urgency,
      ai_summary: summary,
      classification_source: parsed ? 'openai' : 'heuristic_fallback'
    }
  }
];
```

What this node guarantees:

- valid `intent`
- valid `urgency`
- short summary
- fallback even if OpenAI output is bad

---

# STEP 5: IF routing logic

This is the main branching stage.

## 5A. Add the spam IF node

1. Click `+` after `Parse Classification JSON`.
2. Add an `IF` node.
3. Rename it to:

```text
IF Spam?
```

4. Configure the condition:

- `Left Value`:

```javascript
={{ $json.intent }}
```

- `Operator`: `is equal to`
- `Right Value`:

```text
Spam
```

## TRUE and FALSE meaning

- `TRUE` branch:
  - spam
  - log it
  - stop workflow
- `FALSE` branch:
  - not spam
  - continue

## 5B. Add the spam log preparation node

1. Connect the `TRUE` output of `IF Spam?` to a new `Edit Fields` node.
2. Rename it to:

```text
Prepare Spam Log
```

3. Set:
   - `Mode`: `JSON Output`

4. Paste this exact JSON into `JSON Output`:

```json
{
  "Timestamp": "={{ $json.received_at }}",
  "Sender Email": "={{ $json.sender_email }}",
  "Subject": "={{ $json.subject_clean }}",
  "Intent": "Spam",
  "Urgency": "Low",
  "AI Summary": "={{ $json.ai_summary }}",
  "Action Taken": "Ignored - spam",
  "AI Response": ""
}
```

## 5C. Add the spam Google Sheets logger

1. Connect `Prepare Spam Log` to a new `Google Sheets` node.
2. Rename it to:

```text
Google Sheets - Log Spam
```

3. Configure it exactly the same as the ignored logger:

- `Credential to connect with`: your Google Sheets credential
- `Resource`: `Sheet Within Document`
- `Operation`: `Append Row`
- `Document`: `Wildwood & Wax Email Triage`
- `Sheet`: `Email Triage Log`
- `Mapping Column Mode`: `Map Automatically`

4. In `Options`, set:

- `Use Append`: `On`
- `Handling extra fields in input`: `Ignore Them`

The spam branch ends here.

## 5D. Add the urgency IF node on the FALSE branch

1. Connect the `FALSE` output of `IF Spam?` to a new `IF` node.
2. Rename it to:

```text
IF High Urgency?
```

3. Configure the condition:

- `Left Value`:

```javascript
={{ $json.urgency }}
```

- `Operator`: `is equal to`
- `Right Value`:

```text
High
```

## TRUE and FALSE meaning

- `TRUE` branch:
  - send Slack alert
  - still continue to reply generation
- `FALSE` branch:
  - skip Slack
  - continue to reply generation

---

# STEP 6: Slack alert setup

For Slack, use an `HTTP Request` node with a Slack Incoming Webhook. This is simple and reliable.

## Add the Slack webhook node

1. Connect the `TRUE` output of `IF High Urgency?` to a new `HTTP Request` node.
2. Rename it to:

```text
Slack Alert Webhook
```

## Configure the node

Set these fields:

- `Method`: `POST`
- `URL`: your Slack Incoming Webhook URL
- `Send Headers`: `On`

In `Header Parameters`, add:

- `Name`: `Content-Type`
- `Value`: `application/json`

Then set:

- `Send Body`: `On`
- `Specify Body`: `Using JSON`

In `JSON Body`, click `Expression` and paste this exact value:

```javascript
={{
  {
    "text": `:rotating_light: *Wildwood & Wax High-Urgency Email*
*Sender:* ${$json.sender_email}
*Intent:* ${$json.intent}
*Urgency:* ${$json.urgency}
*Subject:* ${$json.subject_clean}
*Summary:* ${$json.ai_summary}
*Action:* Slack alert + auto-reply queued`
  }
}}
```

## Reliability settings

Open the `Settings` tab and set:

- `Retry On Fail`: `On`
- `Max Tries`: `3`
- `Wait Between Tries (ms)`: `2000`
- `On Error`: `Continue`

## Important connection detail

Do not put Slack in the only path to the reply node.

Make these two connections:

1. Connect `TRUE` from `IF High Urgency?` to `Slack Alert Webhook`
2. Also connect `TRUE` from `IF High Urgency?` directly to `OpenAI - Draft Reply`

Then:

3. Connect `FALSE` from `IF High Urgency?` directly to `OpenAI - Draft Reply`

This means:

- High urgency sends Slack and still continues to reply generation.
- Medium/Low skips Slack and still continues to reply generation.

## Example Slack message

```text
:rotating_light: Wildwood & Wax High-Urgency Email
Sender: [email protected]
Intent: Shipping Issue
Urgency: High
Subject: My candle arrived broken
Summary: Customer says their order arrived damaged and needs a fast resolution.
Action: Slack alert + auto-reply queued
```

---

# STEP 7: AI response generation

This step drafts the email reply for all non-spam emails.

## 7A. Add the reply generation OpenAI node

1. Add a new `OpenAI` node.
2. Rename it to:

```text
OpenAI - Draft Reply
```

3. Connect both of these into it:

- `TRUE` from `IF High Urgency?`
- `FALSE` from `IF High Urgency?`

## Configure the node

Set these fields:

- `Credential to connect with`: your OpenAI credential
- `Resource`: `Text`
- `Operation`: `Generate a Model Response`
- `Model`: `gpt-4o-mini`

In `Messages`, add two message rows.

### Message 1

- `Message Type`: `Text`
- `Role`: `System`
- `Text`:

```text
You write customer support emails for Wildwood & Wax, a boutique premium ecommerce brand.

Brand voice:
- warm
- calm
- polished
- premium
- human
- helpful

Rules:
- Acknowledge the customer's issue clearly.
- Do not invent policies, refunds, shipping timelines, discounts, or actions that were not explicitly provided.
- If the issue requires team review, say the team will review and follow up.
- Keep the email between 90 and 160 words.
- Do not sound robotic.
- Do not mention AI, automation, classifiers, or workflow details.
- End with:

Warmly,
Wildwood & Wax Support

- Return JSON only.
- Return exactly one key:
{
  "reply": "full email reply"
}
```

### Message 2

- `Message Type`: `Text`
- `Role`: `User`
- `Text`:

```javascript
=Intent: {{ $('Parse Classification JSON').item.json.intent }}

Urgency: {{ $('Parse Classification JSON').item.json.urgency }}

Summary: {{ $('Parse Classification JSON').item.json.ai_summary }}

Sender Name: {{ $('Parse Classification JSON').item.json.sender_name }}

Subject: {{ $('Parse Classification JSON').item.json.subject_clean }}

Original Email:
{{ $('Parse Classification JSON').item.json.body_clean }}

Write a safe reply for this customer. If information is missing, acknowledge the request and promise review instead of inventing details.
```

## Options

Open `Options` and set:

- `Maximum Number of Tokens`: `500`
- `Output Randomness (Temperature)`: `0.4`
- `Output Format`: `JSON Object`

## Reliability settings

Open the `Settings` tab and set:

- `Retry On Fail`: `On`
- `Max Tries`: `3`
- `Wait Between Tries (ms)`: `2000`
- `On Error`: `Continue`

## 7B. Add the fallback reply Code node

1. Click `+` after `OpenAI - Draft Reply`.
2. Add a `Code` node.
3. Rename it to:

```text
Prepare Final Reply
```

4. Set:
   - `Mode`: `Run Once for Each Item`

5. Paste this exact JavaScript into `JavaScript Code`:

```javascript
const triage = $('Parse Classification JSON').item.json;

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

const fallbackReplies = {
  'Shipping Issue': `Hi ${triage.sender_name || 'there'},

I’m sorry to hear your order arrived with an issue. Thank you for reaching out and letting us know.

Our team is reviewing the details now, and we’ll follow up as quickly as possible with the best next step. If you’re able to reply with any helpful details or photos, that can speed things up, but no need to resend anything if you already included it.

We appreciate your patience and the chance to make this right.

Warmly,
Wildwood & Wax Support`,

  'Subscription Change': `Hi ${triage.sender_name || 'there'},

Thank you for reaching out about your subscription.

We’ve received your request and our team is reviewing the change. We’ll follow up shortly to confirm the next step on our side. If there’s anything specific you’d like updated, you’re welcome to reply with the exact request and we’ll include it in our review.

We appreciate you being part of Wildwood & Wax.

Warmly,
Wildwood & Wax Support`,

  'Wholesale Inquiry': `Hi ${triage.sender_name || 'there'},

Thank you for your interest in Wildwood & Wax.

We’ve received your wholesale inquiry and our team will review it shortly. We’ll follow up with the appropriate next steps once we’ve had a chance to look over your message.

We appreciate you considering Wildwood & Wax and look forward to learning more about your business.

Warmly,
Wildwood & Wax Support`
};

const finalReply =
  (parsed?.reply || '').toString().trim() ||
  fallbackReplies[triage.intent] ||
  `Hi ${triage.sender_name || 'there'},

Thank you for reaching out to Wildwood & Wax.

We’ve received your message and our team will review it shortly. We’ll follow up as soon as possible.

Warmly,
Wildwood & Wax Support`;

return [
  {
    json: {
      ...triage,
      final_reply: finalReply,
      action_taken:
        triage.urgency === 'High' ? 'Slack alert + Auto-reply' : 'Auto-reply'
    }
  }
];
```

This gives you a deterministic fallback email if the AI draft node fails or returns malformed JSON.

## Example AI-generated email reply

```text
Hi Emma,

I’m so sorry to hear your candle arrived broken, and I appreciate you letting us know.

We’ve received your message and our team is reviewing the issue now. We’ll follow up as soon as possible with the next step on our side. If you already included photos, thank you. If not, you are welcome to send them in a reply, though there is no need to resend any details you have already shared.

We truly appreciate your patience and the opportunity to make this right.

Warmly,
Wildwood & Wax Support
```

---

# STEP 8: Send email

## Add the Gmail reply node

1. Click `+` after `Prepare Final Reply`.
2. Add a `Gmail` node.
3. Rename it to:

```text
Gmail - Reply to Sender
```

## Configure the node

Set these fields:

- `Credential to connect with`: your Gmail credential
- `Resource`: `Message`
- `Operation`: `Reply`
- `Message ID`:

```javascript
={{ $('Gmail Trigger').item.json.id }}
```

- `Email Type`: `Text`
- `Message`:

```javascript
={{ $json.final_reply }}
```

In `Reply options`, set:

- `Append n8n attribution`: `Off`
- `Sender Name`: `Wildwood & Wax Support`
- `Reply to Sender Only`: `On`

## Reliability settings

Open the `Settings` tab and set:

- `Retry On Fail`: `On`
- `Max Tries`: `3`
- `Wait Between Tries (ms)`: `2000`
- `On Error`: `Continue`

Why this matters:

- retries handle temporary Gmail API problems
- `Continue` prevents the workflow from hard-stopping after reply generation
- your log step can still run

---

# STEP 9: Google Sheets logging

We already added two Google Sheets nodes for ignored and spam branches.

Now add the success logger for non-spam emails.

## 9A. Add the success log preparation node

1. Click `+` after `Gmail - Reply to Sender`.
2. Add an `Edit Fields` node.
3. Rename it to:

```text
Prepare Success Log
```

4. Set:
   - `Mode`: `JSON Output`

5. Paste this exact JSON into `JSON Output`:

```json
{
  "Timestamp": "={{ $json.received_at }}",
  "Sender Email": "={{ $json.sender_email }}",
  "Subject": "={{ $json.subject_clean }}",
  "Intent": "={{ $json.intent }}",
  "Urgency": "={{ $json.urgency }}",
  "AI Summary": "={{ $json.ai_summary }}",
  "Action Taken": "={{ $json.action_taken }}",
  "AI Response": "={{ $json.final_reply }}"
}
```

## 9B. Add the success Google Sheets logger

1. Connect `Prepare Success Log` to a new `Google Sheets` node.
2. Rename it to:

```text
Google Sheets - Log Success
```

3. Configure it exactly the same as the other two Google Sheets nodes:

- `Credential to connect with`: your Google Sheets credential
- `Resource`: `Sheet Within Document`
- `Operation`: `Append Row`
- `Document`: `Wildwood & Wax Email Triage`
- `Sheet`: `Email Triage Log`
- `Mapping Column Mode`: `Map Automatically`

4. In `Options`, set:

- `Use Append`: `On`
- `Handling extra fields in input`: `Ignore Them`

## Final Google Sheets schema

Your sheet must contain these exact columns:

| Timestamp | Sender Email | Subject | Intent | Urgency | AI Summary | Action Taken | AI Response |
| --- | --- | --- | --- | --- | --- | --- | --- |

## Sample rows

| Timestamp | Sender Email | Subject | Intent | Urgency | AI Summary | Action Taken | AI Response |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 2026-04-16T09:15:12.000Z | [email protected] | My candle arrived broken | Shipping Issue | High | Customer says their order arrived damaged and needs a fast resolution. | Slack alert + Auto-reply | Hi Emma, I’m so sorry to hear your candle arrived broken... |
| 2026-04-16T09:22:08.000Z | [email protected] | Please cancel my subscription | Subscription Change | Medium | Customer wants to cancel or update their subscription. | Auto-reply | Hi Leah, thank you for reaching out about your subscription... |
| 2026-04-16T09:31:44.000Z | [email protected] | Wholesale partnership inquiry | Wholesale Inquiry | Low | Sender is asking about wholesale or business partnership options. | Auto-reply | Hi Daniel, thank you for your interest in Wildwood & Wax... |
| 2026-04-16T09:48:21.000Z | [email protected] | Buy our marketing service | Spam | Low | Message appears to be promotional spam or irrelevant outreach. | Ignored - spam |  |

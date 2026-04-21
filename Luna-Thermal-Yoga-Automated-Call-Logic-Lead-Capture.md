# Luna Thermal Yoga - Automated Call Logic & Lead Capture

This guide is written so you can build the workflow manually in n8n yourself, step by step, without depending on the JSON import.

It follows the same build-it-yourself style as the other workflow guides in this folder.

It assumes:

- You are using a recent n8n version with the current `Webhook`, `Code`, `Set`, `IF`, `Switch`, `Google Sheets`, `Airtable`, `Slack`, `Wait`, and `Error Trigger` nodes.
- You already have these credentials available inside n8n:
  - `Google Sheets OAuth2`
  - `Airtable Personal Access Token`
  - `Slack OAuth2`

Use these exact workflow names:

- `Luna Thermal Yoga - Automated Call Logic & Lead Capture`
- `Luna Thermal Yoga - Error Logger`

## What this workflow does

1. Accepts incoming VOIP-style call events through a webhook.
2. Normalizes the caller phone number and timestamp.
3. Converts the pressed digit into a business-friendly intent.
4. Routes calls into three paths:
   - `Private Session`
   - `Membership Inquiry`
   - `Unknown`
5. Stores urgent private-session leads immediately in Google Sheets.
6. Sends a Slack notification for those urgent leads.
7. Tries to create Airtable records for membership inquiries.
8. Retries Airtable up to three times with timed waits.
9. Falls back to Google Sheets if Airtable still fails.
10. Logs unknown keypresses separately.
11. Uses a second error workflow to capture execution failures centrally.

---

## Before you build

Create the assets first so you do not have to stop halfway through the workflow.

### 1. Google Sheets spreadsheet

Create one spreadsheet. You can name it:

`Luna Thermal Yoga - Call Logs`

Create these four tabs:

1. `Urgent Lead Tracker`
2. `Unknown Calls`
3. `Failed Leads`
4. `Error Logs`

Create these exact header rows.

### Urgent Lead Tracker

```text
Caller ID | Phone | Date/Time | Call Intent
```

### Unknown Calls

```text
Caller ID | Phone | Date/Time | Digit Pressed | Call Intent | Status
```

### Failed Leads

```text
Caller ID | Phone | Date/Time | Digit Pressed | Call Intent | Failure Reason | Retry Count | Logged At
```

### Error Logs

```text
Timestamp | Workflow Name | Node Name | Error Message | Execution Time
```

Keep the spreadsheet ID ready. You will use the same spreadsheet in every Google Sheets node.

### 2. Airtable base

Create or choose one Airtable base for standard call follow-up work.

Recommended base name:

`Admin Tasks`

Create one table for these membership inquiries. You can name it:

`Membership Inquiries`

Create these exact fields:

```text
Caller ID
Phone
Date/Time
Call Intent
Status
```

Keep the Airtable base ID and table ID ready.

### 3. Slack channel

Create or choose one Slack channel for urgent lead notifications.

Example:

`#urgent-leads`

Keep the Slack OAuth credential ready, plus the channel ID you want the node to post into.

### 4. Webhook sample payload

This is the exact test payload you will use while building:

```json
{
  "caller_id": "+11234567890",
  "timestamp": "2026-01-10T18:30:00Z",
  "digit_pressed": "1"
}
```

Meaning:

- `"1"` = `Private Session`
- `"2"` = `Membership Inquiry`
- anything else = `Unknown`

---

# STEP 0: Create the two workflows

## 0A. Create the main workflow

1. Open n8n.
2. Click `Create Workflow`.
3. Rename it to:

```text
Luna Thermal Yoga - Automated Call Logic & Lead Capture
```

4. Save once before adding nodes.

## 0B. Create the error workflow

1. Create a second workflow.
2. Rename it to:

```text
Luna Thermal Yoga - Error Logger
```

3. Save once.

---

# STEP 1: Build the main workflow skeleton

You will build the main workflow in this order:

1. `Webhook - Incoming Call`
2. `Code - Format Phone & Timestamp`
3. `Switch - Route Call Type`
4. `Google Sheets - Add Urgent Lead`
5. `Notify - High Priority Lead`
6. `Set - Confirm High Priority`
7. `Set - Membership Attempt 1`
8. `Airtable - Create Membership Record Attempt 1`
9. `IF - Attempt 1 Failed?`
10. `Wait - Retry Attempt 2`
11. `Set - Membership Attempt 2`
12. `Airtable - Create Membership Record Attempt 2`
13. `IF - Attempt 2 Failed?`
14. `Wait - Retry Attempt 3`
15. `Set - Membership Attempt 3`
16. `Airtable - Create Membership Record Attempt 3`
17. `IF - Attempt 3 Failed?`
18. `Set - Failed Lead Payload`
19. `Google Sheets - Log Failed Lead`
20. `Set - Membership Logged`
21. `Google Sheets - Log Unknown Call`

This gives you one clean entry point and three clear business paths:

- urgent lead path
- membership path with retries
- unknown input fallback path

---

# STEP 2: Create the incoming webhook

## Add the trigger node

1. Add a `Webhook` node.
2. Rename it to:

```text
Webhook - Incoming Call
```

## Configure the webhook

Set these fields exactly:

- `HTTP Method`: `POST`
- `Path`:

```text
luna-thermal-yoga-incoming-call
```

- `Authentication`: `None`
- `Respond`: `Immediately`

If your n8n version shows response settings, use:

- `Response Code`: `200`

You do not need a custom response body for this workflow. Immediate response is enough because the actual processing continues in the background.

## Expected payload

The webhook should accept this JSON body:

```json
{
  "caller_id": "+11234567890",
  "timestamp": "2026-01-10T18:30:00Z",
  "digit_pressed": "1"
}
```

---

# STEP 3: Normalize the incoming call data

This step makes the incoming VOIP payload easier for all later nodes to use.

It does three things:

1. reformats the raw phone number
2. converts the timestamp into a readable string
3. derives the business intent from the digit pressed

## Add the Code node

1. Add a `Code` node after the webhook.
2. Rename it to:

```text
Code - Format Phone & Timestamp
```

3. Set:
   - `Mode`: `Run Once for Each Item`
   - `Language`: `JavaScript`

4. Paste this exact code:

```javascript
const payload = $json.body ?? $json;
const rawCallerId = String(payload.caller_id ?? '').trim();
const rawTimestamp = String(payload.timestamp ?? '').trim();
const digitPressed = String(payload.digit_pressed ?? '').trim();

const digits = rawCallerId.replace(/\D/g, '');
let formattedPhone = rawCallerId;

if (digits.length === 11 && digits.startsWith('1')) {
  formattedPhone = `+1-${digits.slice(1, 4)}-${digits.slice(4, 7)}-${digits.slice(7, 11)}`;
} else if (digits.length === 10) {
  formattedPhone = `+1-${digits.slice(0, 3)}-${digits.slice(3, 6)}-${digits.slice(6, 10)}`;
} else if (digits) {
  formattedPhone = `+${digits}`;
}

const parsedDate = new Date(rawTimestamp);
const readableTime = Number.isNaN(parsedDate.getTime())
  ? (rawTimestamp || new Date().toISOString())
  : parsedDate.toLocaleString('en-US', {
      timeZone: 'UTC',
      year: 'numeric',
      month: 'short',
      day: '2-digit',
      hour: '2-digit',
      minute: '2-digit',
      second: '2-digit',
      hour12: true,
      timeZoneName: 'short',
    });

const intentMap = {
  '1': 'Private Session',
  '2': 'Membership Inquiry',
};

return [{
  json: {
    caller_id: rawCallerId,
    formatted_phone: formattedPhone,
    timestamp: rawTimestamp || new Date().toISOString(),
    readable_time: readableTime,
    digit_pressed: digitPressed,
    call_intent: intentMap[digitPressed] || 'Unknown',
  },
}];
```

## What this node returns

After this node, every item has:

```json
{
  "caller_id": "+11234567890",
  "formatted_phone": "+1-123-456-7890",
  "timestamp": "2026-01-10T18:30:00Z",
  "readable_time": "Jan 10, 2026, 06:30:00 PM UTC",
  "digit_pressed": "1",
  "call_intent": "Private Session"
}
```

---

# STEP 4: Route by call type

This is the main branch controller.

## Add the Switch node

1. Add a `Switch` node after `Code - Format Phone & Timestamp`.
2. Rename it to:

```text
Switch - Route Call Type
```

## Configure rule 1

Create the first rule with:

- `Left Value`:

```javascript
={{ $json.digit_pressed }}
```

- `Operator`: `equals`
- `Right Value`:

```text
1
```

This output is the `Private Session` branch.

## Configure rule 2

Create the second rule with:

- `Left Value`:

```javascript
={{ $json.digit_pressed }}
```

- `Operator`: `equals`
- `Right Value`:

```text
2
```

This output is the `Membership Inquiry` branch.

## Configure fallback

Turn on the fallback output so that any other value goes to a third path.

That fallback output becomes the `Unknown` branch.

## Expected behavior

- output 1 = urgent private-session path
- output 2 = membership path
- fallback output = unknown call logging

---

# STEP 5: Build the high-priority private-session path

This path should feel immediate and safe:

1. write the lead to Google Sheets first
2. notify Slack second
3. mark the item as confirmed

That ordering means the lead is stored before the notification fires.

## 5A. Add the urgent lead Google Sheets node

1. Connect output 1 from `Switch - Route Call Type` to a `Google Sheets` node.
2. Rename it to:

```text
Google Sheets - Add Urgent Lead
```

## Configure the node

Set:

- `Credential to connect with`: your Google Sheets credential
- `Resource`: `Sheet Within Document`
- `Operation`: `Append Row`

Use:

- `Document ID`: your spreadsheet ID
- `Sheet Name`: `Urgent Lead Tracker`

In the manual column mapping, map these fields:

- `Caller ID` -> `{{ $json["caller_id"] }}`
- `Phone` -> `{{ $json["formatted_phone"] }}`
- `Date/Time` -> `{{ $json["readable_time"] }}`
- `Call Intent` -> `{{ $json["call_intent"] }}`

## Reliability settings

In the node settings, turn on:

- `Retry On Fail`
- `Max Tries`: `3`
- `Wait Between Tries (ms)`: `2000`

This path is urgent, so the sheet write should retry automatically.

## 5B. Add the Slack node

1. Connect `Google Sheets - Add Urgent Lead` to a `Slack` node.
2. Rename it to:

```text
Notify - High Priority Lead
```

## Configure the Slack node

Set:

- `Authentication`: your Slack OAuth2 credential
- `Send To`: `Channel`
- `Channel ID`: your urgent-leads Slack channel

For the message text, use this exact expression:

```javascript
=🔥 NEW PRIVATE SESSION LEAD

Caller: {{ $('Code - Format Phone & Timestamp').item.json.formatted_phone }}
Time: {{ $('Code - Format Phone & Timestamp').item.json.readable_time }}
```

This intentionally references the normalization node directly so the message stays stable even if upstream output shapes change.

## Slack reliability settings

In the Slack node settings, turn on:

- `Retry On Fail`
- `Max Tries`: `3`
- `Wait Between Tries (ms)`: `2000`

## 5C. Add the final Set node for the urgent path

1. Connect `Notify - High Priority Lead` to a `Set` node.
2. Rename it to:

```text
Set - Confirm High Priority
```

3. Add these assignments:

- `caller_id` = `{{ $('Code - Format Phone & Timestamp').item.json.caller_id }}`
- `formatted_phone` = `{{ $('Code - Format Phone & Timestamp').item.json.formatted_phone }}`
- `readable_time` = `{{ $('Code - Format Phone & Timestamp').item.json.readable_time }}`
- `call_intent` = `{{ $('Code - Format Phone & Timestamp').item.json.call_intent }}`
- `priority` = `HIGH`
- `status` = `logged`

This node is mostly a clean endpoint that confirms the urgent path completed as intended.

---

# STEP 6: Build the membership inquiry path

This is the most important part of the workflow from a reliability perspective.

The goal is:

1. try Airtable
2. if Airtable fails, wait 5 seconds
3. try again
4. if it fails again, wait 5 seconds
5. try a third time
6. if it still fails, write the lead to Google Sheets fallback storage

## Important design note

When an Airtable node is configured to continue after an error, the returned data mainly contains the error message and not always the original call payload.

That is why we add `Set - Membership Attempt 1`, `Set - Membership Attempt 2`, and `Set - Membership Attempt 3`.

Those Set nodes preserve the original data so every retry stays deterministic.

## 6A. Create the first membership Set node

1. Connect output 2 from `Switch - Route Call Type` to a `Set` node.
2. Rename it to:

```text
Set - Membership Attempt 1
```

3. Add these assignments:

- `caller_id` = `{{ $json["caller_id"] }}`
- `formatted_phone` = `{{ $json["formatted_phone"] }}`
- `readable_time` = `{{ $json["readable_time"] }}`
- `digit_pressed` = `{{ $json["digit_pressed"] }}`
- `call_intent` = `{{ $json["call_intent"] }}`
- `retry_attempt` = `1`
- `status` = `Pending Review`

This creates the stable payload that the whole retry chain will refer back to.

## 6B. Create Airtable attempt 1

1. Connect `Set - Membership Attempt 1` to an `Airtable` node.
2. Rename it to:

```text
Airtable - Create Membership Record Attempt 1
```

## Configure the Airtable node

Set:

- `Authentication`: Airtable Personal Access Token
- `Resource`: `Record`
- `Operation`: `Create`
- `Base`: your Airtable base ID
- `Table`: your Airtable table ID

Map these fields:

- `Caller ID` -> `{{ $json["caller_id"] }}`
- `Phone` -> `{{ $json["formatted_phone"] }}`
- `Date/Time` -> `{{ $json["readable_time"] }}`
- `Call Intent` -> `{{ $json["call_intent"] }}`
- `Status` -> `{{ $json["status"] }}`

## Critical error setting

Open the node settings and set:

- `On Error`: `Continue (Regular Output)`

This is the key setting that allows the workflow to continue into the retry-check IF node instead of stopping.

## 6C. Add the first failure-check IF node

1. Add an `IF` node after the first Airtable node.
2. Rename it to:

```text
IF - Attempt 1 Failed?
```

3. Configure the condition:

- `Left Value`:

```javascript
={{ $json.message || '' }}
```

- `Operator`: `is not empty`

## How this IF works

- `TRUE` means the Airtable node produced an error message, so the write failed
- `FALSE` means there was no error message, so the Airtable create succeeded

## 6D. Add the first Wait node

1. Connect the `TRUE` output of `IF - Attempt 1 Failed?` to a `Wait` node.
2. Rename it to:

```text
Wait - Retry Attempt 2
```

3. Configure:

- `Resume`: `After Time Interval`
- `Amount`: `5`
- `Unit`: `Seconds`

This gives Airtable a short recovery window before the second attempt.

## 6E. Add the second membership Set node

1. Connect `Wait - Retry Attempt 2` to a new `Set` node.
2. Rename it to:

```text
Set - Membership Attempt 2
```

3. Add these assignments:

- `caller_id` = `{{ $('Set - Membership Attempt 1').item.json.caller_id }}`
- `formatted_phone` = `{{ $('Set - Membership Attempt 1').item.json.formatted_phone }}`
- `readable_time` = `{{ $('Set - Membership Attempt 1').item.json.readable_time }}`
- `digit_pressed` = `{{ $('Set - Membership Attempt 1').item.json.digit_pressed }}`
- `call_intent` = `{{ $('Set - Membership Attempt 1').item.json.call_intent }}`
- `retry_attempt` = `2`
- `status` = `Pending Review`

Notice that every value comes from `Set - Membership Attempt 1`, not from the failed Airtable output.

That is intentional.

## 6F. Create Airtable attempt 2

1. Connect `Set - Membership Attempt 2` to a second `Airtable` node.
2. Rename it to:

```text
Airtable - Create Membership Record Attempt 2
```

Use the same Airtable configuration as attempt 1.

Set the same error behavior:

- `On Error`: `Continue (Regular Output)`

## 6G. Add the second failure-check IF node

1. Add an `IF` node after attempt 2.
2. Rename it to:

```text
IF - Attempt 2 Failed?
```

3. Use the same condition:

- `Left Value`:

```javascript
={{ $json.message || '' }}
```

- `Operator`: `is not empty`

## 6H. Add the second Wait node

1. Connect the `TRUE` output of `IF - Attempt 2 Failed?` to another `Wait` node.
2. Rename it to:

```text
Wait - Retry Attempt 3
```

3. Configure:

- `Resume`: `After Time Interval`
- `Amount`: `5`
- `Unit`: `Seconds`

## 6I. Add the third membership Set node

1. Connect `Wait - Retry Attempt 3` to a new `Set` node.
2. Rename it to:

```text
Set - Membership Attempt 3
```

3. Add these assignments:

- `caller_id` = `{{ $('Set - Membership Attempt 1').item.json.caller_id }}`
- `formatted_phone` = `{{ $('Set - Membership Attempt 1').item.json.formatted_phone }}`
- `readable_time` = `{{ $('Set - Membership Attempt 1').item.json.readable_time }}`
- `digit_pressed` = `{{ $('Set - Membership Attempt 1').item.json.digit_pressed }}`
- `call_intent` = `{{ $('Set - Membership Attempt 1').item.json.call_intent }}`
- `retry_attempt` = `3`
- `status` = `Pending Review`

## 6J. Create Airtable attempt 3

1. Connect `Set - Membership Attempt 3` to a third `Airtable` node.
2. Rename it to:

```text
Airtable - Create Membership Record Attempt 3
```

Use the same field mappings and the same error setting:

- `On Error`: `Continue (Regular Output)`

## 6K. Add the third failure-check IF node

1. Add an `IF` node after the third Airtable node.
2. Rename it to:

```text
IF - Attempt 3 Failed?
```

3. Use the same condition again:

- `Left Value`:

```javascript
={{ $json.message || '' }}
```

- `Operator`: `is not empty`

## 6L. Build the final failed-lead fallback

1. Connect the `TRUE` output of `IF - Attempt 3 Failed?` to a `Set` node.
2. Rename it to:

```text
Set - Failed Lead Payload
```

3. Add these assignments:

- `Caller ID` = `{{ $('Set - Membership Attempt 1').item.json.caller_id }}`
- `Phone` = `{{ $('Set - Membership Attempt 1').item.json.formatted_phone }}`
- `Date/Time` = `{{ $('Set - Membership Attempt 1').item.json.readable_time }}`
- `Digit Pressed` = `{{ $('Set - Membership Attempt 1').item.json.digit_pressed }}`
- `Call Intent` = `{{ $('Set - Membership Attempt 1').item.json.call_intent }}`
- `Failure Reason` = `{{ $json.message || 'Airtable create failed after 3 attempts' }}`
- `Retry Count` = `3`
- `Logged At` = `{{ new Date().toISOString() }}`

This node rebuilds a complete fallback record even though the Airtable output may only contain error details.

## 6M. Add the Google Sheets fallback logger

1. Connect `Set - Failed Lead Payload` to a `Google Sheets` node.
2. Rename it to:

```text
Google Sheets - Log Failed Lead
```

Use:

- `Document ID`: your spreadsheet ID
- `Sheet Name`: `Failed Leads`
- `Operation`: `Append Row`

Map:

- `Caller ID` -> `{{ $json["Caller ID"] }}`
- `Phone` -> `{{ $json["Phone"] }}`
- `Date/Time` -> `{{ $json["Date/Time"] }}`
- `Digit Pressed` -> `{{ $json["Digit Pressed"] }}`
- `Call Intent` -> `{{ $json["Call Intent"] }}`
- `Failure Reason` -> `{{ $json["Failure Reason"] }}`
- `Retry Count` -> `{{ $json["Retry Count"] }}`
- `Logged At` -> `{{ $json["Logged At"] }}`

In settings, turn on:

- `Retry On Fail`
- `Max Tries`: `3`
- `Wait Between Tries (ms)`: `2000`

## 6N. Build the membership success endpoint

Now create the success endpoint used by all successful Airtable outcomes.

1. Add a `Set` node.
2. Rename it to:

```text
Set - Membership Logged
```

3. Add these assignments:

- `caller_id` = `{{ $('Set - Membership Attempt 1').item.json.caller_id }}`
- `formatted_phone` = `{{ $('Set - Membership Attempt 1').item.json.formatted_phone }}`
- `readable_time` = `{{ $('Set - Membership Attempt 1').item.json.readable_time }}`
- `call_intent` = `{{ $('Set - Membership Attempt 1').item.json.call_intent }}`
- `priority` = `STANDARD`
- `status` = `logged`

## Connect success outputs

Connect these `FALSE` outputs into `Set - Membership Logged`:

- `FALSE` from `IF - Attempt 1 Failed?`
- `FALSE` from `IF - Attempt 2 Failed?`
- `FALSE` from `IF - Attempt 3 Failed?`

That means:

- if Airtable succeeds on attempt 1, workflow ends at success
- if attempt 1 fails but attempt 2 succeeds, workflow ends at success
- if attempt 1 and 2 fail but attempt 3 succeeds, workflow ends at success

---

# STEP 7: Build the unknown-input path

This path catches anything except `1` or `2`.

## Add the Google Sheets node

1. Connect the fallback output from `Switch - Route Call Type` to a `Google Sheets` node.
2. Rename it to:

```text
Google Sheets - Log Unknown Call
```

## Configure the node

Use:

- `Document ID`: your spreadsheet ID
- `Sheet Name`: `Unknown Calls`
- `Operation`: `Append Row`

Map:

- `Caller ID` -> `{{ $json["caller_id"] }}`
- `Phone` -> `{{ $json["formatted_phone"] }}`
- `Date/Time` -> `{{ $json["readable_time"] }}`
- `Digit Pressed` -> `{{ $json["digit_pressed"] }}`
- `Call Intent` -> `{{ $json["call_intent"] }}`
- `Status` -> `Unknown Input`

In settings, turn on:

- `Retry On Fail`
- `Max Tries`: `3`
- `Wait Between Tries (ms)`: `2000`

This protects you from losing odd or malformed keypad events.

---

# STEP 8: Build the separate error workflow

This workflow is independent from the main flow.

Its only job is to catch workflow failures and write them into the `Error Logs` sheet.

## 8A. Add the Error Trigger node

Inside the second workflow, add an `Error Trigger` node.

Rename it to:

```text
Error Trigger - Global
```

## 8B. Add the Set node for error normalization

1. Add a `Set` node after `Error Trigger - Global`.
2. Rename it to:

```text
Set - Prepare Error Log
```

3. Add these assignments:

- `Timestamp` = `{{ new Date().toISOString() }}`
- `Workflow Name` = `{{ $json.workflow?.name || '' }}`
- `Node Name` = `{{ $json.execution?.lastNodeExecuted || '' }}`
- `Error Message` = `{{ $json.execution?.error?.message || $json.error?.message || $json.execution?.data?.resultData?.error?.message || 'Unknown error' }}`
- `Execution Time` = `{{ $json.execution?.startedAt || $json.execution?.stoppedAt || '' }}`

This gives you a consistent error row even when n8n provides slightly different error payload shapes.

## 8C. Add the Google Sheets error logger

1. Connect `Set - Prepare Error Log` to a `Google Sheets` node.
2. Rename it to:

```text
Google Sheets - Log Error
```

Use:

- `Document ID`: the same spreadsheet ID from the main workflow
- `Sheet Name`: `Error Logs`
- `Operation`: `Append Row`

Map:

- `Timestamp` -> `{{ $json["Timestamp"] }}`
- `Workflow Name` -> `{{ $json["Workflow Name"] }}`
- `Node Name` -> `{{ $json["Node Name"] }}`
- `Error Message` -> `{{ $json["Error Message"] }}`
- `Execution Time` -> `{{ $json["Execution Time"] }}`

In settings, turn on:

- `Retry On Fail`
- `Max Tries`: `3`
- `Wait Between Tries (ms)`: `2000`

## 8D. Link the error workflow to the main workflow

After both workflows are saved:

1. Open the main workflow settings.
2. Find the error workflow setting.
3. Select:

```text
Luna Thermal Yoga - Error Logger
```

This is what makes the global error workflow actually run when the main workflow crashes.

---

# STEP 9: Connection map

If you want to sanity-check your canvas, the connections should look like this.

```text
Webhook - Incoming Call
  -> Code - Format Phone & Timestamp
  -> Switch - Route Call Type

Switch - Route Call Type output 1
  -> Google Sheets - Add Urgent Lead
  -> Notify - High Priority Lead
  -> Set - Confirm High Priority

Switch - Route Call Type output 2
  -> Set - Membership Attempt 1
  -> Airtable - Create Membership Record Attempt 1
  -> IF - Attempt 1 Failed?

IF - Attempt 1 Failed? TRUE
  -> Wait - Retry Attempt 2
  -> Set - Membership Attempt 2
  -> Airtable - Create Membership Record Attempt 2
  -> IF - Attempt 2 Failed?

IF - Attempt 2 Failed? TRUE
  -> Wait - Retry Attempt 3
  -> Set - Membership Attempt 3
  -> Airtable - Create Membership Record Attempt 3
  -> IF - Attempt 3 Failed?

IF - Attempt 3 Failed? TRUE
  -> Set - Failed Lead Payload
  -> Google Sheets - Log Failed Lead

IF - Attempt 1 Failed? FALSE
  -> Set - Membership Logged

IF - Attempt 2 Failed? FALSE
  -> Set - Membership Logged

IF - Attempt 3 Failed? FALSE
  -> Set - Membership Logged

Switch - Route Call Type fallback
  -> Google Sheets - Log Unknown Call
```

Error workflow:

```text
Error Trigger - Global
  -> Set - Prepare Error Log
  -> Google Sheets - Log Error
```

---

# STEP 10: Recommended canvas layout

To keep the workflow readable, lay it out left to right like this:

- top lane = urgent private-session path
- middle lane = membership retry path
- bottom lane = unknown logging path

Recommended positions:

- `Webhook - Incoming Call` far left
- `Code - Format Phone & Timestamp` slightly right
- `Switch - Route Call Type` in the center
- urgent path above the switch
- membership retry chain straight across the middle
- unknown path below the switch

This makes the business logic obvious at a glance.

---

# STEP 11: Testing checklist

Build and test one branch at a time.

## Test 1: urgent private-session path

Send:

```json
{
  "caller_id": "+11234567890",
  "timestamp": "2026-01-10T18:30:00Z",
  "digit_pressed": "1"
}
```

Expected result:

1. webhook receives the payload
2. code node outputs `Private Session`
3. row is appended to `Urgent Lead Tracker`
4. Slack message is posted
5. `Set - Confirm High Priority` outputs `priority = HIGH` and `status = logged`

## Test 2: membership inquiry path

Send:

```json
{
  "caller_id": "+11234567890",
  "timestamp": "2026-01-10T18:30:00Z",
  "digit_pressed": "2"
}
```

Expected result:

1. code node outputs `Membership Inquiry`
2. Airtable record is created in `Membership Inquiries`
3. one of the IF nodes routes to `Set - Membership Logged`
4. no data is written to `Failed Leads`

## Test 3: unknown input path

Send:

```json
{
  "caller_id": "+11234567890",
  "timestamp": "2026-01-10T18:30:00Z",
  "digit_pressed": "9"
}
```

Expected result:

1. switch does not match rule 1 or 2
2. fallback output runs
3. row is appended to `Unknown Calls`

## Test 4: forced Airtable failure path

To test the fallback path safely:

1. temporarily use an invalid Airtable table ID
2. send the `"2"` payload again
3. watch the workflow wait and retry
4. confirm that after the third failure, a row lands in `Failed Leads`

Expected result:

1. attempt 1 fails
2. wait 5 seconds
3. attempt 2 fails
4. wait 5 seconds
5. attempt 3 fails
6. failed lead is written to Google Sheets

## Test 5: error workflow

To test the error workflow:

1. temporarily break a required node in the main workflow
2. trigger the workflow
3. confirm a row is added to the `Error Logs` sheet

---

# STEP 12: Example Slack notification

Your Slack message should look roughly like this:

```text
🔥 NEW PRIVATE SESSION LEAD

Caller: +1-123-456-7890
Time: Jan 10, 2026, 06:30:00 PM UTC
```

---

# STEP 13: Why this workflow is production-safe

This design is stronger than a simple one-shot webhook flow because:

1. urgent leads are stored before notification
2. Airtable writes are retried instead of failing once and disappearing
3. failed membership writes are still captured in Google Sheets
4. unknown keypad input is not discarded
5. execution-level failures are logged by a separate error workflow

That combination prevents silent data loss, which is the real risk in call-routing automations.

---

# STEP 14: Activation checklist

Before activating the workflows, confirm these items:

1. Google Sheets headers exactly match the field names in this guide.
2. Airtable field names exactly match the mapped names.
3. Slack credential and channel ID are valid.
4. The error workflow is assigned to the main workflow.
5. Test webhook payloads succeed in the editor first.
6. Only after that, activate the main workflow.
7. Activate the error workflow too.

---

# Final result

Once built, this system gives Luna Thermal Yoga a small but production-minded call automation flow:

- private-session callers are captured and escalated immediately
- membership inquiries are sent into the admin queue with retry protection
- bad keypad input is logged for follow-up
- failures are centralized in a separate error log

If you want, the next step I can do is create the same kind of long-form build guide for the `ThinkBoard` project prompt you pasted, in a new markdown file matching this style.

# Preschool Enrollment Lead Automation for Little Sprout Academy

This guide is written so you can build the workflow manually in n8n yourself, step by step, without depending on a JSON import.

It follows the same build-it-yourself style as the other workflow guides in this folder.

It assumes:

- You are using a recent n8n version with the current `Google Sheets`, `Code`, `Set`, `IF`, `Switch`, `Merge`, `HubSpot`, `Slack`, and `Error Trigger` nodes.
- You already have these credentials available inside n8n:
  - `Google Sheets OAuth2`
  - `HubSpot API Key or Private App Token`
  - `Slack OAuth2`

Use these exact workflow names:

- `Little Sprout Academy - Enrollment Lead Pipeline`
- `Little Sprout Academy - Error Logger`

---

## What this workflow does

1. Triggers the moment a new row is appended to the Google Sheets enrollment form.
2. Cleans and normalizes raw input data from the sheet.
3. Formats the parent name into proper Title Case.
4. Standardizes the phone number into a consistent E.164-style format.
5. Validates that all required fields are present.
6. Logs invalid or incomplete submissions to a separate sheet and stops processing them.
7. Routes the lead to one of four priority paths based on program interest.
8. Assigns a priority level — High for Infant, Normal for Toddler and Pre-K, Unknown for anything else.
9. Merges all priority paths back into a single clean data stream.
10. Creates or updates a HubSpot Contact using the parent email as the unique identifier.
11. Checks whether the HubSpot contact was created successfully.
12. If the contact creation failed, fires an immediate Slack error alert to the admin channel and stops.
13. Creates a HubSpot Deal in the Initial Inquiry stage if the contact succeeded.
14. Builds a formatted Slack message with all enrollment details and priority flag.
15. Posts the Slack notification to the admissions team channel.
16. Merges both the success path and the error path into a final node for execution tracking.
17. Writes a final log entry to confirm the execution completed.
18. Uses a second separate error workflow to catch any execution-level failures across the entire pipeline.

---

## Before you build

Create all external assets first so you do not have to stop partway through the workflow.

### 1. Google Sheets spreadsheet

Create one spreadsheet and name it:

```text
Little Sprout Academy - Enrollment Pipeline
```

Create these three tabs:

1. `Enrollment Leads`
2. `Invalid Submissions`
3. `Execution Log`

Create these exact header rows.

#### Enrollment Leads

This is the tab your form or manual team will write into. It is also the tab the trigger node watches.

```text
Submission_ID | Parent_Name | Parent_Email | Parent_Phone | Student_Age | Program_Interest | Desired_Start_Date
```

#### Invalid Submissions

This tab captures rows that fail the required fields check.

```text
Submission_ID | Parent_Name | Parent_Email | Parent_Phone | Student_Age | Program_Interest | Desired_Start_Date | Failure_Reason | Logged_At
```

#### Execution Log

This tab captures a confirmation entry after each successful workflow run.

```text
Submission_ID | Parent_Name | Program_Interest | Priority | HubSpot_Contact_ID | Deal_ID | Completed_At
```

Keep the spreadsheet ID ready. You will use it in every Google Sheets node.

### 2. Import the starter data

The file `little_sprout_enrollment_leads_starter.csv` contains 30 sample parent records.

Import it into the `Enrollment Leads` tab.

To do this:

1. Open Google Sheets.
2. Click `File` → `Import`.
3. Choose `Upload` and select `little_sprout_enrollment_leads_starter.csv`.
4. Set `Import location` to `Replace current sheet` if the tab is empty, or `Append to current sheet` if you already have headers.
5. Set `Separator type` to `Comma`.
6. Click `Import data`.

Confirm the columns match exactly:

```text
Submission_ID | Parent_Name | Parent_Email | Parent_Phone | Student_Age | Program_Interest | Desired_Start_Date
```

The starter data contains a wide variety of phone formats and name cases on purpose. This is to test the formatting nodes you will build.

Some examples from the file:

```text
555-010-2233
(555) 010-4455
5550106677
555.010.8899
1-555-010-1122
555 010 3344
```

And name formats:

```text
sarah jenkins         ← all lowercase
REBECCA WHITE         ← all uppercase
Michael Thompson      ← already correct
```

Your formatting nodes will normalize all of these.

### 3. HubSpot setup

#### Create the Deal Pipeline and Stage

If you are using a free HubSpot account:

1. Go to `CRM` → `Deals`.
2. Click the settings gear icon → `Edit pipeline`.
3. Use the default `Sales Pipeline`.
4. Confirm a stage called `Initial Inquiry` exists. If it does not, click `Add stage` and name it exactly:

```text
Initial Inquiry
```

5. Save.

#### Verify contact property availability

The workflow will use these built-in HubSpot contact properties:

- `email` — used as the unique identifier for create or update
- `firstname`
- `lastname`
- `phone`

You will also want one custom contact property for student age. To create it:

1. Go to `Settings` → `Properties` → `Contact properties`.
2. Click `Create property`.
3. Set `Label` to `Student Age`.
4. Set `Internal name` to `student_age`.
5. Set `Field type` to `Number`.
6. Save.

Keep the HubSpot Private App Token or API key ready.

### 4. Slack channels

Create or choose two Slack channels:

- `#admissions-alerts` — for new enrollment notifications
- `#admin-errors` — for workflow failure alerts

Keep both channel IDs ready. You can find a channel ID by opening the channel in Slack, clicking the channel name at the top, and scrolling to the bottom of the About panel.

Keep the Slack OAuth credential ready in n8n.

---

# STEP 0: Create the two workflows

## 0A. Create the main workflow

1. Open n8n.
2. Click `Create Workflow`.
3. Rename it to:

```text
Little Sprout Academy - Enrollment Lead Pipeline
```

4. Save once before adding any nodes.

## 0B. Create the error workflow

1. Create a second workflow.
2. Rename it to:

```text
Little Sprout Academy - Error Logger
```

3. Save once.

---

# STEP 1: Build the main workflow skeleton

You will build the main workflow in this order. This list is your reference for the entire build.

1. `Google Sheets - New Enrollment Lead` ← trigger
2. `Set - Clean Raw Data`
3. `Code - Format Name`
4. `Code - Format Phone`
5. `IF - Required Fields Check`
6. `Set - Mark Invalid Lead` ← FALSE branch from node 5
7. `Google Sheets - Log Invalid Submission` ← continues from node 6
8. `Switch - Route by Program Interest` ← TRUE branch from node 5
9. `Set - Priority High (Infant)` ← output 1 from node 8
10. `Set - Priority Normal (Toddler)` ← output 2 from node 8
11. `Set - Priority Normal (Pre-K)` ← output 3 from node 8
12. `Set - Priority Unknown` ← fallback output from node 8
13. `Merge - Rejoin After Priority`
14. `HubSpot - Create or Update Contact`
15. `IF - Contact Created?`
16. `Set - HubSpot Error Payload` ← FALSE branch from node 15
17. `Slack - Send Admin Error Alert` ← continues from node 16
18. `HubSpot - Create Deal` ← TRUE branch from node 15
19. `Set - Build Slack Message`
20. `Slack - Send Enrollment Notification`
21. `Merge - Final Join`
22. `Set - Final Execution Log`

This gives you one clean trigger, one validation gate, four priority assignment branches, one CRM write with failure handling, one Slack notification path, and one execution logger.

---

# STEP 2: Add the Google Sheets trigger

This is the node that starts the entire workflow.

## Add the node

1. Add a `Google Sheets` node.
2. Rename it to:

```text
Google Sheets - New Enrollment Lead
```

## Configure the node

Set these fields exactly:

- `Operation`: `Trigger`
- `Trigger On`: `Row Added`
- `Credential`: your Google Sheets OAuth2 credential
- `Document`: select your spreadsheet named `Little Sprout Academy - Enrollment Pipeline`
- `Sheet`: `Enrollment Leads`
- `Columns`: leave as All Columns

Under `Options`, set:

- `Poll Times`: every 1 minute (for testing; you can change to 5 minutes in production)

## What this node returns

When a new row is added, the node outputs one item per new row. Each item looks like this:

```json
{
  "Submission_ID": "LS-0001",
  "Parent_Name": "sarah jenkins",
  "Parent_Email": "s.jenkins88@gmail.com",
  "Parent_Phone": "555-010-2233",
  "Student_Age": "0.8",
  "Program_Interest": "Infant",
  "Desired_Start_Date": "2024-09-01"
}
```

All values are strings at this point. The formatting nodes will clean them.

---

# STEP 3: Clean the raw data

This node trims extra whitespace and standardizes empty strings to a consistent null-safe value before any other processing runs.

## Add the node

1. Add a `Set` node after `Google Sheets - New Enrollment Lead`.
2. Rename it to:

```text
Set - Clean Raw Data
```

## Configure the node

Set `Mode` to `Manual Mapping`.

Add these seven fields. Each one uses an expression to trim the raw value.

| Field Name | Expression |
|---|---|
| `submission_id` | `{{ $json["Submission_ID"]?.toString().trim() \|\| "" }}` |
| `parent_name_raw` | `{{ $json["Parent_Name"]?.toString().trim() \|\| "" }}` |
| `parent_email` | `{{ $json["Parent_Email"]?.toString().trim().toLowerCase() \|\| "" }}` |
| `parent_phone_raw` | `{{ $json["Parent_Phone"]?.toString().trim() \|\| "" }}` |
| `student_age` | `{{ $json["Student_Age"]?.toString().trim() \|\| "" }}` |
| `program_interest` | `{{ $json["Program_Interest"]?.toString().trim() \|\| "" }}` |
| `desired_start_date` | `{{ $json["Desired_Start_Date"]?.toString().trim() \|\| "" }}` |

Note that `parent_email` is also lowercased here because HubSpot treats email as case-insensitive and it is cleaner to normalize it early.

## What this node outputs

```json
{
  "submission_id": "LS-0001",
  "parent_name_raw": "sarah jenkins",
  "parent_email": "s.jenkins88@gmail.com",
  "parent_phone_raw": "555-010-2233",
  "student_age": "0.8",
  "program_interest": "Infant",
  "desired_start_date": "2024-09-01"
}
```

---

# STEP 4: Format the parent name

This node converts the name into proper Title Case regardless of whether the original input was all lowercase, all uppercase, or mixed.

## Add the node

1. Add a `Code` node after `Set - Clean Raw Data`.
2. Rename it to:

```text
Code - Format Name
```

3. Set:
   - `Mode`: `Run Once for Each Item`
   - `Language`: `JavaScript`

4. Paste this exact code:

```javascript
const raw = $json.parent_name_raw ?? '';

const titleCase = raw
  .toLowerCase()
  .split(' ')
  .map(word => {
    if (!word) return '';
    return word.charAt(0).toUpperCase() + word.slice(1);
  })
  .join(' ');

const parts = titleCase.trim().split(' ');
const firstName = parts[0] ?? '';
const lastName = parts.slice(1).join(' ') ?? '';

return [{
  json: {
    ...$json,
    parent_name: titleCase.trim(),
    first_name: firstName,
    last_name: lastName,
  }
}];
```

## What this node outputs

It passes through all existing fields and adds three new ones.

For input `"sarah jenkins"`:

```json
{
  "parent_name": "Sarah Jenkins",
  "first_name": "Sarah",
  "last_name": "Jenkins"
}
```

For input `"REBECCA WHITE"`:

```json
{
  "parent_name": "Rebecca White",
  "first_name": "Rebecca",
  "last_name": "White"
}
```

For input `"Kevin O'Brian"`:

```json
{
  "parent_name": "Kevin O'brian",
  "first_name": "Kevin",
  "last_name": "O'brian"
}
```

The split on space means multi-part last names are handled cleanly. The HubSpot mapping will use `first_name` and `last_name` separately.

---

# STEP 5: Format the phone number

This node standardizes the many different phone formats in the starter data into one consistent format.

## Add the node

1. Add a `Code` node after `Code - Format Name`.
2. Rename it to:

```text
Code - Format Phone
```

3. Set:
   - `Mode`: `Run Once for Each Item`
   - `Language`: `JavaScript`

4. Paste this exact code:

```javascript
const raw = $json.parent_phone_raw ?? '';

// Strip everything except digits
const digits = raw.replace(/\D/g, '');

let formatted = raw; // fallback to raw if no format rule matches

if (digits.length === 11 && digits.startsWith('1')) {
  // 1-555-010-1122 → +1-555-010-1122
  formatted = `+1-${digits.slice(1, 4)}-${digits.slice(4, 7)}-${digits.slice(7, 11)}`;
} else if (digits.length === 10) {
  // 5550102233 → +1-555-010-2233
  formatted = `+1-${digits.slice(0, 3)}-${digits.slice(3, 6)}-${digits.slice(6, 10)}`;
} else if (digits.length > 0) {
  // Anything else: just prepend +
  formatted = `+${digits}`;
}

return [{
  json: {
    ...$json,
    parent_phone: formatted,
  }
}];
```

## What this node outputs

It passes through all existing fields and adds `parent_phone`.

| Raw Input | Formatted Output |
|---|---|
| `555-010-2233` | `+1-555-010-2233` |
| `(555) 010-4455` | `+1-555-010-4455` |
| `5550106677` | `+1-555-010-6677` |
| `555.010.8899` | `+1-555-010-8899` |
| `1-555-010-1122` | `+1-555-010-1122` |
| `555 010 3344` | `+1-555-010-3344` |

---

# STEP 6: Validate required fields

This node checks that the three fields HubSpot absolutely requires are present: email, name, and phone. Any row missing any of these gets routed to the invalid submissions path instead of continuing.

## Add the node

1. Add an `IF` node after `Code - Format Phone`.
2. Rename it to:

```text
IF - Required Fields Check
```

## Configure the conditions

Set `Combine` to `AND`.

Add three conditions:

**Condition 1**

- `Value 1`: `{{ $json.parent_email }}`
- `Operation`: `Is Not Empty`

**Condition 2**

- `Value 1`: `{{ $json.parent_name }}`
- `Operation`: `Is Not Empty`

**Condition 3**

- `Value 1`: `{{ $json.parent_phone }}`
- `Operation`: `Is Not Empty`

The `TRUE` output goes to the Switch node in STEP 9.

The `FALSE` output goes to the invalid lead path in STEP 7.

---

# STEP 7: Handle invalid submissions

These two nodes together capture any row that failed the required fields check.

## Add the Set node

1. Add a `Set` node connected to the `FALSE` output of `IF - Required Fields Check`.
2. Rename it to:

```text
Set - Mark Invalid Lead
```

3. Add these two fields:

- `failure_reason` = `Missing required field: email, name, or phone`
- `logged_at` = `{{ new Date().toISOString() }}`

## Add the Google Sheets logging node

1. Add a `Google Sheets` node after `Set - Mark Invalid Lead`.
2. Rename it to:

```text
Google Sheets - Log Invalid Submission
```

3. Configure it:

- `Operation`: `Append Row`
- `Document ID`: your spreadsheet ID
- `Sheet Name`: `Invalid Submissions`

4. Map these columns:

- `Submission_ID` → `{{ $json.submission_id }}`
- `Parent_Name` → `{{ $json.parent_name_raw }}`
- `Parent_Email` → `{{ $json.parent_email }}`
- `Parent_Phone` → `{{ $json.parent_phone_raw }}`
- `Student_Age` → `{{ $json.student_age }}`
- `Program_Interest` → `{{ $json.program_interest }}`
- `Desired_Start_Date` → `{{ $json.desired_start_date }}`
- `Failure_Reason` → `{{ $json.failure_reason }}`
- `Logged_At` → `{{ $json.logged_at }}`

5. In the node settings, enable:

- `Retry On Fail`: on
- `Max Tries`: `3`
- `Wait Between Tries (ms)`: `2000`

This path ends here. Invalid leads do not proceed to HubSpot.

---

# STEP 8: Install the Switch node for program routing

This is the main branching controller. It reads the `program_interest` field and sends the item down one of four outputs.

## Add the node

1. Add a `Switch` node connected to the `TRUE` output of `IF - Required Fields Check`.
2. Rename it to:

```text
Switch - Route by Program Interest
```

## Configure the rules

Set `Mode` to `Rules`.

Set the `Input` field to:

```text
{{ $json.program_interest }}
```

Add these three rules in order:

**Rule 1 — Infant**

- `Operation`: `Equal`
- `Value`: `Infant`
- Output label: `Infant`

**Rule 2 — Toddler**

- `Operation`: `Equal`
- `Value`: `Toddler`
- Output label: `Toddler`

**Rule 3 — Pre-K**

- `Operation`: `Equal`
- `Value`: `Pre-K`
- Output label: `Pre-K`

Enable `Fallback Output` at the bottom. This catches any row where `program_interest` is empty, misspelled, or does not match any of the three known values.

The Switch node will have four outputs:
- Output 1: Infant
- Output 2: Toddler
- Output 3: Pre-K
- Output 4 (Fallback): Unknown

---

# STEP 9: Assign priority — Infant (High)

This node tags Infant leads as High priority.

## Add the node

1. Add a `Set` node connected to Output 1 (Infant) of `Switch - Route by Program Interest`.
2. Rename it to:

```text
Set - Priority High (Infant)
```

3. Add these two fields:

- `priority` = `High`
- `priority_flag` = `🔥 High`

This item will connect to the Merge node in STEP 13.

---

# STEP 10: Assign priority — Toddler (Normal)

1. Add a `Set` node connected to Output 2 (Toddler) of `Switch - Route by Program Interest`.
2. Rename it to:

```text
Set - Priority Normal (Toddler)
```

3. Add these two fields:

- `priority` = `Normal`
- `priority_flag` = `✅ Normal`

This item will also connect to the Merge node in STEP 13.

---

# STEP 11: Assign priority — Pre-K (Normal)

1. Add a `Set` node connected to Output 3 (Pre-K) of `Switch - Route by Program Interest`.
2. Rename it to:

```text
Set - Priority Normal (Pre-K)
```

3. Add these two fields:

- `priority` = `Normal`
- `priority_flag` = `✅ Normal`

This item will also connect to the Merge node in STEP 13.

---

# STEP 12: Assign priority — Unknown fallback

1. Add a `Set` node connected to the Fallback output of `Switch - Route by Program Interest`.
2. Rename it to:

```text
Set - Priority Unknown
```

3. Add these two fields:

- `priority` = `Unknown`
- `priority_flag` = `❓ Unknown`

This item will also connect to the Merge node in STEP 13.

---

# STEP 13: Merge all priority paths back together

All four priority Set nodes rejoin here. This ensures the workflow continues as a single stream regardless of which program interest was detected.

## Add the node

1. Add a `Merge` node.
2. Rename it to:

```text
Merge - Rejoin After Priority
```

3. Set `Mode` to `Append`.

4. Connect all four nodes to this Merge node as inputs:

- `Set - Priority High (Infant)`
- `Set - Priority Normal (Toddler)`
- `Set - Priority Normal (Pre-K)`
- `Set - Priority Unknown`

After this node, every item in the workflow has a `priority` field and a `priority_flag` field regardless of program type.

---

# STEP 14: Create or update the HubSpot contact

This is the primary CRM write. It uses the parent email as the unique identifier, which means if the same parent submits twice, HubSpot will update the existing contact rather than creating a duplicate.

## Add the node

1. Add a `HubSpot` node after `Merge - Rejoin After Priority`.
2. Rename it to:

```text
HubSpot - Create or Update Contact
```

## Configure the node

- `Operation`: `Create or Update`
- `Resource`: `Contact`
- `Credential`: your HubSpot Private App Token credential

Set the `Email` field to:

```text
{{ $json.parent_email }}
```

Then map these Additional Fields:

- `First Name` → `{{ $json.first_name }}`
- `Last Name` → `{{ $json.last_name }}`
- `Phone Number` → `{{ $json.parent_phone }}`

To add the custom Student Age property:

1. Under `Additional Fields`, find or add a `Custom Properties` section.
2. Add one custom property:
   - `Property Name`: `student_age`
   - `Value`: `{{ $json.student_age }}`

## Node settings

In the node settings panel, enable:

- `Retry On Fail`: on
- `Max Tries`: `3`
- `Wait Between Tries (ms)`: `3000`
- `Continue On Fail`: on — this is important because it allows the IF node in STEP 15 to handle the failure gracefully rather than crashing the workflow

## What this node returns on success

```json
{
  "id": "12345678",
  "properties": {
    "email": "s.jenkins88@gmail.com",
    "firstname": "Sarah",
    "lastname": "Jenkins",
    "phone": "+1-555-010-2233"
  }
}
```

The `id` field is the HubSpot Contact ID. You will reference this when creating the Deal.

---

# STEP 15: Check whether the contact was created

Since `Continue On Fail` is enabled on the HubSpot node, failures will pass through as items with error information rather than stopping the workflow. This IF node reads that and routes accordingly.

## Add the node

1. Add an `IF` node after `HubSpot - Create or Update Contact`.
2. Rename it to:

```text
IF - Contact Created?
```

## Configure the condition

Add one condition:

- `Value 1`: `{{ $json.id }}`
- `Operation`: `Is Not Empty`

The `TRUE` output (contact was created) goes to STEP 18.

The `FALSE` output (contact failed) goes to STEP 16.

---

# STEP 16: Package the HubSpot error

This node prepares a structured error payload so the Slack error alert has enough detail to be actionable.

## Add the node

1. Add a `Set` node connected to the `FALSE` output of `IF - Contact Created?`.
2. Rename it to:

```text
Set - HubSpot Error Payload
```

3. Add these fields:

- `error_source` = `HubSpot - Create or Update Contact`
- `error_parent_name` = `{{ $json.parent_name }}`
- `error_parent_email` = `{{ $json.parent_email }}`
- `error_parent_phone` = `{{ $json.parent_phone }}`
- `error_program` = `{{ $json.program_interest }}`
- `error_message` = `{{ $json.error?.message || "HubSpot contact creation returned no ID" }}`
- `error_time` = `{{ new Date().toISOString() }}`

---

# STEP 17: Send the admin error alert to Slack

This node fires only when a HubSpot write fails. It posts directly to the admin channel with enough context for someone to manually create the record.

## Add the node

1. Add a `Slack` node after `Set - HubSpot Error Payload`.
2. Rename it to:

```text
Slack - Send Admin Error Alert
```

## Configure the node

- `Operation`: `Send Message`
- `Credential`: your Slack OAuth2 credential
- `Channel`: your `#admin-errors` channel ID

Set the `Message Text` to:

```text
{{ "⚠️ HubSpot Write Failed\n\n*Parent:* " + $json.error_parent_name + "\n*Email:* " + $json.error_parent_email + "\n*Phone:* " + $json.error_parent_phone + "\n*Program:* " + $json.error_program + "\n*Error:* " + $json.error_message + "\n*Time:* " + $json.error_time + "\n\nPlease create this contact manually in HubSpot." }}
```

The `Merge - Final Join` node in STEP 21 will receive output from this node.

---

# STEP 18: Create the HubSpot deal

This node runs only when the contact was created successfully. It creates a Deal in the Initial Inquiry stage and links it to the contact.

## Add the node

1. Add a `HubSpot` node connected to the `TRUE` output of `IF - Contact Created?`.
2. Rename it to:

```text
HubSpot - Create Deal
```

## Configure the node

- `Operation`: `Create`
- `Resource`: `Deal`
- `Credential`: your HubSpot Private App Token credential

Set these fields:

- `Deal Name` → `{{ $json.parent_name + " Enrollment" }}`

Under `Additional Fields`, add:

- `Pipeline`: `Sales Pipeline` (or the internal ID of your pipeline — find this in HubSpot Settings → Deals → Pipelines)
- `Deal Stage`: `Initial Inquiry` (or the internal stage ID — find this on the same page)
- `Close Date`: `{{ $json.desired_start_date }}`

To associate the deal with the contact, look for the `Associations` section in the Additional Fields:

- `Contact ID` → `{{ $json.id }}`

If your n8n HubSpot node version does not expose a direct Association field for Create Deal, you can add a second HubSpot node after this one using:

- `Operation`: `Associate`
- `Resource`: `Deal`
- `Deal ID`: `{{ $json.id }}` (from the deal creation response)
- `To Object Type`: `Contact`
- `To Object ID`: the contact ID stored from node 14

For the purposes of this guide, the primary deal creation node is this one.

## Node settings

- `Retry On Fail`: on
- `Max Tries`: `3`
- `Wait Between Tries (ms)`: `3000`

## What this node returns on success

```json
{
  "id": "98765432",
  "properties": {
    "dealname": "Sarah Jenkins Enrollment",
    "dealstage": "appointmentscheduled",
    "pipeline": "default"
  }
}
```

The `id` here is the Deal ID. You will pass it to the execution log.

---

# STEP 19: Build the Slack enrollment message

This node prepares all the fields needed for the Slack notification in one clean structured object.

## Add the node

1. Add a `Set` node after `HubSpot - Create Deal`.
2. Rename it to:

```text
Set - Build Slack Message
```

3. Add these fields:

- `slack_contact_id` = `{{ $node["HubSpot - Create or Update Contact"].json.id }}`
- `slack_deal_id` = `{{ $json.id }}`
- `slack_parent_name` = `{{ $node["HubSpot - Create or Update Contact"].json.properties?.firstname + " " + $node["HubSpot - Create or Update Contact"].json.properties?.lastname }}`
- `slack_email` = `{{ $node["Code - Format Phone"].json.parent_email }}`
- `slack_phone` = `{{ $node["Code - Format Phone"].json.parent_phone }}`
- `slack_program` = `{{ $node["Code - Format Phone"].json.program_interest }}`
- `slack_priority_flag` = `{{ $node["Merge - Rejoin After Priority"].json.priority_flag }}`
- `slack_start_date` = `{{ $node["Code - Format Phone"].json.desired_start_date }}`
- `slack_submission_id` = `{{ $node["Code - Format Phone"].json.submission_id }}`

The reason this node references upstream nodes by name using `$node["..."]` is to ensure the values are coming from the cleanest source available even if the HubSpot response reshaped the payload.

---

# STEP 20: Send the enrollment notification to Slack

This node posts the formatted message to the admissions team.

## Add the node

1. Add a `Slack` node after `Set - Build Slack Message`.
2. Rename it to:

```text
Slack - Send Enrollment Notification
```

## Configure the node

- `Operation`: `Send Message`
- `Credential`: your Slack OAuth2 credential
- `Channel`: your `#admissions-alerts` channel ID

Set the `Message Text` to this expression:

```text
{{ "🎓 *New Enrollment Inquiry*\n\n*Submission ID:* " + $json.slack_submission_id + "\n*Parent:* " + $json.slack_parent_name + "\n*Email:* " + $json.slack_email + "\n*Phone:* " + $json.slack_phone + "\n*Program:* " + $json.slack_program + "\n*Priority:* " + $json.slack_priority_flag + "\n*Desired Start:* " + $json.slack_start_date + "\n*HubSpot Contact ID:* " + $json.slack_contact_id + "\n*Deal ID:* " + $json.slack_deal_id }}
```

The resulting Slack message will look like this:

```text
🎓 *New Enrollment Inquiry*

*Submission ID:* LS-0001
*Parent:* Sarah Jenkins
*Email:* s.jenkins88@gmail.com
*Phone:* +1-555-010-2233
*Program:* Infant
*Priority:* 🔥 High
*Desired Start:* 2024-09-01
*HubSpot Contact ID:* 12345678
*Deal ID:* 98765432
```

---

# STEP 21: Merge the success and error paths

The final Merge node brings together both the success path (Slack enrollment notification) and the error path (Slack admin error alert) so the execution log always runs regardless of which path was taken.

## Add the node

1. Add a `Merge` node.
2. Rename it to:

```text
Merge - Final Join
```

3. Set `Mode` to `Append`.

4. Connect both of these nodes as inputs:

- `Slack - Send Enrollment Notification`
- `Slack - Send Admin Error Alert`

---

# STEP 22: Write the final execution log

This node writes a confirmation row to Google Sheets at the end of every successful run.

## Add the node

1. Add a `Set` node after `Merge - Final Join`.
2. Rename it to:

```text
Set - Final Execution Log
```

3. Add these fields:

- `log_submission_id` = `{{ $node["Set - Clean Raw Data"].json.submission_id }}`
- `log_parent_name` = `{{ $node["Code - Format Name"].json.parent_name }}`
- `log_program` = `{{ $node["Set - Clean Raw Data"].json.program_interest }}`
- `log_priority` = `{{ $node["Merge - Rejoin After Priority"].json.priority }}`
- `log_contact_id` = `{{ $node["HubSpot - Create or Update Contact"].json.id || "FAILED" }}`
- `log_deal_id` = `{{ $node["HubSpot - Create Deal"].json.id || "FAILED" }}`
- `log_completed_at` = `{{ new Date().toISOString() }}`

4. After this `Set` node, add a `Google Sheets` node:

Rename it to:

```text
Google Sheets - Write Execution Log
```

Configure:

- `Operation`: `Append Row`
- `Document ID`: your spreadsheet ID
- `Sheet Name`: `Execution Log`

Map:

- `Submission_ID` → `{{ $json.log_submission_id }}`
- `Parent_Name` → `{{ $json.log_parent_name }}`
- `Program_Interest` → `{{ $json.log_program }}`
- `Priority` → `{{ $json.log_priority }}`
- `HubSpot_Contact_ID` → `{{ $json.log_contact_id }}`
- `Deal_ID` → `{{ $json.log_deal_id }}`
- `Completed_At` → `{{ $json.log_completed_at }}`

Enable:

- `Retry On Fail`: on
- `Max Tries`: `3`
- `Wait Between Tries (ms)`: `2000`

---

# STEP 23: Build the separate error workflow

This workflow is completely independent from the main flow. It catches execution-level failures — cases where a node crashed so hard that the main workflow could not recover. These are different from the HubSpot failure handled in STEP 16, which was a recoverable API error inside a running workflow. This error workflow catches situations where n8n itself could not complete the execution.

## 23A. Add the Error Trigger node

Inside the second workflow (`Little Sprout Academy - Error Logger`), add an `Error Trigger` node.

Rename it to:

```text
Error Trigger - Global
```

This node fires automatically whenever the main workflow has an execution-level failure, provided you link the two workflows together in STEP 23D.

## 23B. Add the Set node for error normalization

1. Add a `Set` node after `Error Trigger - Global`.
2. Rename it to:

```text
Set - Prepare Error Log
```

3. Add these fields:

- `err_timestamp` = `{{ new Date().toISOString() }}`
- `err_workflow` = `{{ $json.workflow?.name || "Little Sprout Academy - Enrollment Lead Pipeline" }}`
- `err_node` = `{{ $json.execution?.lastNodeExecuted || "" }}`
- `err_message` = `{{ $json.execution?.error?.message || $json.error?.message || $json.execution?.data?.resultData?.error?.message || "Unknown execution error" }}`
- `err_started_at` = `{{ $json.execution?.startedAt || "" }}`

## 23C. Add the Slack error alert node

1. Add a `Slack` node after `Set - Prepare Error Log`.
2. Rename it to:

```text
Slack - Global Error Alert
```

3. Configure:

- `Operation`: `Send Message`
- `Credential`: your Slack OAuth2 credential
- `Channel`: your `#admin-errors` channel ID

Set the `Message Text` to:

```text
{{ "🚨 *Workflow Execution Failed*\n\n*Workflow:* " + $json.err_workflow + "\n*Failed Node:* " + $json.err_node + "\n*Error:* " + $json.err_message + "\n*Started At:* " + $json.err_started_at + "\n*Logged At:* " + $json.err_timestamp }}
```

## 23D. Link the error workflow to the main workflow

1. Open the main workflow `Little Sprout Academy - Enrollment Lead Pipeline`.
2. Go to workflow settings (the three-dot menu or gear icon).
3. Find the `Error Workflow` dropdown.
4. Select:

```text
Little Sprout Academy - Error Logger
```

5. Save both workflows.

This is the connection that makes n8n automatically trigger the error workflow whenever the main workflow crashes.

---

# STEP 24: Connection map

Use this map to verify every connection on your canvas is correct.

```text
Google Sheets - New Enrollment Lead
  → Set - Clean Raw Data
  → Code - Format Name
  → Code - Format Phone
  → IF - Required Fields Check

IF - Required Fields Check (FALSE)
  → Set - Mark Invalid Lead
  → Google Sheets - Log Invalid Submission
  [end of invalid path]

IF - Required Fields Check (TRUE)
  → Switch - Route by Program Interest

Switch - Route by Program Interest (Output 1: Infant)
  → Set - Priority High (Infant)
  → Merge - Rejoin After Priority

Switch - Route by Program Interest (Output 2: Toddler)
  → Set - Priority Normal (Toddler)
  → Merge - Rejoin After Priority

Switch - Route by Program Interest (Output 3: Pre-K)
  → Set - Priority Normal (Pre-K)
  → Merge - Rejoin After Priority

Switch - Route by Program Interest (Fallback)
  → Set - Priority Unknown
  → Merge - Rejoin After Priority

Merge - Rejoin After Priority
  → HubSpot - Create or Update Contact
  → IF - Contact Created?

IF - Contact Created? (FALSE)
  → Set - HubSpot Error Payload
  → Slack - Send Admin Error Alert
  → Merge - Final Join

IF - Contact Created? (TRUE)
  → HubSpot - Create Deal
  → Set - Build Slack Message
  → Slack - Send Enrollment Notification
  → Merge - Final Join

Merge - Final Join
  → Set - Final Execution Log
  → Google Sheets - Write Execution Log
```

Error workflow:

```text
Error Trigger - Global
  → Set - Prepare Error Log
  → Slack - Global Error Alert
```

---

# STEP 25: Recommended canvas layout

Lay the workflow out left to right in three horizontal lanes so any team member can read the business logic at a glance.

**Top lane — invalid path**

Place `Set - Mark Invalid Lead` and `Google Sheets - Log Invalid Submission` above the main horizontal flow, connected to the FALSE output of the IF validation node.

**Middle lane — main flow**

Run the main pipeline straight across:

```text
Google Sheets Trigger → Set Clean → Code Name → Code Phone → IF Validate → Switch Program → Merge Priority → HubSpot Contact → IF Created? → HubSpot Deal → Set Slack Builder → Slack Notify → Merge Final → Set Log → Google Sheets Log
```

**Bottom lane — error path**

Place `Set - HubSpot Error Payload` and `Slack - Send Admin Error Alert` below the main flow, branching down from the FALSE output of IF Contact Created and rejoining up into Merge Final Join.

**Priority assignment cluster**

Group the four priority Set nodes (High Infant, Normal Toddler, Normal Pre-K, Unknown) in a vertical stack between the Switch node and the Merge node. This makes it visually obvious that all four paths converge.

**Canvas tips**

Add a sticky note to the Switch node that says:

```text
Infant = High Priority → admissions team follows up same day
Toddler / Pre-K = Normal Priority → standard 48hr follow-up
Unknown = review program_interest field in sheet
```

Add a sticky note to the HubSpot Contact node that says:

```text
Uses email as unique key.
Create or Update prevents duplicates.
Continue On Fail = true → IF node handles errors.
```

---

# STEP 26: Testing checklist

Build and test one section at a time. Do not activate the workflow until all five tests pass.

## Test 1: Name formatting

Manually trigger the workflow with this simulated data or add a row to the sheet:

```text
Submission_ID: LS-TEST-01
Parent_Name: JOHN DOE
Parent_Email: john.doe@test.com
Parent_Phone: 5550101234
Student_Age: 1.2
Program_Interest: Infant
Desired_Start_Date: 2025-01-01
```

Expected results:

1. `Set - Clean Raw Data` outputs `parent_name_raw = "JOHN DOE"`.
2. `Code - Format Name` outputs `parent_name = "John Doe"`, `first_name = "John"`, `last_name = "Doe"`.
3. `Code - Format Phone` outputs `parent_phone = "+1-555-010-1234"`.
4. `IF - Required Fields Check` passes TRUE.
5. Switch routes to Output 1 (Infant).
6. `Set - Priority High (Infant)` outputs `priority = "High"`, `priority_flag = "🔥 High"`.
7. Merge node passes the item through.

## Test 2: Phone formatting

Test each of the six phone formats from the starter data by triggering the workflow with one row at a time. For each one, open the execution log after the Code node fires and confirm the `parent_phone` value is in the format `+1-555-010-XXXX`.

The six formats to confirm:

```text
555-010-2233        → +1-555-010-2233
(555) 010-4455      → +1-555-010-4455
5550106677          → +1-555-010-6677
555.010.8899        → +1-555-010-8899
1-555-010-1122      → +1-555-010-1122
555 010 3344        → +1-555-010-3344
```

## Test 3: Invalid submissions path

Add a row to the sheet with an empty email field. Leave `Parent_Email` blank.

Expected results:

1. `IF - Required Fields Check` fires FALSE.
2. `Set - Mark Invalid Lead` adds `failure_reason` and `logged_at`.
3. `Google Sheets - Log Invalid Submission` appends a row to the `Invalid Submissions` tab.
4. The workflow stops. No HubSpot node fires.
5. Confirm the row appears in Google Sheets with the correct failure reason.

## Test 4: HubSpot contact and deal creation

Add a clean row with valid data for an Infant lead.

Expected results:

1. Workflow runs through all 14 nodes in the main path.
2. HubSpot contact is created or updated.
3. `IF - Contact Created?` fires TRUE.
4. HubSpot deal is created with the name `[Parent Name] Enrollment`.
5. Slack message posts to `#admissions-alerts` with 🔥 High priority flag.
6. `Google Sheets - Write Execution Log` appends a row with both the contact ID and deal ID.

To verify in HubSpot:

- Go to `Contacts` and search for the parent email.
- Confirm first name, last name, and phone are correct.
- Click the contact and check the `Deals` panel. The deal should appear as `[Name] Enrollment` in `Initial Inquiry`.

## Test 5: HubSpot failure simulation

To test the error handling path:

1. Temporarily replace the HubSpot API key in the credential with an invalid string.
2. Trigger the workflow with a valid row.
3. Confirm `IF - Contact Created?` fires FALSE.
4. Confirm `Set - HubSpot Error Payload` runs.
5. Confirm the Slack error alert posts to `#admin-errors` with the parent name and error message.
6. Restore the correct HubSpot credential.

## Test 6: Unknown program interest

Add a row with `Program_Interest` set to something that does not match any of the three known values, for example `Kindergarten` or leave it blank.

Expected results:

1. Switch fallback output fires.
2. `Set - Priority Unknown` runs with `priority = "Unknown"` and `priority_flag = "❓ Unknown"`.
3. Merge passes the item through to HubSpot.
4. The Slack message shows `❓ Unknown` for Priority.

## Test 7: Error workflow

To test the global error workflow:

1. Temporarily break the `Google Sheets - New Enrollment Lead` trigger by entering an invalid spreadsheet ID.
2. Trigger the workflow manually.
3. Confirm a Slack message fires to `#admin-errors` from the `Little Sprout Academy - Error Logger` workflow.

---

# STEP 27: Activation checklist

Before activating either workflow, confirm every item on this list.

1. Google Sheets headers in all three tabs exactly match the column names used in this guide.
2. The `Enrollment Leads` tab has the starter data imported correctly and no extra blank rows above the header.
3. HubSpot `student_age` custom property exists and the internal name is `student_age`.
4. HubSpot `Initial Inquiry` pipeline stage exists.
5. Slack channels `#admissions-alerts` and `#admin-errors` exist and the bot has been invited to both.
6. All five test scenarios pass without errors.
7. The error workflow is assigned to the main workflow in Settings → Error Workflow.
8. Save both workflows.
9. Activate `Little Sprout Academy - Error Logger` first.
10. Activate `Little Sprout Academy - Enrollment Lead Pipeline` second.
11. Add one final test row to the sheet and confirm the Slack notification appears in `#admissions-alerts` within two minutes.

---

# STEP 28: Troubleshooting reference

## Google Sheets trigger does not fire

Check these things in order:

1. The workflow is activated, not just saved. An inactive workflow does not poll the sheet.
2. The OAuth2 credential has the correct Google account that owns the spreadsheet.
3. The sheet tab name is exactly `Enrollment Leads` with no leading or trailing space.
4. The spreadsheet has been shared with the Google account used by the credential.
5. If polling is set to 1 minute, wait at least 90 seconds after adding the row.

## HubSpot authentication error

1. Confirm the Private App Token has these scopes enabled in HubSpot: `crm.objects.contacts.write`, `crm.objects.deals.write`, `crm.objects.contacts.read`.
2. Check the token was not accidentally copied with extra whitespace.
3. In n8n, open the credential and click `Test Credential`. If it fails, generate a new token in HubSpot under Settings → Integrations → Private Apps.

## Slack message not posting

1. Confirm the bot has been invited to the target channel. In Slack, type `/invite @YourBotName` inside the channel.
2. Confirm the channel ID is correct. Channel IDs begin with `C` for public channels. Do not use the channel name.
3. In n8n, open the Slack credential and click `Test Credential`.

## Phone format incorrect

1. Open the execution log for the `Code - Format Phone` node.
2. Check what `parent_phone_raw` contained going in.
3. If the raw value has a country code other than 1, the current code will still work but will produce a `+` prefix rather than `+1-`. Adjust the code logic as needed for your country.
4. If the raw value has fewer than 10 digits, the code returns the original raw value. Check for data entry errors in the sheet.

## Name appears in wrong case

1. Open the execution log for `Code - Format Name`.
2. Check the value of `parent_name_raw` going into the node.
3. If the name contains an apostrophe like `O'Brian`, the character after the apostrophe will be lowercase because `.split(' ')` splits only on spaces. This is expected behavior. If you need apostrophe-aware formatting, add a second pass in the code node.

## Missing required fields — valid row treated as invalid

1. Open the execution log for `IF - Required Fields Check`.
2. Check the values of `parent_email`, `parent_name`, and `parent_phone` at that point in the execution.
3. A common cause is extra whitespace in the original sheet cell. The `Set - Clean Raw Data` node trims fields, but if the trim produces an empty string, the field is treated as missing.

## Duplicate HubSpot contacts being created

This happens when the same parent submits with two different email addresses. The `Create or Update` operation uses email as the key, so two different emails will create two different contacts. The solution is to review your intake form and enforce email consistency, or to manually merge duplicates in HubSpot.

## Deal not associated with the contact in HubSpot

If the HubSpot node version you are using does not support inline association on the Create Deal operation, add a second HubSpot node immediately after `HubSpot - Create Deal`:

- `Operation`: `Associate`
- `Resource`: `Deal`
- `Deal ID`: `{{ $json.id }}`
- `To Object Type`: `Contact`
- `To Object ID`: `{{ $node["HubSpot - Create or Update Contact"].json.id }}`

Rename it to `HubSpot - Associate Deal to Contact`.

---

# STEP 29: Pro tips for production

**Use descriptive node names from day one.** Node names show up in error messages, execution logs, and the canvas. Names like `Code - Format Phone` are immediately readable. Names like `Code` or `Node 4` are not.

**Add canvas sticky notes at every branch.** Annotate the Switch node, the two IF nodes, and the Merge nodes. Future you or a teammate will thank you when debugging at midnight.

**Keep retry settings consistent.** Every node that writes to an external service in this workflow uses 3 retries with a 2–3 second wait. Do not lower these. A momentary API timeout should not lose a lead.

**Keep Continue On Fail limited to the HubSpot contact node.** That node uses it intentionally so the downstream IF node can handle the failure gracefully. Do not enable Continue On Fail on every node in the workflow, or silent failures will be hard to catch.

**Test with the full 30-row starter CSV, not just one row.** The starter data was designed to include duplicates (same parent, different emails), varied phone formats, and mixed name cases. Running the full set will expose edge cases before they appear in production.

**Monitor the Execution Log tab in Google Sheets weekly.** A row with `FAILED` in the `HubSpot_Contact_ID` column means the admin error Slack alert fired but was missed. Reviewing the log gives you a second chance to catch dropped leads.

**Before going live, scrub the starter data.** The CSV contains fictional leads. Clear all rows from the `Enrollment Leads` tab (except the header) before you start accepting real parent inquiries.

---

# Final result

Once built and activated, Little Sprout Academy has a fully automated lead pipeline that:

- captures every new form submission the moment it lands in the sheet
- normalizes messy name and phone data automatically
- rejects incomplete submissions before they reach HubSpot
- tags Infant leads as high priority for same-day follow-up
- creates or updates HubSpot contacts without creating duplicates
- opens a Deal in the Initial Inquiry stage linked to the contact
- notifies the admissions team in Slack within seconds
- sends a Slack alert when CRM writes fail so no lead is ever silently dropped
- logs every execution to a Google Sheet for auditing
- captures global execution failures in a separate error workflow

The result is a system where no enrollment inquiry is missed, no data entry is manual, and the admissions team is notified before they have time to check their inbox.
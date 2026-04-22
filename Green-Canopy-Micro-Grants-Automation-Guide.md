# Green Canopy Micro-Grants - Manual n8n Build Guide

This guide is written for building the workflow yourself inside n8n from scratch.

It is intentionally detailed and structured like a real implementation document, so you can follow it node by node without needing to import workflow JSON.

The project automates incoming grant applications and routes them by applicant type:

- `Corporate` applicants become high-priority leads
- `Individual` and `School` applicants go to standard processing

You will build:

1. A main intake and routing workflow
2. A separate error-handling workflow

This guide uses only these n8n nodes:

- `Webhook`
- `Set`
- `Code`
- `Switch`
- `IF`
- `HubSpot`
- `Slack`
- `Wait`
- `Error Trigger`

## What this workflow does

When a grant application is submitted, the workflow:

1. Receives the data in a webhook
2. Normalizes the incoming fields
3. Validates that the required fields are present
4. Tags the applicant as `High Priority` or `Standard`
5. Routes `Corporate` applicants into an urgent partnership lane
6. Routes `Individual` and `School` applicants into standard review
7. Creates or updates a HubSpot contact
8. Creates a HubSpot follow-up task
9. Sends a Slack notification to the team
10. Uses a second workflow to notify Slack if the workflow itself fails

This is a strong pattern for a real intake system because it separates:

- intake
- normalization
- validation
- routing
- CRM handling
- team notification
- workflow-level error alerting

---

## Before you start

Prepare these systems first so the workflow build goes smoothly.

### 1. n8n

Make sure:

- n8n is installed
- you can create and save workflows
- you can activate workflows

### 2. HubSpot

You need a HubSpot account with access to:

- Contacts
- Tasks

Create a custom contact property in HubSpot before building the workflow.

Recommended property:

```text
Property name: priority
Label: Priority
Type: Dropdown select or single-line text
Values:
- High Priority
- Standard
```

This property is important because the workflow will assign:

- `High Priority` for corporate applicants
- `Standard` for individual or school applicants

### 3. Slack

You need:

- a Slack workspace
- a Slack app or credential already usable in n8n
- a channel for intake alerts

Recommended channel:

```text
#grant-alerts
```

### 4. Credentials in n8n

Create or verify these credentials:

- `HubSpot OAuth2 API`
- `Slack OAuth2 API`

### 5. Sample payload

Keep this payload ready for testing:

```json
{
  "name": "Jane Doe",
  "email": "jane@example.com",
  "organization_type": "Corporate",
  "project_budget": 500,
  "message": "We want to sponsor urban gardens"
}
```

---

## Workflow names

Use these exact names so everything stays easy to recognize:

Main workflow:

```text
Green Canopy Micro-Grants - Automated Intake & Lead Routing
```

Error workflow:

```text
Green Canopy Micro-Grants - Error Handler
```

---

## Main workflow overview

You will build the main workflow in this order:

1. `Webhook - Incoming Application`
2. `Code - Structure Application Data`
3. `IF - Validation Failed?`
4. `Slack - Invalid Application Alert`
5. `Set - Normalize Fields`
6. `Switch - Organization Type`
7. `HubSpot - Create or Update Corporate Contact`
8. `Wait - Corporate Sync Buffer`
9. `HubSpot - Create Corporate Task`
10. `Slack - Notify Corporate Lead`
11. `HubSpot - Create or Update Standard Contact`
12. `Wait - Standard Sync Buffer`
13. `HubSpot - Create Standard Task`
14. `Slack - Notify Standard Lead`

This creates:

- one intake point
- one validation gate
- one clean normalization layer
- two clear business lanes

---

## STEP 0: Create the two workflows

### 0A. Create the main workflow

1. Open n8n.
2. Click `Create Workflow`.
3. Rename it to:

```text
Green Canopy Micro-Grants - Automated Intake & Lead Routing
```

4. Save it once before adding nodes.

### 0B. Create the error workflow

1. Create a second workflow.
2. Rename it to:

```text
Green Canopy Micro-Grants - Error Handler
```

3. Save it once.

---

## STEP 1: Add the webhook trigger

### Node 1: Webhook - Incoming Application

Add a `Webhook` node and rename it:

```text
Webhook - Incoming Application
```

Configure it like this:

- `HTTP Method`: `POST`
- `Path`:

```text
green-canopy-micro-grants-intake
```

- `Authentication`: `None`
- `Response Mode`: `On Received`

If your n8n version shows response settings, use:

- `Response Code`: `200`

This keeps the webhook responsive while the workflow continues processing.

### Expected incoming payload

The webhook should accept a JSON body like:

```json
{
  "name": "Jane Doe",
  "email": "jane@example.com",
  "organization_type": "Corporate",
  "project_budget": 500,
  "message": "We want to sponsor urban gardens"
}
```

---

## STEP 2: Structure and normalize the incoming application

### Why this step matters

Raw webhook data can be inconsistent.

This step makes the payload predictable by:

- trimming spaces
- lowercasing the email
- standardizing the organization type
- splitting the applicant name into first and last name
- deciding the routing bucket
- generating a validation error if the input is incomplete

### Node 2: Code - Structure Application Data

Add a `Code` node after the webhook and rename it:

```text
Code - Structure Application Data
```

Set:

- `Mode`: `Run Once for Each Item`
- `Language`: `JavaScript`

Paste this code:

```javascript
const payload = $json.body ?? $json;
const rawName = String(payload.name ?? '').trim();
const rawEmail = String(payload.email ?? '').trim().toLowerCase();
const rawOrganizationType = String(payload.organization_type ?? '').trim();
const rawMessage = String(payload.message ?? '').trim();
const parsedBudget = Number(payload.project_budget ?? 0);
const projectBudget = Number.isFinite(parsedBudget) ? parsedBudget : 0;
const normalizedOrg = rawOrganizationType.toLowerCase();

let organizationType = 'Unknown';
let routingBucket = 'Invalid';

if (['corporate', 'company', 'business'].includes(normalizedOrg)) {
  organizationType = 'Corporate';
  routingBucket = 'Corporate';
} else if (
  ['individual', 'individual/school', 'individual / school', 'individual-school'].includes(normalizedOrg)
) {
  organizationType = 'Individual';
  routingBucket = 'Standard';
} else if (normalizedOrg === 'school') {
  organizationType = 'School';
  routingBucket = 'Standard';
}

const nameParts = rawName.split(/\s+/).filter(Boolean);
const firstName = nameParts[0] ?? '';
const lastName = nameParts.slice(1).join(' ');

const validationErrors = [];

if (!rawName) {
  validationErrors.push('Missing name');
}

if (!rawEmail) {
  validationErrors.push('Missing email');
}

if (routingBucket === 'Invalid') {
  validationErrors.push('Unsupported organization_type');
}

return [{
  json: {
    name: rawName,
    email: rawEmail,
    organization_type: organizationType,
    organization_type_raw: rawOrganizationType,
    project_budget: projectBudget,
    message: rawMessage,
    priority_tag: routingBucket === 'Corporate' ? 'High Priority' : 'Standard',
    routing_bucket: routingBucket,
    first_name: firstName,
    last_name: lastName,
    validation_error: validationErrors.join('; '),
  },
}];
```

### Output of this node

After this node, the data is shaped like:

```json
{
  "name": "Jane Doe",
  "email": "jane@example.com",
  "organization_type": "Corporate",
  "organization_type_raw": "Corporate",
  "project_budget": 500,
  "message": "We want to sponsor urban gardens",
  "priority_tag": "High Priority",
  "routing_bucket": "Corporate",
  "first_name": "Jane",
  "last_name": "Doe",
  "validation_error": ""
}
```

---

## STEP 3: Add validation control

This step prevents incomplete or unsupported data from reaching HubSpot.

### Node 3: IF - Validation Failed?

Add an `IF` node after the Code node and rename it:

```text
IF - Validation Failed?
```

Configure the condition:

- `Left Value`:

```javascript
={{ $json["validation_error"] || '' }}
```

- `Operator`: `is not empty`

Meaning:

- `TRUE` = something is wrong with the payload
- `FALSE` = the payload is valid and can continue

---

## STEP 4: Create the invalid-payload alert path

If the webhook receives a broken payload, the team should know quickly.

### Node 4: Slack - Invalid Application Alert

Connect the `TRUE` output from `IF - Validation Failed?` to a `Slack` node.

Rename it:

```text
Slack - Invalid Application Alert
```

Configure it:

- `Authentication`: your Slack OAuth2 credential
- `Send To`: `Channel`
- `Channel ID`: your Slack alerts channel

Use this message text:

```javascript
=⚠️ Invalid grant application payload

Name: {{ $json["name"] || "Missing" }}
Email: {{ $json["email"] || "Missing" }}
Organization Type: {{ $json["organization_type_raw"] || "Missing" }}
Reason: {{ $json["validation_error"] }}
```

This path ends here.

That is intentional.

If validation fails, the workflow should stop before it writes to HubSpot.

---

## STEP 5: Normalize the fields for the valid path

Now build the clean internal payload used by the routing and CRM steps.

### Node 5: Set - Normalize Fields

Connect the `FALSE` output from `IF - Validation Failed?` to a `Set` node.

Rename it:

```text
Set - Normalize Fields
```

Create these assignments:

- `name` = `{{ $json["name"] }}`
- `email` = `{{ $json["email"] }}`
- `organization_type` = `{{ $json["organization_type"] }}`
- `project_budget` = `{{ $json["project_budget"] }}`
- `message` = `{{ $json["message"] }}`
- `priority_tag` = `{{ $json["priority_tag"] }}`
- `routing_bucket` = `{{ $json["routing_bucket"] }}`
- `first_name` = `{{ $json["first_name"] }}`
- `last_name` = `{{ $json["last_name"] }}`

This node creates the final shared object used by the rest of the workflow.

---

## STEP 6: Route by organization type

This is the main branch controller.

### Node 6: Switch - Organization Type

Add a `Switch` node after `Set - Normalize Fields`.

Rename it:

```text
Switch - Organization Type
```

Create these cases.

### Case 1: Corporate

- `Left Value`:

```javascript
={{ $json["routing_bucket"] }}
```

- `Operator`: `equals`
- `Right Value`:

```text
Corporate
```

### Case 2: Standard

- `Left Value`:

```javascript
={{ $json["routing_bucket"] }}
```

- `Operator`: `equals`
- `Right Value`:

```text
Standard
```

Expected behavior:

- output 1 = corporate path
- output 2 = standard path

You should not need a fallback output because invalid data was already blocked by the IF node.

---

## STEP 7: Build the corporate high-priority path

This lane handles `Corporate` applicants.

The business logic is:

1. create or update the contact
2. wait briefly so downstream operations stay stable
3. create a corporate follow-up task
4. notify Slack

### Node 7: HubSpot - Create or Update Corporate Contact

Connect output 1 from `Switch - Organization Type` to a `HubSpot` node.

Rename it:

```text
HubSpot - Create or Update Corporate Contact
```

Configure it like this:

- `Credential`: your HubSpot OAuth2 credential
- `Resource`: `Contact`
- `Operation`: `Create or Update`

Map the core fields:

- `Email` -> `{{ $json["email"] }}`
- `First Name` -> `{{ $json["first_name"] }}`
- `Last Name` -> `{{ $json["last_name"] }}`

For custom property mapping:

- property `priority` -> `{{ $json["priority_tag"] }}`

Expected value on this branch:

```text
High Priority
```

### Node 8: Wait - Corporate Sync Buffer

Add a `Wait` node after the corporate contact node.

Rename it:

```text
Wait - Corporate Sync Buffer
```

Configure:

- `Resume`: `After Time Interval`
- `Amount`: `2`
- `Unit`: `Seconds`

This small delay gives the contact update a moment to settle before creating the next object.

### Node 9: HubSpot - Create Corporate Task

Add another `HubSpot` node after the wait node.

Rename it:

```text
HubSpot - Create Corporate Task
```

Configure:

- `Credential`: your HubSpot OAuth2 credential
- `Resource`: `Task`
- `Operation`: `Create`

Set the task details:

- `Subject`:

```text
Grant Application - Corporate Lead
```

- `Body`:

```javascript
=Priority: HIGH
Applicant Type: {{ $('Set - Normalize Fields').item.json["organization_type"] }}
Budget: ${{ $('Set - Normalize Fields').item.json["project_budget"] }}
Message: {{ $('Set - Normalize Fields').item.json["message"] }}
```

- `Status`: `NOT_STARTED`

Associate the task with the contact created or updated in the previous HubSpot node.

Depending on your n8n HubSpot node version, use the contact ID field returned by the corporate contact node. If more than one possible contact ID field appears in test output, use the one your node returns reliably.

Recommended expression pattern:

```javascript
{{ $json["id"] || $json["vid"] || $json["canonical-vid"] || $json["contactId"] }}
```

### Node 10: Slack - Notify Corporate Lead

Add a `Slack` node after the corporate task node.

Rename it:

```text
Slack - Notify Corporate Lead
```

Configure:

- `Authentication`: Slack OAuth2
- `Send To`: `Channel`
- `Channel ID`: your `#grant-alerts` channel

Use this message:

```javascript
=🚨 *URGENT: Potential Partner*

Name: {{ $('Set - Normalize Fields').item.json["name"] }}
Email: {{ $('Set - Normalize Fields').item.json["email"] }}
Budget: ${{ $('Set - Normalize Fields').item.json["project_budget"] }}
```

This finishes the corporate lane.

---

## STEP 8: Build the standard processing path

This lane handles both:

- `Individual`
- `School`

The structure is similar, but the business urgency is lower.

### Node 11: HubSpot - Create or Update Standard Contact

Connect output 2 from `Switch - Organization Type` to a `HubSpot` node.

Rename it:

```text
HubSpot - Create or Update Standard Contact
```

Configure:

- `Credential`: your HubSpot OAuth2 credential
- `Resource`: `Contact`
- `Operation`: `Create or Update`

Map:

- `Email` -> `{{ $json["email"] }}`
- `First Name` -> `{{ $json["first_name"] }}`
- `Last Name` -> `{{ $json["last_name"] }}`
- custom property `priority` -> `{{ $json["priority_tag"] }}`

Expected value here:

```text
Standard
```

### Node 12: Wait - Standard Sync Buffer

Add a `Wait` node after the standard contact node.

Rename it:

```text
Wait - Standard Sync Buffer
```

Configure:

- `Resume`: `After Time Interval`
- `Amount`: `2`
- `Unit`: `Seconds`

### Node 13: HubSpot - Create Standard Task

Add a `HubSpot` node after the wait node.

Rename it:

```text
HubSpot - Create Standard Task
```

Configure:

- `Credential`: HubSpot OAuth2
- `Resource`: `Task`
- `Operation`: `Create`

Set:

- `Subject`:

```text
Grant Application - Community Request
```

- `Body`:

```javascript
=Priority: NORMAL
Applicant Type: {{ $('Set - Normalize Fields').item.json["organization_type"] }}
Budget: ${{ $('Set - Normalize Fields').item.json["project_budget"] }}
Message: {{ $('Set - Normalize Fields').item.json["message"] }}
```

- `Status`: `NOT_STARTED`

Associate the task with the contact returned by the standard contact node.

Use the same defensive contact-id pattern if needed:

```javascript
{{ $json["id"] || $json["vid"] || $json["canonical-vid"] || $json["contactId"] }}
```

### Node 14: Slack - Notify Standard Lead

Add a `Slack` node after the standard task node.

Rename it:

```text
Slack - Notify Standard Lead
```

Configure:

- `Authentication`: Slack OAuth2
- `Send To`: `Channel`
- `Channel ID`: your alerts channel

Use this message:

```javascript
=🌱 New Grant Application

Name: {{ $('Set - Normalize Fields').item.json["name"] }}
Type: {{ $('Set - Normalize Fields').item.json["organization_type"] }}
```

This finishes the standard lane.

---

## STEP 9: Build the separate error workflow

This is a separate workflow, not part of the main canvas.

Its purpose is simple:

- if the main workflow crashes
- Slack should receive an alert with the node and error message

### Node E1: Error Trigger - Workflow Failure

In the second workflow, add an `Error Trigger` node.

Rename it:

```text
Error Trigger - Workflow Failure
```

### Node E2: Slack - Notify Workflow Error

Add a `Slack` node after the Error Trigger.

Rename it:

```text
Slack - Notify Workflow Error
```

Configure:

- `Authentication`: Slack OAuth2
- `Send To`: `Channel`
- `Channel ID`: your alerts channel

Use this message:

```javascript
=❌ Workflow Error

Node: {{ ($json["execution"] && $json["execution"]["lastNodeExecuted"]) || ($json["node"] && $json["node"]["name"]) || $json["node"] || "Unknown Node" }}
Error: {{ ($json["execution"] && $json["execution"]["error"] && $json["execution"]["error"]["message"]) || ($json["error"] && $json["error"]["message"]) || "Unknown error" }}
```

This keeps the error workflow small and fast.

---

## STEP 10: Link the error workflow to the main workflow

After both workflows are saved:

1. Open the main workflow
2. Go to workflow `Settings`
3. Find `Error Workflow`
4. Select:

```text
Green Canopy Micro-Grants - Error Handler
```

5. Save again

This is what activates the error workflow as the failure handler for the main one.

---

## STEP 11: Connection map

Use this to verify your canvas.

### Main workflow

```text
Webhook - Incoming Application
  -> Code - Structure Application Data
  -> IF - Validation Failed?

IF - Validation Failed? TRUE
  -> Slack - Invalid Application Alert

IF - Validation Failed? FALSE
  -> Set - Normalize Fields
  -> Switch - Organization Type

Switch - Organization Type output 1
  -> HubSpot - Create or Update Corporate Contact
  -> Wait - Corporate Sync Buffer
  -> HubSpot - Create Corporate Task
  -> Slack - Notify Corporate Lead

Switch - Organization Type output 2
  -> HubSpot - Create or Update Standard Contact
  -> Wait - Standard Sync Buffer
  -> HubSpot - Create Standard Task
  -> Slack - Notify Standard Lead
```

### Error workflow

```text
Error Trigger - Workflow Failure
  -> Slack - Notify Workflow Error
```

---

## STEP 12: Recommended canvas layout

To make the workflow readable:

- place the intake and validation nodes on the far left
- keep the corporate path on the top row
- keep the standard path on the bottom row
- place the error workflow on a separate small canvas

Recommended layout:

- left column:
  - `Webhook - Incoming Application`
  - `Code - Structure Application Data`
  - `IF - Validation Failed?`
- top middle:
  - `Slack - Invalid Application Alert`
- center:
  - `Set - Normalize Fields`
  - `Switch - Organization Type`
- top lane:
  - corporate contact
  - wait
  - corporate task
  - corporate Slack
- bottom lane:
  - standard contact
  - wait
  - standard task
  - standard Slack

This left-to-right layout makes the routing logic obvious.

---

## STEP 13: Testing plan

Test one path at a time.

### Test 1: Corporate application

Send:

```json
{
  "name": "Jane Doe",
  "email": "jane@example.com",
  "organization_type": "Corporate",
  "project_budget": 500,
  "message": "We want to sponsor urban gardens"
}
```

Expected result:

1. Data passes validation
2. `priority_tag` becomes `High Priority`
3. Switch routes to the corporate branch
4. HubSpot contact is created or updated
5. HubSpot task is created
6. Slack receives the urgent notification

### Test 2: Individual application

Send:

```json
{
  "name": "Alex Carter",
  "email": "alex@example.com",
  "organization_type": "Individual",
  "project_budget": 250,
  "message": "We want a pollinator garden for our neighborhood block"
}
```

Expected result:

1. Data passes validation
2. `priority_tag` becomes `Standard`
3. Switch routes to the standard branch
4. HubSpot contact is created or updated
5. HubSpot task is created
6. Slack receives the standard notification

### Test 3: School application

Send:

```json
{
  "name": "Maya Singh",
  "email": "maya@school.edu",
  "organization_type": "School",
  "project_budget": 1200,
  "message": "Our students want to expand the campus food forest"
}
```

Expected result:

1. Data passes validation
2. It goes through the same standard branch
3. Contact and task are created
4. Slack receives the standard message

### Test 4: Invalid application

Send:

```json
{
  "name": "No Email Applicant",
  "organization_type": "Corporate",
  "project_budget": 900,
  "message": "This should trigger validation handling"
}
```

Expected result:

1. Code node sets `validation_error`
2. IF node routes to `TRUE`
3. Slack receives the invalid payload warning
4. HubSpot nodes do not run

### Test 5: Error workflow

To test the error workflow safely:

1. Temporarily break a HubSpot credential or channel setting
2. Send a valid payload
3. Confirm the main workflow fails
4. Confirm the error workflow posts a Slack error message

---

## STEP 14: Deployment checklist

Before activating the workflows, confirm:

1. The HubSpot `priority` property exists
2. HubSpot credentials are connected in all HubSpot nodes
3. Slack credentials are connected in all Slack nodes
4. Slack channel IDs are correct
5. The webhook path is correct
6. The error workflow is linked in main workflow settings
7. All tests pass in editor mode

Then:

1. Activate the error workflow first
2. Activate the main workflow second
3. Copy the production webhook URL
4. Add that webhook URL to the grant application source

---

## STEP 15: Why this design works well

This workflow is stronger than a very small one-branch automation because it adds structure in the places that matter:

1. The webhook receives data quickly and hands off processing cleanly
2. The Code node creates a normalized internal format before branching
3. The IF node prevents bad data from contaminating CRM records
4. The Switch node keeps business routing obvious
5. Separate corporate and standard branches allow different urgency handling
6. The Wait nodes create a small sync buffer between contact updates and task creation
7. A dedicated error workflow keeps operational failures visible

This gives you a more realistic automation project than a simple “webhook to Slack” demo.

---

## STEP 16: Screenshot guide

If you want to document the build for a handoff or portfolio, capture these screenshots:

1. The full main workflow canvas
2. The webhook node configuration
3. The Code node with the normalization script
4. The IF node validation rule
5. The Set node field assignments
6. The Switch node routing cases
7. The corporate HubSpot contact mapping
8. The corporate task configuration
9. The standard task configuration
10. The Slack alert message setup
11. The separate error workflow canvas
12. A successful corporate execution
13. A successful standard execution
14. An invalid-payload execution
15. A Slack error alert from the error workflow

---

## Final result

When you finish this build, you will have a real intake-routing automation in n8n that:

- captures incoming grant applications
- structures the data cleanly
- applies routing rules based on applicant type
- creates or updates HubSpot records
- creates follow-up tasks
- alerts the team in Slack
- flags broken executions through a separate error workflow

If you want, the next step I can take is create a second companion file with an even more visual “click-by-click node setup checklist” for this same Green Canopy workflow.

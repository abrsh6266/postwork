# Inventory and Roast Batch Synchronization System

**Client:** The Copper Filter — Specialty Coffee Roastery & Espresso Bar  
**Workflow Name:** `Copper Filter - Inventory & Roast Batch Sync`  
**Error Workflow Name:** `Copper Filter - Error Logger`

This guide is written so you can build the workflow manually in n8n yourself, step by step, without depending on a JSON import.

It follows a build-it-yourself style. Every node is explained in full. No step says "configure later."

It assumes:

- You are using a recent n8n version with the current `Schedule Trigger`, `Manual Trigger`, `Merge`, `Google Sheets`, `Set`, `Code`, `Split In Batches`, `Airtable`, `IF`, `Aggregate`, `Slack`, `Wait`, `No Operation`, and `Error Trigger` nodes.
- You already have these credentials available inside n8n:
  - `Google Sheets OAuth2`
  - `Airtable Personal Access Token`
  - `Slack OAuth2` or a valid `Slack Incoming Webhook URL`

Use these exact workflow names:

- `Copper Filter - Inventory & Roast Batch Sync`
- `Copper Filter - Error Logger`

---

## What this workflow does

1. Fires every Monday at 7:00 AM on a schedule, and also accepts a manual trigger for testing.
2. Merges both trigger sources into a single stream so the rest of the workflow does not need to know which trigger fired.
3. Reads the weekly warehouse inventory audit from a Google Sheet.
4. Cleans and normalises all numeric fields so they are safe to use in math.
5. Splits rows into individual batches for reliable processing.
6. Calculates available roasted yield using a 15% weight-loss formula.
7. Calculates how many kilograms are needed to reach reorder par.
8. Searches Airtable for each Bean_ID to decide whether to update or create.
9. Routes each record through an upsert: update if found, create if missing.
10. Checks every record against its reorder threshold to detect low stock.
11. Sends a per-item Slack alert for every low-stock bean.
12. Re-joins both stock branches and aggregates a full run summary.
13. Posts a weekly summary report to the Slack operations channel.
14. Logs the successful completion of the run.
15. Uses a separate error workflow to catch and log any execution failure.
16. Retries Airtable writes automatically with a timed wait between attempts.

---

## Before you build

Create all external assets first so you do not have to stop while building nodes.

### 1. Google Sheet

Create one spreadsheet named:

```
The Copper Filter - Warehouse Inventory Audit
```

Create one tab named:

```
Inventory Audit
```

Add these exact column headers in row 1:

```
Bean_ID | Origin | Variety | Processing_Method | Current_Weight_KG | Reorder_Point_KG | Last_Audit_Date
```

Populate at least five rows of sample data. Use this starter data exactly:

```
CF-001 | Ethiopia     | Yirgacheffe  | Washed    | 5.0  | 15.0 | 2026-04-28
CF-002 | Colombia     | Huila        | Natural   | 42.0 | 20.0 | 2026-04-28
CF-003 | Guatemala    | Antigua      | Honey     | 8.5  | 10.0 | 2026-04-28
CF-004 | Kenya        | AA           | Washed    | 3.2  | 12.0 | 2026-04-28
CF-005 | Brazil       | Cerrado      | Natural   | 60.0 | 25.0 | 2026-04-28
CF-006 | Yemen        | Haraaz       | Natural   | 9.8  | 10.0 | 2026-04-28
CF-007 | Peru         | Cajamarca    | Washed    | 2.1  | 8.0  | 2026-04-28
CF-008 | Indonesia    | Sumatra      | Wet-Hulled| 22.5 | 15.0 | 2026-04-28
```

Bean IDs CF-001, CF-004, and CF-007 are intentionally low to trigger alerts.

Keep the spreadsheet Document ID (from its URL) ready. You will use it in every Google Sheets node.

### 2. Airtable base

Create a new Airtable base named:

```
Copper Filter Operations Dashboard
```

Create one table inside it named:

```
Green Bean Inventory
```

Create these exact fields with these types:

| Field Name                  | Airtable Field Type   |
|-----------------------------|-----------------------|
| Bean_ID                     | Single line text      |
| Origin                      | Single line text      |
| Variety                     | Single line text      |
| Processing_Method           | Single line text      |
| Current_Weight_KG           | Number (decimal)      |
| Available_Roasted_Yield_KG  | Number (decimal)      |
| Reorder_Point_KG            | Number (decimal)      |
| Stock_Status                | Single line text      |
| KG_Needed_To_Par            | Number (decimal)      |
| Last_Audit_Date             | Single line text      |
| Last_Synced_At              | Single line text      |

Keep your Airtable base ID and table ID ready. Both appear in the Airtable URL as:
`https://airtable.com/appXXXXXXXX/tblXXXXXXXX/...`

### 3. Slack channels

Create or choose two Slack channels:

- `#coffee-ops-alerts` — receives per-item low stock messages
- `#coffee-ops-weekly` — receives the weekly summary report

Keep your Slack credential and both channel IDs ready.

### 4. Starter CSV (optional)

If you prefer to import data rather than type it, create a file named:

```
copper_filter_inventory_audit_starter.csv
```

with this content:

```csv
Bean_ID,Origin,Variety,Processing_Method,Current_Weight_KG,Reorder_Point_KG,Last_Audit_Date
CF-001,Ethiopia,Yirgacheffe,Washed,5.0,15.0,2026-04-28
CF-002,Colombia,Huila,Natural,42.0,20.0,2026-04-28
CF-003,Guatemala,Antigua,Honey,8.5,10.0,2026-04-28
CF-004,Kenya,AA,Washed,3.2,12.0,2026-04-28
CF-005,Brazil,Cerrado,Natural,60.0,25.0,2026-04-28
CF-006,Yemen,Haraaz,Natural,9.8,10.0,2026-04-28
CF-007,Peru,Cajamarca,Washed,2.1,8.0,2026-04-28
CF-008,Indonesia,Sumatra,Wet-Hulled,22.5,15.0,2026-04-28
```

To use it: open Google Sheets → File → Import → Upload → select the CSV → choose `Replace current sheet`.

---

## STEP 0: Create the two workflows

### 0A. Create the main workflow

1. Open n8n.
2. Click `Create Workflow`.
3. Rename it to:

```
Copper Filter - Inventory & Roast Batch Sync
```

4. Save once before adding nodes.

### 0B. Create the error workflow

1. Create a second workflow.
2. Rename it to:

```
Copper Filter - Error Logger
```

3. Save once.

---

## STEP 1: Understand the full node order

You will build 24 nodes across the main workflow plus 3 nodes in the error workflow.

Build them in this order:

**Trigger section**
1. `Schedule Trigger - Every Monday 7AM`
2. `Manual Trigger - Test Run`
3. `Merge - Combine Triggers`

**Data ingestion**
4. `Google Sheets - Read Inventory Audit`
5. `Set - Clean & Cast Numeric Fields`
6. `Split In Batches - One Bean Per Cycle`

**Calculations**
7. `Code - Calculate Roasted Yield`
8. `Code - Calculate KG Needed To Par`

**Airtable upsert**
9. `Airtable - Search By Bean_ID`
10. `IF - Record Exists In Airtable?`
11. `Airtable - Update Existing Record`
12. `Airtable - Create New Record`

**Stock check and alerts**
13. `Merge - Rejoin After Upsert`
14. `IF - Is Stock Low?`
15. `Set - Build Low Stock Alert Message`
16. `Slack - Send Low Stock Alert`
17. `Set - Mark Healthy Stock`

**Summary and logging**
18. `Merge - Rejoin Stock Branches`
19. `Aggregate - Count Stock Summary`
20. `Set - Build Weekly Summary Message`
21. `Slack - Send Weekly Summary`
22. `No Operation - Run Complete`

**Error handling**
23. `Wait - Retry Delay`  *(used inline in upsert retry path)*
24. `Airtable - Retry Create Record`  *(retry branch)*

**Error workflow (separate)**
25. `Error Trigger - Global`
26. `Set - Prepare Error Payload`
27. `Slack - Admin Failure Alert`

---

## STEP 2: Schedule Trigger — Every Monday 7AM

### Add the node

1. Add a `Schedule Trigger` node.
2. Rename it to:

```
Schedule Trigger - Every Monday 7AM
```

### Configure it

Set these fields:

- `Trigger Interval`: `Weeks`
- `Weeks Between Triggers`: `1`
- `Trigger on Weekdays`: `Monday`
- `At Hour`: `7`
- `At Minute`: `0`

### Purpose

This is the production trigger. It fires automatically every Monday morning before the roastery's operations team arrives, so the Airtable dashboard is ready for the week.

### Example output

```json
{
  "timestamp": "2026-05-04T07:00:00.000Z"
}
```

---

## STEP 3: Manual Trigger — Test Run

### Add the node

1. Add a `Manual Trigger` node.
2. Rename it to:

```
Manual Trigger - Test Run
```

### Configure it

No configuration is needed. This node fires when you click `Test Workflow` in the n8n editor.

### Purpose

Lets you test the full workflow during development without waiting for Monday.

### Why this matters

Without a manual trigger, you would have to edit the schedule every time you want to test. Having both triggers keeps production and development cleanly separated.

---

## STEP 4: Merge — Combine Triggers

### Add the node

1. Add a `Merge` node.
2. Rename it to:

```
Merge - Combine Triggers
```

### Configure it

Set:

- `Mode`: `Combine`
- `Combination Mode`: `Multiplex`

Connect:

- `Schedule Trigger - Every Monday 7AM` → input 0 of this Merge node
- `Manual Trigger - Test Run` → input 1 of this Merge node

### Purpose

No matter which trigger fires, the rest of the workflow receives one unified data stream. This prevents you from having to duplicate all downstream nodes.

### Example output

```json
{
  "timestamp": "2026-05-04T07:00:00.000Z"
}
```

---

## STEP 5: Google Sheets — Read Inventory Audit

### Add the node

1. Add a `Google Sheets` node.
2. Rename it to:

```
Google Sheets - Read Inventory Audit
```

### Configure it

Set:

- `Credential`: your `Google Sheets OAuth2` credential
- `Operation`: `Get Many Rows`
- `Document ID`: your spreadsheet Document ID
- `Sheet Name`: `Inventory Audit`
- `Options > Return All`: enabled

### Purpose

Reads all rows from the Warehouse Inventory Audit sheet. Each row becomes one n8n item flowing to the next node.

### Example output per item

```json
{
  "Bean_ID": "CF-001",
  "Origin": "Ethiopia",
  "Variety": "Yirgacheffe",
  "Processing_Method": "Washed",
  "Current_Weight_KG": "5.0",
  "Reorder_Point_KG": "15.0",
  "Last_Audit_Date": "2026-04-28"
}
```

Note: Google Sheets returns all values as strings. The next node fixes that.

### Node settings

Turn on:

- `Retry On Fail`
- `Max Tries`: `3`
- `Wait Between Tries (ms)`: `3000`

This protects against transient OAuth token refresh failures.

---

## STEP 6: Set — Clean & Cast Numeric Fields

### Add the node

1. Add a `Set` node after `Google Sheets - Read Inventory Audit`.
2. Rename it to:

```
Set - Clean & Cast Numeric Fields
```

### Configure it

Set `Mode` to `Keep Only Set`.

Add these field assignments:

| Output Field Name     | Expression                                                               |
|-----------------------|--------------------------------------------------------------------------|
| Bean_ID               | `{{ $json["Bean_ID"] }}`                                                 |
| Origin                | `{{ $json["Origin"] }}`                                                  |
| Variety               | `{{ $json["Variety"] }}`                                                 |
| Processing_Method     | `{{ $json["Processing_Method"] }}`                                       |
| Current_Weight_KG     | `{{ parseFloat($json["Current_Weight_KG"]) }}`                          |
| Reorder_Point_KG      | `{{ parseFloat($json["Reorder_Point_KG"]) }}`                           |
| Last_Audit_Date       | `{{ $json["Last_Audit_Date"] }}`                                         |

### Purpose

Google Sheets returns every value as a string. If you try to multiply the string `"5.0"` by `0.85`, JavaScript returns `NaN`. This node converts the two weight fields to real JavaScript numbers before any math runs downstream.

### Example output per item

```json
{
  "Bean_ID": "CF-001",
  "Origin": "Ethiopia",
  "Variety": "Yirgacheffe",
  "Processing_Method": "Washed",
  "Current_Weight_KG": 5,
  "Reorder_Point_KG": 15,
  "Last_Audit_Date": "2026-04-28"
}
```

---

## STEP 7: Split In Batches — One Bean Per Cycle

### Add the node

1. Add a `Split In Batches` node after `Set - Clean & Cast Numeric Fields`.
2. Rename it to:

```
Split In Batches - One Bean Per Cycle
```

### Configure it

Set:

- `Batch Size`: `1`

### Purpose

Sends one bean record at a time through all subsequent nodes. This is essential for the Airtable search-then-upsert logic to work correctly. Without it, multiple records would be compared against a single Airtable search result and produce incorrect matches.

### Why batch size 1

Each bean has a unique Bean_ID. The Airtable search node returns the result for one Bean_ID at a time. Sending items one at a time keeps the search-result and the source record perfectly aligned.

---

## STEP 8: Code — Calculate Roasted Yield

### Add the node

1. Add a `Code` node after `Split In Batches - One Bean Per Cycle`.
2. Rename it to:

```
Code - Calculate Roasted Yield
```

3. Set:
   - `Mode`: `Run Once for Each Item`
   - `Language`: `JavaScript`

### Paste this exact code

```javascript
const item = $json;

const currentWeightKG = Number(item.Current_Weight_KG) || 0;

// Specialty coffee loses approximately 15% of its green weight during roasting
// due to moisture evaporation and chaff loss.
// Roasted yield = green weight × 0.85
const roastedYieldKG = parseFloat((currentWeightKG * 0.85).toFixed(2));

return [{
  json: {
    ...item,
    Available_Roasted_Yield_KG: roastedYieldKG,
  },
}];
```

### Yield calculation examples

| Green Weight (KG) | Formula         | Roasted Yield (KG) |
|-------------------|-----------------|--------------------|
| 5.0               | 5.0 × 0.85      | 4.25               |
| 8.5               | 8.5 × 0.85      | 7.23               |
| 20.0              | 20.0 × 0.85     | 17.00              |
| 42.0              | 42.0 × 0.85     | 35.70              |
| 60.0              | 60.0 × 0.85     | 51.00              |

### Example output

```json
{
  "Bean_ID": "CF-001",
  "Origin": "Ethiopia",
  "Variety": "Yirgacheffe",
  "Processing_Method": "Washed",
  "Current_Weight_KG": 5,
  "Reorder_Point_KG": 15,
  "Last_Audit_Date": "2026-04-28",
  "Available_Roasted_Yield_KG": 4.25
}
```

---

## STEP 9: Code — Calculate KG Needed To Par

### Add the node

1. Add a `Code` node after `Code - Calculate Roasted Yield`.
2. Rename it to:

```
Code - Calculate KG Needed To Par
```

3. Set:
   - `Mode`: `Run Once for Each Item`
   - `Language`: `JavaScript`

### Paste this exact code

```javascript
const item = $json;

const currentWeight = Number(item.Current_Weight_KG) || 0;
const reorderPoint = Number(item.Reorder_Point_KG) || 0;

// KG needed to reach reorder par level.
// If current stock is already at or above the reorder point, result is 0.
const kgNeededToPar = currentWeight < reorderPoint
  ? parseFloat((reorderPoint - currentWeight).toFixed(2))
  : 0;

// Determine stock status label.
const stockStatus = currentWeight < reorderPoint ? 'Low Stock' : 'Healthy';

return [{
  json: {
    ...item,
    Stock_Status: stockStatus,
    KG_Needed_To_Par: kgNeededToPar,
    Last_Synced_At: new Date().toISOString(),
  },
}];
```

### Reorder calculation examples

| Current KG | Reorder Point KG | Status     | KG Needed To Par |
|------------|------------------|------------|------------------|
| 5.0        | 15.0             | Low Stock  | 10.0             |
| 3.2        | 12.0             | Low Stock  | 8.8              |
| 2.1        | 8.0              | Low Stock  | 5.9              |
| 42.0       | 20.0             | Healthy    | 0                |
| 60.0       | 25.0             | Healthy    | 0                |

### Example output

```json
{
  "Bean_ID": "CF-001",
  "Origin": "Ethiopia",
  "Variety": "Yirgacheffe",
  "Processing_Method": "Washed",
  "Current_Weight_KG": 5,
  "Reorder_Point_KG": 15,
  "Last_Audit_Date": "2026-04-28",
  "Available_Roasted_Yield_KG": 4.25,
  "Stock_Status": "Low Stock",
  "KG_Needed_To_Par": 10,
  "Last_Synced_At": "2026-05-04T07:01:23.000Z"
}
```

---

## STEP 10: Airtable — Search By Bean_ID

### Add the node

1. Add an `Airtable` node after `Code - Calculate KG Needed To Par`.
2. Rename it to:

```
Airtable - Search By Bean_ID
```

### Configure it

Set:

- `Credential`: your `Airtable Personal Access Token`
- `Operation`: `Search`
- `Base ID`: your Airtable base ID
- `Table ID`: your `Green Bean Inventory` table ID
- `Filter By Formula`:

```
{Bean_ID} = '{{ $json["Bean_ID"] }}'
```

- `Return All`: enabled

### Purpose

For each bean processed, this searches Airtable by the unique Bean_ID field. The result tells the next IF node whether a record already exists (update path) or is missing (create path).

### Node settings

Turn on:

- `Retry On Fail`
- `Max Tries`: `3`
- `Wait Between Tries (ms)`: `4000`

### Example output if record found

```json
{
  "id": "recABCDEFGH",
  "fields": {
    "Bean_ID": "CF-001",
    "Origin": "Ethiopia"
  }
}
```

### Example output if record not found

The node returns zero items. The IF node interprets this as missing.

---

## STEP 11: IF — Record Exists In Airtable?

### Add the node

1. Add an `IF` node after `Airtable - Search By Bean_ID`.
2. Rename it to:

```
IF - Record Exists In Airtable?
```

### Configure it

Add one condition:

- `Value 1`: `{{ $json["id"] }}`
- `Operation`: `Is Not Empty`

### How this works

- If Airtable returned a record, `$json["id"]` contains the Airtable record ID. The condition is true → TRUE branch runs → Update.
- If Airtable returned nothing, `$json["id"]` is undefined/empty. The condition is false → FALSE branch runs → Create.

### Outputs

- `TRUE output` → connect to `Airtable - Update Existing Record`
- `FALSE output` → connect to `Airtable - Create New Record`

---

## STEP 12: Airtable — Update Existing Record

### Add the node

1. Add an `Airtable` node on the TRUE output of `IF - Record Exists In Airtable?`.
2. Rename it to:

```
Airtable - Update Existing Record
```

### Configure it

Set:

- `Credential`: your `Airtable Personal Access Token`
- `Operation`: `Update`
- `Base ID`: your Airtable base ID
- `Table ID`: your `Green Bean Inventory` table ID
- `Record ID`: `{{ $json["id"] }}`

Map these fields:

| Airtable Field              | n8n Expression                                      |
|-----------------------------|-----------------------------------------------------|
| Origin                      | `{{ $('Code - Calculate KG Needed To Par').item.json["Origin"] }}` |
| Variety                     | `{{ $('Code - Calculate KG Needed To Par').item.json["Variety"] }}` |
| Processing_Method           | `{{ $('Code - Calculate KG Needed To Par').item.json["Processing_Method"] }}` |
| Current_Weight_KG           | `{{ $('Code - Calculate KG Needed To Par').item.json["Current_Weight_KG"] }}` |
| Available_Roasted_Yield_KG  | `{{ $('Code - Calculate KG Needed To Par').item.json["Available_Roasted_Yield_KG"] }}` |
| Reorder_Point_KG            | `{{ $('Code - Calculate KG Needed To Par').item.json["Reorder_Point_KG"] }}` |
| Stock_Status                | `{{ $('Code - Calculate KG Needed To Par').item.json["Stock_Status"] }}` |
| KG_Needed_To_Par            | `{{ $('Code - Calculate KG Needed To Par').item.json["KG_Needed_To_Par"] }}` |
| Last_Audit_Date             | `{{ $('Code - Calculate KG Needed To Par').item.json["Last_Audit_Date"] }}` |
| Last_Synced_At              | `{{ $('Code - Calculate KG Needed To Par').item.json["Last_Synced_At"] }}` |

### Why reference the Code node

The Airtable search result only contains the Airtable record ID. The actual inventory data lives upstream in the Code node output. You must reference it by node name using the `$('node name').item.json` syntax.

### Node settings

Turn on:

- `Retry On Fail`
- `Max Tries`: `3`
- `Wait Between Tries (ms)`: `4000`

---

## STEP 13: Airtable — Create New Record

### Add the node

1. Add an `Airtable` node on the FALSE output of `IF - Record Exists In Airtable?`.
2. Rename it to:

```
Airtable - Create New Record
```

### Configure it

Set:

- `Credential`: your `Airtable Personal Access Token`
- `Operation`: `Create`
- `Base ID`: your Airtable base ID
- `Table ID`: your `Green Bean Inventory` table ID

Map these fields:

| Airtable Field              | n8n Expression                                      |
|-----------------------------|-----------------------------------------------------|
| Bean_ID                     | `{{ $json["Bean_ID"] }}`                            |
| Origin                      | `{{ $json["Origin"] }}`                             |
| Variety                     | `{{ $json["Variety"] }}`                            |
| Processing_Method           | `{{ $json["Processing_Method"] }}`                  |
| Current_Weight_KG           | `{{ $json["Current_Weight_KG"] }}`                  |
| Available_Roasted_Yield_KG  | `{{ $json["Available_Roasted_Yield_KG"] }}`         |
| Reorder_Point_KG            | `{{ $json["Reorder_Point_KG"] }}`                   |
| Stock_Status                | `{{ $json["Stock_Status"] }}`                       |
| KG_Needed_To_Par            | `{{ $json["KG_Needed_To_Par"] }}`                   |
| Last_Audit_Date             | `{{ $json["Last_Audit_Date"] }}`                    |
| Last_Synced_At              | `{{ $json["Last_Synced_At"] }}`                     |

### Why this uses $json directly

On the FALSE branch, Airtable returned no records. n8n carries the data from the last node that produced output — which is the Code node. So `$json` here already contains the full calculated item.

### Node settings

Turn on:

- `Retry On Fail`
- `Max Tries`: `2`
- `Wait Between Tries (ms)`: `3000`

---

## STEP 14: Wait — Retry Delay (Airtable Failure Path)

This node is only connected if you want to build a manual retry branch for hard Airtable failures. Connect it downstream of any Airtable create/update error output if your n8n version surfaces error outputs as a separate branch.

### Add the node

1. Add a `Wait` node.
2. Rename it to:

```
Wait - Airtable Retry Delay
```

### Configure it

Set:

- `Resume`: `After Time Interval`
- `Amount`: `10`
- `Unit`: `Seconds`

### Purpose

If the Airtable create fails after its built-in retries, this wait gives the Airtable API time to recover before a final re-attempt is made.

---

## STEP 15: Airtable — Retry Create Record

### Add the node

1. Add an `Airtable` node after `Wait - Airtable Retry Delay`.
2. Rename it to:

```
Airtable - Retry Create Record
```

### Configure it

Use the exact same configuration as `Airtable - Create New Record`.

This is the last-chance write attempt. If this also fails, the error workflow catches it and logs the failure.

---

## STEP 16: Merge — Rejoin After Upsert

### Add the node

1. Add a `Merge` node.
2. Rename it to:

```
Merge - Rejoin After Upsert
```

### Configure it

Set:

- `Mode`: `Append`

Connect:

- Output of `Airtable - Update Existing Record` → input 0
- Output of `Airtable - Create New Record` → input 1

### Purpose

After the upsert decision, both branches re-join into a single stream. Everything that follows — the low-stock check, alerts, and summary — does not need to know whether a record was updated or created.

---

## STEP 17: IF — Is Stock Low?

### Add the node

1. Add an `IF` node after `Merge - Rejoin After Upsert`.
2. Rename it to:

```
IF - Is Stock Low?
```

### Configure it

Add one condition:

- `Value 1`: `{{ $json["Stock_Status"] }}`
- `Operation`: `Equals`
- `Value 2`: `Low Stock`

### Outputs

- `TRUE output` → connect to `Set - Build Low Stock Alert Message`
- `FALSE output` → connect to `Set - Mark Healthy Stock`

---

## STEP 18: Set — Build Low Stock Alert Message

### Add the node

1. Add a `Set` node on the TRUE output of `IF - Is Stock Low?`.
2. Rename it to:

```
Set - Build Low Stock Alert Message
```

### Configure it

Set `Mode` to `Keep All Values`.

Add one field:

- Field Name: `slack_alert_message`
- Expression:

```
☕ *Low Stock Alert – Action Required*

*Bean:* {{ $json["Variety"] }} from {{ $json["Origin"] }}
*Bean ID:* {{ $json["Bean_ID"] }}
*Processing:* {{ $json["Processing_Method"] }}
*Current Stock:* {{ $json["Current_Weight_KG"] }} kg
*Reorder Point:* {{ $json["Reorder_Point_KG"] }} kg
*KG Needed To Par:* {{ $json["KG_Needed_To_Par"] }} kg
*Roastable Yield Remaining:* {{ $json["Available_Roasted_Yield_KG"] }} kg
*Last Audit:* {{ $json["Last_Audit_Date"] }}

⚠️ *Action:* Place a purchase order immediately.
```

### Purpose

Builds a fully formatted Slack message that is unique to each low-stock bean. The next node sends this message to the alert channel.

---

## STEP 19: Slack — Send Low Stock Alert

### Add the node

1. Add a `Slack` node after `Set - Build Low Stock Alert Message`.
2. Rename it to:

```
Slack - Send Low Stock Alert
```

### Configure it

Set:

- `Credential`: your Slack OAuth2 credential
- `Resource`: `Message`
- `Operation`: `Post`
- `Channel`: your `#coffee-ops-alerts` channel ID
- `Text`: `{{ $json["slack_alert_message"] }}`

### Example Slack message rendered

```
☕ Low Stock Alert – Action Required

Bean: Yirgacheffe from Ethiopia
Bean ID: CF-001
Processing: Washed
Current Stock: 5 kg
Reorder Point: 15 kg
KG Needed To Par: 10 kg
Roastable Yield Remaining: 4.25 kg
Last Audit: 2026-04-28

⚠️ Action: Place a purchase order immediately.
```

### Node settings

Turn on:

- `Retry On Fail`
- `Max Tries`: `3`
- `Wait Between Tries (ms)`: `2000`

---

## STEP 20: Set — Mark Healthy Stock

### Add the node

1. Add a `Set` node on the FALSE output of `IF - Is Stock Low?`.
2. Rename it to:

```
Set - Mark Healthy Stock
```

### Configure it

Set `Mode` to `Keep All Values`.

Add one field:

- Field Name: `alert_sent`
- Value: `false`

### Purpose

Healthy-stock records do not need an alert. This Set node passes them through with a flag indicating no alert was sent, so the downstream Aggregate node can count them correctly.

---

## STEP 21: Merge — Rejoin Stock Branches

### Add the node

1. Add a `Merge` node.
2. Rename it to:

```
Merge - Rejoin Stock Branches
```

### Configure it

Set:

- `Mode`: `Append`

Connect:

- Output of `Slack - Send Low Stock Alert` → input 0
- Output of `Set - Mark Healthy Stock` → input 1

### Purpose

Brings low-stock and healthy-stock records back into one stream so the summary aggregation can count all of them.

---

## STEP 22: Aggregate — Count Stock Summary

### Add the node

1. Add an `Aggregate` node after `Merge - Rejoin Stock Branches`.
2. Rename it to:

```
Aggregate - Count Stock Summary
```

### Configure it

Set:

- `Aggregate`: `All Item Data (Into a Single List)`

### Purpose

Collapses all individual bean records into a single item containing an array. The next Code node then counts totals from that array to build the summary report.

After this node, add a `Code` node to compute the summary:

---

## STEP 23: Code — Compute Summary Metrics

### Add the node

1. Add a `Code` node after `Aggregate - Count Stock Summary`.
2. Rename it to:

```
Code - Compute Summary Metrics
```

3. Set:
   - `Mode`: `Run Once for All Items`
   - `Language`: `JavaScript`

### Paste this exact code

```javascript
const allItems = items.map(i => i.json);

const totalSKUs = allItems.length;

const lowStockItems = allItems.filter(i => i.Stock_Status === 'Low Stock');
const healthyItems = allItems.filter(i => i.Stock_Status === 'Healthy');

const totalRoastableKG = allItems.reduce((sum, i) => {
  return sum + (Number(i.Available_Roasted_Yield_KG) || 0);
}, 0);

const lowStockList = lowStockItems.map(i =>
  `• ${i.Variety} (${i.Origin}) — ${i.Current_Weight_KG} kg / needs ${i.KG_Needed_To_Par} kg`
).join('\n');

const summaryMessage = `📦 *Weekly Inventory Sync Report — The Copper Filter*

*Run Date:* ${new Date().toUTCString()}

*Total SKUs Synced:* ${totalSKUs}
*Healthy Stock:* ${healthyItems.length}
*Low Stock:* ${lowStockItems.length}
*Total Roastable KG:* ${totalRoastableKG.toFixed(2)} kg

${lowStockItems.length > 0 ? `⚠️ *Beans Requiring Reorder:*\n${lowStockList}` : '✅ All beans are above reorder threshold.'}

_Sync powered by n8n · Copper Filter Ops_`;

return [{
  json: {
    totalSKUs,
    lowStockCount: lowStockItems.length,
    healthyCount: healthyItems.length,
    totalRoastableKG: parseFloat(totalRoastableKG.toFixed(2)),
    summaryMessage,
  },
}];
```

### Example output

```json
{
  "totalSKUs": 8,
  "lowStockCount": 3,
  "healthyCount": 5,
  "totalRoastableKG": 139.72,
  "summaryMessage": "📦 Weekly Inventory Sync Report..."
}
```

---

## STEP 24: Slack — Send Weekly Summary

### Add the node

1. Add a `Slack` node after `Code - Compute Summary Metrics`.
2. Rename it to:

```
Slack - Send Weekly Summary
```

### Configure it

Set:

- `Credential`: your Slack OAuth2 credential
- `Resource`: `Message`
- `Operation`: `Post`
- `Channel`: your `#coffee-ops-weekly` channel ID
- `Text`: `{{ $json["summaryMessage"] }}`

### Example Slack message rendered

```
📦 Weekly Inventory Sync Report — The Copper Filter

Run Date: Mon, 04 May 2026 07:03:45 GMT

Total SKUs Synced: 8
Healthy Stock: 5
Low Stock: 3
Total Roastable KG: 139.72 kg

⚠️ Beans Requiring Reorder:
• Yirgacheffe (Ethiopia) — 5 kg / needs 10 kg
• AA (Kenya) — 3.2 kg / needs 8.8 kg
• Cajamarca (Peru) — 2.1 kg / needs 5.9 kg

Sync powered by n8n · Copper Filter Ops
```

### Node settings

Turn on:

- `Retry On Fail`
- `Max Tries`: `3`
- `Wait Between Tries (ms)`: `2000`

---

## STEP 25: No Operation — Run Complete

### Add the node

1. Add a `No Operation, do nothing` node after `Slack - Send Weekly Summary`.
2. Rename it to:

```
No Operation - Run Complete
```

### Purpose

This marks the clean end of the workflow. It has no configuration. Its presence makes the execution log easy to read — if this node appears green in the execution history, the full run succeeded.

It also gives you a future-proof attachment point. If you later need to add a final action (such as writing a completion log to Google Sheets or triggering a downstream workflow), you connect it here without restructuring the summary path.

---

## STEP 26: Build the separate error workflow

Open the second workflow you created in Step 0B.

### 26A. Error Trigger — Global

1. Inside `Copper Filter - Error Logger`, add an `Error Trigger` node.
2. Rename it to:

```
Error Trigger - Global
```

This node fires automatically whenever the main workflow crashes at any node. No configuration is needed other than saving it in the correct workflow.

### 26B. Set — Prepare Error Payload

1. Add a `Set` node after `Error Trigger - Global`.
2. Rename it to:

```
Set - Prepare Error Payload
```

Add these assignments:

| Output Field     | Expression                                                                                              |
|------------------|---------------------------------------------------------------------------------------------------------|
| error_timestamp  | `{{ new Date().toISOString() }}`                                                                        |
| workflow_name    | `{{ $json.workflow?.name || 'Copper Filter - Inventory & Roast Batch Sync' }}`                         |
| failed_node      | `{{ $json.execution?.lastNodeExecuted || 'Unknown node' }}`                                             |
| error_message    | `{{ $json.execution?.error?.message || $json.error?.message || 'Unknown error' }}`                      |
| execution_id     | `{{ $json.execution?.id || '' }}`                                                                       |
| admin_alert      | `🚨 *Copper Filter Sync FAILED*\n\n*Workflow:* {{ $json.workflow?.name || 'Copper Filter Sync' }}\n*Failed Node:* {{ $json.execution?.lastNodeExecuted || 'Unknown' }}\n*Error:* {{ $json.execution?.error?.message || 'Unknown error' }}\n*Time:* {{ new Date().toISOString() }}\n\nCheck the n8n execution log immediately.` |

### 26C. Slack — Admin Failure Alert

1. Add a `Slack` node after `Set - Prepare Error Payload`.
2. Rename it to:

```
Slack - Admin Failure Alert
```

Configure it:

- `Credential`: your Slack OAuth2 credential
- `Resource`: `Message`
- `Operation`: `Post`
- `Channel`: your `#coffee-ops-weekly` channel ID (or a private admin channel)
- `Text`: `{{ $json["admin_alert"] }}`

### 26D. Link the error workflow to the main workflow

1. Open `Copper Filter - Inventory & Roast Batch Sync`.
2. Click the three-dot menu or Workflow Settings.
3. Find `Error Workflow`.
4. Select `Copper Filter - Error Logger`.
5. Save the main workflow.

This is what activates the error workflow. Without this step, the error logger never runs.

---

## STEP 27: Full connection map

Use this to verify your canvas before testing.

```
Schedule Trigger - Every Monday 7AM
  → Merge - Combine Triggers

Manual Trigger - Test Run
  → Merge - Combine Triggers

Merge - Combine Triggers
  → Google Sheets - Read Inventory Audit
  → Set - Clean & Cast Numeric Fields
  → Split In Batches - One Bean Per Cycle
  → Code - Calculate Roasted Yield
  → Code - Calculate KG Needed To Par
  → Airtable - Search By Bean_ID
  → IF - Record Exists In Airtable?

IF - Record Exists In Airtable? TRUE
  → Airtable - Update Existing Record
  → Merge - Rejoin After Upsert

IF - Record Exists In Airtable? FALSE
  → Airtable - Create New Record
  → Merge - Rejoin After Upsert

[Optional retry path]
Airtable - Create New Record [on error]
  → Wait - Airtable Retry Delay
  → Airtable - Retry Create Record
  → Merge - Rejoin After Upsert

Merge - Rejoin After Upsert
  → IF - Is Stock Low?

IF - Is Stock Low? TRUE
  → Set - Build Low Stock Alert Message
  → Slack - Send Low Stock Alert
  → Merge - Rejoin Stock Branches

IF - Is Stock Low? FALSE
  → Set - Mark Healthy Stock
  → Merge - Rejoin Stock Branches

Merge - Rejoin Stock Branches
  → Aggregate - Count Stock Summary
  → Code - Compute Summary Metrics
  → Slack - Send Weekly Summary
  → No Operation - Run Complete
```

Error workflow:

```
Error Trigger - Global
  → Set - Prepare Error Payload
  → Slack - Admin Failure Alert
```

---

## STEP 28: Recommended canvas layout

Arrange the canvas in clear horizontal lanes.

```
[FAR LEFT]          [CENTER-LEFT]           [CENTER]              [CENTER-RIGHT]     [RIGHT]

Schedule Trigger ──┐
                   ├── Merge Triggers ── Sheets ── Set Clean ── Split Batches ──┐
Manual Trigger ────┘                                                              │
                                                                                  ↓
                                                               Code: Roasted Yield
                                                                        ↓
                                                               Code: KG Needed To Par
                                                                        ↓
                                                               Airtable: Search
                                                                        ↓
                                                               IF: Exists?
                                                          TRUE ↙         ↘ FALSE
                                                    Airtable:           Airtable:
                                                    Update              Create
                                                          ↘         ↙
                                                       Merge: Rejoin Upsert
                                                                ↓
                                                         IF: Is Stock Low?
                                                    TRUE ↙         ↘ FALSE
                                              Set: Alert         Set: Healthy
                                              Slack: Alert
                                                    ↘               ↙
                                                Merge: Rejoin Stock
                                                        ↓
                                                   Aggregate
                                                        ↓
                                                Code: Summary
                                                        ↓
                                               Slack: Weekly Report
                                                        ↓
                                               No Operation: Done
```

Use sticky notes to label:

- Trigger Zone (yellow)
- Data Ingestion (blue)
- Calculation Engine (purple)
- Airtable Upsert (green)
- Low Stock Routing (orange)
- Summary & Reporting (teal)
- Error Handling (red)

---

## STEP 29: Testing procedure

Test one section at a time.

### Test 1: Trigger merge

1. Click `Test Workflow`.
2. Confirm `Manual Trigger - Test Run` fires.
3. Confirm `Merge - Combine Triggers` outputs one item.

### Test 2: Sheet read and clean

1. Run through Step 5.
2. Inspect `Set - Clean & Cast Numeric Fields` output.
3. Confirm `Current_Weight_KG` is a number, not a string.
4. Confirm `parseFloat("5.0")` returns `5`, not `"5.0"`.

### Test 3: Yield and par calculation

1. Run through Step 8 and Step 9.
2. For CF-001 (5.0 KG at 15.0 reorder):
   - Expected `Available_Roasted_Yield_KG`: `4.25`
   - Expected `Stock_Status`: `Low Stock`
   - Expected `KG_Needed_To_Par`: `10`
3. For CF-002 (42.0 KG at 20.0 reorder):
   - Expected `Available_Roasted_Yield_KG`: `35.7`
   - Expected `Stock_Status`: `Healthy`
   - Expected `KG_Needed_To_Par`: `0`

### Test 4: Airtable create (first run)

1. Clear all records in your Airtable `Green Bean Inventory` table.
2. Run the full workflow.
3. Confirm 8 new records are created.
4. Inspect one record — confirm all 11 fields are populated.
5. Confirm `Last_Synced_At` is a valid ISO timestamp.

### Test 5: Airtable upsert (second run, no duplicates)

1. Without changing the Google Sheet, run the workflow again.
2. Confirm the Airtable table still has exactly 8 records (no duplicates).
3. Confirm `Last_Synced_At` has been updated to the new run time.
4. This confirms the upsert logic is working correctly.

### Test 6: Low stock Slack alerts

1. Check your `#coffee-ops-alerts` Slack channel.
2. Confirm you received three alerts for CF-001, CF-004, and CF-007.
3. Confirm each alert contains the correct KG values and bean names.

### Test 7: Weekly summary

1. Check your `#coffee-ops-weekly` Slack channel.
2. Confirm the summary shows `Total SKUs: 8`, `Low Stock: 3`, `Healthy: 5`.
3. Confirm `Total Roastable KG` is correct.

### Test 8: Error workflow

1. Temporarily set an invalid Airtable Base ID in `Airtable - Search By Bean_ID`.
2. Run the workflow.
3. Confirm the main workflow fails.
4. Confirm the error workflow fires.
5. Confirm a Slack admin alert is posted to your operations channel.
6. Restore the correct Base ID.

---

## STEP 30: Field mapping reference

Use these exact expressions anywhere you need to reference inventory fields:

| Field                       | Expression                                                              |
|-----------------------------|-------------------------------------------------------------------------|
| Bean ID                     | `{{ $json["Bean_ID"] }}`                                                |
| Origin                      | `{{ $json["Origin"] }}`                                                 |
| Variety                     | `{{ $json["Variety"] }}`                                                |
| Processing Method           | `{{ $json["Processing_Method"] }}`                                      |
| Current Weight KG           | `{{ $json["Current_Weight_KG"] }}`                                      |
| Available Roasted Yield KG  | `{{ $json["Available_Roasted_Yield_KG"] }}`                             |
| Reorder Point KG            | `{{ $json["Reorder_Point_KG"] }}`                                       |
| Stock Status                | `{{ $json["Stock_Status"] }}`                                           |
| KG Needed To Par            | `{{ $json["KG_Needed_To_Par"] }}`                                       |
| Last Audit Date             | `{{ $json["Last_Audit_Date"] }}`                                        |
| Last Synced At              | `{{ $json["Last_Synced_At"] }}`                                         |
| Current timestamp           | `{{ $now }}`                                                            |
| Current ISO timestamp       | `{{ new Date().toISOString() }}`                                        |
| Upstream Code node data     | `{{ $('Code - Calculate KG Needed To Par').item.json["Bean_ID"] }}`     |

---

## STEP 31: Troubleshooting

### Google Sheets auth failed

Symptom: `Google Sheets - Read Inventory Audit` shows a 401 error.

Fix:
1. Open Credentials in n8n.
2. Find your Google Sheets OAuth2 credential.
3. Click `Reconnect`.
4. Re-authorize in the Google popup.
5. Re-run the workflow.

### Airtable permission denied

Symptom: Airtable nodes return 403.

Fix:
1. Open your Airtable Personal Access Token.
2. Confirm it has `data.records:read` and `data.records:write` scopes.
3. Confirm the token has access to the correct base.
4. Copy the token again and paste it into your n8n credential.

### Numeric values treated as text

Symptom: `Available_Roasted_Yield_KG` is `NaN` or the yield is wrong.

Fix:
1. Open `Set - Clean & Cast Numeric Fields`.
2. Confirm `Current_Weight_KG` uses `{{ parseFloat($json["Current_Weight_KG"]) }}` not `{{ $json["Current_Weight_KG"] }}`.
3. Run just that node and check the output type.

### Duplicate Bean_ID rows in Airtable

Symptom: Second run creates new records instead of updating.

Fix:
1. Open `Airtable - Search By Bean_ID`.
2. Check the filter formula: `{Bean_ID} = '{{ $json["Bean_ID"] }}'`
3. Make sure the single quotes inside the formula are straight quotes, not curly/smart quotes.
4. Run just the search node and verify it returns the existing record.

### Slack webhook invalid

Symptom: Slack nodes return 400 or `invalid_auth`.

Fix:
1. Go to your Slack OAuth2 credential in n8n.
2. Click `Reconnect` and re-authorize.
3. Confirm the channel ID starts with `C` (not the channel name with `#`).
4. Test the Slack node in isolation by clicking `Test Step` on just that node.

### Empty sheet rows being processed

Symptom: Workflow processes blank rows and creates empty Airtable records.

Fix:
1. In `Google Sheets - Read Inventory Audit`, expand `Options`.
2. Enable `Skip Rows With Empty Values`.
3. Alternatively, add a `Filter` node after the Set node:
   - Filter condition: `{{ $json["Bean_ID"] }}` is not empty.

### Wrong yield math

Symptom: Roasted yield is 85 instead of 4.25 for a 5 KG bean.

Fix:
1. Open `Code - Calculate Roasted Yield`.
2. Confirm the formula is `currentWeightKG * 0.85`, not `currentWeightKG * 85`.
3. Confirm `Number(item.Current_Weight_KG)` returns a number, not `NaN`.
4. Print `console.log(currentWeightKG)` inside the code node and check the execution log.

---

## STEP 32: Deliverables checklist

Use this list when packaging the project for a client or portfolio.

- [ ] Exported workflow JSON file (`Copper Filter - Inventory & Roast Batch Sync.json`)
- [ ] Exported error workflow JSON file (`Copper Filter - Error Logger.json`)
- [ ] Screenshot of full canvas with sticky note labels visible
- [ ] Screenshot of `Code - Calculate Roasted Yield` with correct output data
- [ ] Screenshot of `Code - Calculate KG Needed To Par` with Low Stock / Healthy labels
- [ ] Screenshot of Airtable `Green Bean Inventory` table with all 8 records populated
- [ ] Screenshot of Slack `#coffee-ops-alerts` with at least one low-stock alert
- [ ] Screenshot of Slack `#coffee-ops-weekly` with weekly summary message
- [ ] Screenshot of successful execution with all nodes green
- [ ] Screenshot of second execution confirming zero new Airtable records (upsert confirmed)
- [ ] Airtable base shared as read-only link for client review
- [ ] 1–2 minute Loom walkthrough showing: trigger → sheet read → calculation → upsert → alert → summary

---

## STEP 33: Pro tips for a premium delivery

**Use sticky notes on the canvas**

Group your nodes into sections and add a sticky note above each group. Suggested labels:

- `🟡 TRIGGER ZONE — Fires every Monday 7AM or on demand`
- `🔵 DATA INGESTION — Reads & cleans Google Sheet`
- `🟣 CALCULATION ENGINE — Yield & reorder logic`
- `🟢 AIRTABLE UPSERT — No duplicates guaranteed`
- `🟠 LOW STOCK ROUTING — Alerts only when needed`
- `🔵 WEEKLY SUMMARY — Full ops report to Slack`
- `🔴 ERROR HANDLER — Separate workflow catches all failures`

**Color your node backgrounds**

n8n allows custom node colors. Color-coding by section makes the canvas scannable at a glance for a client demo.

**Use `toFixed(2)` everywhere**

Always round your output numbers to two decimal places. `42.500000000001` in an Airtable cell looks unprofessional. `42.50` looks clean.

**Add before/after screenshots**

Show the Airtable table before the first run (empty) and after (8 records). This makes the automation value concrete.

**Document the starter CSV clearly**

Include the CSV in your delivery folder with a one-paragraph README explaining how to import it. Clients often lose this step.

**Pin the execution log**

After a successful run, pin the execution record in n8n. This lets you show the client a timestamped proof of the run history without them needing to run it themselves.

---

## Final result

Once built, this system gives The Copper Filter a production-grade weekly automation that:

- never creates duplicate records across weekly audit cycles
- automatically surfaces low-stock beans before the roast team discovers them manually
- delivers a clean dashboard-ready Airtable base the operations manager can open on any Monday morning
- sends immediate Slack alerts for every bean that needs purchasing attention
- posts a summary report covering the full inventory health of the roastery
- logs every workflow failure to an admin channel so nothing fails silently
- can be tested on demand during any development or review session using the manual trigger

The workflow is designed to be extended. Future additions could include a Google Sheets summary tab, a Discord mirror of the alerts, automated purchase order drafts via Gmail, or a secondary workflow that generates roast schedule recommendations based on the available yield data.

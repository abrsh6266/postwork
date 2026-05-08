# 🖥️ Nexus Blue — IT Hardware Refresh Tracker
### Complete Airtable Implementation Guide
### *Enterprise-Grade Step-by-Step Documentation for Project Managers & Technicians*

---

> **Document Version:** 1.0  
> **Project:** 50-Unit Workstation Deployment — Legal Firm Client  
> **Platform:** Airtable (Free or Pro Plan)  
> **Skill Level Required:** Beginner (No prior Airtable knowledge needed)  
> **Estimated Build Time:** 2–3 hours  

---

## 📋 Table of Contents

1. [Project Overview](#1-project-overview)
2. [Business Goal](#2-business-goal)
3. [Architecture Overview](#3-architecture-overview)
4. [Prerequisites & Account Setup](#4-prerequisites--account-setup)
5. [Creating the Airtable Base](#5-creating-the-airtable-base)
6. [Creating the Deployment Inventory Table](#6-creating-the-deployment-inventory-table)
7. [Field-by-Field Configuration](#7-field-by-field-configuration)
8. [View Creation](#8-view-creation)
9. [Automation Setup](#9-automation-setup)
10. [Data Import Instructions](#10-data-import-instructions)
11. [Testing Procedures](#11-testing-procedures)
12. [Screenshot Instructions](#12-screenshot-instructions)
13. [Video Walkthrough Guide](#13-video-walkthrough-guide)
14. [Share Link & Submission](#14-share-link--submission)
15. [Acceptance Criteria Checklist](#15-acceptance-criteria-checklist)
16. [QA Checklist](#16-qa-checklist)
17. [Troubleshooting](#17-troubleshooting)
18. [Best Practices](#18-best-practices)
19. [Scalability Recommendations](#19-scalability-recommendations)
20. [Future Improvements](#20-future-improvements)

---

## 1. Project Overview

**Nexus Blue IT** is a managed services provider that specializes in hardware procurement, configuration, and deployment for small-to-medium businesses. This project involves a **50-unit workstation refresh** for a legal firm client. The firm is replacing aging laptops across its entire workforce with new, standardized devices.

The Project Manager requires a centralized, real-time tracking system to:

- Know exactly which device is assigned to which employee at any moment
- Track the configuration status of every device (from unboxing to deployment)
- Monitor which technician is responsible for each unit
- Automatically capture when a device is officially deployed
- Produce clean reports and share progress visibility with stakeholders

This Airtable base is the **single source of truth** for the entire deployment operation.

---

## 2. Business Goal

| Goal | Description |
|---|---|
| **Asset Accountability** | Every device is tagged and linked to a specific employee |
| **Pipeline Visibility** | All stakeholders can see real-time deployment progress |
| **Technician Coordination** | Workload is visible per technician to avoid bottlenecks |
| **Audit Trail** | Every status change is automatically timestamped |
| **Scalability** | The system can scale from 50 to 500+ devices with minimal changes |
| **Client Reporting** | A shareable read-only view gives the legal firm live progress updates |

---

## 3. Architecture Overview

This implementation uses a **single Airtable base** with one primary table and three views. An automation handles timestamp capture on deployment. The structure is intentionally simple so a non-technical project manager can operate it daily without training.

```
Airtable Base: Nexus Blue — Hardware Refresh Tracker
│
└── Table: Deployment Inventory
    │
    ├── Fields:
    │   ├── Asset Number       (Autonumber)
    │   ├── Asset ID           (Formula: "NB-" prefix)
    │   ├── Device Model       (Single Select)
    │   ├── Serial Number      (Single Line Text)
    │   ├── Assigned User      (Single Line Text)
    │   ├── Technician         (Single Select)
    │   ├── Deployment Status  (Single Select)
    │   ├── Last Updated       (Last Modified Time)
    │   └── Deployment Date    (Date)
    │
    ├── Views:
    │   ├── Master Grid View
    │   ├── Work-In-Progress Kanban
    │   └── Tech Gallery
    │
    └── Automations:
        └── Auto-stamp Deployment Date when Status → Deployed
```

---

## 4. Prerequisites & Account Setup

### 4.1 What You Need Before Starting

- A computer with an internet browser (Google Chrome recommended)
- An Airtable account (free tier is sufficient for this project)
- The starter CSV file: `nexus_blue_hardware_refresh_starter.csv`

### 4.2 Creating a Free Airtable Account

If you do not yet have an Airtable account:

1. Open your browser and navigate to **https://airtable.com**
2. Click the blue **"Sign up for free"** button in the top-right corner
3. Enter your email address and create a password, or sign up with Google
4. Complete email verification if prompted
5. On the onboarding screen, you may be asked what you will use Airtable for — select **"Project management"**
6. Click through the onboarding and land on the **Airtable Home** dashboard

> **Note:** The free Airtable plan supports up to 1,000 records per base and 5 editors, which is fully sufficient for this 50-unit project.

---

## 5. Creating the Airtable Base

### 5.1 Navigate to Your Workspace

1. After logging in, you will land on the **Airtable Home** page
2. In the left sidebar, locate your workspace name (it usually defaults to your name, e.g., "John's Workspace")
3. Click on the workspace name to expand it

### 5.2 Create a New Base

1. In the main content area, look for the large blue **"+ Create"** button or a **"+ Add a base"** card
2. Click **"Start from scratch"** from the options that appear
3. A new base will be created with a default name like **"Untitled Base"**

### 5.3 Rename the Base

1. The base name appears at the top-left of the screen, typically highlighted for editing immediately after creation
2. If it is not already in edit mode, **click once on the base name** at the top of the screen
3. Clear the existing text and type exactly:
   ```
   Nexus Blue — Hardware Refresh Tracker
   ```
4. Press **Enter** to confirm the name

> **Tip:** Use the em dash (—) by typing `--` on most systems or copy-pasting it. This matches professional naming conventions.

### 5.4 Rename the Default Table

Airtable creates a default table called **"Table 1"** automatically.

1. At the bottom of the screen, you will see a tab labeled **"Table 1"**
2. **Double-click** on the tab labeled "Table 1"
3. The name will become editable
4. Clear the text and type exactly:
   ```
   Deployment Inventory
   ```
5. Press **Enter** to confirm

---

## 6. Creating the Deployment Inventory Table

When you open the base for the first time, Airtable pre-populates the table with several default fields: **Name**, **Notes**, **Attachments**, and a few others. You will need to delete these defaults before building your custom fields.

### 6.1 Delete All Default Fields

1. Right-click on the **"Name"** column header at the top of the grid
2. A dropdown menu will appear
3. Click **"Delete field"**
4. A confirmation dialog will appear — click **"Delete"** to confirm
5. Repeat this process for **every default field** in the table (Notes, Attachments, etc.)
6. Continue until the table is completely empty — no columns remain

> **Warning:** Make sure you have deleted ALL default columns before adding your custom fields. Mixing default fields with your custom fields can cause confusion during automation setup.

---

## 7. Field-by-Field Configuration

You will now build every field from scratch. Follow each subsection exactly in the order presented.

---

### 7.1 Field 1 — Asset Number (Autonumber)

**Purpose:** This is the underlying numeric counter that Airtable manages automatically. It increments by 1 for each new record. It is not the user-facing Asset ID — it is the raw number that the formula field (next) will use to build the prefixed ID.

**Why two fields?** Airtable's native Autonumber field produces plain integers (1, 2, 3…). It cannot add text prefixes like "NB-" directly. The solution is to use this Autonumber as the data source and then a Formula field to combine the prefix with the number.

#### Steps to Create:

1. Click the **"+"** icon to the right of all existing column headers (or click **"Add field"** at the top)
2. A field configuration panel will appear on the right side of the screen
3. In the **"Field name"** input at the top of the panel, type:
   ```
   Asset Number
   ```
4. Below the name field, locate the **"Field type"** dropdown
5. Click on it to open the field type picker
6. Scroll through the list and click **"Autonumber"**
7. The configuration panel will update — Autonumber has no additional settings to configure
8. Click the **"Save field"** button at the bottom of the panel

**Expected Result:** A new column called "Asset Number" will appear in the grid. As you add records, this column will automatically display 1, 2, 3… and so on.

---

### 7.2 Field 2 — Asset ID (Formula)

**Purpose:** This is the user-facing device identifier displayed on asset labels, reports, and communications. It combines the prefix "NB-" with the Autonumber to produce IDs like NB-1, NB-2, NB-50.

**Example values:**

| Asset Number | Asset ID |
|---|---|
| 1 | NB-1 |
| 2 | NB-2 |
| 15 | NB-15 |
| 50 | NB-50 |

#### Steps to Create:

1. Click the **"+"** icon again to add another field
2. In the **"Field name"** input, type:
   ```
   Asset ID
   ```
3. Click the **"Field type"** dropdown
4. Scroll through the list and click **"Formula"**
5. The panel will now show a formula editor text area
6. Click inside the formula editor and type the following formula **exactly**:
   ```
   "NB-" & {Asset Number}
   ```
7. As you type, Airtable may auto-suggest field names — this is normal. Confirm that `{Asset Number}` refers to the Autonumber field you created in the previous step
8. Below the formula editor, a **preview** section will show you what the output will look like. If you see something like `NB-1`, the formula is working correctly
9. Click **"Save field"**

> **Note:** The `&` operator in Airtable formulas concatenates (joins) text strings. `"NB-"` is a literal text prefix and `{Asset Number}` dynamically inserts the autonumber value. Do not add spaces inside the quotes unless you want them in the output.

> **Warning:** If the preview shows an error (red text), double-check that you spelled `{Asset Number}` exactly as the Autonumber field is named. Field references in formulas are case-sensitive.

---

### 7.3 Field 3 — Device Model (Single Select)

**Purpose:** Tracks which hardware model each unit is. Using a Single Select (dropdown) instead of free text ensures consistent naming — technicians cannot accidentally enter "macbook pro" vs "MacBook Pro" inconsistently.

**Why Single Select?** Free-text fields for categorical data create filtering and grouping problems. Single Select enforces a controlled vocabulary, making views, filters, and statistics reliable.

#### Steps to Create:

1. Click the **"+"** icon to add a new field
2. In **"Field name"**, type:
   ```
   Device Model
   ```
3. Click the **"Field type"** dropdown and select **"Single select"**
4. The panel will show an **"Options"** section with an input to add choices
5. Click **"Add an option"** and type:
   ```
   Dell Latitude
   ```
6. Press **Enter** to confirm and click **"Add an option"** again
7. Type:
   ```
   MacBook Pro
   ```
8. Press **Enter** and click **"Add an option"** again
9. Type:
   ```
   Lenovo ThinkPad
   ```
10. Press **Enter** to confirm

#### Recommended Color Assignments:

Airtable allows you to assign colors to Single Select options. Click the colored circle to the left of each option to set its color:

| Option | Recommended Color |
|---|---|
| Dell Latitude | Blue |
| MacBook Pro | Gray |
| Lenovo ThinkPad | Red |

11. After adding all three options and assigning colors, click **"Save field"**

---

### 7.4 Field 4 — Serial Number (Single Line Text)

**Purpose:** Captures the manufacturer's unique serial number printed on the device. This is essential for warranty claims, theft recovery, and device identification in the event of a discrepancy between the asset tag and the physical device.

**Why unique tracking matters:** In large deployments, devices are physically indistinguishable from one another. The serial number is the only field that ties a specific physical unit to its record — even if the asset label is damaged or missing.

**Suggested validation practices:**
- Serial numbers are typically 8–15 alphanumeric characters
- Always confirm the serial number by checking both the sticker on the bottom of the device AND the operating system's system information screen
- Do not use spaces in serial numbers

#### Steps to Create:

1. Click **"+"** to add a new field
2. In **"Field name"**, type:
   ```
   Serial Number
   ```
3. Click the **"Field type"** dropdown and select **"Single line text"**
4. No additional configuration is needed
5. Click **"Save field"**

---

### 7.5 Field 5 — Assigned User (Single Line Text)

**Purpose:** Records the full name of the employee at the legal firm who will receive this device. This field ensures that every device has a designated owner before it leaves the technician's hands.

**Naming conventions:** Enter names in **First Last** format (e.g., "Harvey Specter"). This ensures consistent sorting and display throughout the base. Do not use nicknames or email addresses in this field.

**Why this field matters for deployment:** During physical delivery, technicians must confirm that the device being handed over matches the Assigned User in the record. This prevents devices from reaching the wrong employee, which is especially critical in a law firm where confidential client data may be pre-configured on devices.

#### Steps to Create:

1. Click **"+"** to add a new field
2. In **"Field name"**, type:
   ```
   Assigned User
   ```
3. Click the **"Field type"** dropdown and select **"Single line text"**
4. No additional configuration is needed
5. Click **"Save field"**

---

### 7.6 Field 6 — Technician (Single Select) — Option A (Recommended)

**Purpose:** Identifies which Nexus Blue technician is responsible for preparing and delivering this specific device. This field enables workload visibility — you can see at a glance how many devices each technician owns.

There are two ways to implement this field. **Option A** (Single Select) is recommended for this project. **Option B** (Linked Table) is documented as an advanced alternative below.

---

#### Option A — Single Select (Recommended for This Project)

**Why Single Select is appropriate here:** For a team of three known technicians working on a fixed-scope project, a Single Select is faster to configure, easier to maintain, and simpler for non-technical users to operate. It also enables grouping in the Gallery view without requiring linked-record lookups.

##### Steps to Create:

1. Click **"+"** to add a new field
2. In **"Field name"**, type:
   ```
   Technician
   ```
3. Click the **"Field type"** dropdown and select **"Single select"**
4. Click **"Add an option"** and type:
   ```
   Marcus
   ```
5. Press **Enter** and add:
   ```
   Sarah
   ```
6. Press **Enter** and add:
   ```
   Elena
   ```

##### Recommended Color Assignments:

| Option | Recommended Color |
|---|---|
| Marcus | Purple |
| Sarah | Green |
| Elena | Orange |

7. Click **"Save field"**

---

#### Option B — Linked Team Table (Advanced, for Future Scalability)

> **Note:** Do not implement Option B now unless you are comfortable with Airtable and have completed Option A first. Option B is documented here as a scalability upgrade path.

**Why a Linked Table is more powerful at scale:** When you link the Technician field to a separate Team table, each technician becomes a full record with their own metadata (phone number, email, certifications, availability). You can then use Lookup fields to pull technician contact info into the Deployment Inventory view, and Rollup fields to automatically count how many devices each technician owns.

##### Steps to Create Option B (Advanced):

**Step 1 — Create the Team Table:**
1. At the bottom of the screen, click the **"+"** button next to the "Deployment Inventory" tab
2. Select **"Create new table"**
3. Name the table:
   ```
   Team
   ```
4. Add fields to the Team table:
   - **Name** (Single Line Text — the default field, rename it to "Technician Name")
   - **Email** (Email field type)
   - **Phone** (Phone Number field type)
   - **Role** (Single Line Text)
5. Add records for Marcus, Sarah, and Elena

**Step 2 — Link the Technician Field:**
1. Return to the **Deployment Inventory** table
2. Add a new field with type **"Link to another record"**
3. Name it:
   ```
   Technician
   ```
4. In the configuration, choose the **"Team"** table as the linked table
5. Click **"Save field"**
6. Now when entering a technician for a record, you select from the linked Team table records

**Benefits of Option B:**
- Centralized technician contact directory
- Automatic device count rollups per technician
- Easy to add or remove technicians without changing field options
- Foundation for future HR or scheduling integrations

---

### 7.7 Field 7 — Deployment Status (Single Select)

**Purpose:** This is the operational heart of the tracker. The Deployment Status field represents where each device sits in the preparation-to-delivery pipeline. It powers the Kanban view and triggers the automation that stamps the deployment date.

**Status pipeline logic:** Devices always move forward through the pipeline — never backward in normal operations:

```
Unboxed → Imaged → Ready for Delivery → Deployed
```

| Status | Meaning |
|---|---|
| **Unboxed** | Device has been received and physically removed from packaging. Not yet configured. |
| **Imaged** | Technician has installed the OS image, software, and security policies on the device. |
| **Ready for Delivery** | Device is fully configured, tested, and staged for handoff to the user. |
| **Deployed** | Device has been physically delivered to and signed for by the assigned user. |

#### Steps to Create:

1. Click **"+"** to add a new field
2. In **"Field name"**, type:
   ```
   Deployment Status
   ```
3. Click the **"Field type"** dropdown and select **"Single select"**
4. Add the following options in order:
   - `Unboxed`
   - `Imaged`
   - `Ready for Delivery`
   - `Deployed`

##### Recommended Color Assignments:

| Status | Recommended Color |
|---|---|
| Unboxed | Gray |
| Imaged | Yellow |
| Ready for Delivery | Blue |
| Deployed | Green |

> **Tip:** Using a traffic-light-inspired color scheme (gray → yellow → blue → green) makes the pipeline progression intuitive at a glance.

5. Click **"Save field"**

---

### 7.8 Field 8 — Last Updated (Last Modified Time)

**Purpose:** Automatically records the exact date and time that any field in the record was last changed. This field requires zero manual input — Airtable maintains it entirely automatically.

**Why this matters for audit tracking:** In a managed services deployment, you may need to prove when a device's status was changed — for compliance, for SLA validation, or in the event of a dispute with the client. The Last Updated field provides an immutable, automatic audit timestamp.

#### Steps to Create:

1. Click **"+"** to add a new field
2. In **"Field name"**, type:
   ```
   Last Updated
   ```
3. Click the **"Field type"** dropdown and select **"Last modified time"**
4. A configuration panel appears with a **"Fields to watch"** section
5. Click the **"Fields to watch"** dropdown
6. You will see an option for **"All editable fields"** — select this option
7. This ensures that any change to any editable field in the record updates this timestamp
8. Leave the **date format** as the default local format
9. Click **"Save field"**

> **Note:** You cannot manually edit the Last Updated field — Airtable controls it. If you try to click into the field in a record, it will not open for editing. This is by design.

---

### 7.9 Field 9 — Deployment Date (Date)

**Purpose:** Records the official date that the device was delivered to the user. This field is primarily populated by the automation you will configure in Section 9, not by manual entry.

**Why manual editing should be minimized:** Manual date entry is error-prone. Technicians may enter the wrong date, forget to update the field, or use inconsistent formats. By having the automation populate this field the moment the status is changed to "Deployed," you eliminate these risks and ensure the timestamp is always accurate.

#### Steps to Create:

1. Click **"+"** to add a new field
2. In **"Field name"**, type:
   ```
   Deployment Date
   ```
3. Click the **"Field type"** dropdown and select **"Date"**
4. In the configuration panel, locate the **"Include time field"** toggle
5. Make sure this toggle is set to **OFF** (grey/disabled) — we only want the date, not the time
6. For **"Date format"**, select **"Local"** from the dropdown (this will display the date in the format standard for your region, e.g., MM/DD/YYYY in the US or DD/MM/YYYY in the UK)
7. Click **"Save field"**

---

### 7.10 Final Field Order Summary

After creating all fields, your table should have the following columns from left to right:

| # | Field Name | Field Type |
|---|---|---|
| 1 | Asset Number | Autonumber |
| 2 | Asset ID | Formula |
| 3 | Device Model | Single Select |
| 4 | Serial Number | Single Line Text |
| 5 | Assigned User | Single Line Text |
| 6 | Technician | Single Select |
| 7 | Deployment Status | Single Select |
| 8 | Last Updated | Last Modified Time |
| 9 | Deployment Date | Date |

**Reordering fields (if needed):**
If your fields are out of order, you can drag them to reorder:
1. Hover over the column header of the field you want to move
2. A **drag handle** (six dots arranged in a grid) will appear to the left of the column name
3. Click and hold the drag handle
4. Drag the column to the correct position
5. Release to drop it in place

---

## 8. View Creation

Airtable supports multiple types of views of the same data. You will now create three views, each serving a different operational purpose.

---

### 8.1 View 1 — Master Grid View

**Purpose:** This is the primary operational view showing every record in a traditional spreadsheet layout. It is the first view to open when entering the base and the view used for data entry, record editing, and full-table oversight.

**Why this is the operational master view:** Grid view provides maximum data density — you can see all fields for all records simultaneously. Sorting by Asset ID ensures records always appear in a logical, numbered sequence regardless of the order in which they were entered.

#### Steps to Create:

1. Look at the **left sidebar** of the screen — this is the Views panel
2. You should already see a default view called **"Grid view"** at the top of the panel
3. **Double-click** on "Grid view" to rename it
4. Type:
   ```
   Master Grid View
   ```
5. Press **Enter** to confirm

#### Configure Sorting:

1. With the Master Grid View active, click the **"Sort"** button in the toolbar at the top of the screen (it looks like an icon with horizontal lines of different lengths)
2. The Sort panel will open
3. Click **"+ Add a sort"**
4. In the **"Field"** dropdown, select **"Asset ID"**
5. In the **"Direction"** dropdown, confirm it is set to **"A → Z"** (ascending)
6. Click anywhere outside the Sort panel to close it and apply the sort

> **Expected Result:** All records are now displayed in ascending order by Asset ID (NB-1, NB-2, NB-3…). New records added to the base will automatically sort into the correct position.

---

### 8.2 View 2 — Work-In-Progress Kanban

**Purpose:** The Kanban view visualizes the deployment pipeline as a board with vertical columns — one column per status. Devices (cards) are physically dragged from one column to the next as their status changes. This view is primarily used by technicians during active deployment work.

**Why Kanban is useful for deployment pipelines:** A Kanban board provides instant visual comprehension of the entire pipeline state. At a glance, a project manager can see how many devices are stuck in "Imaged" vs. how many are "Ready for Delivery." Bottlenecks become immediately visible. Unlike a grid view where you must read row-by-row, the Kanban gives you a spatial overview.

**How drag-and-drop works:** When a technician finishes imaging a device, they find the device's card in the "Imaged" column and drag it to the "Ready for Delivery" column. This action automatically updates the Deployment Status field in the record — no manual field editing required. This is how technicians interact with the system most naturally.

#### Steps to Create:

1. In the left **Views** panel, look for the **"+ Add view"** link at the bottom of the views list
2. Click **"+ Add view"**
3. A menu will appear showing all available view types
4. Click **"Kanban"**
5. Airtable will ask you to configure the Kanban

#### Configure Kanban Stacking:

1. In the **"Stack by"** dropdown, select **"Deployment Status"**
2. This tells Airtable to create one column for each status option: Unboxed, Imaged, Ready for Delivery, Deployed
3. Click **"Create view"** or equivalent confirm button

#### Rename the View:

1. The new view will appear in the sidebar with a default name like "Kanban"
2. **Double-click** on the view name in the sidebar
3. Type:
   ```
   Work-In-Progress Kanban
   ```
4. Press **Enter**

#### Configure Card Fields:

1. In the Kanban view, click the **"Fields"** button in the toolbar
2. This controls which fields appear on each card
3. Turn **ON** the following fields:
   - Assigned User
   - Technician
   - Device Model
4. Turn **OFF** fields you don't need on cards (like Serial Number, Last Updated) to keep cards clean
5. Close the Fields panel

> **Expected Result:** The Kanban board now shows four columns (Unboxed, Imaged, Ready for Delivery, Deployed). Each device appears as a card in the column matching its current status. Cards show the assigned user and technician name.

**How technicians move devices through stages:**
- A technician physically unboxes a device → finds its card in "Unboxed" → drags it to "Imaged"
- After completing imaging → drags it to "Ready for Delivery"
- After physical handoff to the user → drags it to "Deployed" (this triggers the automation)

> **Important:** When a card is dragged to the **"Deployed"** column, this will trigger the automation you configure in Section 9, which will automatically populate the Deployment Date field.

---

### 8.3 View 3 — Tech Gallery

**Purpose:** The Gallery view displays each device as a visual card in a grid layout, grouped by technician. This gives technicians and the project manager a workload-per-technician view — an at-a-glance overview of how many devices each team member owns.

**Why gallery view helps workload visibility:** The Gallery's card-based layout makes it easy to count and scan records visually. When grouped by Technician, you immediately see each person's queue. If Marcus has 20 cards and Elena has 5, workload rebalancing is immediately obvious.

#### Steps to Create:

1. In the left **Views** panel, click **"+ Add view"**
2. From the view type menu, click **"Gallery"**
3. Airtable will create the view — rename it immediately:
4. **Double-click** on the view name in the sidebar and type:
   ```
   Tech Gallery
   ```
5. Press **Enter**

#### Configure Grouping:

1. With the Tech Gallery view active, click the **"Group"** button in the toolbar
2. The Group panel will open
3. Click **"+ Add grouping"**
4. In the field dropdown, select **"Technician"**
5. Leave the direction as the default (A → Z)
6. Close the Group panel

#### Configure Visible Fields on Cards:

1. Click the **"Fields"** button in the toolbar
2. Turn **ON** the following fields for the gallery cards:
   - Asset ID
   - Device Model
   - Assigned User
   - Deployment Status
3. Turn **OFF** all other fields to keep the gallery cards clean and scannable
4. Close the Fields panel

> **Expected Result:** The Gallery view displays all device cards grouped into three sections — one section per technician (Marcus, Sarah, Elena). Each card shows the Asset ID, Device Model, Assigned User, and Deployment Status.

---

## 9. Automation Setup

This automation monitors the Deployment Status field. When a record's status changes to "Deployed," the automation immediately updates the Deployment Date field with today's date. This eliminates the need for technicians to manually record deployment timestamps.

---

### 9.1 Opening the Automations Panel

1. Look at the top navigation bar of the Airtable interface
2. Click the **"Automations"** button — it has a lightning bolt (⚡) icon
3. A panel or full screen will open showing the Automations workspace
4. Click **"+ New automation"** or **"Create an automation"** to begin building

---

### 9.2 Naming the Automation

1. At the top of the automation editor, you will see a text field with a default name like "Untitled automation"
2. Click on this name and replace it with:
   ```
   Auto-Stamp Deployment Date on Status → Deployed
   ```
3. This name clearly communicates what the automation does to anyone reviewing it later

---

### 9.3 Configuring the Trigger

The trigger is the event that causes the automation to run. In this case, the trigger is a specific field change.

1. In the automation editor, you will see a **"Trigger"** section at the top
2. Click **"Choose a trigger"** or the dropdown that appears there
3. From the list of available triggers, select:
   ```
   When a record is updated
   ```
4. After selecting this trigger type, a configuration panel will appear on the right side

#### Configure the Trigger Settings:

**Step 1 — Select the Table:**
1. Under **"Table"**, click the dropdown
2. Select **"Deployment Inventory"**

**Step 2 — Select the Watched Field:**
1. Under **"Fields to watch"** (or "When these fields change"), click the dropdown
2. **Uncheck "All fields"** if it is selected by default
3. Check only the **"Deployment Status"** field
4. This tells Airtable: "Only fire this trigger when the Deployment Status field changes — not when any other field changes."

**Step 3 — Add a Condition:**
The trigger fires when any update occurs to Deployment Status. But you only want the automation to run when the status becomes specifically "Deployed." To restrict this:

1. Look for a **"Conditions"** or **"Filter"** section within the trigger configuration
2. Click **"+ Add condition"** or **"Add filter"**
3. Set the condition:
   - **Field:** Deployment Status
   - **Condition:** `is`
   - **Value:** `Deployed`
4. This means: "Only proceed if the updated value of Deployment Status is exactly 'Deployed'"

> **Note:** Without this condition, the automation would fire every time the status changes (e.g., from Unboxed to Imaged), which is not desired.

---

### 9.4 Configuring the Action

The action is what the automation does after the trigger fires. You will configure it to update the Deployment Date field.

1. Below the Trigger section in the automation editor, click **"+ Add action"**
2. From the list of action types, select:
   ```
   Update record
   ```

#### Configure the Action Settings:

**Step 1 — Select the Table:**
1. Under **"Table"**, select **"Deployment Inventory"**

**Step 2 — Select the Record:**
1. Under **"Record"** or **"Record ID"**, you need to tell Airtable which record to update — it should be the same record that triggered the automation
2. Click the dropdown and select the option that references the trigger record. This is typically labeled:
   ```
   Trigger → Record ID
   ```
   or simply
   ```
   Record ID from trigger
   ```
3. This ensures that the exact record whose status was just changed to "Deployed" is the one that gets its date updated

**Step 3 — Set the Field to Update:**
1. In the **"Fields"** section of the action, click **"+ Add field"**
2. From the field dropdown, select **"Deployment Date"**

---

### 9.5 Setting the Deployment Date Value

There are two methods for setting the date value. **Method A is recommended.**

---

#### Method A — Recommended: Use Airtable's "Current time" Dynamic Value

This is the simplest and most reliable approach.

1. After selecting "Deployment Date" as the field to update, a value input will appear to the right
2. Click inside the value input
3. Look for a **"dynamic values"** option, often represented by a blue **"+"** or a lightning bolt icon in the input field
4. Click on dynamic values
5. Select **"Current time"** or **"Today"** from the options
6. Airtable will insert a dynamic reference that resolves to the current date when the automation runs

**Pros of Method A:**
- Simple to configure
- No formula knowledge required
- Reliable — Airtable handles date formatting automatically
- Always uses the server-side date when the automation fires

**Cons of Method A:**
- In rare cases, "Current time" may include a time component — verify the Deployment Date field is configured as Date-only to strip the time

---

#### Method B — Advanced: Use Automation Timestamp Variables

This method uses Airtable's built-in formula/variable system within automations.

1. In the Deployment Date value input, instead of selecting "Current time," click on the formula or variable option
2. Enter the following expression:
   ```
   TODAY()
   ```
   or reference the automation's own execution timestamp variable if available in your Airtable plan

**Pros of Method B:**
- More explicit control over date formatting
- Can be used in more complex automation chains (e.g., if you later add multiple date-stamping steps)

**Cons of Method B:**
- Requires understanding of Airtable formula syntax
- Not available on all Airtable plan tiers
- Slightly more complex to configure and debug

> **Recommendation:** Use **Method A** for this project. It requires no formula knowledge and works reliably on all Airtable plan levels.

---

### 9.6 Enable and Test the Automation

1. After configuring both the trigger and the action, look for the automation's **on/off toggle** at the top of the automation editor
2. The toggle may say **"Off"** by default — click it to switch it to **"On"**
3. The automation is now live and will fire whenever a record's Deployment Status is changed to "Deployed"

> **Warning:** Do not leave the automation in the disabled (Off) state after setup. An off automation will never fire, and deployment dates will not be stamped. Always verify the toggle is green/on before using the base in production.

---

## 10. Data Import Instructions

You have been provided with a starter CSV file containing 15 pre-populated device records. You will import this file into the Deployment Inventory table.

---

### 10.1 Understanding the Starter CSV

The file `nexus_blue_hardware_refresh_starter.csv` contains the following 15 records:

| Asset ID | Device Model | Serial Number | Assigned User | Technician | Deployment Status | Last Updated | Deployment Date |
|---|---|---|---|---|---|---|---|
| NB-0001 | Dell Latitude 5420 | SN-000101X | Harvey Specter | Marcus | Deployed | 2024-05-01 09:00:00 | 2024-05-01 |
| NB-0002 | MacBook Pro 14" | SN-000102X | Jessica Pearson | Elena | Deployed | 2024-05-01 10:30:00 | 2024-05-01 |
| NB-0003 | Lenovo ThinkPad X1 Carbon | SN-000103X | Louis Litt | Sarah | Deployed | 2024-05-02 14:15:00 | 2024-05-02 |
| NB-0004 | Dell Latitude 5420 | SN-000104X | Mike Ross | Marcus | Deployed | 2024-05-03 11:00:00 | 2024-05-03 |
| NB-0005 | MacBook Pro 16" | SN-000105X | Rachel Zane | Elena | Deployed | 2024-05-04 16:45:00 | 2024-05-04 |
| NB-0006 | Lenovo ThinkPad T14 | SN-000106X | Donna Paulsen | Sarah | Ready for Delivery | 2024-05-06 09:20:00 | *(blank)* |
| NB-0007 | Dell Latitude 7430 | SN-000107X | Robert Zane | Marcus | Ready for Delivery | 2024-05-06 13:00:00 | *(blank)* |
| NB-0008 | Lenovo ThinkPad X1 Carbon | SN-000108X | Katrina Bennett | Elena | Ready for Delivery | 2024-05-07 10:00:00 | *(blank)* |
| NB-0009 | Dell Latitude 5420 | SN-000109X | Alex Williams | Sarah | Ready for Delivery | 2024-05-07 11:30:00 | *(blank)* |
| NB-0010 | MacBook Pro 14" | SN-000110X | Samantha Wheeler | Marcus | Imaged | 2024-05-08 15:00:00 | *(blank)* |
| NB-0011 | Lenovo ThinkPad T14 | SN-000111X | Daniel Hardman | Elena | Imaged | 2024-05-09 08:45:00 | *(blank)* |
| NB-0012 | Dell Latitude 7430 | SN-000112X | Sheila Sazs | Sarah | Imaged | 2024-05-09 14:00:00 | *(blank)* |
| NB-0013 | MacBook Pro 16" | SN-000113X | Gretchen Bodinski | Marcus | Imaged | 2024-05-10 10:15:00 | *(blank)* |
| NB-0014 | Dell Latitude 5420 | SN-000114X | Harold Gunderson | Elena | Imaged | 2024-05-10 16:30:00 | *(blank)* |
| NB-0015 | Lenovo ThinkPad X1 Carbon | SN-000115X | Travis Tanner | Sarah | Unboxed | 2024-05-12 09:00:00 | *(blank)* |

> **Note:** The Asset ID column in the CSV (NB-0001, NB-0002…) is a reference. Since your Airtable base uses an Autonumber + Formula to generate Asset IDs, you will **not** import the Asset ID column directly. The Asset ID will be auto-generated by Airtable. Map the CSV columns to the appropriate Airtable fields as instructed below.

---

### 10.2 Importing the CSV into Airtable

#### Method — Using Airtable's CSV Import Feature

1. With the **Deployment Inventory** table open and the **Master Grid View** active, click the **"+"** icon at the very bottom of the screen (below the last record row)

   > **Alternative method:** In some Airtable versions, you can also access CSV import by clicking the **"..."** (more options) button next to the table name tab and selecting **"Import data"** or **"Import CSV"**

2. Alternatively, look for the **"Add or import"** option in the toolbar area. The exact location varies by Airtable version but is usually near the record list

3. A dialog box will appear — select **"CSV file"** or **"Import a spreadsheet"**

4. Click **"Choose file"** or drag the `nexus_blue_hardware_refresh_starter.csv` file into the dialog box

5. Airtable will read the file and show you a **preview** of the data and a field mapping interface

---

### 10.3 Mapping CSV Columns to Airtable Fields

This is the most critical step of the import. You must tell Airtable which CSV column corresponds to which Airtable field.

In the mapping screen, you will see each CSV column header on the left, and a dropdown on the right where you select the matching Airtable field.

**Map as follows:**

| CSV Column | Map To (Airtable Field) |
|---|---|
| Asset ID | **Do NOT import** — Select "Skip this field" |
| Device Model | Device Model |
| Serial Number | Serial Number |
| Assigned User | Assigned User |
| Technician | Technician |
| Deployment Status | Deployment Status |
| Last Updated | **Do NOT import** — Select "Skip this field" |
| Deployment Date | Deployment Date |

> **Why skip Asset ID?** Because your Airtable base auto-generates Asset IDs using the Autonumber + Formula combination. Importing the CSV's Asset ID values would conflict with this system. Let Airtable generate fresh Asset IDs (NB-1 through NB-15).

> **Why skip Last Updated?** The Last Modified Time field is automatically controlled by Airtable and cannot be set manually during import.

---

### 10.4 Completing the Import

1. After mapping all columns, click **"Import records"** or **"Confirm import"**
2. Airtable will process the file and add 15 new records to the Deployment Inventory table
3. A success confirmation will appear showing how many records were imported

---

### 10.5 Verifying Imported Records

After the import completes:

1. Return to the **Master Grid View**
2. You should now see 15 records in the table
3. Scroll through each record and verify:
   - Device Model values match the CSV (Dell Latitude, MacBook Pro, Lenovo ThinkPad variants)
   - Serial Numbers are populated correctly
   - Assigned User names are correct
   - Technician names show as colored Single Select tags (Marcus, Sarah, Elena)
   - Deployment Status values are correct (Deployed, Ready for Delivery, Imaged, Unboxed)
   - Deployment Date is populated for the 5 "Deployed" records and blank for the rest
4. Check that the **Asset ID** formula field auto-generated values like NB-1, NB-2, NB-3…

---

### 10.6 Fixing Mapping Mistakes

If records imported with incorrect field values:

1. Click **"Undo"** in the top-left (or press **Ctrl+Z** / **Cmd+Z**) to revert the import if you catch the error immediately
2. Re-import using the correct column mapping
3. If the undo is no longer available, manually delete all imported records:
   - Click the checkbox at the far left of the first record
   - Hold **Shift** and click the last record's checkbox to select all
   - Right-click and select **"Delete records"**
4. Then re-import the CSV with correct mappings

---

### 10.7 Extending to 50 Units

The starter CSV contains 15 records. To reach the full 50-unit deployment:

**Option A — Manual Entry:**
1. Click the **"+"** row at the bottom of the grid to add a new blank record
2. Fill in Device Model, Serial Number, Assigned User, Technician, and Deployment Status
3. Repeat for records 16 through 50

**Option B — Duplicate Records:**
1. Click on an existing record's row number (far left) to select it
2. Right-click and select **"Duplicate record"**
3. Edit the duplicated record's details (Serial Number, Assigned User) to make it unique
4. Repeat until you have 50 records total

**Option C — Extend the CSV:**
1. Open the CSV file in Microsoft Excel, Google Sheets, or a text editor
2. Add rows 16 through 50 with real or placeholder device data
3. Save the file and re-import only the new rows
   > Use the "Merge" import option if available to avoid duplicating existing records

---

## 11. Testing Procedures

Before putting the base into production use, you must verify that all components work correctly.

---

### 11.1 Testing the Asset ID Formula

1. Open the **Master Grid View**
2. Look at the **Asset ID** column
3. Verify that values display as **NB-1, NB-2, NB-3…** (not as plain numbers)
4. If Asset IDs show as plain numbers, verify the formula field configuration:
   - Click on the Asset ID column header
   - Click **"Customize field type"** or **"Edit field"**
   - Confirm the formula is exactly `"NB-" & {Asset Number}`
   - Confirm that `{Asset Number}` references the correct Autonumber field

---

### 11.2 Testing the Kanban View

1. Click on **"Work-In-Progress Kanban"** in the Views panel
2. Verify that four columns appear: Unboxed, Imaged, Ready for Delivery, Deployed
3. Verify that device cards appear in the correct column based on their current status
4. Test drag-and-drop:
   - Find a card in the **Imaged** column
   - Click and hold it, then drag it to the **Ready for Delivery** column
   - Release the card
   - The card should now sit in the "Ready for Delivery" column
   - Navigate to the Master Grid View — the record's Deployment Status should now show "Ready for Delivery"
5. Drag the same card back to "Imaged" to restore its original status

---

### 11.3 Testing the Automation (Critical Test)

This is the most important test. It validates that the auto-stamping of the Deployment Date works correctly.

**Pre-test setup:**
1. Identify one record that is currently **NOT** in "Deployed" status (use a record with status "Ready for Delivery" or "Imaged")
2. Note the record's current **Deployment Date** value — it should be blank

**Test steps:**
1. Open the record by clicking on it (click the expand icon — the diagonal arrow — to open the full record detail panel)
2. Locate the **Deployment Status** field
3. Click on it and change the value from its current status to **"Deployed"**
4. Close the record panel
5. **Wait 15–30 seconds** — Airtable automations are not instant; they run on a short delay
6. Re-open the same record (click the expand icon again)
7. Look at the **Deployment Date** field

**Expected Result:**
The Deployment Date field should now be populated with today's date (the date you performed this test). It will not be blank anymore.

---

### 11.4 Troubleshooting Automation Test Failures

If the Deployment Date did NOT auto-populate:

1. **Check if the automation is enabled:**
   - Go to the Automations panel
   - Verify the automation toggle is set to **"On"** (green)

2. **Check the automation run history:**
   - In the Automations panel, click on your automation
   - Look for a **"Run history"** or **"Recent runs"** tab
   - Check if the automation shows any recent runs — if the run shows an error, the error message will tell you what went wrong

3. **Check the trigger configuration:**
   - Verify that the trigger watches the **"Deployment Status"** field specifically
   - Verify the condition is set to: Deployment Status `is` "Deployed"

4. **Check the action configuration:**
   - Verify that the action updates the **"Deployment Date"** field
   - Verify the record ID is mapped to the trigger record

5. **Re-test:** After making corrections, change the status back to "Imaged" and then to "Deployed" again to re-trigger the automation

---

### 11.5 Testing the Gallery View

1. Click on **"Tech Gallery"** in the Views panel
2. Verify that records are grouped by technician name
3. You should see three groups: Marcus, Sarah, and Elena
4. Each group should show cards for only that technician's assigned devices
5. Verify that the correct fields are visible on the cards (Asset ID, Device Model, Assigned User, Deployment Status)

---

## 12. Screenshot Instructions

Professional screenshots are required for project documentation and submission. The following screenshots must be captured.

---

### Screenshot 1 — Kanban View with 3+ Populated Columns

**What to show:**
- The **Work-In-Progress Kanban** view
- At least 3 of the 4 status columns must contain device cards
- Cards should be visible with technician and user information

**How to make it clean:**
1. Ensure no record editing panel is open — close all modals
2. Zoom your browser to 80–90% (Ctrl+minus / Cmd+minus) to show more of the board
3. Ensure all four column headers are visible: Unboxed, Imaged, Ready for Delivery, Deployed
4. Take a full-browser screenshot (not just a cropped area)

**How to capture:**
- **Windows:** Press `Win + Shift + S` to open Snipping Tool, then drag to select the Kanban board
- **Mac:** Press `Cmd + Shift + 4`, then drag to select the area
- **Browser Extension:** Use GoFullPage or similar

---

### Screenshot 2 — Automation Trigger Configuration Screen

**What to show:**
- The Automations panel open
- The trigger configuration panel visible
- The "When a record is updated" trigger type visible
- The "Deployment Status" watched field visible
- The condition "Deployment Status is Deployed" visible

**How to capture:**
1. Open the Automations panel by clicking the ⚡ icon
2. Click on your automation to open its editor
3. Click on the Trigger block to expand its configuration
4. Ensure all configuration details are visible before screenshotting

---

### Screenshot 3 — Automation Action Configuration Screen

**What to show:**
- The action configuration panel visible
- The "Update record" action type visible
- The table set to "Deployment Inventory"
- The "Deployment Date" field being updated
- The "Current time" dynamic value visible

**How to capture:**
1. While still in the automation editor, scroll down to the Action block
2. Click on it to expand the configuration
3. Ensure the field mapping is fully visible before screenshotting

---

### Screenshot 4 — Master Grid View

**What to show:**
- The full Master Grid View
- All 9 columns visible (scroll to verify if needed)
- Multiple populated records visible
- Asset IDs showing the NB- prefix correctly
- Deployment Status color badges visible

**How to capture:**
1. Click on "Master Grid View" in the Views panel
2. Zoom out slightly to show more columns if they are cut off
3. Take a full-screen screenshot

---

### Screenshot 5 — Tech Gallery View

**What to show:**
- The Tech Gallery view
- Records clearly grouped by Technician (Marcus, Sarah, Elena sections visible)
- Device cards visible with field information
- Group headers clearly labeled

**How to capture:**
1. Click on "Tech Gallery" in the Views panel
2. Scroll so that all three technician groups are visible simultaneously
3. Take the screenshot

---

## 13. Video Walkthrough Guide

A short video demonstration of the Airtable base in action is required. The video should be under 2 minutes and demonstrate the key features of the system.

---

### 13.1 Required Content to Demonstrate

Your walkthrough video must show, in order:

1. **Opening the base** and landing on the Master Grid View (5 seconds)
2. **Scrolling through the records** to show all imported data (10 seconds)
3. **Switching to the Kanban view** — show the four columns and device cards (15 seconds)
4. **Dragging a Kanban card** from one column to another — for example, from "Imaged" to "Ready for Delivery" (10 seconds)
5. **Switching to the Tech Gallery view** — show the three technician groups (10 seconds)
6. **Demonstrating the automation firing** — change a record's status to "Deployed" and show the Deployment Date auto-populating (20–30 seconds; note: include a brief pause while the automation runs)
7. **Return to Master Grid View** and confirm the Deployment Date is filled (10 seconds)

---

### 13.2 Recommended Recording Tools

#### Loom (Recommended for Beginners)
- **Website:** https://www.loom.com
- Free plan available; allows recordings up to 5 minutes
- Install the Loom Chrome Extension or desktop app
- Click the Loom icon, select "Screen + Camera" or "Screen only"
- Click "Start Recording," perform your walkthrough, then click "Stop"
- Loom automatically generates a shareable link

#### OBS Studio (Recommended for Advanced Users)
- **Website:** https://obsproject.com (free, open-source)
- Provides the highest video quality
- Configure a "Screen Capture" scene
- Press "Start Recording," complete the walkthrough, press "Stop Recording"
- Export as MP4

#### Zoom Screen Recording
- Start a Zoom meeting with only yourself
- Click "Record" in the meeting toolbar
- Perform the walkthrough
- End the meeting — Zoom will save the recording as an MP4 file

---

### 13.3 Suggested Narration Script

```
"This is the Nexus Blue Hardware Refresh Tracker, built in Airtable 
for our 50-unit laptop deployment at [Client Name].

Here in the Master Grid View, we can see all 15 starter records 
imported, each with their Asset ID, device model, serial number, 
assigned user, technician, and status.

Switching to the Work-In-Progress Kanban, you can see devices 
grouped by their current stage in the deployment pipeline — 
Unboxed, Imaged, Ready for Delivery, and Deployed.

I'll drag this device from Imaged to Ready for Delivery — 
notice the status updates automatically.

In the Tech Gallery view, devices are grouped by technician, 
giving us instant workload visibility.

Now watch what happens when I mark a device as Deployed — 
the Deployment Date field auto-populates with today's date, 
triggered by our Airtable automation.

This base is now ready to track all 50 units through 
to full deployment."
```

---

### 13.4 Recording Tips

- **Close unnecessary browser tabs** before recording to avoid accidental notifications appearing
- **Use full-screen browser mode** (press F11) for a cleaner recording
- **Speak clearly and at a moderate pace** — do not rush; the goal is clarity
- **Pause briefly before each section** of the demo to give viewers time to register what they're seeing
- **Pre-test your automation** before recording — ensure it works so you don't get a failed automation on camera

---

### 13.5 Export Settings

- **Format:** MP4 (H.264)
- **Resolution:** 1080p (1920x1080) or 720p (1280x720)
- **Duration:** Under 2 minutes (aim for 90 seconds)
- **File size:** Aim under 50MB for easy sharing

---

## 14. Share Link & Submission

### 14.1 Generating a Read-Only Share Link

A read-only share link allows stakeholders and evaluators to view the base without the ability to edit, delete, or modify any records.

#### Steps:

1. In the top-right area of the Airtable interface, click the **"Share"** button
2. A sharing dialog will appear
3. Under the **"Create a shared link to this view"** or **"Share base"** section, click **"Enable shared link"** or **"Create link"**
4. Airtable will generate a unique URL

#### Permission Settings:

1. Verify that the permission level is set to **"Read only"** — this ensures that anyone with the link can view the data but cannot make changes
2. Do **not** enable "Allow editors to add collaborators" or similar options
3. Do **not** share with edit or comment permissions for external stakeholders

---

### 14.2 Verifying the Share Link

1. Copy the generated share link
2. Open a **private/incognito browser window** (so you are not logged into Airtable)
3. Paste the share link and press Enter
4. Verify:
   - The base opens and is viewable
   - You cannot edit any records (clicking on cells should not open editing)
   - All three views are visible in the sidebar
   - The data appears complete and professional

---

### 14.3 Security Recommendations

- **Never share the base editor link** — only share the read-only share link
- **Do not include sensitive personal data** (social security numbers, passwords, etc.) in the base
- In a real client engagement, consider adding a **password** to the shared link (available on Airtable Pro plans)
- Periodically **audit who has access** via the Sharing settings panel

---

## 15. Acceptance Criteria Checklist

Use this checklist to verify that all project requirements have been met before submission.

| # | Requirement | Status |
|---|---|---|
| 1 | Asset IDs correctly display NB- prefix (NB-1, NB-2…) | ☐ |
| 2 | All 9 required fields are present and correctly typed | ☐ |
| 3 | Deployment Status Single Select has all 4 options with colors | ☐ |
| 4 | Technician Single Select has Marcus, Sarah, Elena options | ☐ |
| 5 | Master Grid View exists and sorts by Asset ID ascending | ☐ |
| 6 | Work-In-Progress Kanban exists and stacks by Deployment Status | ☐ |
| 7 | Kanban supports drag-and-drop (tested manually) | ☐ |
| 8 | Tech Gallery View exists and groups by Technician | ☐ |
| 9 | Starter CSV imported successfully (15 records visible) | ☐ |
| 10 | Automation is enabled (toggle is ON) | ☐ |
| 11 | Automation timestamps Deployment Date when Status → Deployed | ☐ |
| 12 | Automation tested and working (verified manually) | ☐ |
| 13 | Read-only share link generated and verified | ☐ |
| 14 | Screenshot 1 captured (Kanban with 3+ columns populated) | ☐ |
| 15 | Screenshot 2 captured (Automation trigger configuration) | ☐ |
| 16 | Screenshot 3 captured (Automation action configuration) | ☐ |
| 17 | Screenshot 4 captured (Master Grid View) | ☐ |
| 18 | Screenshot 5 captured (Tech Gallery View) | ☐ |
| 19 | Video walkthrough recorded and under 2 minutes | ☐ |
| 20 | Video demonstrates all required features | ☐ |

---

## 16. QA Checklist

Use this additional quality assurance checklist to verify data integrity and system accuracy.

### Data Integrity

| Check | Expected Result | Pass/Fail |
|---|---|---|
| Asset Number column contains integers only (1, 2, 3…) | No text, no gaps | |
| Asset ID column contains "NB-" prefix on every record | NB-1 through NB-15+ | |
| No two records share the same Serial Number | All serial numbers unique | |
| No record has a blank Assigned User | Every record has a name | |
| Deployment Date is blank for non-Deployed records | Only Deployed records have dates | |
| All 5 "Deployed" records in CSV have Deployment Date populated | Dates visible | |

### View Integrity

| Check | Expected Result | Pass/Fail |
|---|---|---|
| Master Grid View is sorted A→Z by Asset ID | NB-1 appears at top, NB-15 at bottom | |
| Kanban has exactly 4 columns | Unboxed, Imaged, Ready for Delivery, Deployed | |
| Gallery is grouped by exactly 3 technicians | Marcus, Sarah, Elena groups visible | |
| No records appear in wrong Kanban column | Each card's column matches its Deployment Status | |

### Automation Integrity

| Check | Expected Result | Pass/Fail |
|---|---|---|
| Automation shows as "On" in Automations panel | Green enabled toggle | |
| Automation run history shows successful runs | No error runs | |
| Deployment Date populated within 30 seconds of status change | Date appears automatically | |
| Automation does NOT fire for non-Deployed status changes | Only "Deployed" triggers the date | |

---

## 17. Troubleshooting

### 17.1 Automation Not Firing

**Symptom:** You change the Deployment Status to "Deployed" but the Deployment Date never populates.

**Solutions:**
1. **Verify automation is enabled:** Go to Automations → check that the toggle next to your automation is green/On
2. **Check the trigger field:** Open the automation editor → Trigger configuration → confirm "Deployment Status" is the watched field
3. **Check the condition:** Confirm the condition is: Deployment Status `is` `Deployed` (exact spelling, exact case)
4. **Wait longer:** Automations can take up to 60 seconds to fire on the free plan
5. **Check run history:** In the automation editor, click "Run history" — look for failed runs and read the error message

---

### 17.2 Deployment Date Not Updating

**Symptom:** The automation fires (shows in run history as "Success") but the Deployment Date field remains blank.

**Solutions:**
1. **Check the action field mapping:** Open the automation → Action configuration → verify the field being updated is "Deployment Date" (not a different date field)
2. **Check the value:** Verify the value is set to the dynamic "Current time" value and not left blank
3. **Check the record ID mapping:** Verify the Record ID in the action is set to pull from the trigger record (not hardcoded to a specific record)
4. **Manually check the field type:** Confirm that Deployment Date is a Date field type, not a text field

---

### 17.3 Wrong Field Mappings After CSV Import

**Symptom:** After importing the CSV, some records show values in the wrong columns (e.g., Serial Numbers appear in the Assigned User column).

**Solutions:**
1. Undo the import immediately with Ctrl+Z / Cmd+Z
2. If undo is unavailable, select all imported records, right-click, and choose "Delete records"
3. Re-import the CSV — in the mapping screen, carefully verify each column mapping before clicking "Import"
4. Cross-reference with the mapping table in Section 10.3 of this guide

---

### 17.4 Kanban Not Grouping Properly

**Symptom:** The Kanban view shows one big column instead of four status columns, or records appear in the wrong column.

**Solutions:**
1. **Check the "Stack by" setting:** Click the Kanban view → look for "Group by" or "Stack by" in the toolbar → confirm it is set to "Deployment Status"
2. **Check for blank status values:** If any records have a blank Deployment Status, they will appear in an unlabeled column. Open those records and assign a status
3. **Verify Single Select options:** Confirm that the Deployment Status field has exactly the four options listed in this guide. Extra or misspelled options will create extra columns

---

### 17.5 Formula Errors in Asset ID

**Symptom:** The Asset ID field shows an error message (red text) instead of NB-1, NB-2, etc.

**Solutions:**
1. Click on the Asset ID column header and select "Edit field"
2. Check the formula — it must be exactly: `"NB-" & {Asset Number}`
3. Common issues:
   - Curly braces around the wrong field name
   - Quotation marks are "smart quotes" (from copy-pasting) instead of straight quotes — delete and retype
   - The Autonumber field was renamed — update the formula to match the current name
4. After correcting, click "Save field" and verify the preview shows NB-1

---

### 17.6 CSV Import Problems

**Symptom:** CSV import fails, shows an error, or imports 0 records.

**Solutions:**
1. **Check file encoding:** Ensure the CSV is saved as UTF-8. Open in a text editor and save as UTF-8
2. **Check for special characters:** The MacBook Pro records in the CSV contain `"` (double quote inside the device model name). If the import fails, try opening the CSV in Excel, cleaning those cells, and re-saving
3. **Check for blank rows:** Remove any completely blank rows from the CSV before importing
4. **Use the correct Airtable import button:** In some versions, the import option is found via the "..." menu next to the table tab, under "Import data"

---

### 17.7 Missing Single Select Options

**Symptom:** After importing the CSV, some Deployment Status or Technician values appear as plain text (not colored badges) or the import creates extra unwanted options.

**Solutions:**
1. If new options were auto-created by the import, go to the field configuration → remove the unwanted options
2. If values appear as plain text without a matching option, it means the CSV value didn't match any existing option. Check for typos or spacing differences between the CSV values and your configured options
3. Manually update affected records by clicking the field and selecting the correct option from the dropdown

---

### 17.8 Record Sorting Issues

**Symptom:** Records in the Master Grid View are not sorted by Asset ID.

**Solutions:**
1. With the Master Grid View active, click the "Sort" button in the toolbar
2. Verify the sort rule is: Asset ID → A → Z (ascending)
3. If the sort rule is missing, click "+ Add a sort" and configure it
4. If the sort order looks wrong (NB-10 appears before NB-2), this is because the sort is treating Asset ID as text, not a number. To fix: add a secondary sort by "Asset Number" (Autonumber) ascending to get true numeric ordering

---

## 18. Best Practices

### 18.1 Naming Conventions

- **Asset IDs:** Always use the NB- prefix format. Do not manually assign Asset IDs — always let the Autonumber + Formula system generate them
- **Assigned User:** Always use "First Last" format. This ensures consistent alphabetical sorting and avoids duplicates when two users share a first name
- **Serial Numbers:** Use the exact serial number as printed on the device. Do not modify, abbreviate, or pad with zeros
- **Field names:** If you add custom fields in the future, use clear, descriptive names without abbreviations

---

### 18.2 Status Management

- **Only advance statuses — never go backward** in normal operations. If a device must be returned to a previous stage (e.g., an imaging failure), add a note to the record before changing the status
- **Set status to "Deployed" only after physical handoff** — not when the device is ready for delivery
- **Review the Kanban board daily** during active deployment to catch bottlenecks early
- **Archive completed deployments:** At the end of the project, filter for all "Deployed" records and move them to a new view called "Archived — Deployed" to keep the active Kanban clean

---

### 18.3 Technician Ownership

- **One technician per device** — do not split responsibility between two technicians for the same device
- **Review workload balance weekly** — use the Tech Gallery view to ensure no single technician is overloaded
- **Document technician handoffs** — if a device changes hands between technicians, update the Technician field and add a note with the reason

---

### 18.4 Data Cleanliness

- **Validate data on entry** — spot-check newly entered records immediately after creation
- **Do not use abbreviations** in free-text fields like Assigned User or Serial Number
- **Delete test records** before sharing the base with stakeholders — test records pollute reporting
- **Use consistent device model names** — the Single Select field enforces this for Device Model, but the model variants in the Serial Number or notes fields should also follow a consistent convention

---

### 18.5 Device Lifecycle Tracking

- **Keep all records, even after deployment** — historical records provide accountability and support warranty claims
- **Track return/replacements** — if a device is returned, add a "Returned" status option and update the record (do not delete it)
- **Document damage or defects** in a Notes or Condition field (add this as a future enhancement)

---

### 18.6 Audit Logging

- The **Last Updated** field provides a basic audit trail showing when each record was last modified
- For more comprehensive auditing, Airtable Pro plan users can enable **Revision history**, which shows a full changelog of every edit made to each record
- Export the base to CSV monthly as an offline backup of the audit trail

---

### 18.7 Scaling Beyond 50 Devices

- The current architecture can scale to Airtable's record limit (1,000 on free, 50,000+ on paid plans) without any structural changes
- For 100+ devices, consider:
  - Adding a **"Location/Floor"** field to track physical staging areas
  - Adding a **"Batch"** field to group deliveries into waves
  - Creating filtered views per technician so each person only sees their own devices

---

## 19. Scalability Recommendations

As the Nexus Blue practice grows and project sizes increase, consider these architectural upgrades:

### 19.1 Add a Clients Table

Create a **Clients** table with one record per client company. Link the Deployment Inventory to the Clients table so all devices for a given client can be filtered and reported independently.

```
Clients Table
├── Client Name
├── Industry
├── Primary Contact
├── Total Devices Ordered
└── Project Start Date
```

### 19.2 Add a Devices / Asset Register Table

Create a permanent **Device Register** that persists beyond individual projects. Each physical device (identified by serial number) has one record that tracks its full lifetime — every project it has been assigned to, every reimaging, and its current location.

### 19.3 Multi-Project Support

The current base is scoped to one project. For managing multiple simultaneous deployments:
- Add a **"Project"** field (Linked Record to a Projects table)
- Filter all views by the current project
- Use the Projects table to aggregate metrics across all active deployments

### 19.4 Role-Based Access Control

On Airtable Pro or Business plans, use **"Shared views"** with specific field visibility restrictions to give technicians access to only their relevant columns, preventing accidental edits to sensitive fields.

### 19.5 API Integration

Airtable provides a REST API that allows external systems to read and write records. This enables:
- Automatic record creation when a purchase order is received from a procurement system
- Status updates pushed from a mobile scanning app
- Nightly exports to a business intelligence tool like Tableau or Power BI

---

## 20. Future Improvements

### 20.1 Barcode / QR Asset Labels

Generate QR codes for each Asset ID and print them as labels to affix to devices. Scanning the QR code with a mobile device opens the Airtable record directly. This enables:
- Instant device-to-record lookup without searching
- Faster physical audits
- Reduced mis-tagging errors

**Recommended tools:** Avery Design & Print, QR Code Monkey, Dymo label printers

---

### 20.2 SLA Tracking

Add a **"Target Delivery Date"** field and a formula field called **"Days to Deadline"** that calculates how many days remain until the target date:

```
DATETIME_DIFF({Target Delivery Date}, TODAY(), 'days')
```

Add a formula field called **"SLA Status"** that flags at-risk devices:

```
IF({Days to Deadline} < 2, "🔴 At Risk", IF({Days to Deadline} < 5, "🟡 Warning", "🟢 On Track"))
```

---

### 20.3 Email Notification Automations

Add an additional automation that sends an email to the Assigned User when their device's status changes to "Ready for Delivery." This notifies employees that their new laptop is ready for pickup without requiring manual outreach from the project manager.

**In Airtable Automations:**
- **Trigger:** When Deployment Status changes to "Ready for Delivery"
- **Action:** Send email
- **To:** {Assigned User Email} (requires adding an Email field to the Assigned User or linking to an HR directory)
- **Subject:** "Your new laptop is ready for pickup"
- **Body:** Personalized message with device details

---

### 20.4 Slack Integration

Connect Airtable to Slack (via native integration or Zapier/n8n) to post a message to a deployment channel every time a device is deployed. This gives the team real-time progress visibility without opening Airtable.

**Example Slack message:**
```
✅ NB-23 (Dell Latitude) has been deployed to Harvey Specter.
Technician: Marcus | Date: May 9, 2025
```

---

### 20.5 Device Warranty Tracking

Add the following fields for lifecycle management:

| Field | Type | Purpose |
|---|---|---|
| Purchase Date | Date | When the device was procured |
| Warranty Expiry Date | Date | Manufacturer warranty end date |
| Days Until Warranty Expires | Formula | `DATETIME_DIFF({Warranty Expiry Date}, TODAY(), 'days')` |
| Warranty Status | Formula | Flag if warranty expires within 90 days |

---

### 20.6 Client Portal via Shared View

Create a **"Client Progress View"** — a shared view that shows only high-level deployment progress (no serial numbers or internal notes) that can be shared with the client's IT manager as a read-only link. This eliminates the need for manual status email updates.

---

### 20.7 Inventory Aging Dashboard

Use Airtable's **Summary Bar** (click the Σ icon at the bottom of the grid) to display:
- Count of devices in each status
- Average days from Unboxed to Deployed
- Number of devices per technician

Combine with Airtable's **Gantt view** (Pro plan) to visualize the deployment timeline across all 50 units.

---

### 20.8 Integration with Jira

If your team uses Jira for project management, use the **Airtable Jira integration** (or n8n/Zapier) to:
- Automatically create a Jira ticket when a new device is added to Airtable
- Sync Jira ticket status with Airtable Deployment Status
- Link Airtable records to Jira epics for full project traceability

---

### 20.9 n8n Workflow Automation

[n8n](https://n8n.io) is an open-source workflow automation tool that can create powerful multi-step workflows connecting Airtable to any other system:
- When a device is Deployed in Airtable → update the asset register in Snipe-IT
- When a device is returned → create a refurbishment ticket in your ticketing system
- Nightly: export all "Deployed" records to a Google Sheet for client reporting

---

*End of Document*

---

**Document prepared for:** Nexus Blue Managed Services — IT Hardware Refresh Program  
**Base name:** Nexus Blue — Hardware Refresh Tracker  
**Implementation guide version:** 1.0  
**Platform:** Airtable  

---

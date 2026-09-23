# HubSpot Data Hygiene Auditor (n8n template)

A scheduled n8n workflow that audits contacts and companies in HubSpot for missing or invalid data, scores each record, writes the score back to HubSpot, and logs a report to Google Sheets.

Built and tested end-to-end by Anat Gilaad against a live HubSpot free CRM instance and self-hosted n8n — free to import and adapt.

## What it does

Every week (configurable), the workflow:
1. Pulls all contacts and companies from HubSpot
2. Checks each one against three hygiene rules
3. Writes a `data_quality_score` (0–100) and `data_quality_issues` list back onto the record in HubSpot
4. Logs every scored record, plus a run summary, to a Google Sheet

**Contacts are checked for:** missing or invalid email, missing phone, missing lifecycle stage.
**Companies are checked for:** missing domain, missing industry, missing phone.

Each missing item costs ~33 points, so a record missing all three checks scores 0, and a clean record scores 100.

## A note on HubSpot API versions

This template targets HubSpot's **older/legacy Contacts and Companies API shape** (contacts identified by `vid`, companies by `companyId`, properties nested as `properties.<field>.value`), because that's what n8n's HubSpot node returns by default in current versions. If your n8n version returns the newer flat `id` / `properties.email` shape instead, the two Code nodes (Score Contacts, Score Companies) will need small adjustments — the scoring logic itself doesn't change, just how the raw fields are read.

Contacts are written back using HubSpot's **email-based upsert** ("Create or Update"), since this n8n version doesn't expose a plain "update by ID" option for contacts. Companies are updated directly **by ID**, since we already have it from the initial fetch.

## Setup

### 1. Create two custom properties in HubSpot
Go to **Settings → Properties** and create these for **both** Contacts and Companies:

| Property name | Type |
|---|---|
| `data_quality_score` | Number |
| `data_quality_issues` | Single-line text |

Double-check the exact property name after creating it — HubSpot occasionally auto-adjusts what you type, and a mismatch here will make the workflow silently fail to attach scores.

### 2. Get a HubSpot access token
In HubSpot, go to **Settings → Integrations → Private Apps** (also labeled "Service Keys" on some accounts — HubSpot is mid-transition on this naming). Create one, and under scopes enable read/write for both `crm.objects.contacts` and `crm.objects.companies`. Copy the key — HubSpot only shows it in full once.

### 3. Create your Google Sheet
Create a Google Sheet with three tabs: `Contacts`, `Companies`, `Summary`.

Add header rows matching the fields the workflow writes:
- `Contacts` tab: `id`, `email`, `name`, `recordType`, `score`, `issuesText`
- `Companies` tab: `id`, `domain`, `name`, `recordType`, `score`, `issuesText`
- `Summary` tab: `date`, `totalRecords`, `flagged`, `critical`, `averageScore`

Copy the sheet's ID from its URL (the long string between `/d/` and `/edit`).

### 4. Import the workflow
In n8n: **Workflows → Import from File** → select `hubspot-data-hygiene-auditor.json`.

### 5. Connect your credentials
Open the **Get Contacts**, **Get Companies**, **Update Contact Score**, and **Update Company Score** nodes and attach a HubSpot credential using your access token from step 2. Depending on your n8n version, this auth method may be labeled "Private App Token," "App Token," or "Service Key" — they're the same thing.

Open the three Google Sheets nodes and attach your Google Sheets credential, then replace `PASTE_YOUR_GOOGLE_SHEET_ID_HERE` in each node's Document field with your sheet's ID from step 3.

### 6. Sanity-check node field names after import
n8n node UIs and API response shapes change between versions. After importing:
- Open **Get Contacts** / **Get Companies** and confirm the properties-to-fetch list is populated (email, phone, lifecyclestage, firstname, lastname for contacts; domain, industry, phone, name for companies). If empty, re-add them.
- Open the **Score Contacts** / **Score Companies** Code nodes and run one item through manually (use n8n's "Execute step" on the upstream Get node first, then this node) to confirm `id`, `name`, `score`, and `issuesText` populate correctly. If they come out blank, your HubSpot API response shape differs from what this template expects — see the note above.
- Open each **Google Sheets** node and confirm "Mapping Column Mode" is set to **Map Automatically** — some n8n versions default this to manual and won't run until it's switched.

### 7. Test it
Click **Execute Workflow** to run it manually before trusting the schedule. Check that:
- Scores appear on a few contacts/companies in HubSpot
- Rows appear in your Google Sheet across all three tabs
- The Summary tab's `totalRecords` equals your contact count plus your company count (if it looks too low, your Merge node may be set to pair records by position instead of appending both lists — check its mode)

## How the scoring logic works

The logic lives in the **Score Contacts** and **Score Companies** Code nodes — plain JavaScript, easy to extend. Want to check for more fields (e.g. missing job title, missing LinkedIn URL)? Add a check to the `issues` array and adjust the point deduction (currently `100 / 3` per issue, per object type).

## License

Free to use, copy, and adapt.

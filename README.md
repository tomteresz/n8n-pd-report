# 📟 n8n PagerDuty Weekly Report

Automated **n8n** workflow that fetches PagerDuty incidents from the last 7 days
across multiple services and teams, processes the data, generates a ranked
**Top 20 alerts** report, and delivers it every Monday morning as an
**HTML email with a CSV attachment** via Gmail.

---

## 📋 Overview

Keeping track of recurring incidents across multiple PagerDuty services manually
is time-consuming. This workflow automates the entire weekly reporting cycle -
from pulling raw incident data via the PagerDuty API, through aggregation and
ranking, to generating multi-format reports (CSV, HTML table, Markdown) and
sending them directly to the team's inbox.

---

## ⚙️ How It Works

The workflow is organised into 6 logical stages:

### 1. 🗓️ Setup Initial Config

Two triggers are available - a **manual trigger** for on-demand runs and a
**Schedule Trigger** (cron: every Monday at 06:00) for automated weekly
delivery. A `Last7days` node dynamically calculates the `since` and `until`
date range using the current timestamp, so no manual date adjustments are
ever needed.

### 2. 📥 Pull Data from PagerDuty

Two parallel HTTP requests fetch incidents from the PagerDuty REST API v2
(`/incidents`) for different services and teams:

- **On Call - Data Center** - incidents for the Data Center service and teams
- **Uptime Synthetic Transaction** - incidents for the Synthetic Monitoring service

Both requests use header-based authentication (`PD-by-header` credentials),
support **automatic pagination** (offset-based, up to 100 records per page),
and filter by `service_ids` and `team_ids` for each data source.

### 3. 🔀 Prepare Data from Different Services

Each data stream is aggregated independently and then merged into a single
unified dataset using the `Merge-PD-Services` node, combining incidents from
both services into one collection for further processing.

### 4. 🔍 Extract from Prepared Data

A JavaScript code node (`Get-Incidents-details`) loops through the merged
dataset and extracts the relevant fields for each incident:
`ID`, `Incident Number`, `Title`, `Service`, `Created`, `Resolved`,
`Status`, and `Incident URL`. A second node (`Extract-id-and-title`)
further narrows the fields down to `Incident ID` and `Incident Title`
for the ranking step.

### 5. 📊 Prepare the Report

Three parallel output formats are generated simultaneously:

- **CSV file** (`incidents-list.csv`) - full incident list for the email
  attachment, containing all extracted fields.
- **HTML table** - the **Top 20 most frequent alert titles** ranked by
  occurrence count, styled with inline CSS and embedded directly in the
  email body.
- **Markdown file** (`incidents-list-markdown`) - the same HTML table
  converted to Markdown format, useful for posting in wikis or Slack
  messages.

The CSV data and HTML table are then merged together to prepare the final
email payload.

### 6. 📧 Send the Report

A **Gmail** node sends the weekly report email with:

- **Subject:** `PD Trends for Week {week_number}` (dynamically generated)
- **Body:** greeting + top 20 HTML alert table embedded inline
- **Attachment:** full `incidents-list.csv` file

---

## 🛠️ Prerequisites

- **n8n** instance (self-hosted or cloud)
- **PagerDuty API token** configured as an HTTP Header credential
  (`Authorization: Token token=<your-api-key>`)
- **Gmail OAuth2** credential configured in n8n
- Target PagerDuty **Service IDs** and **Team IDs** updated in the HTTP
  request nodes

---

## 🔧 Configuration

### 1. 🔑 PagerDuty Credentials

In n8n, create a new **Header Auth** credential named `PD-by-header`.
Set the header name to `Authorization` and the value to
`Token token=<your-pagerduty-api-key>`. Update the `service_ids[]` and
`team_ids[]` query parameters in both HTTP request nodes to match your
PagerDuty environment.

### 2. 📬 Gmail Credentials

Create a **Gmail OAuth2** credential in n8n and authorize it with the
Google account that will send the weekly reports. Update the `sendTo`
field in the `Prepare-and-send-an-email` node with the recipient email
address.

### 3. 🗓️ Schedule

The Schedule Trigger is set to run every **Monday at 06:00** (cron:
`0 6 * * 1`). Enable the trigger node and activate the workflow when
ready for production. Until then, use the **Run-manually** trigger for
testing.

---

## 📤 Output Formats

| Format | Node | Purpose |
|---|---|---|
| 📎 CSV | `Create-csv-file` | Full incident list as email attachment |
| 🌐 HTML Table | `Create-html-table` | Top 20 alerts embedded in email body |
| 📝 Markdown | `Markdown-file` | Portable format for wikis or chat tools |

---

## ✅ Pre-run Checklist

- [ ] 🔑 PagerDuty Header Auth credential created and assigned to both HTTP nodes
- [ ] 🔍 `service_ids[]` and `team_ids[]` updated in both HTTP request nodes
- [ ] 📬 Gmail OAuth2 credential authorized and assigned
- [ ] 📧 Recipient email address updated in `Prepare-and-send-an-email`
- [ ] 🗓️ Schedule Trigger enabled and workflow activated for automated runs
- [ ] 🧪 Manual test run completed successfully before going live

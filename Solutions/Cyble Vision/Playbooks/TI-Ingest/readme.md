# Cyble Threat Intelligence Ingest Playbook

![Cyble](https://raw.githubusercontent.com/Azure/Azure-Sentinel/master/Logos/CybleLogo.svg)

## Summary

This playbook imports Indicators of Compromise (IoCs) from the Cyble Vision Threat Intelligence API and stores them as Threat Intelligence indicators in Microsoft Sentinel for detection.

On a recurring schedule (default: every 60 minutes), the playbook:

1. Builds a time window from `now - schedule interval` to `now`.
2. Calls the Cyble IOC API (`GET https://api.cyble.ai/engine/api/v2/y/iocs`) for IoCs last seen in that window, sorted by `last_seen` descending, 100 IoCs per page.
3. Pages through the remaining results until all IoCs in the window are collected.
4. Queries the `ThreatIntelIndicators` table for the last 7 days and skips IoCs that already exist (de-duplication).
5. Skips IoCs whose `last_seen` timestamp is outside the current time window.
6. Uploads new IoCs to Microsoft Sentinel using **Upload Indicators v2**, with `sourcesystem` set to `CybleVision`.

Each uploaded indicator includes:

| Cyble field | Sentinel field |
| --- | --- |
| `ioc` | Name, display name, and indicator value |
| `ioc_type` | Indicator type and STIX pattern (`ipv4address`, `filehash-sha256`, or `url`) |
| `risk_score` | Confidence and severity (mapped 1–5) |
| `confidence_rating` | Tag |
| `behaviour_tags` | Labels |
| Fixed values | Threat type `malicious-activity`, description `IOC from Cyble`, expiration **7 days** |

Risk score mapping:

- ≥ 80 → 5
- ≥ 60 → 4
- ≥ 40 → 3
- ≥ 20 → 2
- else → 1

Supported Cyble `ioc_type` values are **IPv4**, **FileHash-SHA256**, and **URL**. Any other type is uploaded with a URL STIX pattern.

## Pre-requisites for deployment

1. A Microsoft Sentinel workspace with [Threat Intelligence](https://learn.microsoft.com/azure/sentinel/understand-threat-intelligence) enabled. The playbook writes indicators into this workspace and reads `ThreatIntelIndicators` for de-duplication.
2. A valid Cyble Vision API bearer token with access to the Threat Intelligence IOC API. Generate an access key from the Cyble Vision portal at [https://cyble.ai](https://cyble.ai) (**Utilities → Access API**), or request a token from your Cyble Customer Success Manager. Treat the token as a secret.
3. Permission to deploy Logic Apps and API connections in the target subscription and resource group.
4. The Log Analytics workspace name and workspace ID of the Microsoft Sentinel workspace.

## Deployment Instructions

1. Deploy the playbook by clicking the **Deploy to Azure** button. This opens the ARM Template wizard.

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FAzure%2FAzure-Sentinel%2Fmaster%2FSolutions%2FCyble%2520Vision%2FPlaybooks%2FTI-Ingest%2Fazuredeploy.json)
[![Deploy to Azure Gov](https://aka.ms/deploytoazuregovbutton)](https://portal.azure.us/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FAzure%2FAzure-Sentinel%2Fmaster%2FSolutions%2FCyble%2520Vision%2FPlaybooks%2FTI-Ingest%2Fazuredeploy.json)

2. Fill in the required parameters for deploying the playbook.

| Parameter | Description |
| --- | --- |
| **Playbook Name** | Logic App name, without spaces. Default: `Cyble-TI-Ingest-Playbook` |
| **Schedule Frequency In Minutes** | How often the playbook polls Cyble for new IoCs. Default: `60` |
| **Cyble Api Bearer Token** | Cyble Vision API bearer token used in the `Authorization: Bearer <token>` header |
| **Workspace ID** | Microsoft Sentinel / Log Analytics workspace ID used by Upload Indicators v2 |
| **log Analytics Workspace Name** | Log Analytics workspace name used to query existing indicators for de-duplication |

# Post-Deployment Instructions

## a. Authorize API connections

After deployment completes, authorize both API connections created with the playbook.

### Microsoft Sentinel connection

1. Open the Logic App, then open **API connections**.
2. Select the Microsoft Sentinel (`azuresentinel`) connection.
3. Click **Edit API connection**.
4. Click **Authorize**, sign in with an account that can manage Threat Intelligence in the workspace, and click **Save**.

### Azure Monitor Logs connection

1. Select the Azure Monitor Logs (`azuremonitorlogs`) connection.
2. Click **Edit API connection**.
3. Click **Authorize**, sign in with an account that can query the Log Analytics workspace, and click **Save**.

## b. Assign roles to the playbook managed identity

The Logic App uses a system-assigned managed identity.

1. Open the playbook (Logic App) → **Identity** → **System assigned**.
2. Click **Azure role assignments** → **Add role assignment**.
3. Grant these roles on the Microsoft Sentinel workspace (or its resource group):
   - **Microsoft Sentinel Contributor** — required to upload Threat Intelligence indicators.
   - **Log Analytics Reader** — required to query `ThreatIntelIndicators` for de-duplication.
4. Click **Save**. Role assignments can take a few minutes to apply.

## c. Configurations in Sentinel

- Confirm Threat Intelligence is enabled on the workspace so uploaded indicators appear in **Threat Intelligence** and the `ThreatIntelIndicators` table.
- Keep the Logic App **Enabled**. The recurrence trigger starts automatically using **Schedule Frequency In Minutes**.
- Optionally create analytics rules that match Cyble indicators (`sourcesystem == "CybleVision"`) against your telemetry.

# Playbook steps explained

## Recurrence trigger

Runs on the configured interval (default 60 minutes) and starts an ingest cycle.

## Initialize StartDateTimeFull / EndDateTimeFull

Sets the Cyble query window to the last schedule interval, in UTC (`yyyy-MM-ddTHH:mm:ssZ`).

## HTTP Get IOC Data

Calls Cyble `GET /engine/api/v2/y/iocs` with `sortBy=last_seen`, `order=desc`, `limit=100`, and the start/end dates. The first response is used to calculate total pages from `data.pagination.total_count`.

## Get Iocs Per Page

For each page:

1. Fetches that page of IoCs from Cyble.
2. Appends each IoC to `rawIocs`.
3. Builds a KQL query against `ThreatIntelIndicators` for the collected values.
4. Appends matching `ObservableValue` results to `existingIocs` (last 7 days).

## For Each IOC

For every collected IoC:

- If the value is already in `existingIocs`, increment `skippedCounter` and do not upload.
- If `last_seen` is outside the current start/end window, skip the upload.
- Otherwise, upload the IoC with **Upload Indicators v2**, including STIX pattern, confidence, severity, tags, labels, and a 7-day expiration. Retries use exponential backoff (5 attempts, 15-second interval).

## Usage and triggering

This playbook is **schedule-triggered**. It does not run from an incident.

- **Automatic:** enabled by default after deployment. Change the interval with **Schedule Frequency In Minutes**, or edit the recurrence trigger in the Logic App designer.
- **Manual:** open the Logic App and click **Run Trigger** to ingest the latest window immediately.
- Use this playbook to keep Sentinel Threat Intelligence current. For per-incident Cyble context, use [Cyble IOC Enrichment](../IoC-Enrichment/readme.md).

## Troubleshooting

| Issue | What to check |
| --- | --- |
| Logic App run fails on `HTTP_Get_IOC_Data` with 401/403 | The Cyble API token is missing, expired, or not entitled for the IOC API. Regenerate the key in Cyble Vision and update the **Cyble Api Bearer Token** parameter. |
| No indicators appear in Sentinel | Confirm the Microsoft Sentinel connection is authorized, the managed identity has **Microsoft Sentinel Contributor**, and **Workspace ID** is correct. Open a successful run and inspect **Upload_Indicators_v2**. |
| Every IoC is skipped | De-duplication found the values in `ThreatIntelIndicators` for the last 7 days, or `last_seen` is outside the schedule window. Check `skippedCounter` in the run history. |
| Azure Monitor Logs action fails | Authorize the Azure Monitor Logs connection and confirm **log Analytics Workspace Name** matches the Sentinel workspace. The identity or signed-in user needs **Log Analytics Reader**. |
| Only some IoC types look correct | The playbook maps `IPv4` and `FileHash-SHA256` explicitly. Other Cyble types are uploaded as URL patterns. |
| Run times out or is slow | Large windows increase paging. Lower **Schedule Frequency In Minutes** so each run fetches a smaller set. |
| Indicators expire quickly | Uploaded indicators expire 7 days after ingest. Keep the schedule running so still-active IoCs are refreshed. |

Check run history on the Logic App for action-level inputs, outputs, and HTTP status codes. Enable Logic App diagnostic logs if you need a longer audit trail.

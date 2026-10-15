# Cyble IOC Enrichment Playbook

![Cyble](https://raw.githubusercontent.com/Azure/Azure-Sentinel/master/Logos/CybleLogo.svg)

## Summary

When a Microsoft Sentinel incident is created, this playbook enriches IP, host, URL, file hash, and mailbox entities with Cyble Vision Threat Intelligence and posts the results as an incident comment.

The playbook:

1. Triggers on **Microsoft Sentinel incident creation**.
2. Reads the incident and its related entities.
3. For each supported entity, queries the Cyble IOC API (`GET https://api.cyble.ai/engine/api/v2/y/iocs?ioc=<value>`).
4. Formats risk score, confidence, sources, malware, threat actors, behavior tags, first/last seen, and whitelist status.
5. Posts a single incident comment titled **IOC Enrichment Results**. If Cyble returns no data for an entity, the comment includes `No IOCs found.` If there is nothing to post, the comment is skipped.

This playbook does **not** ingest a Threat Intelligence feed. It only adds analyst context to the incident. For scheduled IOC ingest into Sentinel Threat Intelligence, use [Cyble Threat Intelligence Ingest](../TI-Ingest/readme.md).

Supported entity kinds and the Cyble lookup value:

| Incident entity kind | Value sent to Cyble |
| --- | --- |
| `ip` | `properties.address` |
| `host` | `properties.hostName` |
| `url` | `properties.url` |
| `filehash` | `properties.hashValue` |
| `mailbox` | `properties.mailboxPrimaryAddress` |

## Pre-requisites for deployment

1. A Microsoft Sentinel workspace that creates incidents with at least one of these entities: IP, host, URL, file hash, or mailbox.
2. A valid Cyble Vision API bearer token with access to the Threat Intelligence IOC API. Generate an access key from the Cyble Vision portal at [https://cyble.ai](https://cyble.ai) (**Utilities → Access API**), or request a token from your Cyble Customer Success Manager. Treat the token as a secret.
3. Permission to deploy Logic Apps and API connections in the target subscription and resource group.
4. An account that can authorize the Microsoft Sentinel Logic App connection and create automation rules.

## Deployment Instructions

1. Deploy the playbook by clicking the **Deploy to Azure** button. This opens the ARM Template wizard.

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FAzure%2FAzure-Sentinel%2Fmaster%2FSolutions%2FCyble%2520Vision%2FPlaybooks%2FIoC-Enrichment%2Fazuredeploy.json)
[![Deploy to Azure Gov](https://aka.ms/deploytoazuregovbutton)](https://portal.azure.us/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FAzure%2FAzure-Sentinel%2Fmaster%2FSolutions%2FCyble%2520Vision%2FPlaybooks%2FIoC-Enrichment%2Fazuredeploy.json)

2. Fill in the required parameters for deploying the playbook.

| Parameter | Description |
| --- | --- |
| **Playbook Name** | Logic App name, without spaces. Default: `Cyble-IOC-Enrichment-Playbook` |
| **Schedule Frequency In Minutes** | Reserved deployment parameter. This playbook is incident-triggered and does not use a recurrence schedule. Default: `60` |
| **Cyble Api Bearer Token** | Cyble Vision API bearer token used in the `Authorization: Bearer <token>` header |
| **Workspace ID** | Microsoft Sentinel workspace ID (required by the deployment template) |

# Post-Deployment Instructions

## a. Authorize API connection

After deployment completes, authorize the Microsoft Sentinel connection.

1. Open the Logic App, then open **API connections**.
2. Select the Microsoft Sentinel (`azuresentinel`) connection.
3. Click **Edit API connection**.
4. Click **Authorize**.
5. Sign in with an account that can read incidents and add comments.
6. Click **Save**.

## b. Assign roles to the playbook managed identity

The Logic App uses a system-assigned managed identity.

1. Open the playbook (Logic App) → **Identity** → **System assigned**.
2. Click **Azure role assignments** → **Add role assignment**.
3. Grant **Microsoft Sentinel Responder** on the Microsoft Sentinel workspace (or its resource group). This is required to add incident comments.
4. Click **Save**. Role assignments can take a few minutes to apply.

## c. Configurations in Sentinel

- Analytics rules (or other detections) must create incidents that include IP, host, URL, file hash, or mailbox entities. Entities of other kinds are queried with an empty IOC value and will not return useful Cyble data.
- Create an automation rule so the playbook runs when an incident is created:
  1. In Microsoft Sentinel, go to **Automation** → **Create** → **Automation rule**.
  2. Set the trigger to **When incident is created**.
  3. Add the action **Run playbook** and select `Cyble-IOC-Enrichment-Playbook` (or the name you deployed).
  4. Optionally filter by analytics rule, severity, or entity type so only relevant incidents are enriched.
- You can also run the playbook manually from an incident: **Actions** → **Run playbook**.

# Playbook steps explained

## When Microsoft Sentinel incident is created

The incident-creation webhook starts the playbook and passes the full incident object.

## Parse Full Incident

Reads the incident payload, including `id` (used later as `incidentArmId`) and `Properties.relatedEntities`.

## Parse Entities

Parses `relatedEntities` into an array of entity objects.

## Initialize Enrichment Array

Creates an empty `FormattedComments` array to collect one comment block per entity.

## For each Entity

For every related entity:

1. **HTTP Get IOC Data** — looks up the entity value in Cyble (`?ioc=<value>`).
2. **Parse IOC Response** — reads `success` and the first object in `data.iocs`.
3. **Compose Formatted Comment** — builds a markdown block with:
   - Entity
   - IOC and type
   - Risk score
   - Confidence rating
   - Sources
   - Related malware
   - Related threat actors
   - Behavior tags
   - First seen / last seen
   - Is whitelisted
4. If the API call fails or returns no IoCs, the block records `No IOCs found.` or `API returned an error or no data available.`
5. **Append Comment to Array** — adds the block to `FormattedComments`.

## Check If Comments Not Empty

- If `FormattedComments` has at least one entry, **Add comment to incident (V3)** posts:

  `**IOC Enrichment Results**` plus every entity block.
- If the array is empty, the playbook skips the comment (`No enriched IOCs found; skipping comment.`).

## Usage and triggering

This playbook is **incident-triggered**.

- **Automatic:** attach it to a Sentinel automation rule with trigger **When incident is created**.
- **Manual:** open an incident and choose **Run playbook**.
- Best results come from incidents that already have IP, host, URL, hash, or mailbox entities mapped.

## Troubleshooting

| Issue | What to check |
| --- | --- |
| Playbook never runs | Confirm the Microsoft Sentinel connection is authorized and an automation rule (or a manual run) is configured for this Logic App. |
| Comment is not added | Grant **Microsoft Sentinel Responder** to the Logic App managed identity. Inspect **Add_comment_to_incident_(V3)** in run history. |
| Comment says `No IOCs found.` | Cyble has no record for that entity, or the entity kind/property was empty. Confirm the incident entity mapping (IP address, URL, hash, host name, or mailbox). |
| HTTP action returns 401/403 | The Cyble API token is missing, expired, or not entitled for the IOC API. Regenerate the key in Cyble Vision and update **Cyble Api Bearer Token**. |
| Unsupported entities produce empty lookups | Only `ip`, `host`, `url`, `filehash`, and `mailbox` are mapped. Other entity kinds send an empty `ioc` query parameter. |
| Enrichment runs on every incident | Narrow the automation rule with analytics-rule, severity, or entity filters. |

Check run history on the Logic App for action-level inputs, outputs, and HTTP status codes. Enable Logic App diagnostic logs if you need a longer audit trail.

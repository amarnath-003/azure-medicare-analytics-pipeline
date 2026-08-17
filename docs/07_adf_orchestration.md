# Day 7–8 — ADF Orchestration & Schedule Trigger

## Objective
Connect ADF to Databricks so that one ADF pipeline run triggers the full
Bronze → Silver → Gold transformation automatically. Add a monthly schedule trigger.

---

## Architecture

```
ADF Pipeline: pl_healthcare_full_pipeline
│
├── Activity 1: Copy Providers CSV → ADLS   (Copy Activity)
├── Activity 2: Copy Drugs CSV → ADLS       (Copy Activity)
├── Activity 3: Copy Inpatient CSV → ADLS   (Copy Activity)
│              ↓  (on success)
├── Activity 4: Run Bronze Notebook         (Databricks Notebook Activity)
│              ↓  (on success)
├── Activity 5: Run Silver Notebook         (Databricks Notebook Activity)
│              ↓  (on success)
└── Activity 6: Run Gold Notebook           (Databricks Notebook Activity)
```

---

## Step 1 — Create Databricks Access Token

1. Go to your **Databricks workspace**
2. Top right → click your profile icon → **Settings**
3. Left menu → **Developer** → **Access Tokens**
4. Click **Generate new token**
5. Name: `adf_databricks_token`
6. Lifetime: 90 days
7. Click **Generate**
8. **Copy the token immediately** — it won't show again
9. Save it temporarily (paste in Notepad)

---

## Step 2 — Create ADF Linked Service for Databricks

1. ADF Studio → **Manage** → **Linked Services** → **+ New**
2. Search: `Azure Databricks`
3. Click **Continue**
4. Fill in:
   - Name: `ls_databricks`
   - Databricks workspace URL: (copy from Databricks — looks like `https://adb-xxxx.azuredatabricks.net`)
   - Cluster: **Existing interactive cluster** → select your cluster
   - Access token: paste the token from Step 1
5. Click **Test Connection** → should show "Connection successful"
6. Click **Create**

---

## Step 3 — Add Notebook Activities to Pipeline

1. Open pipeline `pl_healthcare_full_load`
2. Drag **Databricks Notebook** activity onto canvas
3. Name: `run_bronze_notebook`
4. Settings tab:
   - Linked Service: `ls_databricks`
   - Notebook path: `/Users/your_email/01_bronze_ingestion`
     (copy path from Databricks notebook → File → Copy path)
5. Connect: Copy activities → Bronze notebook (on success arrow)

Repeat for Silver and Gold notebooks.

---

## Step 4 — Chain Activities in Order

Connect activities using the **green arrow (on success)**:

```
copy_providers ──┐
copy_drugs     ──┼──→ run_bronze_notebook
copy_inpatient ──┘         ↓ (on success)
                     run_silver_notebook
                           ↓ (on success)
                     run_gold_notebook
```

To connect:
1. Hover over an activity → green arrow appears on the right edge
2. Drag the green arrow to the next activity

---

## Step 5 — Validate and Publish

1. Click **Validate** — fix any errors
2. Click **Publish All**

---

## Step 6 — Test Full Pipeline Run

1. Click **Add Trigger** → **Trigger Now**
2. Go to **Monitor** tab
3. Watch all 6 activities complete
4. Expected total run time: 20–40 minutes (Bronze reads 2M rows, Silver joins, Gold aggregates)

---

## Step 7 — Create Schedule Trigger

1. ADF Studio → **Manage** → **Triggers** → **+ New**
2. Fill in:
   - Name: `tr_monthly_healthcare`
   - Type: **Schedule**
   - Start date: first day of next month
   - Time zone: your local time zone
   - Recurrence: **1 Month**
   - At: 06:00 AM
3. Click **Next** → select pipeline `pl_healthcare_full_pipeline`
4. Click **OK**
5. Click **Publish All**
6. Trigger should show status: **Started**

---

## Trigger Configuration Summary

| Setting | Value |
|---------|-------|
| Name | tr_monthly_healthcare |
| Type | Schedule |
| Frequency | Monthly |
| Day | 1st of each month |
| Time | 06:00 AM |
| Pipeline | pl_healthcare_full_pipeline |
| Status | Active |

---

## What This Demonstrates on Your Resume

```
"Built Azure Data Factory pipeline with Databricks Notebook Activity and monthly
schedule trigger, enabling zero-touch end-to-end automation from raw file ingestion
through Gold layer analytics."
```

---

## Status

- [x] Databricks access token generated (adf_databricks_token, all scopes, 90 days)
- [x] ADF Linked Service created: ls_databricks (Compute tab → Azure Databricks)
- [x] Notebook Activity added: run_bronze_notebook
- [x] Notebook Activity added: run_silver_notebook
- [x] Notebook Activity added: run_gold_notebook
- [x] Activities chained in correct order
- [ ] Full pipeline test run — blocked by Azure free account CPU quota (4 cores limit)
- [x] Schedule trigger created: tr_monthly_healthcare
- [x] Trigger activated — Status: Started

---

## Errors Encountered

| Error | Cause | Fix |
|---|---|---|
| Cluster configuration required | ADF Notebook Activity does not support serverless clusters | Changed linked service to New Job Cluster |
| Sorry, cannot run Cluster — exhausted credits | Databricks 14-day free DBU trial expired | Upgraded workspace from Trial to Premium Pay-As-You-Go |
| Upgrade workspace failed — AjaxError | Azure portal glitch | Retried in different browser — succeeded |
| CLOUD_PROVIDER_RESOURCE_STOCKOUT — Standard_DS3_v2 | VM not available in Central India at that moment | Changed node type to Standard_D4s_v3 |
| Spark version 12.2.x not supported | Workspace has legacy features disabled — needs Runtime 13.3+ | Changed cluster version to 13.3.x-photon-scala2.12 |
| AZURE_QUOTA_EXCEEDED — Total Regional Cores limit 4 | Free account CPU quota = 4 cores, D4s_v3 needs 8 | Attempted quota increase — still investigating |

## Notes

- ADF Notebook Activity requires a classic job cluster — serverless is not supported
- Job cluster billing uses Databricks DBUs — separate from Azure credits
- Upgrading to Premium Pay-As-You-Go routes DBU billing through Azure credits
- Azure free account CPU quota in Central India = 4 cores (blocks most VM sizes)
- Pipeline architecture is fully correct and validated — quota is an infrastructure constraint not a skill gap

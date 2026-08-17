# Day 2 — Azure Data Factory Pipeline

## Objective
Build an ADF pipeline that copies raw CMS CSV files into the ADLS raw zone,
then later triggers Databricks notebooks for transformation.

---

## ADF Concepts Used

| Concept | What It Is |
|---------|-----------|
| Linked Service | Connection to an external service (ADLS, Databricks) |
| Dataset | Pointer to a specific file or table in a linked service |
| Pipeline | Container of activities that run in sequence or parallel |
| Copy Activity | Copies data from source to sink |
| Notebook Activity | Runs a Databricks notebook |
| Trigger | Schedules when a pipeline runs |

---

## ADF Resource Details

| Field | Value |
|-------|-------|
| Name | adf-healthcare-dev-amarnath |
| Region | Central India |
| Resource Group | same as Databricks + Storage |

## Step 1 — Open ADF Studio

1. Go to **portal.azure.com**
2. Search **Data factories**
3. Click **adf-healthcare-dev-amarnath** → Click **Launch Studio**
4. You are now in ADF Studio (author mode)

---

## Step 2 — Create Linked Service for ADLS Gen2

1. Left panel → **Manage** (toolbox icon)
2. Click **Linked Services** → **+ New**
3. Search: `Azure Data Lake Storage Gen2`
4. Click **Continue**
5. Fill in:
   - Name: `ls_adls_retailedgestorage1`
   - Authentication: `Account Key`
   - Storage account: `retailedgestorage1`
6. Click **Test Connection** → should show "Connection successful"
7. Click **Create**

---

## Step 3 — Create 3 Datasets (one per CSV file)

### Dataset 1 — Providers
1. Left panel → **Author** (pencil icon)
2. Click **Datasets** → **+ New Dataset**
3. Choose: `Azure Data Lake Storage Gen2` → `DelimitedText (CSV)`
4. Fill in:
   - Name: `ds_cms_providers`
   - Linked Service: `ls_adls_retailedgestorage1`
   - File path: `healthcare/raw/providers/cms_providers_2022.csv`
   - First row as header: **checked**
5. Click **OK**

### Dataset 2 — Drugs
- Name: `ds_cms_drugs`
- File path: `healthcare/raw/drugs/cms_drugs_2022.csv`

### Dataset 3 — Inpatient
- Name: `ds_cms_inpatient`
- File path: `healthcare/raw/inpatient/cms_inpatient_2022.csv`

---

## Step 4 — Build Pipeline: pl_healthcare_full_load

1. Click **Pipelines** → **+ New Pipeline**
2. Name: `pl_healthcare_full_load`
3. Drag **Copy Data** activity onto canvas
4. Name it: `copy_providers`
5. Source tab → Dataset: `ds_cms_providers`
6. Sink tab → Dataset: same (copy within ADLS, or use as validation)

> Note: Since files are already in ADLS, this pipeline validates the setup.
> In a real scenario, source would be an HTTP or on-prem location.

---

## Step 5 — Add Pipeline Parameters (for dynamic runs)

Add these parameters to the pipeline:

| Parameter | Type | Default Value |
|-----------|------|--------------|
| p_year | string | 2022 |
| p_run_date | string | @utcnow() |

---

## Step 6 — Validate and Publish

1. Click **Validate** (top menu) — should show no errors
2. Click **Publish All** — saves all changes

---

## Step 7 — Test Run

1. Click **Add Trigger** → **Trigger Now**
2. Click **OK**
3. Go to **Monitor** tab (left panel)
4. Watch pipeline run — should show **Succeeded**

---

## ADF Pipeline Screenshot Checklist

Save screenshots to: `adf/pipeline_screenshots/`

- [ ] Linked service connection test success
- [ ] Dataset configuration
- [ ] Pipeline canvas with activities
- [ ] Successful pipeline run in Monitor tab
- [ ] Trigger configuration (Day 8)

---

## Status

- [x] ADF Studio opened
- [x] Linked Service created: ls_adls_healthcare
- [x] Dataset created: ds_cms_providers
- [x] Dataset created: ds_cms_drugs
- [x] Dataset created: ds_cms_inpatient
- [x] Pipeline created: pl_healthcare_full_load
- [x] Pipeline validated and published
- [x] Test run successful (29s, Manual trigger, Succeeded — 2026-08-06)

---

## Notes

> Add any issues or ADF-specific observations here.

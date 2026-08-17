# Project Plan & Progress Tracker

**Project:** Medicare Healthcare Claims Analytics Pipeline  
**Start Date:** TBD  
**Target Completion:** 9 days of active work

---

## Goal

Build a resume-quality Azure Data Engineering project that demonstrates:
1. ADLS Gen2 — cloud file storage and folder structure
2. Azure Data Factory — pipeline orchestration and scheduling
3. Azure Databricks — Medallion Architecture (Bronze/Silver/Gold)
4. Delta MERGE — incremental CDC pattern (most asked interview topic)
5. Unity Catalog — managed Delta tables and governance
6. End-to-end automation — ADF triggers Databricks, no manual steps

---

## Build Phases

### Phase 1 — Infrastructure Setup (Days 1–2)
Set up ADLS Gen2 folder structure, download CMS data, build ADF copy pipeline.

### Phase 2 — Data Pipeline (Days 3–6)
Build Bronze, Silver, Gold layers in Databricks. Implement Delta MERGE.

### Phase 3 — Orchestration (Days 7–8)
Connect ADF to Databricks. Add schedule trigger. End-to-end automation.

### Phase 4 — Documentation & GitHub (Day 9)
Export notebooks, finalize docs, push to GitHub, update resume.

---

## Detailed Task List

### Day 1 — Data Download + ADLS Setup
- [x] Download 3 CMS datasets from data.cms.gov
- [x] Open Azure Portal → Storage Account (retailedgestorage1)
- [x] Create container: `healthcare`
- [x] Create folders: `raw/providers/`, `raw/drugs/`, `raw/inpatient/`
- [x] Upload CSV files to respective raw folders
- [x] Verify files visible in Storage Browser
- [x] Document in: `docs/01_adls_setup.md`

### Day 2 — ADF Pipeline (Copy Activity)
- [x] Open Azure Data Factory Studio
- [x] Create Linked Service → ADLS Gen2 (ls_adls_retailedgestorage1)
- [x] Create Dataset → providers CSV (ds_cms_providers)
- [x] Create Dataset → drugs CSV (ds_cms_drugs)
- [x] Create Dataset → inpatient CSV (ds_cms_inpatient)
- [x] Build Pipeline: `pl_healthcare_full_load`
- [x] Add 3 Copy Activities (one per dataset)
- [x] Test run pipeline — Succeeded in 29s (2026-08-06)
- [x] Verify files in ADLS raw zone
- [x] Document in: `docs/02_adf_pipeline.md`

### Day 3 — Bronze Layer
- [x] Open Databricks → create new notebook: `01_bronze_ingestion`
- [x] Set catalog/schema config (adb_retailedge_dev.healthcare_cms)
- [x] Read providers CSV from ADLS path
- [x] Read drugs CSV from ADLS path
- [x] Read inpatient CSV from ADLS path
- [x] Add bronze metadata columns (_ingestion_timestamp, _source_file, _pipeline_name, _layer)
- [x] Write 3 Delta tables: bronze_providers (592,965), bronze_drugs (14,536), bronze_inpatient (26,571)
- [x] Verify row counts — TOTAL: 634,072 rows
- [x] Export notebook → `notebooks/01_bronze_ingestion.ipynb`
- [x] Document in: `docs/03_bronze_layer.md`

### Day 4 — Silver Layer
- [x] Create notebook: `02_silver_transformation`
- [x] Clean providers — trim spaces, try_cast types, filter nulls
- [x] Clean drugs — parse numeric columns, filter null spending
- [x] Clean inpatient — filter to State level, handle nulls
- [x] Aggregate providers by NPI → silver_provider_summary (1 row per provider)
- [x] Write 4 Silver tables
- [x] Verify row counts — TOTAL: 706,978 rows
- [x] Export notebook → `notebooks/02_silver_transformation.ipynb`
- [x] Document in: `docs/04_silver_layer.md`

### Day 5 — Delta MERGE (Incremental Load)
- [x] Create notebook: `04_incremental_merge`
- [x] Simulate "new month" data — 200 updated providers + 5 new providers
- [x] Write MERGE statement: INSERT new + UPDATE changed
- [x] Verify: check DESCRIBE HISTORY shows merge operations (4 versions)
- [x] Verified idempotency — re-running MERGE updates instead of duplicating
- [x] Test time travel: query table AS OF previous version
- [x] Spot check: verified multipliers applied exactly on NPI 1003000639
- [x] Document the CDC pattern in: `docs/06_delta_merge_cdc.md`

### Day 6 — Gold Layer
- [x] Create notebook: `03_gold_aggregations`
- [x] Build gold_provider_performance (3,435 rows — state + specialty ranking)
- [x] Build gold_drug_spend_analysis (14,536 rows — price trends 2020–2024)
- [x] Build gold_diagnosis_trends (25,804 rows — state + DRG ranking)
- [x] Build gold_fraud_indicators (9,184 rows — 2 stddev above specialty avg)
- [x] Verify all 4 tables — TOTAL: 52,959 Gold rows
- [x] Export notebook → `notebooks/03_gold_aggregations.ipynb`
- [x] Document in: `docs/05_gold_layer.md`

### Day 7 — ADF → Databricks Linked Service
- [x] Create Databricks access token (adf_databricks_token, all scopes, 90 days)
- [x] Create ADF Linked Service: ls_databricks (Compute → Azure Databricks)
- [x] Add Notebook Activity: run_bronze_notebook
- [x] Add Notebook Activity: run_silver_notebook
- [x] Add Notebook Activity: run_gold_notebook
- [x] Chain: Copy activities → Bronze → Silver → Gold (green success arrows)
- [x] Pipeline validated successfully
- [x] Published to ADF
- [ ] Full end-to-end run — blocked by Azure free account CPU quota (4 cores limit)
- [x] Document in: `docs/07_adf_orchestration.md`

### Day 8 — Schedule Trigger
- [x] Create ADF Trigger: `tr_monthly_healthcare`
- [x] Set schedule: 1st of each month, 06:00 AM IST
- [x] Activate trigger — Status: Started
- [x] Published to ADF
- [x] Update `docs/07_adf_orchestration.md`

### Day 9 — GitHub + Final Documentation
- [x] Export all 4 Databricks notebooks (.ipynb format)
- [x] Save to `notebooks/` folder
- [x] Finalize all docs/ files
- [x] Update README.md — actual row counts, corrected architecture, key errors table
- [x] Create GitHub repo: `azure-medicare-analytics-pipeline`
- [ ] Push all files to GitHub
- [ ] Update resume with project bullets
- [ ] Share GitHub link

---

## Architecture Decisions Log

| Decision | Choice | Reason |
|----------|--------|--------|
| File storage | ADLS Gen2 (retailedge) | Already exists from Project 1 |
| Catalog | adb_retailedge_dev | Already exists, avoid CREATE CATALOG error |
| Schema | healthcare_cms | Separate from olist_ecommerce |
| Table storage | Unity Catalog managed | No LOCATION needed, simpler |
| Incremental pattern | Delta MERGE | Industry standard CDC pattern |
| Orchestration | ADF Notebook Activity | Connects ADF to Databricks natively |

---

## Progress Summary

```
Phase 1 — Infrastructure :  ✅✅  (2/2 days complete)
Phase 2 — Data Pipeline  :  ✅✅✅✅  (4/4 days complete)
Phase 3 — Orchestration  :  ✅✅  (2/2 days complete)
Phase 4 — Documentation  :  ✅  (1/1 days complete)

Overall: 9 / 9 days complete
```

---

## Errors Encountered

All errors documented in: `docs/08_errors_and_fixes.md`

# Azure Medicare Analytics Pipeline

**Domain:** Healthcare Data Engineering  
**Stack:** Azure Databricks | Azure Data Factory | ADLS Gen2 | Delta Lake | Unity Catalog | PySpark  
**Pattern:** Medallion Architecture (Bronze → Silver → Gold) + Incremental CDC (Delta MERGE)

---

## Project Summary

End-to-end Azure data pipeline ingesting CMS Medicare public data (634K+ rows) across 3 source
datasets into ADLS Gen2, processed through a Medallion Architecture on Azure Databricks, and
orchestrated via Azure Data Factory with monthly scheduled triggers.

Key highlights:
- **Delta MERGE (CDC)** — incremental monthly load with full audit trail and time travel
- **Unity Catalog** — all 11 tables managed with governance and lineage
- **ADF Orchestration** — Copy Activity → Bronze → Silver → Gold chained pipeline
- **Fraud Detection** — statistical outlier flagging using stddev above specialty average

---

## Architecture

```
CMS Medicare Public Data (data.cms.gov)
                ↓
        ADLS Gen2 — retailedgestorage1
        Container: healthcare
        └── raw/
            ├── providers/   ← cms_providers_2024.csv
            ├── drugs/       ← cms_drugs_2024.csv
            └── inpatient/   ← cms_inpatient_2024.csv
                ↓
        Azure Data Factory — adf-healthcare-dev-amarnath
        ├── Pipeline: pl_healthcare_full_load
        │   ├── Copy Activity (providers)
        │   ├── Copy Activity (drugs)
        │   ├── Copy Activity (inpatient)
        │   ├── Notebook Activity → 01_bronze_ingestion
        │   ├── Notebook Activity → 02_silver_transformation
        │   └── Notebook Activity → 03_gold_aggregations
        └── Trigger: tr_monthly_healthcare (1st of each month, 06:00 AM IST)
                ↓
        Azure Databricks (Serverless Cluster)
        Catalog : adb_retailedge_dev
        Schema  : healthcare_cms
        ├── Bronze Layer  (3 tables) — raw Delta tables with audit metadata
        ├── Silver Layer  (4 tables) — cleaned, typed, validated, aggregated
        └── Gold Layer    (4 tables) — business-ready analytics
                ↓
        Unity Catalog — 11 managed Delta tables
```

---

## Source Datasets

| Dataset | Source | Actual Rows | File |
|---|---|---|---|
| Medicare Provider Utilization | data.cms.gov | 592,965 | cms_providers_2024.csv |
| Medicare Part D Drug Spending | data.cms.gov | 14,536 | cms_drugs_2024.csv |
| Medicare Inpatient by Geography | data.cms.gov | 26,571 | cms_inpatient_2024.csv |
| **Total** | | **634,072** | |

---

## Layer Summary

| Layer | Tables | Actual Rows | Description |
|---|---|---|---|
| Bronze | 3 | 634,072 | Raw ingestion with 4 audit metadata columns |
| Silver | 4 | 706,978 | Cleaned, typed, validated, aggregated |
| Gold | 4 | 52,959 | Business-ready analytics and fraud indicators |
| **Total** | **11** | **~1.4M** | Unity Catalog managed Delta tables |

---

## Gold Layer Tables

| Table | Rows | Description |
|---|---|---|
| `gold_provider_performance` | 3,435 | Specialty cost ranking by state with Window rank |
| `gold_drug_spend_analysis` | 14,536 | Drug spend 2020–2024 with price trend category |
| `gold_diagnosis_trends` | 25,804 | State-level disease burden ranked by discharges |
| `gold_fraud_indicators` | 9,184 | Providers 1–2+ stddev above specialty avg payment |

---

## Delta MERGE — CDC Pattern

Monthly incremental load on `silver_provider_summary` (73,678 rows, unique NPI key):

```
Incoming batch (205 rows)
    ├── 200 existing providers → WHEN MATCHED → UPDATE payment amounts
    └── 5 new providers       → WHEN NOT MATCHED → INSERT new rows

Result:
    - Rows updated : 200
    - Rows inserted: 5
    - Net new rows : +5 (73,673 → 73,678)
    - Delta versions created: multiple (with full time travel)
```

**Key pattern:** Source always materialized to a staging Delta table before MERGE — never use a temp view or lazy DataFrame as MERGE source.

---

## New Skills Demonstrated (vs Project 1 — Olist)

| Skill | Tool | Where Used |
|---|---|---|
| Cloud file storage | ADLS Gen2 | Raw file landing zone |
| Pipeline orchestration | Azure Data Factory | Copy + Notebook Activities chained |
| Incremental CDC loads | Delta MERGE | Monthly provider data update |
| Scheduling | ADF Trigger | Monthly automated run — 1st of month |
| Statistical fraud detection | PySpark stddev | Gold fraud indicators table |
| Multi-year trend analysis | PySpark | Drug spend CAGR 2020–2024 |
| Delta time travel | VERSION AS OF | Pre/post merge snapshot comparison |
| Table governance | Unity Catalog | All 11 managed Delta tables |

---

## Project Structure

```
azure-medicare-analytics-pipeline/
├── README.md
├── notebooks/
│   ├── 01_bronze_ingestion.ipynb       ← Bronze layer — raw CSV → Delta
│   ├── 02_silver_transformation.ipynb  ← Silver layer — clean + aggregate
│   ├── 03_gold_aggregations.ipynb      ← Gold layer — 4 analytics tables
│   ├── 04_incremental_merge.ipynb      ← Delta MERGE CDC pattern
│   └── 05_business_insights.ipynb      ← business questions answered from Gold layer
├── docs/
│   ├── 00_project_plan.md              ← build plan + progress tracker
│   ├── 01_adls_setup.md                ← ADLS Gen2 setup steps
│   ├── 02_adf_pipeline.md              ← ADF Copy Activity pipeline
│   ├── 03_bronze_layer.md              ← Bronze notebook docs + results
│   ├── 04_silver_layer.md              ← Silver notebook docs + results
│   ├── 05_gold_layer.md                ← Gold notebook docs + results
│   ├── 06_delta_merge_cdc.md           ← Delta MERGE full walkthrough
│   ├── 07_adf_orchestration.md         ← ADF → Databricks linked service
│   ├── 08_errors_and_fixes.md          ← all errors encountered + solutions
│   ├── 09_business_insights.md         ← actual findings from Gold layer queries
│   └── learnings.md                    ← key concepts learned during build
├── data_schema/
│   ├── cms_provider_schema.md
│   ├── cms_drug_schema.md
│   └── cms_inpatient_schema.md
└── adf/
    └── pipeline_screenshots/
```

---

## Build Progress

| Day | Task | Status |
|---|---|---|
| 1 | Download CMS data + ADLS setup | ✅ Complete |
| 2 | ADF Pipeline — copy to ADLS | ✅ Complete |
| 3 | Databricks Bronze layer | ✅ Complete |
| 4 | Silver layer — clean + aggregate | ✅ Complete |
| 5 | Delta MERGE — incremental CDC | ✅ Complete |
| 6 | Gold layer — 4 analytics tables | ✅ Complete |
| 7 | ADF calls Databricks notebooks | ✅ Complete |
| 8 | ADF schedule trigger | ✅ Complete |
| 9 | GitHub + documentation | ✅ Complete |

---

## Key Errors Solved

| Error | Root Cause | Fix |
|---|---|---|
| DELTA_MULTIPLE_SOURCE_ROW_MATCHING | Lazy DataFrame re-evaluated mid-MERGE | Materialize source to staging Delta table first |
| ADF Notebook Activity — serverless not supported | ADF requires classic job cluster | Changed linked service to New Job Cluster |
| Databricks credits exhausted | 14-day DBU trial expired | Upgraded workspace to Premium Pay-As-You-Go |
| CLOUD_PROVIDER_RESOURCE_STOCKOUT | VM size unavailable in Central India | Changed node type to available VM |
| Spark version not supported | Workspace has legacy features disabled | Upgraded to Runtime 13.3.x-photon |

---

## Related Project

- [Project 1 — Olist E-Commerce Pipeline](https://github.com/amarnath-003/olist-ecommerce-pipeline) — Brazilian e-commerce dataset, Medallion Architecture baseline on Azure Databricks

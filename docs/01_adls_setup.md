# Day 1 — ADLS Gen2 Setup & Data Download

## Objective
Download 3 CMS Medicare datasets and upload them to ADLS Gen2 in the correct folder structure.

---

## Step 1 — Download CMS Data

Go to **data.cms.gov** and download these 3 files:

### Dataset 1 — Medicare Provider Utilization
- URL: https://data.cms.gov/provider-summary-by-type-of-service/medicare-physician-other-practitioners/medicare-physician-other-practitioners-by-provider
- Download option: "Latest Dataset Only (2024)"
- Rename to: `cms_providers_2024.csv`
- Expected size: ~3 GB

### Dataset 2 — Medicare Part D Drug Spending
- URL: https://data.cms.gov/summary-statistics-on-use-and-payments/medicare-medicaid-spending-by-drug/medicare-part-d-spending-by-drug
- Download option: "Latest Dataset Only (2024)"
- Rename to: `cms_drugs_2024.csv`
- Expected size: ~10–50 MB

### Dataset 3 — Medicare Inpatient Hospitals
- URL: https://data.cms.gov/provider-summary-by-type-of-service/medicare-inpatient-hospitals/medicare-inpatient-hospitals-by-geography-and-service
- Download option: "Latest Dataset Only (2024)"
- Rename to: `cms_inpatient_2024.csv`
- Expected size: ~2–5 MB

---

## Step 2 — ADLS Gen2 Folder Structure

### Storage Account Details
```
Storage Account : retailedgestorage1 (already exists from Project 1)
Container       : healthcare  ← create this new container
```

### Folder Structure to Create
```
healthcare/
├── raw/
│   ├── providers/
│   │   └── cms_providers_2024.csv
│   ├── drugs/
│   │   └── cms_drugs_2024.csv
│   └── inpatient/
│       └── cms_inpatient_2024.csv
```

---

## Step 3 — Create Container in Azure Portal

1. Go to **portal.azure.com**
2. Search for your **Storage Account** → `retailedgestorage1`
3. Left menu → **Containers**
4. Click **+ Container**
5. Name: `healthcare`
6. Access level: **Private**
7. Click **Create**

---

## Step 4 — Create Folder Structure

1. Click into the `healthcare` container
2. Click **+ Add Directory** → type `raw`
3. Click into `raw` → **+ Add Directory** → type `providers`
4. Repeat for `drugs` and `inpatient`

---

## Step 5 — Upload CSV Files

1. Click into `raw/providers/`
2. Click **Upload**
3. Select `cms_providers_2024.csv`
4. Click **Upload**
5. Repeat for drugs and inpatient folders

---

## Step 6 — Verify

After upload, your Storage Browser should show:
```
healthcare
└── raw
    ├── providers
    │   └── cms_providers_2024.csv   ✓
    ├── drugs
    │   └── cms_drugs_2024.csv       ✓
    └── inpatient
        └── cms_inpatient_2024.csv   ✓
```

---

## ADLS Paths for Databricks

These are the paths you will use in Databricks notebooks:

```python
STORAGE_ACCOUNT = "retailedgestorage1"
CONTAINER       = "healthcare"

PROVIDERS_PATH  = f"abfss://{CONTAINER}@{STORAGE_ACCOUNT}.dfs.core.windows.net/raw/providers/cms_providers_2024.csv"
DRUGS_PATH      = f"abfss://{CONTAINER}@{STORAGE_ACCOUNT}.dfs.core.windows.net/raw/drugs/cms_drugs_2024.csv"
INPATIENT_PATH  = f"abfss://{CONTAINER}@{STORAGE_ACCOUNT}.dfs.core.windows.net/raw/inpatient/cms_inpatient_2024.csv"
```

---

## Status

- [x] CMS Provider data downloaded (~3 GB)
- [x] CMS Drug data downloaded (~5 MB)
- [x] CMS Inpatient data downloaded (~3 MB)
- [x] `healthcare` container created in ADLS
- [x] Folder structure created (raw/providers, raw/drugs, raw/inpatient)
- [x] All 3 CSV files uploaded to correct folders
- [x] Paths verified in Storage Browser

---

## Notes

> Add any issues or observations here as you work through this step.

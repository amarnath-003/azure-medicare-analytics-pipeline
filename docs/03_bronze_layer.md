# Day 3 — Bronze Layer

## Objective
Read all 3 CMS CSV files from ADLS Gen2 and write them as raw Delta tables
in Unity Catalog with audit metadata columns. No transformation — raw faithful copy.

---

## Notebook
Name: `01_bronze_ingestion`
Catalog: `adb_retailedge_dev`
Schema: `healthcare_cms`

---

## Bronze Design Principle

Bronze = raw copy of source with audit columns added. No cleaning, no type casting.

```
CSV File (ADLS)
      ↓
spark.read.csv(inferSchema=False)   ← all columns as string
      ↓
Add 4 audit columns
      ↓
.saveAsTable(bronze_table)          ← Delta managed table
```

### Why inferSchema=False
- Bronze loads everything as string — no type assumptions
- Prevents crashes on messy real-world data
- Single pass through file — faster
- Type casting happens in Silver using try_cast

### 4 Audit Columns Added to Every Bronze Table

| Column | Value | Purpose |
|--------|-------|---------|
| `_ingestion_timestamp` | current_timestamp() | When row was loaded |
| `_source_file` | filename string | Which file it came from |
| `_pipeline_name` | "healthcare_cms_pipeline" | Pipeline identifier |
| `_layer` | "bronze" | Which layer |

---

## Cell by Cell

### Cell 1 — Configuration
```python
CATALOG        = "adb_retailedge_dev"
SCHEMA         = "healthcare_cms"
STORAGE_ACCT   = "retailedgestorage1"
CONTAINER      = "healthcare"

BASE_PATH      = f"abfss://{CONTAINER}@{STORAGE_ACCT}.dfs.core.windows.net/raw"
PROVIDERS_PATH = f"{BASE_PATH}/providers/cms_providers_2024.csv"
DRUGS_PATH     = f"{BASE_PATH}/drugs/cms_drugs_2024.csv"
INPATIENT_PATH = f"{BASE_PATH}/inpatient/cms_inpatient_2024.csv"
```

### Cell 2 — Create Schema
```python
spark.sql(f"CREATE SCHEMA IF NOT EXISTS {CATALOG}.{SCHEMA}")
```

### Cell 3 — Bronze Writer Function
```python
from pyspark.sql.functions import current_timestamp, lit

def write_bronze(df, table_name, source_file):
    bronze_df = df \
        .withColumn("_ingestion_timestamp", current_timestamp()) \
        .withColumn("_source_file", lit(source_file)) \
        .withColumn("_pipeline_name", lit("healthcare_cms_pipeline")) \
        .withColumn("_layer", lit("bronze"))

    bronze_df.write \
        .format("delta") \
        .mode("overwrite") \
        .option("overwriteSchema", "true") \
        .saveAsTable(f"{CATALOG}.{SCHEMA}.bronze_{table_name}")

    count = spark.table(f"{CATALOG}.{SCHEMA}.bronze_{table_name}").count()
    print(f"bronze_{table_name}: {count:,} rows written")
```

### Cell 4 — Load Providers
```python
df_providers = spark.read.csv(PROVIDERS_PATH, header=True, inferSchema=False)
write_bronze(df_providers, "providers", "cms_providers_2024.csv")
```

### Cell 5 — Load Drugs
```python
df_drugs = spark.read.csv(DRUGS_PATH, header=True, inferSchema=False)
write_bronze(df_drugs, "drugs", "cms_drugs_2024.csv")
```

### Cell 6 — Load Inpatient
```python
df_inpatient = spark.read.csv(INPATIENT_PATH, header=True, inferSchema=False)
write_bronze(df_inpatient, "inpatient", "cms_inpatient_2024.csv")
```

### Cell 7 — Verification
```python
tables = ["providers", "drugs", "inpatient"]
total = 0
for t in tables:
    count = spark.table(f"{CATALOG}.{SCHEMA}.bronze_{t}").count()
    total += count
    print(f"bronze_{t:<12}: {count:>10,} rows")
print(f"TOTAL: {total:,} rows")
```

---

## Results

| Table | Rows | Columns | Source File |
|-------|------|---------|-------------|
| bronze_providers | 592,965 | 28 | cms_providers_2024.csv |
| bronze_drugs | 14,536 | 46 | cms_drugs_2024.csv |
| bronze_inpatient | 26,571 | 9 | cms_inpatient_2024.csv |
| **TOTAL** | **634,072** | | |

---

## Errors Encountered

| Error | Fix |
|-------|-----|
| PATH_NOT_FOUND for inpatient | File uploaded as `.CSV` (uppercase) — renamed in ADLS to `.csv` |

---

## Status

- [x] Schema created: adb_retailedge_dev.healthcare_cms
- [x] bronze_providers: 592,965 rows
- [x] bronze_drugs: 14,536 rows
- [x] bronze_inpatient: 26,571 rows
- [x] All 3 tables verified in Unity Catalog

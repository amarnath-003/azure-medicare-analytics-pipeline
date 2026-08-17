# Day 4 — Silver Layer

## Objective
Read Bronze Delta tables, clean and transform the data, cast columns to
correct types, and write 4 Silver tables ready for Gold aggregations.

---

## Notebook
Name: `02_silver_transformation`
Catalog: `adb_retailedge_dev`
Schema: `healthcare_cms`

---

## Silver Design Principles

1. **Select only needed columns** — drop irrelevant columns from Bronze
2. **Rename columns** — clean snake_case names instead of CMS abbreviations
3. **Cast types** — use `try_cast` via `expr()` — bad values become NULL not crashes
4. **Filter nulls** — remove rows with null on key columns
5. **Aggregate where needed** — collapse provider rows by NPI

---

## Key PySpark Operations Used

| Operation | Purpose |
|-----------|---------|
| `trim(col())` | Remove leading/trailing spaces from string columns |
| `expr("try_cast(col as type)")` | Safe type casting — bad values → NULL |
| `.filter()` | Remove null or invalid rows |
| `.groupBy().agg()` | Aggregate multiple rows into one |
| `spark_sum()`, `spark_avg()`, `spark_count()` | Aggregation functions |

### Why try_cast Instead of cast?
```python
# cast — crashes if any value is invalid
col("Tot_Benes").cast("int")      # one bad value → whole job fails

# try_cast — bad values become NULL instead
expr("try_cast(Tot_Benes as int)") # bad values → NULL, job continues
```

### Why Import Explicitly?
```python
# BAD — causes conflicts with Python built-ins
from pyspark.sql.functions import *

# GOOD — explicit imports, no conflicts
from pyspark.sql.functions import col, trim, expr
from pyspark.sql.functions import sum as spark_sum, count as spark_count
```

---

## Cell by Cell

### Cell 1 — Configuration
```python
CATALOG = "adb_retailedge_dev"
SCHEMA  = "healthcare_cms"
```

### Cell 2 — Silver Providers
```python
from pyspark.sql.functions import col, trim, expr

df_providers = spark.table(f"{CATALOG}.{SCHEMA}.bronze_providers")

silver_providers = df_providers \
    .select(
        trim(col("Rndrng_NPI")).alias("provider_npi"),
        trim(col("Rndrng_Prvdr_Last_Org_Name")).alias("provider_name"),
        trim(col("Rndrng_Prvdr_First_Name")).alias("provider_first_name"),
        trim(col("Rndrng_Prvdr_City")).alias("city"),
        trim(col("Rndrng_Prvdr_State_Abrvtn")).alias("state"),
        trim(col("Rndrng_Prvdr_Zip5")).alias("zip_code"),
        trim(col("Rndrng_Prvdr_Type")).alias("provider_type"),
        trim(col("HCPCS_Cd")).alias("hcpcs_code"),
        trim(col("HCPCS_Desc")).alias("hcpcs_description"),
        trim(col("Place_Of_Srvc")).alias("place_of_service"),
        expr("try_cast(Tot_Benes as int)").alias("total_beneficiaries"),
        expr("try_cast(Tot_Srvcs as double)").alias("total_services"),
        expr("try_cast(Avg_Mdcr_Pymt_Amt as double)").alias("avg_medicare_payment"),
        expr("try_cast(Avg_Sbmtd_Chrg as double)").alias("avg_submitted_charge"),
        expr("try_cast(Avg_Mdcr_Alowd_Amt as double)").alias("avg_allowed_amount")
    ) \
    .filter(col("provider_npi").isNotNull()) \
    .filter(col("state").isNotNull())

silver_providers.write.format("delta").mode("overwrite") \
    .option("overwriteSchema", "true") \
    .saveAsTable(f"{CATALOG}.{SCHEMA}.silver_providers")
```

### Cell 3 — Silver Provider Summary (Aggregated by NPI)
```python
from pyspark.sql.functions import sum as spark_sum, avg as spark_avg
from pyspark.sql.functions import count as spark_count, round as spark_round

silver_provider_summary = spark.table(f"{CATALOG}.{SCHEMA}.silver_providers") \
    .groupBy("provider_npi", "provider_name", "provider_first_name",
             "city", "state", "zip_code", "provider_type") \
    .agg(
        spark_count("hcpcs_code").alias("total_procedures"),
        spark_sum("total_beneficiaries").alias("total_beneficiaries"),
        spark_sum("total_services").alias("total_services"),
        spark_round(spark_avg("avg_medicare_payment"), 2).alias("avg_medicare_payment"),
        spark_round(spark_avg("avg_submitted_charge"), 2).alias("avg_submitted_charge"),
        spark_round(spark_avg("avg_allowed_amount"), 2).alias("avg_allowed_amount")
    )

silver_provider_summary.write.format("delta").mode("overwrite") \
    .option("overwriteSchema", "true") \
    .saveAsTable(f"{CATALOG}.{SCHEMA}.silver_provider_summary")
```

### Cell 4 — Silver Drugs
```python
from pyspark.sql.functions import col, trim, expr

silver_drugs = spark.table(f"{CATALOG}.{SCHEMA}.bronze_drugs") \
    .select(
        trim(col("Brnd_Name")).alias("brand_name"),
        trim(col("Gnrc_Name")).alias("generic_name"),
        trim(col("Mftr_Name")).alias("manufacturer"),
        expr("try_cast(Tot_Mftr as int)").alias("total_manufacturers"),
        expr("try_cast(Tot_Clms_2024 as int)").alias("total_claims_2024"),
        expr("try_cast(Tot_Benes_2024 as int)").alias("total_beneficiaries_2024"),
        expr("try_cast(Tot_Spndng_2024 as double)").alias("total_spending_2024"),
        expr("try_cast(Avg_Spnd_Per_Clm_2024 as double)").alias("avg_spend_per_claim_2024"),
        expr("try_cast(Avg_Spnd_Per_Bene_2024 as double)").alias("avg_spend_per_bene_2024"),
        expr("try_cast(Tot_Spndng_2023 as double)").alias("total_spending_2023"),
        expr("try_cast(Tot_Spndng_2022 as double)").alias("total_spending_2022"),
        expr("try_cast(Tot_Spndng_2021 as double)").alias("total_spending_2021"),
        expr("try_cast(Tot_Spndng_2020 as double)").alias("total_spending_2020"),
        expr("try_cast(CAGR_Avg_Spnd_Per_Dsg_Unt_20_24 as double)").alias("cagr_2020_2024"),
        col("Outlier_Flag_2024").alias("outlier_flag")
    ) \
    .filter(col("brand_name").isNotNull()) \
    .filter(col("total_spending_2024").isNotNull())

silver_drugs.write.format("delta").mode("overwrite") \
    .option("overwriteSchema", "true") \
    .saveAsTable(f"{CATALOG}.{SCHEMA}.silver_drugs")
```

### Cell 5 — Silver Inpatient
```python
from pyspark.sql.functions import col, trim, expr

silver_inpatient = spark.table(f"{CATALOG}.{SCHEMA}.bronze_inpatient") \
    .select(
        trim(col("Rndrng_Prvdr_Geo_Lvl")).alias("geo_level"),
        trim(col("Rndrng_Prvdr_Geo_Cd")).alias("geo_code"),
        trim(col("Rndrng_Prvdr_Geo_Desc")).alias("geo_description"),
        trim(col("DRG_Cd")).alias("drg_code"),
        trim(col("DRG_Desc")).alias("drg_description"),
        expr("try_cast(Tot_Dschrgs as int)").alias("total_discharges"),
        expr("try_cast(Avg_Submtd_Cvrd_Chrg as double)").alias("avg_submitted_charge"),
        expr("try_cast(Avg_Tot_Pymt_Amt as double)").alias("avg_total_payment"),
        expr("try_cast(Avg_Mdcr_Pymt_Amt as double)").alias("avg_medicare_payment")
    ) \
    .filter(col("geo_level") == "State") \
    .filter(col("drg_code").isNotNull()) \
    .filter(col("total_discharges").isNotNull())

silver_inpatient.write.format("delta").mode("overwrite") \
    .option("overwriteSchema", "true") \
    .saveAsTable(f"{CATALOG}.{SCHEMA}.silver_inpatient")
```

---

## Results

| Table | Rows | Description |
|-------|------|-------------|
| silver_providers | 592,965 | Cleaned provider + procedure rows |
| silver_provider_summary | 73,673 | One row per provider — aggregated |
| silver_drugs | 14,536 | Drug spending 2020–2024 |
| silver_inpatient | 25,804 | State-level inpatient claims |
| **TOTAL** | **706,978** | |

---

## Errors Encountered

| Error | Fix |
|-------|-----|
| TypeError: 'int' object is not callable | Used `from pyspark.sql.functions import *` — conflicted with Python built-in `count`. Fixed by explicit imports with aliases |
| TABLE_OR_VIEW_NOT_FOUND silver_provider_summary | Saved as `silver_provider_summery` (typo). Fixed with `ALTER TABLE ... RENAME TO` |
| TABLE_OR_VIEW_NOT_FOUND bronze_durgs | Typo in table name — fixed to `bronze_drugs` |

---

## Status

- [x] silver_providers: 592,965 rows
- [x] silver_provider_summary: 73,673 rows
- [x] silver_drugs: 14,536 rows
- [x] silver_inpatient: 25,804 rows
- [x] All 4 tables verified in Unity Catalog

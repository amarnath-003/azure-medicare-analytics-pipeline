# Day 6 — Gold Layer

## Objective
Read Silver Delta tables and build 4 business-ready analytics tables.
Gold = pre-aggregated answers, not raw data. Built for dashboards and reports.

---

## Notebook
Name: `03_gold_aggregations`
Catalog: `adb_retailedge_dev`
Schema: `healthcare_cms`

---

## Gold vs Silver — Why Gold Exists

Silver is clean data. Gold is a business answer.

Silver still requires someone to write a query to get meaning.
Gold is pre-computed so a dashboard can read it directly without any SQL.
In production, Gold tables connect to Power BI, Tableau, or Looker.

---

## Cell by Cell

### Cell 1 — Configuration
```python
CATALOG = "adb_retailedge_dev"
SCHEMA  = "healthcare_cms"

SILVER_PROVIDER  = f"{CATALOG}.{SCHEMA}.silver_provider_summary"
SILVER_DRUGS     = f"{CATALOG}.{SCHEMA}.silver_drugs"
SILVER_INPATIENT = f"{CATALOG}.{SCHEMA}.silver_inpatient"
```

---

### Cell 2 — gold_provider_performance
**Business question:** Which specialties and states cost Medicare the most?

**Logic:**
- Group `silver_provider_summary` by `state + provider_type`
- Aggregate: total providers, avg payment, avg charge, total services, total beneficiaries
- Add `rank_in_state` — rank each specialty within its state by avg Medicare payment (Window function)

```python
from pyspark.sql.functions import count, round as spark_round
from pyspark.sql.functions import sum as spark_sum, avg as spark_avg
from pyspark.sql.window import Window
from pyspark.sql.functions import rank

df_providers = spark.table(SILVER_PROVIDER)

gold_provider_performance = df_providers \
    .groupBy("state", "provider_type") \
    .agg(
        count("provider_npi").alias("total_providers"),
        spark_round(spark_avg("avg_medicare_payment"), 2).alias("avg_medicare_payment"),
        spark_round(spark_avg("avg_submitted_charge"), 2).alias("avg_submitted_charge"),
        spark_round(spark_sum("total_services"), 0).alias("total_services"),
        spark_round(spark_sum("total_beneficiaries"), 0).alias("total_beneficiaries")
    )

window_spec = Window.partitionBy("state").orderBy(
    gold_provider_performance["avg_medicare_payment"].desc()
)

gold_provider_performance = gold_provider_performance \
    .withColumn("rank_in_state", rank().over(window_spec))

gold_provider_performance.write.format("delta").mode("overwrite") \
    .option("overwriteSchema", "true") \
    .saveAsTable(f"{CATALOG}.{SCHEMA}.gold_provider_performance")
```

Output: `gold_provider_performance: 3,435 rows written`

---

### Cell 3 — gold_drug_spend_analysis
**Business question:** Which drugs cost Medicare the most? Which prices are rising fastest?

**Logic:**
- Select key spending columns from `silver_drugs`
- Add `price_trend_category` using CAGR:
  - > 10% CAGR → "High Growth"
  - > 5% CAGR → "Moderate Growth"
  - ≤ 5% CAGR → "Stable"
- Order by total_spending_2024 descending

```python
from pyspark.sql.functions import col, round as spark_round, when

gold_drug_spend_analysis = spark.table(SILVER_DRUGS) \
    .select(
        "brand_name", "generic_name", "manufacturer",
        "total_claims_2024", "total_beneficiaries_2024",
        spark_round(col("total_spending_2024"), 2).alias("total_spending_2024"),
        spark_round(col("avg_spend_per_claim_2024"), 2).alias("avg_spend_per_claim_2024"),
        spark_round(col("avg_spend_per_bene_2024"), 2).alias("avg_spend_per_bene_2024"),
        spark_round(col("total_spending_2023"), 2).alias("total_spending_2023"),
        spark_round(col("total_spending_2022"), 2).alias("total_spending_2022"),
        spark_round(col("cagr_2020_2024"), 4).alias("cagr_2020_2024"),
        "outlier_flag",
        when(col("cagr_2020_2024") > 0.10, "High Growth")
        .when(col("cagr_2020_2024") > 0.05, "Moderate Growth")
        .when(col("cagr_2020_2024") <= 0.05, "Stable")
        .otherwise("Unknown").alias("price_trend_category")
    ) \
    .filter(col("total_spending_2024").isNotNull()) \
    .orderBy(col("total_spending_2024").desc())

gold_drug_spend_analysis.write.format("delta").mode("overwrite") \
    .option("overwriteSchema", "true") \
    .saveAsTable(f"{CATALOG}.{SCHEMA}.gold_drug_spend_analysis")
```

Output: `gold_drug_spend_analysis: 14,536 rows written`

---

### Cell 4 — gold_diagnosis_trends
**Business question:** Which diseases are most common state by state? Which states have the highest hospital costs?

**Logic:**
- Group `silver_inpatient` by `geo_code + geo_description + drg_code + drg_description`
- Aggregate: total discharges, avg Medicare payment, avg submitted charge, avg total payment
- Add `rank_in_state` — rank each diagnosis within its state by total discharges

```python
from pyspark.sql.functions import col, round as spark_round, rank
from pyspark.sql.window import Window

gold_diagnosis_trends = spark.table(SILVER_INPATIENT) \
    .groupBy("geo_code", "geo_description", "drg_code", "drg_description") \
    .agg(
        spark_round(spark_sum("total_discharges"), 0).alias("total_discharges"),
        spark_round(spark_avg("avg_medicare_payment"), 2).alias("avg_medicare_payment"),
        spark_round(spark_avg("avg_submitted_charge"), 2).alias("avg_submitted_charge"),
        spark_round(spark_avg("avg_total_payment"), 2).alias("avg_total_payment")
    )

window_spec = Window.partitionBy("geo_code").orderBy(col("total_discharges").desc())

gold_diagnosis_trends = gold_diagnosis_trends \
    .withColumn("rank_in_state", rank().over(window_spec))

gold_diagnosis_trends.write.format("delta").mode("overwrite") \
    .option("overwriteSchema", "true") \
    .saveAsTable(f"{CATALOG}.{SCHEMA}.gold_diagnosis_trends")
```

Output: `gold_diagnosis_trends: 25,804 rows written`

---

### Cell 5 — gold_fraud_indicators
**Business question:** Which providers bill abnormally above their specialty average — possible fraud?

**Logic:**
- Calculate avg and stddev of `avg_medicare_payment` per `state + provider_type`
- Join back to provider level
- Flag providers more than 1 or 2 standard deviations above their specialty average:
  - > 2 stddev → "High Risk"
  - > 1 stddev → "Medium Risk"
  - Otherwise → "Normal" (excluded from Gold table)
- Order by stddev_multiplier descending (highest risk first)

```python
from pyspark.sql.functions import col, round as spark_round, avg as spark_avg, stddev, when

df_providers = spark.table(SILVER_PROVIDER)

specialty_stats = df_providers \
    .groupBy("state", "provider_type") \
    .agg(
        spark_round(spark_avg("avg_medicare_payment"), 2).alias("specialty_avg_payment"),
        spark_round(stddev("avg_medicare_payment"), 2).alias("specialty_stddev_payment")
    )

gold_fraud_indicators = df_providers \
    .join(specialty_stats, on=["state", "provider_type"], how="left") \
    .withColumn("deviation_from_avg",
        spark_round(col("avg_medicare_payment") - col("specialty_avg_payment"), 2)) \
    .withColumn("stddev_multiplier",
        spark_round(
            (col("avg_medicare_payment") - col("specialty_avg_payment")) /
            col("specialty_stddev_payment"), 2)) \
    .withColumn("fraud_risk_flag",
        when(col("stddev_multiplier") > 2, "High Risk")
        .when(col("stddev_multiplier") > 1, "Medium Risk")
        .otherwise("Normal")) \
    .filter(col("fraud_risk_flag") != "Normal") \
    .select(
        "provider_npi", "provider_name", "provider_first_name",
        "city", "state", "provider_type",
        "avg_medicare_payment", "specialty_avg_payment",
        "deviation_from_avg", "stddev_multiplier", "fraud_risk_flag",
        "total_services", "total_beneficiaries"
    ) \
    .orderBy(col("stddev_multiplier").desc())

gold_fraud_indicators.write.format("delta").mode("overwrite") \
    .option("overwriteSchema", "true") \
    .saveAsTable(f"{CATALOG}.{SCHEMA}.gold_fraud_indicators")
```

Output: `gold_fraud_indicators: 9,184 rows written`

---

### Cell 6 — Verify all Gold tables
```python
gold_tables = [
    "gold_provider_performance",
    "gold_drug_spend_analysis",
    "gold_diagnosis_trends",
    "gold_fraud_indicators"
]

total = 0
for table in gold_tables:
    count = spark.table(f"{CATALOG}.{SCHEMA}.{table}").count()
    total += count
    print(f"{table:<35}: {count:>10,} rows")

print(f"\nTOTAL Gold rows: {total:,}")
```

Output:
```
gold_provider_performance          :      3,435 rows
gold_drug_spend_analysis           :     14,536 rows
gold_diagnosis_trends              :     25,804 rows
gold_fraud_indicators              :      9,184 rows

TOTAL Gold rows: 52,959
```

---

## Results

| Table | Rows | Source | Business Use |
|---|---|---|---|
| gold_provider_performance | 3,435 | silver_provider_summary | Specialty cost ranking by state |
| gold_drug_spend_analysis | 14,536 | silver_drugs | Drug spend + price trend 2020–2024 |
| gold_diagnosis_trends | 25,804 | silver_inpatient | Disease burden by state |
| gold_fraud_indicators | 9,184 | silver_provider_summary | Abnormal billing detection |
| **TOTAL** | **52,959** | | |

---

## Full Project Table Count (All Layers)

| Layer | Tables | Rows |
|---|---|---|
| Bronze | 3 | 634,072 |
| Silver | 4 | 706,978 |
| Gold | 4 | 52,959 |
| **Total** | **11** | **~1.4M** |

---

## Key PySpark Concepts Used

| Concept | Where Used |
|---|---|
| `Window.partitionBy().orderBy()` | Ranking specialties within each state |
| `rank().over(window_spec)` | Row number per partition |
| `stddev()` | Calculate billing variability per specialty |
| `when().otherwise()` | Classify price trend and fraud risk categories |
| `.join()` | Bring specialty stats back to provider level |

---

## Interview Q&A

| Question | Answer |
|---|---|
| What is the Gold layer for? | Pre-aggregated business answers — connected directly to dashboards, no SQL needed |
| What is a Window function? | A calculation across a set of rows related to the current row — here used to rank specialties within each state without collapsing the rows |
| How did you detect fraud? | Calculated stddev of payments per specialty+state, flagged providers more than 2 stddev above their peer group average |
| Why filter out "Normal" from fraud table? | Gold tables should contain only actionable rows — 73K normal providers would overwhelm the fraud analyst |

---

## Status

- [x] gold_provider_performance: 3,435 rows
- [x] gold_drug_spend_analysis: 14,536 rows
- [x] gold_diagnosis_trends: 25,804 rows
- [x] gold_fraud_indicators: 9,184 rows
- [x] All 4 tables verified in Unity Catalog

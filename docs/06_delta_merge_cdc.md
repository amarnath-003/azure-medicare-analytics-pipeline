# Day 5 — Delta MERGE & Incremental CDC Pattern

## What is CDC?

**CDC = Change Data Capture**

In the real world, data changes every day:
- New claims arrive
- Provider information gets updated
- Drug prices change

A naive pipeline would reload everything from scratch every day — slow and expensive.
CDC captures only what **changed** and applies those changes incrementally.

---

## Delta MERGE — The SQL Syntax

```sql
MERGE INTO target_table AS target
USING source_data AS source
ON target.join_key = source.join_key

WHEN MATCHED THEN UPDATE SET
    target.col1 = source.col1,
    target.col2 = source.col2

WHEN NOT MATCHED THEN INSERT (col1, col2, col3)
VALUES (source.col1, source.col2, source.col3)
```

### The 3 MERGE conditions:
| Condition | Meaning | Action |
|---|---|---|
| WHEN MATCHED | Row exists in target, key found in source | UPDATE |
| WHEN NOT MATCHED | New row in source, key not in target | INSERT |
| WHEN NOT MATCHED BY SOURCE | Row in target, key missing in source | DELETE (optional) |

---

## Why silver_provider_summary is the MERGE Target

`silver_providers` has many rows per NPI (one per procedure billed).
MERGE requires exactly **one row per key** in the target table.

`silver_provider_summary` aggregates all procedures per NPI into one row using `groupBy(provider_npi)` — making it safe to MERGE on `provider_npi` as the unique key.

| Table | Rows | Rows per NPI | MERGE safe? |
|---|---|---|---|
| silver_providers | 592,965 | Many | ❌ No |
| silver_provider_summary | 73,673 | 1 | ✅ Yes |

---

## Our Implementation — Monthly Provider Update

### Scenario
On the 1st of each month, CMS releases updated provider billing data.
Some providers have updated payment amounts. Some new providers are added.
We UPDATE changed providers and INSERT new ones — without reloading all 73K rows.

---

## Notebook: 04_incremental_merge

### Cell 1 — Configuration
```python
CATALOG = "adb_retailedge_dev"
SCHEMA  = "healthcare_cms"
TARGET  = f"{CATALOG}.{SCHEMA}.silver_provider_summary"

print(f"Target table : {TARGET}")
```
Output:
```
Target table : adb_retailedge_dev.healthcare_cms.silver_provider_summary
```

---

### Cell 2 — Snapshot row count BEFORE merge
```python
df_before    = spark.table(TARGET)
count_before = df_before.count()
print(f"Rows BEFORE merge: {count_before:,}")
```
Output:
```
Rows BEFORE merge: 73,673
```

---

### Cell 3 — Simulate incoming "new month" data
```python
from pyspark.sql.functions import col, round as spark_round
from pyspark.sql import Row

# 200 existing providers with updated billing numbers
df_updates = spark.table(TARGET) \
    .limit(200) \
    .withColumn("avg_medicare_payment", spark_round(col("avg_medicare_payment") * 1.05, 2)) \
    .withColumn("avg_submitted_charge", spark_round(col("avg_submitted_charge") * 1.03, 2)) \
    .withColumn("avg_allowed_amount",   spark_round(col("avg_allowed_amount")   * 1.04, 2)) \
    .withColumn("total_services",       spark_round(col("total_services")       * 1.10, 2)) \
    .withColumn("total_beneficiaries",  (col("total_beneficiaries") * 1.08).cast("long")) \
    .withColumn("total_procedures",     (col("total_procedures") + 5).cast("long"))

# 5 brand new providers not in target
new_providers = spark.createDataFrame([
    Row(provider_npi="9990000001", provider_name="NEW HEALTH CLINIC",     provider_first_name="JAMES",
        city="AUSTIN",   state="TX", zip_code="78701", provider_type="Internal Medicine",
        total_procedures=12, total_beneficiaries=320,  total_services=980.0,
        avg_medicare_payment=185.50, avg_submitted_charge=310.00, avg_allowed_amount=210.75),
    Row(provider_npi="9990000002", provider_name="SUNRISE MEDICAL GROUP", provider_first_name="SARA",
        city="PHOENIX",  state="AZ", zip_code="85001", provider_type="Family Practice",
        total_procedures=8,  total_beneficiaries=210,  total_services=640.0,
        avg_medicare_payment=162.00, avg_submitted_charge=275.00, avg_allowed_amount=190.20),
    Row(provider_npi="9990000003", provider_name="VALLEY CARE CENTER",    provider_first_name="MICHAEL",
        city="DENVER",   state="CO", zip_code="80201", provider_type="Cardiology",
        total_procedures=20, total_beneficiaries=450,  total_services=1200.0,
        avg_medicare_payment=320.00, avg_submitted_charge=510.00, avg_allowed_amount=365.00),
    Row(provider_npi="9990000004", provider_name="LAKESIDE PHYSICIANS",   provider_first_name="EMILY",
        city="CHICAGO",  state="IL", zip_code="60601", provider_type="Orthopedic Surgery",
        total_procedures=15, total_beneficiaries=280,  total_services=890.0,
        avg_medicare_payment=425.00, avg_submitted_charge=720.00, avg_allowed_amount=490.00),
    Row(provider_npi="9990000005", provider_name="COAST MEDICAL ASSOC",   provider_first_name="DAVID",
        city="SEATTLE",  state="WA", zip_code="98101", provider_type="Neurology",
        total_procedures=18, total_beneficiaries=390,  total_services=1100.0,
        avg_medicare_payment=380.00, avg_submitted_charge=620.00, avg_allowed_amount=430.00),
])

df_incoming = df_updates.union(new_providers)

print(f"Incoming batch size : {df_incoming.count():,} rows")
print(f"  - Updated records : 200")
print(f"  - New records     : 5")
```
Output:
```
Incoming batch size : 205 rows
  - Updated records : 200
  - New records     : 5
```

---

### Cell 4 — Register as Temp View
```python
df_incoming.createOrReplaceTempView("incoming_providers")
print("Temp view 'incoming_providers' registered.")
```
Output:
```
Temp view 'incoming_providers' registered.
```

**Why:** Delta MERGE uses SQL syntax. SQL cannot read a PySpark DataFrame directly —
it must be registered as a temp view first.

---

### Cell 5 — Execute Delta MERGE
```python
spark.sql(f"""
    MERGE INTO {TARGET} AS target
    USING incoming_providers AS source
    ON target.provider_npi = source.provider_npi

    WHEN MATCHED THEN UPDATE SET
        target.avg_medicare_payment = source.avg_medicare_payment,
        target.avg_submitted_charge = source.avg_submitted_charge,
        target.avg_allowed_amount   = source.avg_allowed_amount,
        target.total_services       = source.total_services,
        target.total_beneficiaries  = source.total_beneficiaries,
        target.total_procedures     = source.total_procedures

    WHEN NOT MATCHED THEN INSERT (
        provider_npi,
        provider_name,
        provider_first_name,
        city,
        state,
        zip_code,
        provider_type,
        total_procedures,
        total_beneficiaries,
        total_services,
        avg_medicare_payment,
        avg_submitted_charge,
        avg_allowed_amount
    ) VALUES (
        source.provider_npi,
        source.provider_name,
        source.provider_first_name,
        source.city,
        source.state,
        source.zip_code,
        source.provider_type,
        source.total_procedures,
        source.total_beneficiaries,
        source.total_services,
        source.avg_medicare_payment,
        source.avg_submitted_charge,
        source.avg_allowed_amount
    )
""")

print("MERGE complete.")
```
Output:
```
MERGE complete.
```

**Common mistake:** Do NOT put a trailing comma after the last column in UPDATE SET.
SQL parser throws PARSE_SYNTAX_ERROR if you do.

---

### Cell 6 — Verify row counts after merge
```python
count_after = spark.table(TARGET).count()
new_rows    = count_after - count_before

print(f"Rows BEFORE merge : {count_before:,}")
print(f"Rows AFTER  merge : {count_after:,}")
print(f"Net new rows      : {new_rows:,}  (expected: 5)")
```
Output:
```
Rows BEFORE merge : 73,673
Rows AFTER  merge : 73,678
Net new rows      : 5  (expected: 5)
```

---

### Cell 7 — DESCRIBE HISTORY
```python
display(spark.sql(f"DESCRIBE HISTORY {TARGET}"))
```

**Actual history observed:**

| Version | Operation | Key Metrics |
|---|---|---|
| 0 | CREATE OR REPLACE | 73,673 rows written (original Silver) |
| 1 | CREATE OR REPLACE | 73,673 rows (Silver re-run) |
| 2 | MERGE | 200 updated, 5 inserted ✅ |
| 3 | MERGE | 205 updated, 0 inserted (idempotent re-run) |
| 4 | MERGE | 205 updated, 0 inserted (idempotent re-run) |

**Key observation — MERGE is idempotent:**
Re-running the same MERGE never creates duplicates. The 5 new providers inserted in version 2
were found via WHEN MATCHED in versions 3 and 4 — they got updated instead of re-inserted.
This is why MERGE is safe to retry if a pipeline fails.

---

### Cell 8 — Get version numbers for time travel
```python
history_df    = spark.sql(f"DESCRIBE HISTORY {TARGET}")
latest_version    = history_df.selectExpr("max(version)").collect()[0][0]
pre_merge_version = latest_version - 1

print(f"Latest version    : {latest_version}  (post-merge)")
print(f"Pre-merge version : {pre_merge_version}")
```
Output:
```
Latest version    : 4  (post-merge)
Pre-merge version : 3
```

---

### Cell 9 — Time travel: query pre-merge snapshot
```python
df_time_travel = spark.sql(f"""
    SELECT provider_npi, provider_name, state, avg_medicare_payment
    FROM {TARGET} VERSION AS OF {pre_merge_version}
    LIMIT 5
""")

print(f"Pre-merge snapshot (version {pre_merge_version}):")
display(df_time_travel)
```
Output (sample):
```
provider_npi  | provider_name  | state | avg_medicare_payment
1003002049    | Srinivasan     | CA    | 97.77
1003016957    | Mathur         | CT    | 193.16
1003019019    | Dudney         | MO    | 200.60
```

---

### Cell 10 — Confirm new providers across versions
```python
new_npi_list = "('9990000001','9990000002','9990000003','9990000004','9990000005')"

count_current   = spark.sql(f"""
    SELECT COUNT(*) FROM {TARGET}
    WHERE provider_npi IN {new_npi_list}
""").collect()[0][0]

count_pre_merge = spark.sql(f"""
    SELECT COUNT(*) FROM {TARGET} VERSION AS OF 1
    WHERE provider_npi IN {new_npi_list}
""").collect()[0][0]

print(f"New providers in current version  : {count_current}   (expected: 5)")
print(f"New providers in version 1        : {count_pre_merge}  (expected: 0)")
```
Output:
```
New providers in current version  : 5   (expected: 5)
New providers in version 1        : 0   (expected: 0)
```

---

### Cell 11 — Spot check: compare one provider before and after MERGE
```python
sample_npi = spark.sql(f"""
    SELECT provider_npi FROM {TARGET} VERSION AS OF 1 LIMIT 1
""").collect()[0][0]

print(f"Spot-checking NPI: {sample_npi}\n")

print("--- BEFORE merge (version 1) ---")
display(spark.sql(f"""
    SELECT provider_npi, avg_medicare_payment, avg_submitted_charge, total_services
    FROM {TARGET} VERSION AS OF 1
    WHERE provider_npi = '{sample_npi}'
"""))

print("--- AFTER merge (current) ---")
display(spark.sql(f"""
    SELECT provider_npi, avg_medicare_payment, avg_submitted_charge, total_services
    FROM {TARGET}
    WHERE provider_npi = '{sample_npi}'
"""))
```
Output (NPI: 1003000639):
```
BEFORE: avg_medicare_payment=423.90  avg_submitted_charge=3522.61  total_services=128
AFTER:  avg_medicare_payment=445.09  avg_submitted_charge=3628.29  total_services=140.8
```

Verification:
- 423.90 × 1.05 = 445.09 ✅
- 3522.61 × 1.03 = 3628.29 ✅
- 128 × 1.10 = 140.8 ✅

---

## Results Summary

| Step | Result |
|---|---|
| Incoming batch | 205 rows (200 updates + 5 new) |
| MERGE: MATCHED → UPDATE | 200 existing providers updated |
| MERGE: NOT MATCHED → INSERT | 5 new providers inserted |
| Row count before | 73,673 |
| Row count after | 73,678 (+5) |
| Delta versions created | 4 (including idempotent re-runs) |
| Time travel verified | Pre-merge snapshot queryable via VERSION AS OF |
| Spot check | Multipliers applied exactly ✅ |

---

## Errors Encountered

| Error | Cause | Fix |
|---|---|---|
| NameError: spark_round not defined | Import line missing from cell | Always put imports at top of each cell |
| PARSE_SYNTAX_ERROR: missing EQ | Trailing comma after last column in UPDATE SET | Remove comma after last SET column |
| NameError: count_pre_merge not defined | Typo — written as `ount_pre_merge` | Fixed variable name |
| Spot check returned empty "before" table | `.limit(1)` picked a fake NPI (9990000005) inserted by MERGE — doesn't exist in version 1 | Query `VERSION AS OF 1` to get a real CMS provider NPI |

---

## Interview Q&A

| Question | Answer |
|---|---|
| How do you handle incremental loads in Databricks? | Delta MERGE — upsert pattern on a unique key |
| What is MERGE idempotency? | Re-running MERGE with same data updates existing rows instead of duplicating them — safe to retry on pipeline failure |
| What is SCD Type 1? | Overwrite old value with new — exactly what our MERGE UPDATE does |
| What is SCD Type 2? | Keep old row, insert new row with is_current flag and timestamps — needs extra columns |
| How do you audit changes in Delta Lake? | DESCRIBE HISTORY shows every operation with row-level metrics |
| How do you recover from a bad MERGE? | Time travel — RESTORE TABLE to a previous version |
| Why use silver_provider_summary instead of silver_providers as the MERGE target? | MERGE needs one row per key — silver_providers has many rows per NPI (one per procedure). summary has exactly one row per NPI. |

---

## Status

- [x] Incremental dataset simulated (200 updates + 5 new providers)
- [x] MERGE statement written and tested
- [x] DESCRIBE HISTORY confirms MERGE operation
- [x] Idempotent re-run verified (no duplicates on second run)
- [x] Time travel query tested (VERSION AS OF)
- [x] Row counts verified before and after MERGE — net +5 rows
- [x] Spot check: multipliers verified on real NPI (1003000639)

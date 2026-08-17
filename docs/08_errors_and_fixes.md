# Errors & Fixes Log

All errors encountered during the build — documented with root cause, fix, and lesson learned.

---

## Error 1 — ADLS Access from Serverless Compute
**Day:** Day 3 — Bronze layer  
**Step:** Testing ADLS connection from Databricks notebook  
**Error:** `AZURE_INVALID_CREDENTIALS_CONFIGURATION — Invalid configuration value detected for fs.azure.account.key`  
**Root Cause:** Unity Catalog with serverless compute blocks `spark.conf.set()` for storage keys — this method only works on classic clusters.  
**Fix:**
1. Created Access Connector for Azure Databricks (`connector-healthcare`) in Azure Portal
2. Granted Storage Blob Data Contributor IAM role on `retailedgestorage1`
3. Created Storage Credential in Unity Catalog (`credential_healthcare`)
4. Created External Location (`loc_healthcare`) → `abfss://healthcare@retailedgestorage1.dfs.core.windows.net/`

**Lesson:** On serverless + Unity Catalog, always use External Location + Storage Credential. Never use `spark.conf.set()` with storage keys.

---

## Error 2 — PATH_NOT_FOUND for Inpatient File
**Day:** Day 3 — Bronze layer  
**Step:** Reading inpatient CSV from ADLS  
**Error:** `PATH_NOT_FOUND: Path does not exist: .../raw/inpatient/cms_inpatient_2024.csv`  
**Root Cause:** File was uploaded with uppercase `.CSV` extension but code referenced lowercase `.csv`. ADLS paths are case-sensitive.  
**Fix:** Renamed file in ADLS to lowercase `.csv`  
**Lesson:** Always verify exact filenames using `dbutils.fs.ls("abfss://...")` before hardcoding paths. ADLS is case-sensitive unlike Windows.

---

## Error 3 — TypeError: 'int' object is not callable
**Day:** Day 4 — Silver layer  
**Step:** silver_provider_summary groupBy/agg  
**Error:** `TypeError: 'int' object is not callable`  
**Root Cause:** `from pyspark.sql.functions import *` overwrites Python built-ins. `count`, `sum`, `round` become PySpark functions but conflict when re-imported.  
**Fix:** Import explicitly with aliases: `from pyspark.sql.functions import count as spark_count, sum as spark_sum`  
**Lesson:** Never use `from pyspark.sql.functions import *` — always import explicitly to avoid conflicts with Python built-ins (count, sum, round, min, max).

---

## Error 4 — TABLE_OR_VIEW_NOT_FOUND silver_provider_summery
**Day:** Day 4 — Silver layer  
**Step:** Reading silver_provider_summary in subsequent cells  
**Error:** `TABLE_OR_VIEW_NOT_FOUND: silver_provider_summery`  
**Root Cause:** Typo when saving — wrote `summery` instead of `summary`  
**Fix:** `ALTER TABLE silver_provider_summery RENAME TO silver_provider_summary`  
**Lesson:** Double-check table names before writing. Use a config variable at the top of the notebook instead of repeating string literals.

---

## Error 5 — NameError: spark_round not defined
**Day:** Day 5 — Delta MERGE  
**Step:** Cell 3 — building incoming batch DataFrame  
**Error:** `NameError: name 'spark_round' is not defined`  
**Root Cause:** Import line was missing from the cell. Each Databricks cell runs independently — imports from previous cells do not carry over if the kernel was restarted.  
**Fix:** Added `from pyspark.sql.functions import round as spark_round` at the top of the cell  
**Lesson:** Always put imports inside the cell that uses them, not in a separate earlier cell.

---

## Error 6 — PARSE_SYNTAX_ERROR in MERGE statement
**Day:** Day 5 — Delta MERGE  
**Step:** Cell 5 — executing MERGE SQL  
**Error:** `PARSE_SYNTAX_ERROR: Syntax error at or near 'NOT': missing EQ`  
**Root Cause:** Trailing comma after the last column in `UPDATE SET` block — SQL does not allow comma after last item  
**Fix:** Removed the trailing comma after `target.total_procedures = source.total_procedures`  
**Lesson:** In SQL MERGE, UPDATE SET columns are comma-separated — no trailing comma after the last one.

---

## Error 7 — DELTA_MULTIPLE_SOURCE_ROW_MATCHING_TARGET_ROW_IN_MERGE
**Day:** Day 5 — Delta MERGE  
**Step:** Cell 5 — MERGE execution  
**Error:** `DELTA_MULTIPLE_SOURCE_ROW_MATCHING_TARGET_ROW_IN_MERGE: multiple source rows matched and attempted to modify the same target row`  
**Root Cause:** `df_updates = spark.table(TARGET).limit(200)` reads from the same table being merged into. PySpark DataFrames are lazy — they re-evaluate multiple times during MERGE execution. When the target table is being modified mid-execution, different rows are returned on each evaluation, causing apparent duplicate matches.  
**Fix:** Materialized the incoming DataFrame to a staging Delta table first, then ran MERGE from that stable snapshot:
```python
df_incoming.write.format("delta").mode("overwrite") \
    .saveAsTable(f"{CATALOG}.{SCHEMA}.staging_incoming_providers")

spark.sql("MERGE INTO target USING staging_incoming_providers ...")
```
**Lesson:** Never use a temp view or lazy DataFrame built from the target table as the MERGE source. Always materialize to a staging Delta table first — a committed snapshot is stable and gives MERGE the same data every time.

---

## Error 8 — ADF Notebook Activity: Cluster Configuration Required
**Day:** Day 7 — ADF Orchestration  
**Step:** Pipeline validation after adding Notebook Activities  
**Error:** `Cluster configuration is required for this activity. Please select a non-serverless option`  
**Root Cause:** ADF Notebook Activity does not support Databricks serverless clusters — it can only control classic job clusters where it specifies VM size and node count.  
**Fix:** Changed linked service `ls_databricks` from serverless to New Job Cluster with Runtime 13.3.x-photon and Standard_D4s_v3 node type  
**Lesson:** ADF Notebook Activity requires a classic job cluster. Serverless is only usable interactively from Databricks notebooks — not via ADF.

---

## Error 9 — Databricks Credits Exhausted
**Day:** Day 7 — ADF Orchestration  
**Step:** First pipeline test run  
**Error:** `Sorry, cannot run Cluster because you've exhausted your available credits`  
**Root Cause:** Databricks 14-day free DBU trial had expired. Azure credits (₹16,008) and Databricks DBU credits are two separate billing systems — Azure credits do not automatically cover Databricks DBU costs.  
**Fix:** Upgraded Databricks workspace from Trial to Premium Pay-As-You-Go — this routes DBU billing through Azure credits  
**Lesson:** When using Azure free account with Databricks, upgrade to Premium Pay-As-You-Go early so DBU costs are covered by Azure credits. The 14-day DBU trial expires independently of Azure credits.

---

## Error 10 — CLOUD_PROVIDER_RESOURCE_STOCKOUT
**Day:** Day 7 — ADF Orchestration  
**Step:** Pipeline test run after billing fix  
**Error:** `Standard_DS3_v2 is currently not available in CentralIndia`  
**Root Cause:** VM size not available in Central India region at that time — cloud capacity issue  
**Fix:** Changed node type from `Standard_DS3_v2` to `Standard_D4s_v3`  
**Lesson:** Always have a backup VM size ready. Central India region has limited VM availability on free/trial accounts.

---

## Error 11 — Spark Version Not Supported
**Day:** Day 7 — ADF Orchestration  
**Step:** Pipeline test run  
**Error:** `Spark version 12.2.x-scala2.12 isn't supported because both legacy access and legacy DBFS are disabled`  
**Root Cause:** Workspace has legacy features disabled — requires Runtime 13.3 LTS or above  
**Fix:** Changed cluster version to `13.3.x-photon-scala2.12`  
**Lesson:** When Unity Catalog is enabled with legacy features disabled, always use Runtime 13.3 LTS or above.

---

## Error 12 — AZURE_QUOTA_EXCEEDED
**Day:** Day 7 — ADF Orchestration  
**Step:** Pipeline test run  
**Error:** `Operation could not be completed as it results in exceeding approved Total Regional Cores quota. Current Limit: 4, Additional Required: 8`  
**Root Cause:** Azure free account has a 4 vCore quota limit in Central India. Standard_D4s_v3 requires 4 cores per node × 2 nodes = 8 cores total — exceeds quota.  
**Fix:** Could not resolve within free account limits — pipeline architecture is fully built and validated. Full end-to-end run blocked by infrastructure quota constraint only.  
**Lesson:** Azure free accounts have strict regional CPU quotas (4 cores in Central India). For job clusters, use the smallest available 2-core VM or request a quota increase.

---

## Known Issues to Watch For

| Issue | What Happens | Prevention |
|---|---|---|
| `from pyspark.sql.functions import *` | Conflicts with Python built-ins (count, sum, round) | Always import explicitly with aliases |
| `.save(path)` + LOCATION | Fails with Unity Catalog | Use `.saveAsTable("catalog.schema.table")` |
| `/dbfs/` or `dbfs:/` paths | Fails on Unity Catalog workspaces | Use `abfss://` paths only |
| `CREATE CATALOG` on Azure | Requires storage root URL | Use existing catalog — never create new |
| `cast()` on dirty data | Crashes on invalid values | Use `expr("try_cast(col as type)")` |
| Temp view as MERGE source | DELTA_MULTIPLE_SOURCE_ROW_MATCHING | Materialize to staging Delta table first |
| Serverless cluster in ADF | Notebook Activity not supported | Use New Job Cluster in linked service |

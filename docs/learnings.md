# Project Learnings & Concepts

Key concepts learned during the build — explained simply with interview answers.
This file grows as we encounter new things throughout the project.

---

## Index

1. [How Databricks Connects to ADLS Gen2](#1-how-databricks-connects-to-adls-gen2)
2. [Delta MERGE — Never Read From the Target Table as Source](#2-delta-merge--never-read-from-the-target-table-as-source)
3. [ADF + Databricks — Two Separate Billing Systems](#3-adf--databricks--two-separate-billing-systems)
4. [ADF Notebook Activity — Serverless Not Supported](#4-adf-notebook-activity--serverless-not-supported)

---

## 1. How Databricks Connects to ADLS Gen2

**When encountered:** Day 3 — Bronze layer setup  
**Error that triggered it:** `AZURE_INVALID_CREDENTIALS_CONFIGURATION`

---

### The Problem

Databricks needs permission to read files from Azure Storage. On serverless compute with Unity Catalog, you cannot use `spark.conf.set()` with storage keys — Unity Catalog requires a proper managed identity instead.


---

### Visual Flow — How Everything Connects

```
┌─────────────────────────────────────────────────────────────────┐
│                        AZURE PORTAL                             │
│                                                                 │
│   ┌─────────────────────────┐                                   │
│   │  retailedgestorage1     │  ← your ADLS storage account      │
│   │  (ADLS Gen2)            │                                   │
│   │                         │                                   │
│   │  Container: healthcare  │                                   │
│   │  └── raw/               │                                   │
│   │      ├── providers/     │                                   │
│   │      ├── drugs/         │                                   │
│   │      └── inpatient/     │                                   │
│   └────────────┬────────────┘                                   │
│                │                                                │
│                │  IAM Role Assignment                           │
│                │  Storage Blob Data Contributor                 │
│                │  ↓ grants read/write permission to ↓           │
│                │                                                │
│   ┌────────────▼────────────┐                                   │
│   │  Access Connector       │  ← managed identity for          │
│   │  connector-healthcare   │    Databricks (no password)       │
│   └────────────┬────────────┘                                   │
│                │                                                │
│                │  Resource ID copied from here                  │
└────────────────┼────────────────────────────────────────────────┘
                 │
                 │ paste Resource ID
                 ↓
┌─────────────────────────────────────────────────────────────────┐
│                    UNITY CATALOG (Databricks)                   │
│                                                                 │
│   ┌─────────────────────────┐                                   │
│   │  Storage Credential     │  ← tells Unity Catalog            │
│   │  credential_healthcare  │    "use this identity"            │
│   └────────────┬────────────┘                                   │
│                │                                                │
│                │  paired with URL                               │
│                ↓                                                │
│   ┌─────────────────────────────────────────────────┐          │
│   │  External Location: loc_healthcare              │          │
│   │  URL: abfss://healthcare@                       │          │
│   │       retailedgestorage1.dfs.core.windows.net/  │          │
│   └────────────┬────────────────────────────────────┘          │
│                │                                                │
└────────────────┼────────────────────────────────────────────────┘
                 │
                 │ now any notebook can use this path
                 ↓
┌─────────────────────────────────────────────────────────────────┐
│                   DATABRICKS NOTEBOOK                           │
│                                                                 │
│   spark.read.csv(                                               │
│     "abfss://healthcare@retailedgestorage1                      │
│      .dfs.core.windows.net/raw/drugs/..."                       │
│   )                                                             │
│                                                                 │
│   ✓ Unity Catalog matches path → loc_healthcare                 │
│   ✓ Uses credential_healthcare → connector-healthcare           │
│   ✓ connector-healthcare has Storage Blob Data Contributor      │
│   ✓ File read successfully                                      │
└─────────────────────────────────────────────────────────────────┘
```

---

### Setup Order (What to Create First)

```
STEP 1 ──► Create Access Connector       (Azure Portal)
               │
               ▼
STEP 2 ──► Assign IAM Role               (Azure Portal → Storage Account)
           Storage Blob Data Contributor
           on retailedgestorage1
               │
               ▼
STEP 3 ──► Create Storage Credential     (Unity Catalog)
           paste Access Connector ID
               │
               ▼
STEP 4 ──► Create External Location      (Unity Catalog)
           paste ADLS URL + pick credential
               │
               ▼
STEP 5 ──► Read from ADLS in Notebook    (Databricks)
           spark.read.csv("abfss://...")  ✓
```

---

![[Pasted image 20260806215956.png]]
  Corrected Steps in Simple Terms:

  1. To read ADLS from Databricks → need an External Location in Unity Catalog

  2. External Location needs two things:
     → a Storage Credential (who has access)
     → a URL (which storage path)

  3. To create Storage Credential → need an Access Connector ID

  4. So first created Access Connector for Azure Databricks
     (this is a managed identity — like a service account for Databricks)

  5. Then went to the STORAGE ACCOUNT (retailedgestorage1) → IAM
     → Added role: Storage Blob Data Contributor
     → Assigned to: connector-healthcare
     (this gives the Access Connector permission to read/write the storage)

  6. Copied the Access Connector Resource ID
     → Pasted into Unity Catalog → Storage Credential (credential_healthcare)

  7. Created External Location (loc_healthcare)
     → URL: abfss://healthcare@retailedgestorage1.dfs.core.windows.net/
     → Credential: credential_healthcare

  8. Now any notebook can read from that ADLS path ✓

  ---
  The key distinction to remember for interviews:

  ┌─────────────────────┬────────────────────────────────┐
  │        Step         │        Where it happens        │
  ├─────────────────────┼────────────────────────────────┤
  │ Access Connector    │ Azure Portal                   │
  ├─────────────────────┼────────────────────────────────┤
  │ IAM role assignment │ Azure Portal → Storage Account │
  ├─────────────────────┼────────────────────────────────┤
  │ Storage Credential  │ Databricks Unity Catalog       │
  ├─────────────────────┼────────────────────────────────┤
  │ External Location   │ Databricks Unity Catalog       │
  └─────────────────────┴────────────────────────────────┘

![[Screenshot 2026-08-06 212334.png|507]]

![[Screenshot 2026-08-06 210722.png|504]]

![[Screenshot 2026-08-06 211209.png]]
### Step 1 — Access Connector for Azure Databricks

**What it is:** A special Azure resource that acts as a **managed identity** — an identity that Azure recognizes without needing a username/password.

**Why needed:** Serverless Databricks cannot use `spark.conf.set()` with storage keys. Unity Catalog requires a proper managed identity instead.

**What we did:** Created `connector-healthcare` in Azure Portal (Central India region).

---

### Step 2 — IAM Role Assignment on Storage Account

**What it is:** A permission that says *"this identity is allowed to read/write this storage account"*.

**Why needed:** Just creating the Access Connector is not enough — Azure doesn't automatically trust it. We had to explicitly grant it rights.

**What we did:**
```
retailedgestorage1
→ Access Control (IAM)
→ Add role assignment
→ Storage Blob Data Contributor
→ Assigned to: connector-healthcare (managed identity)
```

---

### Step 3 — Storage Credential in Unity Catalog

**What it is:** Unity Catalog's internal record of the Access Connector identity.

**Why needed:** Unity Catalog manages all data access centrally — individual notebooks cannot connect to storage directly. Everything goes through Unity Catalog.

**What we did:** Created `credential_healthcare` in:
```
Databricks → Catalog → create a credential → Storage Credentials
→ Pasted the Access Connector Resource ID
```

---

### Step 4 — External Location in Unity Catalog

**What it is:** A pointer that maps a specific ADLS path to a Storage Credential.

**Why needed:** The Storage Credential alone doesn't tell Databricks which storage path to use. The External Location connects the credential to a specific ADLS path.

**What we did:** Created `loc_healthcare` pointing to:
```
abfss://healthcare@retailedgestorage1.dfs.core.windows.net/
```

---

### How It Works End to End

When you run `spark.read.csv("abfss://healthcare@retailedgestorage1...")`:

1. Databricks sees the `abfss://healthcare@retailedgestorage1` path
2. Checks Unity Catalog → finds External Location `loc_healthcare` matches this path
3. Uses Storage Credential `credential_healthcare` to get the Access Connector identity
4. Uses that identity (which has Storage Blob Data Contributor on retailedgestorage1) to read the file
5. Returns the data to your notebook

---

### Interview Answer

> "In Unity Catalog with serverless compute, you connect Databricks to ADLS using three components:
> First, create an **Access Connector for Azure Databricks** in Azure Portal and assign it the
> **Storage Blob Data Contributor** IAM role on the storage account.
> Second, register that Access Connector as a **Storage Credential** in Unity Catalog.
> Third, create an **External Location** in Unity Catalog that maps the ADLS path to the Storage Credential.
> After that, any notebook can read from that ADLS path directly — no keys or passwords needed."

---

### Resources Created

| Resource | Name | Where |
|----------|------|-------|
| Access Connector | `connector-healthcare` | Azure Portal |
| IAM Role | Storage Blob Data Contributor | retailedgestorage1 → IAM |
| Storage Credential | `credential_healthcare` | Unity Catalog |
| External Location | `loc_healthcare` | Unity Catalog |

---

---

## 2. Delta MERGE — Never Read From the Target Table as Source

**When encountered:** Day 5 — Delta MERGE incremental load  
**Error that triggered it:** `DELTA_MULTIPLE_SOURCE_ROW_MATCHING_TARGET_ROW_IN_MERGE`

---

### The Problem

We built the incoming batch like this:

```python
df_updates = spark.table(TARGET).limit(200)  # reads from target table
df_incoming = df_updates.union(new_providers)
df_incoming.createOrReplaceTempView("incoming_providers")

spark.sql("MERGE INTO target USING incoming_providers ...")
```

Even though `df_incoming.count()` showed 205 rows with 0 duplicates, MERGE still threw:

```
DELTA_MULTIPLE_SOURCE_ROW_MATCHING_TARGET_ROW_IN_MERGE:
multiple source rows matched and attempted to modify the same target row
```

---

### Why It Happened

PySpark DataFrames are **lazy** — they don't actually read data when you define them.
They only read when an action is triggered (`.count()`, `.show()`, `.collect()`).

When MERGE executes, it evaluates the source DataFrame **multiple times internally**.
Since `df_updates` reads from the same table being merged into (`TARGET`), and that table
is being modified mid-execution, Spark sees different rows on different evaluations —
causing apparent duplicates that MERGE cannot resolve.

```
MERGE execution start
  → evaluate source (reads TARGET at time T1) → 205 rows, no duplicates
  → begin modifying target rows
  → re-evaluate source (reads TARGET at time T2, partially modified) → different rows
  → same NPI found in two source evaluations → AMBIGUOUS MATCH → ERROR
```

---

### The Fix — Materialize Source to a Staging Table First

```python
# Write incoming batch to a stable staging Delta table
df_incoming.write \
    .format("delta") \
    .mode("overwrite") \
    .option("overwriteSchema", "true") \
    .saveAsTable(f"{CATALOG}.{SCHEMA}.staging_incoming_providers")

# MERGE from the stable staging table — not from a temp view or DataFrame
spark.sql(f"""
    MERGE INTO {TARGET} AS target
    USING {CATALOG}.{SCHEMA}.staging_incoming_providers AS source
    ON target.provider_npi = source.provider_npi
    ...
""")
```

A Delta table is a **stable, committed snapshot** — MERGE reads the same data every time
it evaluates the source, no ambiguity possible.

---

### Why Temp View Did Not Fix It

A temp view is just a **pointer to the DataFrame** — not a materialized copy.
`createOrReplaceTempView("incoming_providers")` does not actually execute the DataFrame.
When MERGE queries the temp view, it still re-evaluates the underlying lazy DataFrame
multiple times → same problem as before.

```
Temp View  →  still lazy  →  still reads TARGET mid-merge  →  still ambiguous
Delta Table  →  committed snapshot  →  stable  →  MERGE succeeds
```

---

### Production Best Practice

```
Step 1 — Build incoming DataFrame (from any source)
Step 2 — Write to staging Delta table  ← materialize here
Step 3 — Run MERGE from staging table
Step 4 — (Optional) Drop staging table after MERGE
```

Always materialize the source before MERGE — especially when source data is derived
from the same table being merged into.

---

### Interview Answer

> "Delta MERGE requires that each target row matches at most one source row.
> If you build the source by reading from the same table you are merging into,
> Spark re-evaluates the lazy DataFrame multiple times during MERGE execution,
> which can produce different results on each evaluation and cause ambiguous matches.
> The fix is to always materialize the source into a staging Delta table first —
> a committed Delta snapshot is stable and gives MERGE the same data every time."

---

### Key Rule to Remember

| Source Type | Stable for MERGE? | Why |
|---|---|---|
| `spark.table(TARGET).limit(200)` | ❌ No | Lazy — re-evaluated mid-merge |
| `createOrReplaceTempView(...)` | ❌ No | Still lazy — just a pointer |
| Staging Delta table | ✅ Yes | Committed snapshot — always same data |

---

## 3. ADF + Databricks — Two Separate Billing Systems

**When encountered:** Day 7 — ADF Orchestration  
**Error that triggered it:** `Sorry, cannot run Cluster — you've exhausted your available credits`

---

### The Problem

Azure free account had ₹16,008 in credits but the job cluster still failed to start with a billing error.

---

### Why It Happened

Azure credits and Databricks DBU credits are **two completely separate billing systems:**

```
Azure Credits (₹16,008)
→ Pays for: VMs, Storage, ADF, Networking
→ Does NOT automatically cover Databricks DBU costs

Databricks DBU Credits (14-day free trial)
→ Pays for: Databricks cluster runtime, notebook execution
→ Expires independently of Azure credits
```

Even though Azure credits were full, the Databricks DBU trial had expired → cluster blocked.

**Indian analogy:**
- Azure credits = Jio recharge (pays for the internet/data)
- Databricks DBUs = Hotstar subscription (pays for the content)
- Even if Jio is full, expired Hotstar means you can't watch

---

### The Fix

Upgrade Databricks workspace from **Trial** to **Premium Pay-As-You-Go**:

```
Azure Portal → Azure Databricks → your workspace
→ Overview → Pricing Tier: Trial (Click to change)
→ Select: Premium (Pay-As-You-Go)
```

This routes Databricks DBU billing through your Azure subscription → covered by Azure credits.

---

### Interview Answer

> "Azure credits and Databricks DBU credits are separate billing systems.
> Azure credits cover Azure infrastructure (VMs, storage, networking).
> Databricks DBUs cover cluster compute time — billed separately on top of Azure.
> To use Azure credits for Databricks, you must upgrade to Premium Pay-As-You-Go,
> which routes DBU costs through the Azure subscription."

---

## 4. ADF Notebook Activity — Serverless Not Supported

**When encountered:** Day 7 — ADF Orchestration  
**Error that triggered it:** `Cluster configuration is required. Please select a non-serverless option`

---

### The Problem

Added Databricks Notebook Activity to ADF pipeline → validation failed with cluster configuration error.

---

### Why It Happened

**Serverless clusters** are fully managed by Databricks — you cannot configure their VM type, size, or startup. ADF Notebook Activity needs to **spin up and control** the cluster itself, which requires a classic job cluster where you specify the VM.

```
Serverless cluster  →  Databricks manages everything  →  ADF cannot control  →  ❌
Classic job cluster →  You specify VM + config        →  ADF can spin up     →  ✅
```

---

### The Fix

In the ADF linked service (`ls_databricks`), change cluster type to **New Job Cluster** and specify:

| Setting | Value |
|---|---|
| Cluster version | 13.3.x-photon-scala2.12 (LTS, legacy features disabled) |
| Node type | Smallest available VM in your region |
| Workers | 1 (minimum for Spark commands) |

---

### Key Distinctions

| Cluster Type | Use Case | ADF Compatible? |
|---|---|---|
| Serverless | Interactive notebooks, fast startup | ❌ No |
| All-purpose (interactive) | Development, shared notebooks | ❌ No |
| Job cluster (new) | Scheduled jobs, ADF pipelines | ✅ Yes |

**Job cluster advantage:** Spins up only when ADF triggers it, runs notebooks, then shuts down automatically. You are only charged for the runtime — not for idle time.

---

### Interview Answer

> "ADF Notebook Activity does not support Databricks serverless clusters — it requires a
> classic job cluster where ADF controls the VM configuration.
> The job cluster spins up when ADF triggers the pipeline and shuts down after the notebooks complete,
> so you only pay for actual compute time. This is the standard pattern for
> production ADF → Databricks orchestration."

*More learnings will be added here as we build through the project.*

# Business Insights — Medicare Analytics Findings

Answers to 4 key business questions derived from the Gold layer analytics tables.

---

## Q1 — Which Specialties Cost Medicare the Most?

**Source:** `gold_provider_performance`  
**Query:** Group by provider_type nationally, rank by avg_medicare_payment

### Results — Top 10 Most Expensive Specialties

| Specialty | Avg Medicare Payment | Avg Submitted Charge | Total Providers |
|---|---|---|---|
| Ambulatory Surgical Center | $1,104 | $6,387 | 334 |
| Thoracic Surgery | $404 | $1,852 | 141 |
| Ambulance Service Provider | $347 | $1,804 | 557 |
| Cardiac Surgery | $317 | $1,676 | 57 |
| Radiation Therapy Center | $302 | $3,655 | 5 |
| Micrographic Dermatologic Surgery | $216 | $739 | 27 |
| Neurosurgery | $214 | $1,452 | 326 |
| Opioid Treatment Program | $166 | $217 | 56 |
| IDTF | $164 | $1,271 | 114 |
| Vascular Surgery | $154 | $647 | 238 |

### Key Findings
- Ambulatory Surgical Centers charge $6,387 but Medicare only pays $1,104 — a 5.8x markup
- Cardiac Surgery submitted charges are 5.3x the Medicare payment
- Opioid Treatment Programs have the smallest markup (1.3x) — most fairly priced specialty

---

## Q2 — Which Drugs Cost Medicare the Most?

**Source:** `gold_drug_spend_analysis`  
**Query:** Order by total_spending_2024 descending + CAGR analysis

### Q2A — Top 10 Drugs by Total Medicare Spending (2024)

| Drug | Generic | Total Spending 2024 | Total Claims | Avg per Claim |
|---|---|---|---|---|
| Eliquis | Apixaban | $20.7 billion | 24,061,332 | $863 |
| Ozempic | Semaglutide | $12.9 billion | 10,417,182 | $1,245 |
| Jardiance | Empagliflozin | $11.4 billion | 11,368,280 | $1,006 |
| Mounjaro | Tirzepatide | $6.3 billion | 5,105,397 | $1,241 |
| Xarelto | Rivaroxaban | $6.2 billion | 6,660,246 | $936 |
| Trulicity | Dulaglutide | $5.4 billion | 4,305,141 | $1,267 |
| Trelegy Ellipta | Fluticasone combo | $5.3 billion | 6,250,872 | $847 |
| Farxiga | Dapagliflozin | $5.3 billion | 5,634,650 | $939 |
| Humira CF Pen | Adalimumab | $4.3 billion | 490,303 | $8,829 |
| Revlimid | Lenalidomide | $4.2 billion | 239,843 | $17,426 |

### Q2B — Fastest Growing Drug Prices (CAGR 2020–2024)

| Drug | Generic | CAGR | Total Spending 2024 |
|---|---|---|---|
| Jynneos | Smallpox/Mpox Vaccine | 8.72% | $337K |
| Lagevrio (EUA) | Molnupiravir (COVID) | 6.72% | $120.7M |
| Amphotericin B Liposome | Amphotericin B | 1.54% | $1.9M |
| Recarbrio | Imipenem combo | 1.26% | $1.2M |

### Key Findings
- **Eliquis alone = $20.7 billion** — more than the GDP of some small countries
- **Ozempic + Mounjaro (GLP-1s)** = $19.3 billion combined — fastest growing category by volume
- **Revlimid** costs $17,426 per prescription — cancer drugs are the most expensive per claim
- **Humira** costs $8,829 per claim — biologic drugs are 10x more expensive than traditional drugs
- **Mpox vaccine** has highest price growth at 8.72% annual CAGR

---

## Q3 — Which Diseases Are Most Common and Most Expensive?

**Source:** `gold_diagnosis_trends`  
**Query:** Aggregate nationally by DRG code, rank by discharges and by avg Medicare payment

### Q3A — Top 10 Most Common Diagnoses (National)

| DRG | Diagnosis | National Discharges |
|---|---|---|
| 871 | Septicemia (blood poisoning) without MV | 578,073 |
| 291 | Heart Failure and Shock with MCC | 306,135 |
| 177 | Respiratory Infections with MCC | 144,162 |
| 193 | Simple Pneumonia with MCC | 139,034 |
| 872 | Septicemia without MCC | 107,314 |
| 690 | Kidney/Urinary Tract Infections without MCC | 95,596 |
| 189 | Pulmonary Edema and Respiratory Failure | 93,765 |
| 392 | Gastroenteritis and Digestive Disorders | 90,637 |
| 280 | Acute Heart Attack (discharged alive) with MCC | 87,661 |
| 689 | Kidney/Urinary Tract Infections with MCC | 84,058 |

### Q3B — Top 5 Most Expensive Diagnoses (Avg Medicare Payment)

| DRG | Diagnosis | Avg Medicare Payment |
|---|---|---|
| 018 | CAR-T Cell Immunotherapy (Cancer) | $441,724 |
| 927 | Extensive Burns with MV + Skin Graft | $345,341 |
| 001 | Heart Transplant with MCC | $261,486 |
| 003 | ECMO or Tracheostomy with MV >96 hours | $175,946 |
| 007 | Lung Transplant | $136,472 |

### Key Findings
- **Septicemia (blood poisoning) is the #1 admission** — 578K cases nationally
- **Heart failure is #2** at 306K cases — cardiovascular disease dominates Medicare admissions
- **CAR-T cell therapy** at $441,724 per admission is 25x more expensive than the average inpatient stay
- **Heart transplant** ($261K) and **lung transplant** ($136K) are the most expensive surgical procedures
- Top 10 most common diagnoses are all chronic/acute conditions — sepsis, heart, lungs, kidneys

---

## Q4 — Which Providers Have Abnormal Billing Patterns?

**Source:** `gold_fraud_indicators`  
**Query:** Filter High Risk (>2 stddev above specialty average), rank by stddev_multiplier

### Q4A — Top 10 Highest Fraud Risk Providers

| Provider | State | Specialty | Avg Payment | Specialty Avg | Stddev Multiplier |
|---|---|---|---|---|---|
| Hyde, Tiffany | IL | Nurse Practitioner | $1,204 | $65 | 18.81x |
| Garrison, Lyddia | IL | Physical Therapist | $337 | $31 | 14.78x |
| Williams, Nikia | FL | Nurse Practitioner | $548 | $66 | 14.61x |
| Chapler, Chana | NJ | Nurse Practitioner | $593 | $70 | 13.70x |
| Smith-Litchfield, Nadia | NY | Nurse Practitioner | $388 | $61 | 12.28x |
| Bishop, Yelena | CA | Physician Assistant | $577 | $72 | 11.80x |
| Roark, Sarah | CO | Nurse Practitioner | $538 | $65 | 11.39x |
| Prasad, Keerthi | TX | Diagnostic Radiology | $866 | $45 | 11.32x |
| Niku, Soheil | CA | Diagnostic Radiology | $876 | $60 | 11.28x |
| Lakhanpal, Gaurav | MD | Internal Medicine | $891 | $86 | 11.04x |

### Q4B — States with Most High Risk Providers

| State | High Risk Providers |
|---|---|
| CA | 180 |
| TX | 170 |
| NY | 159 |
| FL | 139 |
| PA | 122 |
| OH | 96 |
| NC | 79 |
| MI | 72 |
| MA | 67 |
| IL | 64 |

### Q4C — Specialties with Most High Risk Providers

| Specialty | High Risk Providers |
|---|---|
| Nurse Practitioner | 335 |
| Physician Assistant | 206 |
| Internal Medicine | 136 |
| CRNA | 119 |
| Family Practice | 110 |
| Physical Therapist | 103 |
| Anesthesiology | 91 |
| Optometry | 70 |
| Diagnostic Radiology | 69 |
| Orthopedic Surgery | 50 |

### Key Findings
- **Tiffany Hyde (IL)** bills 18.81x above the Nurse Practitioner average — highest anomaly in the dataset
- **Nurse Practitioners dominate** the fraud risk list — 335 high-risk providers vs 206 Physician Assistants
- **California, Texas, New York** have the most high-risk providers — correlates with largest Medicare populations
- All top 10 providers bill **10–18x above their specialty peer average** — far beyond normal variation
- **Diagnostic Radiologists** Prasad and Niku both in top 10 — imaging fraud is a known Medicare issue

---

## Notebook

Name: `05_business_insights`  
Source: Gold layer tables only — no Silver or Bronze reads needed

---

## Interview Talking Points

| Question | Answer from This Data |
|---|---|
| What business insight did your pipeline produce? | Identified Eliquis as Medicare's top drug spend at $20.7B; flagged Tiffany Hyde as billing 18.81x above NP average |
| How did you detect fraud? | Calculated stddev of payments per specialty+state in Gold layer; flagged providers >2 stddev above peer average |
| What was your most surprising finding? | CAR-T cell cancer therapy costs $441K per admission — 25x the average inpatient Medicare payment |
| Why does the pipeline matter? | Medicare spends $1T+ annually — even 1% fraud reduction saves $10 billion |

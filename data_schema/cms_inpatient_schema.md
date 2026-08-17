# CMS Inpatient Claims Dataset — Schema Reference

**File:** cms_inpatient_2022.csv  
**Source:** data.cms.gov — Medicare Inpatient Hospitals by Geography and Service  
**Rows:** ~500K  

---

## Key Columns

| Column | Type | Description |
|--------|------|-------------|
| Rndrng_Prvdr_CCN | string | CMS Certification Number — hospital ID |
| Rndrng_Prvdr_Org_Name | string | Hospital/facility name |
| Rndrng_Prvdr_City | string | Hospital city |
| Rndrng_Prvdr_State_Abrvtn | string | Hospital state (2-letter code) |
| Rndrng_Prvdr_Zip5 | string | Hospital zip code |
| DRG_Cd | string | Diagnosis Related Group code |
| DRG_Desc | string | DRG description (diagnosis + procedure) |
| Tot_Dschrgs | integer | Total discharges for this DRG |
| Avg_Submtd_Cvrd_Chrg | decimal | Average submitted covered charge ($) |
| Avg_Tot_Pymt_Amt | decimal | Average total payment ($) |
| Avg_Mdcr_Pymt_Amt | decimal | Average Medicare payment ($) |

---

## Join Keys

- Join to providers on: `Rndrng_Prvdr_State_Abrvtn` + `Rndrng_Prvdr_Zip5` for geographic analysis
- `DRG_Cd` is the key for diagnosis trend analysis in Gold layer

---

## DRG Code Notes

DRG (Diagnosis Related Group) codes group hospital cases by diagnosis.  
Example:
- DRG 470 — Major Joint Replacement (hip/knee)
- DRG 291 — Heart Failure with MCC
- DRG 392 — Esophagitis, Gastroenteritis

For Gold layer diagnosis trends, group by DRG_Cd prefix or category.

---

## Data Quality Notes

- `Avg_Submtd_Cvrd_Chrg` may have formatting — strip $ and commas before casting
- `DRG_Cd` is string — do NOT cast to integer (has leading zeros)
- `Tot_Dschrgs` below 11 is suppressed by CMS for privacy — these appear as NULL or "*"
- `Rndrng_Prvdr_State_Abrvtn` includes territories (PR, VI, GU) — decide whether to include

# CMS Drug Spending Dataset — Schema Reference

**File:** cms_drugs_2022.csv  
**Source:** data.cms.gov — Medicare Part D Spending by Drug  
**Rows:** ~500K  

---

## Key Columns

| Column | Type | Description |
|--------|------|-------------|
| Brnd_Name | string | Brand name of the drug |
| Gnrc_Name | string | Generic name of the drug |
| Mftr_Name | string | Manufacturer name |
| Tot_Mftr | integer | Total manufacturers for this drug |
| Tot_Spndng | decimal | Total Medicare Part D spending ($) |
| Tot_Dsgn_Unts | decimal | Total dosage units dispensed |
| Tot_Clms | integer | Total claims |
| Tot_Benes | integer | Total beneficiaries |
| Avg_Spnd_Per_Dsgn_Unt_Wghtd | decimal | Avg spending per dosage unit |
| Avg_Spnd_Per_Clm | decimal | Avg spending per claim |
| Avg_Spnd_Per_Bene | decimal | Avg spending per beneficiary |
| Outlier_Flag | string | Flag for unusual spending pattern |

---

## Join Keys

- This dataset is provider-level aggregated — join to providers via `Rndrng_NPI` if using the by-provider version
- For total spending analysis, this table is standalone (national level)

---

## Data Quality Notes

- `Tot_Spndng` may be formatted with commas — strip before casting to decimal
- `Outlier_Flag` values: check unique values in Bronze before Silver logic
- `Brnd_Name` and `Gnrc_Name` may have trailing spaces — use trim() in Silver
- NULL `Mftr_Name` is valid — some generics have no brand manufacturer

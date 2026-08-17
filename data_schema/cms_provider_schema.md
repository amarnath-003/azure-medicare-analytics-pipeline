# CMS Provider Dataset — Schema Reference

**File:** cms_providers_2022.csv  
**Source:** data.cms.gov — Medicare Physician & Other Practitioners by Provider  
**Rows:** ~1 million  

---

## Key Columns

| Column | Type | Description |
|--------|------|-------------|
| Rndrng_NPI | string | National Provider Identifier — unique provider ID (join key) |
| Rndrng_Prvdr_Last_Org_Name | string | Provider last name or organization name |
| Rndrng_Prvdr_First_Name | string | Provider first name |
| Rndrng_Prvdr_City | string | Provider city |
| Rndrng_Prvdr_State_Abrvtn | string | Provider state (2-letter code) |
| Rndrng_Prvdr_Zip5 | string | Provider zip code |
| Rndrng_Prvdr_Type | string | Provider specialty type |
| Rndrng_Prvdr_Gndr | string | Provider gender (M/F) |
| HCPCS_Cd | string | Healthcare procedure code |
| HCPCS_Desc | string | Procedure description |
| HCPCS_Drug_Ind | string | Is this a drug service? (Y/N) |
| Place_Of_Srvc | string | Office (O) or Facility (F) |
| Tot_Benes | integer | Total Medicare beneficiaries served |
| Tot_Srvcs | integer | Total services rendered |
| Tot_Sbmtd_Chrg | decimal | Total submitted charges ($) |
| Tot_Mdcr_Alowd_Amt | decimal | Total Medicare allowed amount ($) |
| Tot_Mdcr_Pymt_Amt | decimal | Total Medicare payment amount ($) |
| Tot_Mdcr_Stdzd_Amt | decimal | Standardized Medicare payment ($) |

---

## Join Keys

- Join to inpatient data on: `Rndrng_NPI`
- Join to drug data on: `Rndrng_NPI`

---

## Data Quality Notes

- `Rndrng_NPI` should be 10-digit numeric string — validate in Silver
- `Tot_Sbmtd_Chrg` may have $ signs or commas in raw CSV — cast carefully
- `Rndrng_Prvdr_State_Abrvtn` should be 2-letter code — filter out invalid states
- Some providers appear multiple times (once per procedure code) — aggregate in Silver

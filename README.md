# Medicare Data Analysis and Dashboard

## Table of Contents
1. [Introduction](#introduction)
2. [Tools Used and Dashboard Preview](#tools-used-and-dashboard-preview)
3. [Data Cleaning and Preparation](#data-cleaning-and-preparation)
4. [Exploratory Data Analysis (EDA)](#exploratory-data-analysis-eda)
5. [Results and Recommendations](#results-and-recommendations)
6. [Limitations](#limitations)

## Introduction
This project is inspired by my experience in rare disease marketing at Sanofi, where the complexity of the diagnostic landscape for rare disease patients made it challenging for organizations to determine the most effective allocation of resources to promote accurate diagnoses. Understanding healthcare provider distribution and behavior is essential for optimizing resource allocation, particularly in regions with limited patient populations and specialized needs. The goal of this project is to create a dashboard that offers a data-driven view of provider activity and influence across different areas and specialties, helping to address these challenges.

The data used in this analysis comes from the Centers for Medicare & Medicaid Services (CMS) and includes the Provider Utilization and Payment Data Physician and Other Practitioners Dataset. This dataset provides comprehensive information on services and procedures provided to Medicare beneficiaries by physicians and other healthcare professionals, including utilization, payment, and charges, organized by National Provider Identifier (NPI), Healthcare Common Procedure Coding System (HCPCS) code, and place of service. It covers 100% final-action Part B non-institutional claims for the Medicare fee-for-service population, excluding claims processed by Durable Medical Equipment, Prosthetic, Orthotics, and Supplies (DMEPOS) Medicare Administrative Contractors (MACs). The data originates from CMS administrative claims and the Chronic Condition Data Warehouse (CCW), with additional provider demographics sourced from the National Plan & Provider Enumeration System (NPPES).

The repository includes SQL scripts and analyses of Medicare data to examine provider behavior and trends. Key objectives include data cleaning, handling missing values, identifying outliers, and performing exploratory data analysis. The resulting user-friendly dashboard provides insights into healthcare spending across various provider types and individual healthcare professionals. It is designed to assist decision-makers in making more informed choices about resource allocation and strategic planning.

**Link to dashboard**: https://public.tableau.com/app/profile/chung.hsi.shan/vizzes

## Tools Used and Dashboard Preview
- **PostgreSQL**: Used for all data cleaning, transformation and analysis.
- **Tableau**: Dashboard creation and data visualization.
- Below are screenshots taken of the Medicare Dashboard created using Tableau:
![Dashboard 1](https://github.com/user-attachments/assets/15ae6085-f600-411d-86e1-1595e4b22a4e)
![Dashboard 2](https://github.com/user-attachments/assets/a5ab3ca1-c61a-4471-868f-0287209d61dc)


## Data Cleaning and Preparation
The data cleaning process involves several steps to ensure the dataset's quality and usability:

1. **Remove Duplicates**: Identifies and removes duplicate records based on a unique combination of columns.
2. **Handle Missing Values**: Detects missing values in crucial fields and considers possible actions.
3. **Validate Data**: Checks for data consistency and correctness (e.g., negative values in numeric fields and ensures the right ID length).
4. **Standardizing Data Formats**: Check for data consistency and apply the right format to the different fields. 

### Code Snippets

```sql
-- 1. Remove duplicates based on the defined unique fields
DELETE FROM public.medicare_data
WHERE rndrng_npi NOT IN (
    SELECT MIN(rndrng_npi)
    FROM public.medicare_data
    GROUP BY rndrng_npi, hcpcs_cd, tot_benes, tot_srvcs, tot_bene_day_srvcs, Avg_Sbmtd_Chrg
);

-- 2. Check for missing values
SELECT *
FROM public.medicare_data
WHERE rndrng_npi IS NULL 
    OR rndrng_prvdr_last_org_name IS NULL
    OR rndrng_prvdr_first_name IS NULL
    OR rndrng_prvdr_gndr IS NULL 
    OR rndrng_prvdr_city IS NULL
    OR rndrng_prvdr_type IS NULL
    OR rndrng_prvdr_state_abrvtn IS NULL 
    OR hcpcs_cd IS NULL
    OR tot_benes IS NULL
    OR tot_srvcs IS NULL
    OR tot_bene_day_srvcs IS NULL
    OR avg_sbmtd_chrg IS NULL
    OR avg_mdcr_alowd_amt IS NULL 
    OR avg_mdcr_pymt_amt IS NULL
    OR avg_mdcr_stdzd_amt IS NULL;


-- 3. Validate data to ensure data fits within constraints 
SELECT *
FROM public.medicare_data
WHERE tot_benes < 0
    OR tot_srvcs < 0
    OR tot_bene_day_srvcs < 0
    OR avg_sbmtd_chrg < 0
    OR avg_mdcr_alowd_amt < 0
    OR avg_mdcr_pymt_amt < 0
    OR avg_mdcr_stdzd_amt < 0;

-- Ensure all NPI code is 10 digits long
SELECT DISTINCT LENGTH(CAST(rndrng_npi AS TEXT)) AS digit_count
FROM public.medicare_data;

-- 4. Standardizing Data Formats by trimming the white space within the text based fields
UPDATE public.medicare_data
SET 
    rndrng_prvdr_last_org_name = TRIM(rndrng_prvdr_last_org_name),
    rndrng_prvdr_first_name = TRIM(rndrng_prvdr_first_name),
    rndrng_prvdr_gndr = TRIM(rndrng_prvdr_gndr),
    rndrng_prvdr_city = TRIM(rndrng_prvdr_city),
    rndrng_prvdr_type = TRIM(rndrng_prvdr_type),
    rndrng_prvdr_state_abrvtn = TRIM(rndrng_prvdr_state_abrvtn);

```


## Exploratory Data Analysis (EDA)
Exploratory Data Analysis aims to summarize the main characteristics of the dataset using statistical graphics and other data visualization methods:

1. **Provider Type Analysis**: Understand the distribution of various provider types in the Medicare data.
2. **Service Type Analysis**: the number of services rendered and the corresponding costs incurred on the Medicare program. 
3. **Ranking Analysis**: Rank healthcare providers based on the total amount claimed.

### Code Snippets
```sql
-- 1. Provider Type Analysis

-- Taking a look at the number of distinct provider types, there are 103 distinct provider types

SELECT
    DISTINCT rndrng_prvdr_type
FROM public.medicare_data;

--Taking a look at the number of unique HCPs for each provider type
SELECT
    rndrng_prvdr_type,
    COUNT (DISTINCT rndrng_npi)
FROM public.medicare_data
GROUP BY rndrng_prvdr_type 
ORDER BY COUNT (DISTINCT rndrng_npi) DESC;

-- 2. Service Type Analysis

-- Calculate how many distinct HSPCS code there are, there are a total of 6326 unique HSPCS code in total. 

SELECT
    COUNT (DISTINCT hcpcs_cd)
FROM public.medicare_data

-- Calculate how much Medicare spent for each HSPCS code in 2022

With spent_each_code AS (
	SELECT 
		hcpcs_cd,
		tot_srvcs * avg_mdcr_pymt_amt AS paid_amount
	FROM public.medicare_data
)

SELECT 
	hcpcs_cd,
	SUM (paid_amount) AS total_paid
FROM spent_each_code
GROUP BY hcpcs_cd
ORDER BY total_paid DESC;

/* Analyzing the percentage of outliers for each HCPCS code based on the Average
Medicare Standardized Payment Amount. This field standardizes payments by removing
geographic differences, such as local wages or input prices, making Medicare payments
comparable across different areas. This allows us to see variations that reflect
differences in factors like physicians' practice patterns and beneficiaries' access to
and desire for care. By filtering for HCPCS codes with more than 100 entries, this
eliminates the HCPCS codes that can be easily influenced by extreme values due to
their small sample sizes */

WITH q1_q3 as (
	SELECT 
        -- Calculating Q1 (25th percentile) and Q3 (75th percentile) for each column
		hcpcs_cd,
        percentile_cont(0.25) WITHIN GROUP (ORDER BY avg_mdcr_stdzd_amt) AS Q1_avg_mdcr_stdzd_amt,
        percentile_cont(0.75) WITHIN GROUP (ORDER BY avg_mdcr_stdzd_amt) AS Q3_avg_mdcr_stdzd_amt
	FROM public.medicare_data
	GROUP BY hcpcs_cd
),
iqr_table as (
	SELECT 
		hcpcs_cd,
		Q3_avg_mdcr_stdzd_amt,
		Q1_avg_mdcr_stdzd_amt,
		Q3_Avg_Mdcr_Stdzd_Amt - Q1_Avg_Mdcr_Stdzd_Amt as iqr
	FROM q1_q3
),
outliers_table as (
	SELECT 
		df.*,
		i.Q3_avg_mdcr_stdzd_amt,
		i.Q1_avg_mdcr_stdzd_amt,
		i.Q3_Avg_Mdcr_Stdzd_Amt - Q1_Avg_Mdcr_Stdzd_Amt as iqr,
		CASE 
	        WHEN df.Avg_Mdcr_Stdzd_Amt < i.Q1_Avg_Mdcr_Stdzd_Amt - 1.5 * (i.Q3_Avg_Mdcr_Stdzd_Amt - i.Q1_Avg_Mdcr_Stdzd_Amt) OR 
	             df.Avg_Mdcr_Stdzd_Amt > i.Q3_Avg_Mdcr_Stdzd_Amt + 1.5 * (i.Q3_Avg_Mdcr_Stdzd_Amt - i.Q1_Avg_Mdcr_Stdzd_Amt) THEN 1
	        ELSE 0
	    END AS Avg_Mdcr_Stdzd_Amt_Outlier
	FROM public.medicare_data as df
		LEFT JOIN iqr_table as i
			ON df.hcpcs_cd = i.hcpcs_cd
)

SELECT 
	hcpcs_cd,
	SUM(Avg_Mdcr_Stdzd_Amt_Outlier) as sum_outliers,
	COUNT(*) AS total_count,
	ROUND(SUM(Avg_Mdcr_Stdzd_Amt_Outlier) / CAST(COUNT(*) AS FLOAT)::numeric, 2) AS rounded_outlier_percentage
FROM outliers_table
GROUP BY hcpcs_cd
HAVING COUNT(*)>100
ORDER BY rounded_outlier_percentage DESC;

-- 3. Ranking Analysis

-- The following ranks NPIs according to their service type and state, based on the total claim amount for the year 2022. The results generated can be further filtered down to the desired service provider type and their state to identify regional top spenders. 

WITH table_total_amt AS (
    SELECT 
        rndrng_npi,
        rndrng_prvdr_type,
        rndrng_prvdr_state_abrvtn,
        tot_srvcs * Avg_Mdcr_Alowd_Amt AS total_amt_2022
    FROM public.medicare_data
    ORDER BY total_amt_2022 DESC
),
cte2 AS (
    SELECT 
        rndrng_npi,
        rndrng_prvdr_type,
        rndrng_prvdr_state_abrvtn,
        ROUND(SUM(total_amt_2022)::NUMERIC,0) AS total_claim
    FROM table_total_amt
    GROUP BY rndrng_npi, rndrng_prvdr_type, rndrng_prvdr_state_abrvtn
),
ranking_table AS (
    SELECT 
        df.*,
        cte2.total_claim,
        DENSE_RANK() OVER(PARTITION BY df.rndrng_prvdr_type, df.rndrng_prvdr_state_abrvtn ORDER BY total_claim DESC) AS ranking
    FROM public.medicare_data AS df
    LEFT JOIN cte2 ON cte2.rndrng_npi = df.rndrng_npi 
        AND cte2.rndrng_prvdr_type = df.rndrng_prvdr_type
        AND cte2.rndrng_prvdr_state_abrvtn = df.rndrng_prvdr_state_abrvtn
    ORDER BY ranking
)
SELECT DISTINCT
    rndrng_npi,
    rndrng_prvdr_type,
    rndrng_prvdr_last_org_name,
    rndrng_prvdr_first_name,
    rndrng_prvdr_crdntls,
    rndrng_prvdr_state_abrvtn,
    rndrng_prvdr_city,
    total_claim,
    ranking
FROM ranking_table
WHERE rndrng_prvdr_state_abrvtn != 'AA'
    AND rndrng_prvdr_state_abrvtn != 'AE'
ORDER BY rndrng_prvdr_state_abrvtn, rndrng_prvdr_type, ranking;

-- Lastly, the following selects for service provider types that are relevant to the disease area of rare diseases. The table generated is used to create the tableau dashboard.

SELECT 
	rndrng_npi AS npi,
	rndrng_prvdr_last_org_name ||' '|| rndrng_prvdr_first_name AS full_name,
	CASE 
		WHEN rndrng_prvdr_st2 = '' THEN rndrng_prvdr_st1
		ELSE rndrng_prvdr_st1||', '|| rndrng_prvdr_st2 
	END AS full_address,
	rndrng_prvdr_city AS city,
	rndrng_prvdr_state_abrvtn AS state_abbreviation,
	rndrng_prvdr_type AS provider_type,
	hcpcs_cd AS hcpcs_code,
	hcpcs_desc AS hcpcs_description, 
	tot_benes AS num_medicare_patients,
	tot_srvcs AS total_serviced,
	tot_srvcs * avg_sbmtd_chrg AS total_cost
FROM public.medicare_data
WHERE 
    rndrng_prvdr_type = 'Medical Genetics and Genomics' OR
    rndrng_prvdr_type = 'Neurology' OR
    rndrng_prvdr_type = 'Hematology' OR
    rndrng_prvdr_type = 'Rheumatology' OR
    rndrng_prvdr_type = 'Endocrinology' OR
    rndrng_prvdr_type = 'Pediatric Medicine';
```
## Results and Recommendations
### Results 
- **Resource Allocation**: By understanding where KOLs are located and how they influence the healthcare landscape, organizations can more effectively allocate resources, target outreach efforts, and design educational programs.
- **Market Analysis and Strategy**: The insights derived from this data analysis can help inform market strategies, identify unmet needs, and guide product development and distribution plans.
- **Improving Patient Outcomes**: By identifying trends in healthcare provider behavior, stakeholders can implement strategies to improve patient outcomes, particularly in underserved or high-need areas.
- **Policy Development**: Policy makers can use the data to understand healthcare utilization patterns and develop policies that address gaps in care or inequities in service provision.

### Recommendations 

## Limitations
- Only Medicare Part B (Outpatient Services) non-institutional claims (excluding
DMEPOS) are used for the creation of the dashboard, meaning Part A (Inpatient
Services), C (Medicare Advantage Plan), and D (Prescription Drug Coverage) part of
Medicare was not included, thus the analysis can only be generalized to Part B of
Medicare.
- The dataset only includes healthcare providers who are not affiliated with an
institution. 
- To protect the privacy of Medicare beneficiaries, any aggregated records that are
derived from 10 or fewer beneficiaries are excluded. Therefore volume HSPCS codes are
not part of the analysis and the total sum calculated may be a slight underestimation
of the true Medicare Part B totals. 
- Private insurance and other insurance claims are not taken into account in the analysis.
- Some providers bill under both an individual NPI and an organizational 
NPI. In this case, users cannot determine a provider’s actual total because there is
no way to identify the individual’s portion when billed under their organization.
- For the reasons above, the data do not represent a healthcare provider's entire
practice, and omit providers who are institution-affiliated.

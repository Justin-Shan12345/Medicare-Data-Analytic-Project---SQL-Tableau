# Medicare Data Analysis

## Table of Contents
1. [Introduction](#introduction)
2. [Tools Used](#tools-used)
3. [Data Cleaning and Preparation](#data-cleaning-and-preparation)
4. [Exploratory Data Analysis (EDA)](#exploratory-data-analysis-eda)
5. [Results and Recommendations](#results-and-recommendations)
6. [Limitations](#limitations)

## Introduction
This repository contains SQL scripts and analysis for Medicare data to explore healthcare provider behavior and trends. The primary objectives include data cleaning, handling missing values, identifying outliers, and performing exploratory data analysis to derive actionable insights.

## Tools Used
- **PostgreSQL**: Used for all data handling, cleaning, and analysis.
- **SQL**: Structured Query Language for data manipulation and retrieval.
- **GitHub**: Version control and collaboration.

## Data Cleaning and Preparation
The data cleaning process involves several steps to ensure the dataset's quality and usability:

1. **Remove Duplicates**: Identifies and removes duplicate records based on a unique combination of columns.
2. **Handle Missing Values**: Detects missing values in crucial fields and considers possible actions.
3. **Validate Data**: Checks for data consistency and correctness (e.g., negative values in numeric fields).
4. **Outlier Detection**: Identifies outliers using the Interquartile Range (IQR) method for numerical fields.

### Code Snippets

```sql
-- Remove duplicates
DELETE FROM public.medicare_data
WHERE rndrng_npi NOT IN (
    SELECT MIN(rndrng_npi)
    FROM public.medicare_data
    GROUP BY rndrng_npi, hcpcs_cd, tot_benes, tot_srvcs, tot_bene_day_srvcs, Avg_Sbmtd_Chrg
);

-- Handle missing values
SELECT *
FROM public.medicare_data
WHERE Rndrng_NPI IS NULL 
    OR Rndrng_Prvdr_Last_Org_Name IS NULL
    OR Rndrng_Prvdr_First_Name IS NULL
    OR Rndrng_Prvdr_Gndr IS NULL 
    OR Rndrng_Prvdr_City IS NULL
    OR Rndrng_Prvdr_RUCA IS NULL
    OR Rndrng_Prvdr_Cntry IS NULL
    OR Rndrng_Prvdr_Type IS NULL
    OR HCPCS_Cd IS NULL
    OR Tot_Benes IS NULL
    OR Tot_Srvcs IS NULL
    OR Tot_Bene_Day_Srvcs IS NULL
    OR Avg_Sbmtd_Chrg IS NULL
    OR Avg_Mdcr_Alowd_Amt IS NULL 
    OR Avg_Mdcr_Pymt_Amt IS NULL
    OR Avg_Mdcr_Stdzd_Amt IS NULL;

-- Validate data
SELECT *
FROM public.medicare_data
WHERE Tot_Benes < 0
    OR Tot_Srvcs < 0
    OR Tot_Bene_Day_Srvcs < 0
    OR Avg_Sbmtd_Chrg < 0
    OR Avg_Mdcr_Alowd_Amt < 0
    OR Avg_Mdcr_Pymt_Amt < 0
    OR Avg_Mdcr_Stdzd_Amt < 0;

-- Outlier Detection
WITH iqr_calculations AS (
    SELECT 
        percentile_cont(0.25) WITHIN GROUP (ORDER BY Tot_Benes) AS Q1_Tot_Benes,
        percentile_cont(0.75) WITHIN GROUP (ORDER BY Tot_Benes) AS Q3_Tot_Benes,
        percentile_cont(0.25) WITHIN GROUP (ORDER BY Tot_Srvcs) AS Q1_Tot_Srvcs,
        percentile_cont(0.75) WITHIN GROUP (ORDER BY Tot_Srvcs) AS Q3_Tot_Srvcs,
        percentile_cont(0.25) WITHIN GROUP (ORDER BY Tot_Bene_Day_Srvcs) AS Q1_Tot_Bene_Day_Srvcs,
        percentile_cont(0.75) WITHIN GROUP (ORDER BY Tot_Bene_Day_Srvcs) AS Q3_Tot_Bene_Day_Srvcs,
        percentile_cont(0.25) WITHIN GROUP (ORDER BY Avg_Sbmtd_Chrg) AS Q1_Avg_Sbmtd_Chrg,
        percentile_cont(0.75) WITHIN GROUP (ORDER BY Avg_Sbmtd_Chrg) AS Q3_Avg_Sbmtd_Chrg,
        percentile_cont(0.25) WITHIN GROUP (ORDER BY Avg_Mdcr_Alowd_Amt) AS Q1_Avg_Mdcr_Alowd_Amt,
        percentile_cont(0.75) WITHIN GROUP (ORDER BY Avg_Mdcr_Alowd_Amt) AS Q3_Avg_Mdcr_Alowd_Amt,
        percentile_cont(0.25) WITHIN GROUP (ORDER BY Avg_Mdcr_Pymt_Amt) AS Q1_Avg_Mdcr_Pymt_Amt,
        percentile_cont(0.75) WITHIN GROUP (ORDER BY Avg_Mdcr_Pymt_Amt) AS Q3_Avg_Mdcr_Pymt_Amt,
        percentile_cont(0.25) WITHIN GROUP (ORDER BY Avg_Mdcr_Stdzd_Amt) AS Q1_Avg_Mdcr_Stdzd_Amt,
        percentile_cont(0.75) WITHIN GROUP (ORDER BY Avg_Mdcr_Stdzd_Amt) AS Q3_Avg_Mdcr_Stdzd_Amt
    FROM public.medicare_data
)
SELECT 
    t.*, 
    Q3_Tot_Benes - Q1_Tot_Benes AS IQR_Tot_Benes,
    Q3_Tot_Srvcs - Q1_Tot_Srvcs AS IQR_Tot_Srvcs,
    Q3_Tot_Bene_Day_Srvcs - Q1_Tot_Bene_Day_Srvcs AS IQR_Tot_Bene_Day_Srvcs,
    Q3_Avg_Sbmtd_Chrg - Q1_Avg_Sbmtd_Chrg AS IQR_Avg_Sbmtd_Chrg,
    Q3_Avg_Mdcr_Alowd_Amt - Q1_Avg_Mdcr_Alowd_Amt AS IQR_Avg_Mdcr_Alowd_Amt,
    Q3_Avg_Mdcr_Pymt_Amt - Q1_Avg_Mdcr_Pymt_Amt AS IQR_Avg_Mdcr_Pymt_Amt,
    Q3_Avg_Mdcr_Stdzd_Amt - Q1_Avg_Mdcr_Stdzd_Amt AS IQR_Avg_Mdcr_Stdzd_Amt,
    CASE 
        WHEN t.Tot_Benes < Q1_Tot_Benes - 1.5 * (Q3_Tot_Benes - Q1_Tot_Benes) OR 
             t.Tot_Benes > Q3_Tot_Benes + 1.5 * (Q3_Tot_Benes - Q1_Tot_Benes) THEN 'Outlier'
        ELSE 'Normal' 
    END AS Tot_Benes_Outlier,
    CASE 
        WHEN t.Tot_Srvcs < Q1_Tot_Srvcs - 1.5 * (Q3_Tot_Srvcs - Q1_Tot_Srvcs) OR 
             t.Tot_Srvcs > Q3_Tot_Srvcs + 1.5 * (Q3_Tot_Srvcs - Q1_Tot_Srvcs) THEN 'Outlier'
        ELSE 'Normal' 
    END AS Tot_Srvcs_Outlier,
    CASE 
        WHEN t.Tot_Bene_Day_Srvcs < Q1_Tot_Bene_Day_Srvcs - 1.5 * (Q3_Tot_Bene_Day_Srvcs - Q1_Tot_Bene_Day_Srvcs) OR 
             t.Tot_Bene_Day_Srvcs > Q3_Tot_Bene_Day_Srvcs + 1.5 * (Q3_Tot_Bene_Day_Srvcs - Q1_Tot_Bene_Day_Srvcs) THEN 'Outlier'
        ELSE 'Normal' 
    END AS Tot_Bene_Day_Srvcs_Outlier,
    CASE 
        WHEN t.Avg_Sbmtd_Chrg < Q1_Avg_Sbmtd_Chrg - 1.5 * (Q3_Avg_Sbmtd_Chrg - Q1_Avg_Sbmtd_Chrg) OR 
             t.Avg_Sbmtd_Chrg > Q3_Avg_Sbmtd_Chrg + 1.5 * (Q3_Avg_Sbmtd_Chrg - Q1_Avg_Sbmtd_Chrg) THEN 'Outlier'
        ELSE 'Normal' 
    END AS Avg_Sbmtd_Chrg_Outlier,
    CASE 
        WHEN t.Avg_Mdcr_Alowd_Amt < Q1_Avg_Mdcr_Alowd_Amt - 1.5 * (Q3_Avg_Mdcr_Alowd_Amt - Q1_Avg_Mdcr_Alowd_Amt) OR 
             t.Avg_Mdcr_Alowd_Amt > Q3_Avg_Mdcr_Alowd_Amt + 1.5 * (Q3_Avg_Mdcr_Alowd_Amt - Q1_Avg_Mdcr_Alowd_Amt) THEN 'Outlier'
        ELSE 'Normal' 
    END AS Avg_Mdcr_Alowd_Amt_Outlier,
    CASE 
        WHEN t.Avg_Mdcr_Pymt_Amt < Q1_Avg_Mdcr_Pymt_Amt - 1.5 * (Q3_Avg_Mdcr_Pymt_Amt - Q1_Avg_Mdcr_Pymt_Amt) OR 
             t.Avg_Mdcr_Pymt_Amt > Q3_Avg_Mdcr_Pymt_Amt + 1.5 * (Q3_Avg_Mdcr_Pymt_Amt - Q1_Avg_Mdcr_Pymt_Amt) THEN 'Outlier'
        ELSE 'Normal' 
    END AS Avg_Mdcr_Pymt_Amt_Outlier,
    CASE 
        WHEN t.Avg_Mdcr_Stdzd_Amt < Q1_Avg_Mdcr_Stdzd_Amt - 1.5 * (Q3_Avg_Mdcr_Stdzd_Amt - Q1_Avg_Mdcr_Stdzd_Amt) OR 
             t.Avg_Mdcr_Stdzd_Amt > Q3_Avg_Mdcr_Stdzd_Amt + 1.5 * (Q3_Avg_Mdcr_Stdzd_Amt - Q1_Avg_Mdcr_Stdzd_Amt) THEN 'Outlier'
        ELSE 'Normal' 
    END AS Avg_Mdcr_Stdzd_Amt_Outlier
FROM public.medicare_data t, iqr_calculations;
```

## Exploratory Data Analysis (EDA)
Exploratory Data Analysis aims to summarize the main characteristics of the dataset using statistical graphics and other data visualization methods:

1. **Provider Type Analysis**: Understand the distribution of various provider types in the Medicare data.
2. **Service Analysis: Analyze**: the number of services rendered and the corresponding costs.
3. **Ranking Analysis**: Rank healthcare providers based on the total amount claimed.

## Code Snippets
```sql
-- Provider Type Analysis
SELECT rndrng_prvdr_type, COUNT(*)
FROM public.medicare_data
GROUP BY rndrng_prvdr_type
ORDER BY COUNT(*) DESC;

-- Ranking Analysis
CREATE VIEW top10_kols AS
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
WHERE ranking <= 10
    AND rndrng_prvdr_state_abrvtn != 'AA'
    AND rndrng_prvdr_state_abrvtn != 'AE'
ORDER BY rndrng_prvdr_state_abrvtn, rndrng_prvdr_type, ranking;
```

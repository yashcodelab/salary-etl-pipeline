# Glassdoor India Software Salaries — End-to-End PySpark ETL Pipeline

<div align="center">

![PySpark](https://img.shields.io/badge/PySpark-E25A1C?style=for-the-badge&logo=apachespark&logoColor=white)
![Python](https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white)
![Spark SQL](https://img.shields.io/badge/Spark%20SQL-E25A1C?style=for-the-badge&logo=apachespark&logoColor=white)
![Parquet](https://img.shields.io/badge/Parquet-50ABF1?style=for-the-badge&logo=apachehadoop&logoColor=white)
![Pandas](https://img.shields.io/badge/Pandas-150458?style=for-the-badge&logo=pandas&logoColor=white)
![Jupyter](https://img.shields.io/badge/Jupyter-F37626?style=for-the-badge&logo=jupyter&logoColor=white)

![rows](https://img.shields.io/badge/Records-22%2C770-1D9E75?style=flat-square)
![stages](https://img.shields.io/badge/Pipeline%20Stages-10-185FA5?style=flat-square)
![features](https://img.shields.io/badge/Features%20Engineered-7-854F0B?style=flat-square)
![spark](https://img.shields.io/badge/Spark-4.1.1-E25A1C?style=flat-square)

</div>

---

## Overview

A **production-pattern PySpark ETL pipeline** built on 22,770 Glassdoor India software salary records.
The pipeline covers every stage a Data Engineer handles in the real world — from raw ingestion and
schema profiling all the way through to partitioned Parquet storage and Spark SQL analytics.

> **Why this project?** Most PySpark tutorials stop at "load a CSV and group by one column."
> This pipeline mirrors what actually happens in production: schema enforcement, null audits,
> outlier removal, star-schema joins, window functions, partition tuning, and SQL views — end to end.

---

## Key results

| Metric | Value |
|--------|-------|
| Raw rows ingested | 22,770 |
| After cleaning & dedup | 22,399 |
| Outliers removed | 371 (salary outside 1–500 LPA) |
| Features engineered | 7 new columns |
| Final enriched columns | 25 |
| Parquet partition key | `job_role × employment_status` |
| Spark SQL temp view | `salary_data` |

### Business insights from the data

| Finding | Result |
|---------|--------|
| Top paying role | Database — **9.6 LPA avg** |
| Top paying city | Mumbai — **9.6 LPA avg** |
| Senior vs Junior gap (Backend) | 15.4 LPA → 3.09 LPA — **12.3 LPA gap** |
| Management avg salary | **16.3 LPA** |
| Junior avg salary | **4.0 LPA** |

---

## Pipeline architecture

```
glassdoor_salary.csv  (22,770 rows — fact table)
city_dim              (19 rows — Tier 1/2/3 + region + state)
role_dim              (21 rows — job role → category + is_tech flag)
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│  Stage 0  │  Spark Session (Spark 4.1.1, AQE enabled)       │
├─────────────────────────────────────────────────────────────┤
│  Stage 1  │  Ingest + Profile  (schema, nulls %, distinct)  │
│  Stage 1B │  Dimension Tables  (star schema — city + role)  │
├─────────────────────────────────────────────────────────────┤
│  Stage 2  │  Clean  (snake_case rename, INR→LPA, dedup,     │
│           │          outlier removal, string trimming)       │
├─────────────────────────────────────────────────────────────┤
│  Stage 3  │  Transform  (salary_band, rating_band,          │
│           │  title_seniority, is_reliable, weighted_salary) │
├─────────────────────────────────────────────────────────────┤
│  Stage 4  │  Aggregate  (by role, company, city, seniority) │
├─────────────────────────────────────────────────────────────┤
│  Stage 5  │  Joins  (left join city_dim + role_dim,         │
│           │          anti-join data quality check)           │
├─────────────────────────────────────────────────────────────┤
│  Stage 6  │  Window Functions  (rank in role, rank in city, │
│           │                     % of role max salary)        │
├─────────────────────────────────────────────────────────────┤
│  Stage 7  │  Performance  (cache, repartition, coalesce,    │
│           │                explain plan)                     │
├─────────────────────────────────────────────────────────────┤
│  Stage 8  │  Storage  (Parquet partitioned + CSV summary)   │
├─────────────────────────────────────────────────────────────┤
│  Stage 9  │  Spark SQL  (5 queries — CTEs, window SQL,      │
│           │              percentiles, seniority pay gap)     │
├─────────────────────────────────────────────────────────────┤
│  Stage 10 │  Pipeline Summary                               │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
  ./output/salary_clean.parquet   (partitioned by job_role, employment_status)
  ./output/salary_summary.csv     (aggregated summary)
```

---

## Tech stack

| Layer | Technology |
|-------|-----------|
| Distributed processing | PySpark 4.1.1 |
| SQL analytics | Spark SQL (CTEs, window functions, percentiles) |
| Data storage | Parquet (partitioned), CSV |
| Data wrangling | Pandas (summary export) |
| Schema enforcement | PySpark StructType / StructField |
| Notebook environment | Jupyter Notebook |
| Language | Python 3 |

---

## What each stage covers

### Stage 1 — Ingestion & profiling
- Load CSV with explicit schema inference
- NULL audit: % missing per column (not raw counts — percentages are meaningful)
- Distinct value counts to identify categorical vs continuous columns
- Distribution check: job role, employment type, location

### Stage 1B — Dimension tables (star schema)
- `city_dim`: 19 Indian cities with Tier (1/2/3), region, and state
- `role_dim`: 21 job roles mapped to category (Backend, Data, Mobile...) + `is_tech` flag
- Created in-code with explicit `StructType` schemas — no external dependency

### Stage 2 — Data cleaning
- Renamed all columns to `snake_case` (fixes `AnalysisException` from space-delimited names)
- Converted raw INR salary → **LPA (Lakhs Per Annum)** — Indian industry standard
- Removed 371 outliers (salaries outside 1–500 LPA range)
- Deduplication check (0 dupes found — good data quality)
- String trimming across all string columns

### Stage 3 — Feature engineering (7 new columns)

| Feature | Logic |
|---------|-------|
| `salary_band` | LPA quartile bucket: Entry / Mid / Senior / Lead |
| `rating_band` | Company rating: Poor / Average / Good / Excellent |
| `title_seniority` | NLP-based: Junior / Mid-Level / Senior / Management from job title |
| `is_reliable` | Flag: salaries_reported ≥ 3 |
| `weighted_salary` | salary_lpa × log(salaries_reported + 1) |
| `salary_vs_avg` | Deviation from role mean |
| `high_rating` | Boolean: rating ≥ 4.0 |

### Stage 4 — Aggregations
- Salary by job role (count, avg, min, max, stddev)
- Top 15 companies with 10+ salary reports (reliability filter — same as SQL HAVING)
- Salary by city (top 10)
- Salary by seniority tier

### Stage 5 — Joins
- Left join `city_dim` on location → preserves all 22,399 fact rows
- Left join `role_dim` on job_role → enriches with category + is_tech flag
- **Anti-join** data quality check → identified 4,332 unmatched city rows

### Stage 6 — Window functions
- `RANK() OVER (PARTITION BY job_role ORDER BY salary_lpa DESC)` — rank within role
- `ROW_NUMBER() OVER (PARTITION BY city ORDER BY salary_lpa DESC)` — top earner per city
- `salary_lpa / MAX(salary_lpa) OVER (PARTITION BY job_role)` — % of role maximum

### Stage 7 — Performance optimisation
- `cache()` — materialise hot DataFrame in memory before repeated reads
- `repartition(4, "job_role")` — co-locate same role on same partition before groupBy
- `coalesce(2)` — reduce shuffle partitions for write
- `explain()` — physical plan review and verification

### Stage 8 — Storage
- **Parquet** output partitioned by `job_role × employment_status`
- CSV summary for non-Spark consumers
- Why Parquet: columnar reads, compression, partition pruning on downstream queries

### Stage 9 — Spark SQL (5 analytical queries)
1. Top roles by average salary (min 5 reports reliability filter)
2. Top 5 companies per city — window SQL
3. **Seniority pay gap** — CTE pivot (Senior vs Junior gap up to 12.3 LPA)
4. Salary percentiles (P25 / Median / P75) per role × seniority
5. Reliability-filtered benchmark across all roles

---

## Sample outputs

### Seniority pay gap (CTE query — Stage 9)
```
+--------+----------+-------+----------+-----------------+
|job_role|senior_lpa|mid_lpa|junior_lpa|senior_junior_gap|
+--------+----------+-------+----------+-----------------+
|Backend |     15.40|   7.73|      3.09|            12.31|
|Frontend|     14.63|   5.86|      3.49|            11.14|
|SDE     |     12.75|   8.81|      4.52|             8.23|
|Database|     12.91|   8.59|      4.75|             8.16|
|Android |     10.46|   5.29|      3.01|             7.45|
+--------+----------+-------+----------+-----------------+
```

### Top paying roles & cities
```
=== TOP PAYING ROLES ===       === TOP PAYING CITIES ===
job_role   avg_lpa             location   avg_lpa
Database       9.6             Mumbai         9.6
  Mobile       9.1             Bangalore      7.5
     SDE       8.5             Kolkata        7.2

=== SALARY BY SENIORITY ===
title_seniority   avg_lpa
     Management      16.3
         Senior      11.1
      Mid-Level       6.9
         Junior       4.0
```

### Pipeline summary output
```
=======================================================
  GLASSDOOR SALARY ETL — COMPLETE
=======================================================
  Raw rows ingested     :   22,770
  After cleaning        :   22,399
  Features added        :        7
  Enriched columns      :       21
  Window cols added     :        4
  Parquet partition key : job_role x employment_status
  Spark SQL view        : salary_data
  Output location       : ./output/
=======================================================
```

---

## Folder structure

```
salary-etl-pipeline/
│
├── salary_etl_pipeline.ipynb     ← main notebook (10 stages, 53 cells)
├── README.md                     ← this file
├── requirements.txt              ← pip dependencies
│
├── data/
│   ├── glassdoor_salary.csv      ← raw dataset (see Dataset section)
│   └── README.md                 ← column descriptions + data source
│
└── output/                       ← auto-created when notebook runs
    ├── salary_clean.parquet/     ← partitioned: job_role / employment_status
    └── salary_summary.csv        ← aggregated summary for reporting
```

---

## How to run

### Option 1 — Local (PySpark installed)

```bash
# 1. Clone the repo
git clone https://github.com/yashcodelab/salary-etl-pipeline.git
cd salary-etl-pipeline

# 2. Install dependencies
pip install -r requirements.txt

# 3. Add the dataset
# Download glassdoor_salary.csv and place inside data/

# 4. Launch notebook
jupyter notebook salary_etl_pipeline.ipynb
```

### Option 2 — Google Colab (no local install needed)

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/yashcodelab/salary-etl-pipeline/blob/main/salary_etl_pipeline.ipynb)

Add this to the first cell in Colab:
```python
!pip install pyspark
```

---

## Requirements

```
pyspark>=4.0.0
pandas>=2.0.0
jupyter>=1.0.0
```

---

## Dataset

- **Source:** Glassdoor India Software Salaries
- **Records:** 22,770 rows × 8 columns
- **Columns:** `Rating` · `Company Name` · `Job Title` · `Salary (INR)` · `Salaries Reported` · `Location` · `Employment Status` · `Job Roles`
- **Key note:** Salary stored as raw INR integers — pipeline converts to LPA (Lakhs Per Annum) in Stage 2

> Dataset is not included in this repo due to size. Download from Kaggle and place at `data/glassdoor_salary.csv`.

---

## Resume bullet points

> Copy these directly into your resume or LinkedIn project section.

- Built end-to-end **PySpark ETL pipeline** on 22,770 Glassdoor India salary records across 10 modular stages — ingestion, profiling, cleaning, feature engineering, star-schema joins, window functions, and partitioned Parquet storage
- Engineered **7 derived features** including LPA conversion, NLP-based title seniority detection, reliability scoring, and weighted salary; enriched dataset from 8 → 25 columns via dimensional joins
- Implemented **Spark SQL analytics** using CTEs and window functions — revealed Backend Senior→Junior pay gap of **12.3 LPA** and Database as top-paying role at **9.6 LPA avg**
- Applied **production performance patterns** — `cache()`, `repartition()`, `coalesce()`, AQE, and physical plan review via `explain()`; stored output as Parquet partitioned by `job_role × employment_status`

---

## Connect

Built by **[Yash Phadtare](https://github.com/yashcodelab)** — Data Engineer | SQL Developer | ETL Specialist

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0A66C2?style=flat&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/yash-phadtare-b28139241/)
[![GitHub](https://img.shields.io/badge/GitHub-181717?style=flat&logo=github&logoColor=white)](https://github.com/yashcodelab)
[![Email](https://img.shields.io/badge/Email-D14836?style=flat&logo=gmail&logoColor=white)](mailto:yashphadtare927@gmail.com)

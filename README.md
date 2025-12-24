# Car_Sales — Azure Data Engineering Project

A complete end-to-end data engineering solution for ingesting, processing and modeling car sales data using Azure Data Factory, Azure Databricks and file-based datasets.  
This repository contains sample data, Databricks notebooks (exported .dbc), Azure Data Factory pipeline exports, and architecture & screenshots used to build a Bronze → Silver → Gold data pipeline for analytics.

Table of Contents
- [Project Overview](#project-overview)
- [Architecture](#architecture)
- [Data Sources](#data-sources)
- [ETL / Pipeline Overview](#etl--pipeline-overview)
- [How to run / reproduce](#how-to-run--reproduce)
- [Databricks Notebooks & ADF Pipelines](#databricks-notebooks--adf-pipelines)
- [Incremental Load Strategy](#incremental-load-strategy)
- [Sample DDL & Queries](#sample-ddl--queries)
- [Screenshots & Visuals](#screenshots--visuals)
- [Possible Improvements](#possible-improvements)
- [License & Contact](#license--contact)

---

## Project Overview
This project demonstrates an Azure-based data engineering pipeline for Car Sales analytics. It ingests CSV data (static and incremental), processes and transforms the data using Databricks notebooks, and contains exported Azure Data Factory pipeline definitions to orchestrate the flow. The output is a set of curated gold tables (dimensions + fact) suitable for BI and analytics.

Key technologies:
- Azure Data Factory (ADF) — orchestration / ingestion
- MS SQL Server Data Base (MS SQL Server DB) — Incremenatl Load
- Azure Data Lake Storage (ADLS) or blob storage — data lake (bronze/silver/gold)
- Databricks (PySpark / SQL notebooks) — transformations & modeling
- Target analytical model: dimensional (dimensions + fact table)

---

## Architecture
The repository contains an architecture diagram describing the end-to-end flow. High level flow:
- Raw CSV files land in a landing/bronze directory in ADLS.
- Databricks notebooks perform cleaning, deduplication and transformation to create silver datasets.
- Gold notebooks create dimension tables (branch / date / dealer / model) and a fact_sales table.
- ADF pipelines orchestrate copying/importing and can support incremental loads.

View the architecture diagram:
- [Architecture Diagram.png](https://github.com/Koushiksai2127/Car_sales-Azure-Data-Engineering-Project/blob/main/Architecture%20%26%20Screenshots/Architecture%20Diagram.png)

---

## Data Sources
This repo includes two example CSV files used as sample datasets for the pipeline:
- CarSales Data/SalesData.csv — full historical sales records (sample)
- CarSales Data/IncrementalSales.csv — incremental records (sample for incremental pipeline)

Both CSVs include the following columns (example row headers):
Branch_ID, Dealer_ID, Model_ID, Revenue, Units_Sold, Date_ID, Day, Month, Year, BranchName, DealerName, Product_Name

These are the raw inputs used by the provided Databricks notebooks and ADF pipelines.

---

## ETL / Pipeline Overview
Typical pipeline flow (implemented across ADF + Databricks notebooks in this repo):

1. Ingest: ADF pipelines copy source CSVs from source (or Git) to landing ADLS (bronze).
2. Bronze: Persist raw files as-is (partitioned by ingestion time / source).
3. Silver: Databricks notebooks parse, validate, normalize, and deduplicate raw data. Columns are cleaned and typed.
4. Gold: Databricks notebooks create dimension tables and the fact sales table, applying any required transformations, surrogate keys and joins.
5. Incremental: ADF + Databricks apply incremental logic to detect new/changed rows (using Date_ID or ingestion watermark) and MERGE into gold tables.

View the architecture diagram:
- [Incremenatl pipeline.png](https://github.com/Koushiksai2127/Car_sales-Azure-Data-Engineering-Project/blob/main/Architecture%20%26%20Screenshots/Incremental%20Pipeline.png)

---

## How to run / reproduce

Prerequisites
- Azure subscription with:
  - Azure Data Lake Storage Gen2 (ADLS) or Blob Storage
  - MS SQL Server Database
  - Azure Databricks workspace
  - Azure Data Factory (For orchestration)
- Databricks runtime supporting PySpark / SQL

Steps (high level)
1. Clone repo
   - git clone https://github.com/Koushiksai2127/Car_sales-Azure-Data-Engineering-Project.git
2. Upload sample CSVs to your storage account (or use ADF copy pipelines included):
   - Place SalesData.csv and IncrementalSales.csv into a landing container/folder (e.g., adls://<container>/bronze/carsales/)
3. Import Databricks notebooks:
   - In Databricks workspace import the `.dbc` files from `Databricks Notebooks` (Workspace → Import → choose .dbc).
   - Recommended execution order:
     1. Schema_Notebook — sets up schemas / config variables
     2. silver_notebook — ingestion/cleaning transformations to create silver data
     3. gold_dim_* notebooks — build dimension tables (branch, date, dealer, model)
     4. gold_fact_sales — build sales fact table (aggregations / merges)
4. (Optional) Import ADF pipelines:
   - Use the zip files in `Data factory pipelines/` → Import into Azure Data Factory (Manage → Git configuration or Import ARM template depending on the ZIP format).
   - Adjust linked services (credentials, storage account, Databricks workspace token) to match your environment.
5. Run notebooks/pipelines and validate outputs in your gold database / Data Lake.

Notes:
- Set workspace secrets or notebook-scoped variables for storage paths, DB/warehouse names, and credentials.
- The Databricks notebooks are exported in `.dbc` form — open them to view/modify the cells to match your environment.

---

## Incremental Load Strategy
This repository includes an `IncrementalSales.csv` sample and an incremental pipeline export. A recommended incremental approach:
1. Landing: Put incremental CSVs into a designated landing folder (bronze/incremental).
2. Detection: Use Date_ID (or ingestion timestamp) as watermark to identify new rows.
3. Merge: Use Databricks Delta (recommended) and MERGE INTO to upsert new/updated rows into silver/gold tables (MERGE on Branch_ID + Dealer_ID + Model_ID + Date_ID).
4. Idempotency: Keep processed-file logs (or store file ingestion metadata) to avoid reprocessing the same file twice.

Refer to:
- [Incremenatl pipeline.png](https://github.com/Koushiksai2127/Car_sales-Azure-Data-Engineering-Project/blob/main/Architecture%20%26%20Screenshots/Incremental%20Pipeline.png)

---

## Sample DDL & Example MERGE

Example CREATE TABLE DDLs (Delta/parquet-compatible):

CREATE TABLE dim_date (
  date_id STRING,
  day INT,
  month INT,
  year INT,
  full_date DATE
);

CREATE TABLE dim_branch (
  branch_id STRING,
  branch_name STRING
);

CREATE TABLE dim_dealer (
  dealer_id STRING,
  dealer_name STRING
);

CREATE TABLE dim_model (
  model_id STRING,
  product_name STRING
);

CREATE TABLE fact_sales (
  sale_id BIGINT GENERATED BY DEFAULT AS IDENTITY,
  branch_id STRING,
  dealer_id STRING,
  model_id STRING,
  date_id STRING,
  revenue DOUBLE,
  units_sold INT
);

Example MERGE (Databricks/Delta) — upsert incremental records into fact_sales:
MERGE INTO gold.fact_sales AS tgt
USING incremental_sales AS src
ON tgt.branch_id = src.branch_id AND tgt.dealer_id = src.dealer_id AND tgt.model_id = src.model_id AND tgt.date_id = src.date_id
WHEN MATCHED THEN UPDATE SET tgt.revenue = src.revenue, tgt.units_sold = src.units_sold
WHEN NOT MATCHED THEN INSERT (branch_id, dealer_id, model_id, date_id, revenue, units_sold) VALUES (src.branch_id, src.dealer_id, src.model_id, src.date_id, src.revenue, src.units_sold)

---

## Screenshots & Visuals
- Architecture Diagram: [Architecture Diagram.png](https://github.com/Koushiksai2127/Car_sales-Azure-Data-Engineering-Project/blob/main/Architecture%20%26%20Screenshots/Architecture%20Diagram.png)
- Databricks Workflow: [Databricks workflow.png](https://github.com/Koushiksai2127/Car_sales-Azure-Data-Engineering-Project/blob/main/Architecture%20%26%20Screenshots/Databricks%20workflow.png)
- Incremental Pipeline flow: [Incremental Pipeline.png](https://github.com/Koushiksai2127/Car_sales-Azure-Data-Engineering-Project/blob/main/Architecture%20%26%20Screenshots/Incremental%20Pipeline.png)
- Catalog screenshot: [Carsales_catalog.png](https://github.com/Koushiksai2127/Car_sales-Azure-Data-Engineering-Project/blob/main/Architecture%20%26%20Screenshots/Carsales_catalog.png)

---

## Possible Improvements / Next Steps
- Add unit-tests / data quality checks for incoming files (record counts, null checks, schema validation).
- Add CI/CD for Databricks notebooks (use repo-backed workspace or Databricks Repos).
- Implement SCD Type 2 logic for dimensions if historical attributes must be tracked.
- Add parameterization and a configuration file (JSON / YAML) for environment-specific settings.
- Convert `.dbc` to `.py` or source-controlled notebooks (Git friendly) for easier diffs and PRs.
- Provide a sample ARM or Bicep template to automatically deploy ADF pipelines and storage resources.

---

## License & Contact
This repository is provided as-is for learning and demonstration purposes. Please contact the repository owner for questions, improvements or contributions.

Repository owner: [Koushiksai2127](https://github.com/Koushiksai2127)

---

If you'd like, I can:
- Produce a polished ARM/Bicep template for deploying required Azure resources,
- Convert the Databricks `.dbc` exports into notebook files (.ipynb / .py) with run-order and parameter cells,
- Create a short runbook for importing the ADF zips and updating linked services step-by-step.

# Car Sales - End-to-End Azure Data Engineering Project

An end-to-end Azure data engineering solution that ingests, processes, and models car sales data using Azure Data Factory, Azure Databricks, and Azure Data Lake Storage (ADLS Gen2).
The project implements a Bronze → Silver → Gold medallion architecture with incremental loading, governed lakehouse design, and analytics-ready dimensional modeling.
<br><br>
[![Azure Data Factory](https://img.shields.io/badge/Azure_Data_Factory-0078D4?logo=azure&logoColor=white)](https://learn.microsoft.com/en-us/azure/data-factory/)
[![Azure Databricks](https://img.shields.io/badge/Azure_Databricks-FF3900?logo=databricks&logoColor=white)](https://azure.microsoft.com/en-us/products/databricks)
[![ADLS Gen2](https://img.shields.io/badge/ADLS_Gen2-0089D6?logo=microsoftazure&logoColor=white)](https://learn.microsoft.com/en-us/azure/storage/blobs/data-lake-storage-introduction)
[![PySpark](https://img.shields.io/badge/PySpark-E25A1C?logo=apachespark&logoColor=white)](https://spark.apache.org/docs/latest/api/python/)
[![Delta Lake](https://img.shields.io/badge/Delta_Lake-00A651?logo=delta&logoColor=white)](https://delta.io/)

## Table of Contents
- [Technology Stack](#Technology-Stack)
- [Project Overview](#project-overview)
- [Architecture](#architecture)
- [Data Sources](#Data-Sources)
- [Implementation Details](#Implementation-Details)
- [How to run / reproduce](#how-to-run--reproduce)
- [Screenshots & Visuals](#Screenshots-&-Visuals)
- [Key Takeaways & Learnings](#Key-Takeaways-&-Learnings)
- 

---
## Technology Stack

| Category | Technology | Usage |
| :--- | :--- | :--- |
| **Orchestration** | **Azure Data Factory (ADF)** | Pipeline orchestration, triggering, and incremental ingestion logic. |
| **Compute** | **Azure Databricks** | PySpark & SQL for data transformation and modeling. |
| **Storage** | **ADLS Gen2** | Hierarchical namespace storage for the Data Lake. |
| **Governance** | **Unity Catalog** | Centralized metadata, access control, and lineage. |
| **Format** | **Delta Lake** | ACID transactions, Schema Enforcement, and Time Travel. |
| **Source** | **Azure SQL DB** | Source system for incremental load simulation. |

---
## Project Overview
This project demonstrates an enterprise-grade Azure data engineering pipeline for Car Sales analytics. It orchestrates the ingestion of static and incremental CSV data, transforms it using Databricks (PySpark/SQL), and serves it via a Gold layer suitable for BI reporting.

**Key Features:**
* **Medallion Architecture:** Bronze (Raw), Silver (Cleaned), Gold (Star Schema).
* **Incremental Loading:** Efficiently processes only new records using Watermarking.
* **Governance:** Utilizes Unity Catalog for centralized metadata management.
* **Automation:** Fully automated end-to-end via Azure Data Factory pipelines.

---

## Architecture
The repository contains an architecture diagram describing the end-to-end flow. High level flow:
- Raw CSV files land in a landing/bronze directory in ADLS.
- Databricks notebooks perform cleaning, deduplication and transformation to create silver datasets.
- Gold notebooks create dimension tables (branch / date / dealer / model) and a fact_sales table.
- ADF pipelines orchestrate copying/importing and can support incremental loads.

View the architecture diagram:
- ![Architecture Diagram.png](https://github.com/Koushiksai2127/Car_sales-Azure-Data-Engineering-Project/blob/main/Architecture%20%26%20Screenshots/Architecture%20Diagram.png)

---

## Data Sources
The project utilizes two primary datasets provided in the `CarSales Data` folder:
1.  `SalesData.csv`: Full historical sales records.
2.  `IncrementalSales.csv`: New records to demonstrate the incremental pipeline.

Both CSVs include the following columns (example row headers):
Branch_ID, Dealer_ID, Model_ID, Revenue, Units_Sold, Date_ID, Day, Month, Year, BranchName, DealerName, Product_Name

---
## Implementation Details

### Azure Infrastructure
* **Storage:** ADLS Gen2 with Hierarchical Namespace (HNS) enabled; secured via Entra ID.
* **Security:** Managed Identity (MI) used for all service-to-service authentication (ADF to SQL, ADF to Storage).

### Azure Data Factory Configuration
* **Integration:** Linked Services configured for SQL, ADLS, and GitHub.
* **Pattern:** Utilizes **Lookup** (to check watermarks) + **Copy Activity** (to move data) + **Stored Procedures** (to update watermarks).

### Databricks & Unity Catalog
* **Metastore:** Account-level Unity Catalog enabled with a regional metastore.
* **Storage Credentials:** Access Connectors configured for secure access to Bronze, Silver, and Gold containers.
* **Tables:** All data assets are stored as **Delta Tables** for performance and reliability.

### Data Processing Layers
* **🥉 Bronze:** Raw ingestion landing zone.
* **🥈 Silver:** Data cleaning, type casting, deduplication, and standardization.
* **🥇 Gold:** Dimensional modeling (Star Schema) with Fact and Dimension tables.


## ETL / Pipeline Overview

1.  **Ingest:** ADF pipelines copy source data to the **Bronze** layer.
2.  **Cleanse (Silver):** Databricks notebooks read Bronze data, validate schemas, remove duplicates, and write to **Silver** tables.
3.  **Model (Gold):** Transformations are applied to create Dimension tables (`Dim_Branch`, `Dim_Dealer`, `Dim_Model`, `Dim_Date`) and the Fact table (`Fact_Sales`).
4.  **Upsert:** The pipeline utilizes the `MERGE INTO` Delta command to handle updates and insertions seamlessly.

![Incremental Pipeline](https://github.com/Koushiksai2127/Car_sales-Azure-Data-Engineering-Project/blob/main/Architecture%20%26%20Screenshots/Incremental%20Pipeline.png)

---

## Incremental Load Strategy
This repository includes an `IncrementalSales.csv` sample and an incremental pipeline export. A recommended incremental approach:
1. Landing: Put incremental CSVs into a designated landing folder (bronze/incremental).
2. Detection: Use Date_ID (or ingestion timestamp) as watermark to identify new rows.
3. Merge: Use Databricks Delta (recommended) and MERGE INTO to upsert new/updated rows into silver/gold tables (MERGE on Branch_ID + Dealer_ID + Model_ID + Date_ID).
4. Idempotency: Keep processed-file logs (or store file ingestion metadata) to avoid reprocessing the same file twice.

Refer to:
- <img width="975" height="683" alt="image" src="https://github.com/user-attachments/assets/a14fdf95-2184-4b41-a111-77a568ca3986" />

---

## How to run / reproduce

### Prerequisites
* Azure Subscription (ADLS Gen2, ADF, SQL DB, Databricks Workspace).
* Databricks Runtime 13.3 LTS or higher.

### Steps
1.  **Clone the Repository:**
    ```bash
    git clone [https://github.com/Koushiksai2127/Car_sales-Azure-Data-Engineering-Project.git]
    ```
2.  **Setup Storage:**
    * Create containers: `bronze`, `silver`, `gold`, `scripts`.
    * Upload `SalesData.csv` to the landing path.
3.  **Import Notebooks:**
    * Import `.dbc` files from the `Databricks Notebooks` folder into your workspace.
    * Update configuration variables (storage paths) in `Schema_Notebook`.
4.  **Configure ADF:**
    * Import pipelines from `Data factory pipelines/`.
    * Update Linked Services to point to your specific Azure resources.
5.  **Execute:**
    * Run the `Master_Pipeline` in ADF to trigger the full flow.

Notes:
- Set workspace secrets or notebook-scoped variables for storage paths, DB/warehouse names, and credentials.
- The Databricks notebooks are exported in `.dbc` form — open them to view/modify the cells to match your environment.

---

## Screenshots & Visuals
- Architecture Diagram: [Architecture Diagram.png](https://github.com/Koushiksai2127/Car_sales-Azure-Data-Engineering-Project/blob/main/Architecture%20%26%20Screenshots/Architecture%20Diagram.png)
- Databricks Workflow: [Databricks workflow.png](https://github.com/Koushiksai2127/Car_sales-Azure-Data-Engineering-Project/blob/main/Architecture%20%26%20Screenshots/Databricks%20workflow.png)
- Incremental Pipeline flow: [Incremental Pipeline.png](https://github.com/Koushiksai2127/Car_sales-Azure-Data-Engineering-Project/blob/main/Architecture%20%26%20Screenshots/Incremental%20Pipeline.png)
- Catalog screenshot: [Carsales_catalog.png](https://github.com/Koushiksai2127/Car_sales-Azure-Data-Engineering-Project/blob/main/Architecture%20%26%20Screenshots/Carsales_catalog.png)

---
## Key Takeaways & Learnings

### Solving Managed Identity for Azure SQL
One of the critical challenges I overcame was authenticating Azure Data Factory with Azure SQL Database using Managed Identity. I learned that unlike other resources, SQL Server does not have a built-in RBAC role for MSI.
* **Solution:** I had to explicitly create the ADF Managed Identity as a "Contained User" in the database using `CREATE USER [MyDF] FROM EXTERNAL PROVIDER` and assign roles manually.
* **Reference:** I followed this [Microsoft Q&A guide](https://learn.microsoft.com/en-us/answers/questions/1180796/adding-azure-data-factory-managed-identity-to-azur) to resolve the permission errors.

### Unity Catalog & Data Governance
Implementing **Unity Catalog** gave me a clear understanding of how to decouple metadata from physical storage. I learned the distinct roles of:
* **Metastore:** Acts as the "brain" for permissions and schema definitions (Metadata) .
* **Storage Credentials & External Locations:** The bridge that securely allows the metastore to access ADLS Gen2 without exposing access keys.
* **Medallion Architecture Governance:** By creating separate external locations for Bronze, Silver, and Gold, I established a secure lineage where different teams can have isolated access levels.

### Hybrid Orchestration (ADF + Databricks Workflows)
I moved beyond simple orchestration by integrating **Databricks Workflows** within Azure Data Factory.
* While ADF handled the ingestion and watermarking logic, I leveraged Databricks Workflows to run the transformation notebooks (Silver → Gold) in parallel.
* This hybrid approach optimized cost and performance, ensuring the Star Schema (Fact/Dimensions) was populated efficiently before the Power BI refresh.

---

## Final Thoughts

<div align="center">
  <p>
    Building this project has improved my practical skills in <b>Azure Data Engineering</b> and strengthened my understanding of cloud architecture design. It reflects my curiosity, persistence, and passion for developing real, end-to-end cloud data solutions—from raw CSV ingestion to <b>Power BI</b> visualization.
  </p>

  <p>
    <i>More Azure, Databricks, and data platform projects will be added soon.</i>
  </p>

  <h3>Let's Connect 🤝</h3>
  <p>
    If you have suggestions, feedback, or opportunities, feel free to reach out! I’m always excited to learn, collaborate, and work on meaningful data projects.
  </p>
</div>

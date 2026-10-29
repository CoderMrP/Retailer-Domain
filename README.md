# Enterprise GCP Retail Data Lakehouse
An end-to-end, metadata-driven data engineering pipeline designed for a multi-source retail ecosystem on Google Cloud Platform. The architecture ingests relational snapshots, incremental deltas (Cloud SQL MySQL), and external REST API feeds into Google Cloud Storage (GCS). Data is transformed using PySpark on Cloud Dataproc and modeled in BigQuery through a Medallion Architecture (Bronze → Silver → Gold). Workflows are orchestrated via Cloud Composer (Apache Airflow) with automated CI/CD using GitHub and GCP Cloud Build.

---

## Architecture Overview

```text
[ DATA SOURCES ]              [ INGESTION LAYER ]             [ STORAGE & WAREHOUSE ]           [ ORCHESTRATION & CI/CD ]

+------------------+         +---------------------+         +----------------------+         +------------------------+
| Retailer MySQL   | ------> |                     | ------> | GCS (Landing/Archive)|         | Cloud Composer         |
| Supplier MySQL   | ------> | PySpark on Dataproc |         +----------+-----------+         | (Managed Airflow)      |
| Customer API     | ------> | (Metadata-Driven &  |                    |                     +-----------+------------+
+------------------+         |  Watermark Logic)   |                    v                                 |
                             +---------------------+         +----------------------+                             v
                                                             | BigQuery Bronze      | <------- Orchestrates All Tasks
                                                             | (External Tables)    |
                                                             +----------+-----------+
                                                                        |
                                                                        v
                                                             +----------------------+
                                                             | BigQuery Silver      |
                                                             | (Cleaned / SCD Type2)|
                                                             +----------+-----------+
                                                                        |
                                                                        v
+------------------+                                         +----------------------+         +------------------------+
| GitHub Repo      | --------------------------------------> | BigQuery Gold        | ------> | Analytics &            |
| (cloudbuild.yaml)|        (Automated Deployment via        | (KPI Aggregation)    |         | BI Dashboards          |
+------------------+         Cloud Build Trigger)            +----------------------+         +------------------------+




## Core Pipeline Features

### 1. Data Ingestion via Cloud Dataproc & PySpark
- Extracts relational data directly from Cloud SQL (MySQL databases) into Google Cloud Storage (GCS) landing buckets using PySpark jobs running on Cloud Dataproc.
- Driven by GCS configuration files (`retailer_config.csv`, `supplier_config.csv`) to dynamically orchestrate extractions, handling both `FULL` loads (snapshot replacements) and `INCREMENTAL` loads.

### 2. Audit-Based Watermarking & Archival
- Executes high-watermark delta queries by fetching the latest `load_timestamp` from the BigQuery audit log (`temp_dataset.audit_log`).
- Automates historical file archiving in GCS (`landing/<source>/archive/<table_name>/YYYY/MM/DD/`) prior to landing new snapshots.

### 3. Medallion Architecture & Custom Gold Layer KPI Models
Organized into BigQuery layers (Bronze $\rightarrow$ Silver $\rightarrow$ Gold), with primary emphasis on business insight aggregation in the **Gold Layer**:

* **Bronze Layer (Raw Storage):** External BigQuery tables mapped directly over raw GCS JSON and Parquet landing files.
* **Silver Layer (Conformed Data):** Standardized, deduplicated, and cleansed tables handling `NULL` values and maintaining historical tracking via SCD Type 2 logic.
* **Gold Layer (Business Analytics & KPI Models):** Tailored analytics tables designed for direct BI dashboard consumption:
  * **Sales & Revenue Metrics (`gold_sales_summary`):** Aggregates daily/monthly sales volume, total revenue, and Average Order Value (AOV) by joining `orders`, `order_items`, and `products`.
  * **Supplier Performance & Margin Analysis (`gold_supplier_performance`):** Evaluates supply margins by comparing supplier wholesale pricing (`supply_price`) against retail pricing across products and regions.
  * **Product & Category Performance (`gold_product_performance`):** Identifies top-performing SKUs, category-level sales trends, and cross-selling patterns.
  * **Customer Engagement & Sentiment (`gold_customer_review_summary`):** Combines ingested REST API review data with customer transaction histories to calculate average product ratings and feedback volume.

### 4. Continuous Integration & Deployment (CI/CD)
- GitHub PR merges trigger GCP Cloud Build (`cloudbuild.yaml`) to run DAG syntax validation.
- Automatically deploys updated Airflow workflows to the Cloud Composer environment via `utils/add_dags_to_composer.py`.

# Fraud_Guard

An end-to-end fraud detection system built on Databricks, processing 6.3M+ financial transactions through a medallion architecture with automated ML training, live model serving, and a monitoring dashboard.

**Dataset:** [PaySim1](https://www.kaggle.com/datasets/ealaxi/paysim1) (synthetic mobile-money transactions, ~0.13% fraud rate)

---

## Architecture

```
PaySim CSV
   │
   ▼
Bronze (raw, incremental append)
   │
   ▼
Silver (Structured Streaming, cleaned & deduped)
   │
   ▼
Feature Engineering (balance ratios + sliding-window velocity features)
   │
   ├──▶ Model Training (XGBoost + SMOTE) → MLflow → Unity Catalog Registry → Model Serving API
   │
   └──▶ Gold / Compliance Layer (SQL aggregates + data quality checks)
                │
                ▼
        Lakeview Dashboard (9 visuals)

All stages orchestrated via Databricks Workflows (DAG-based, scheduled)
```

## Tech Stack

Databricks (Serverless) · Delta Lake · Spark Structured Streaming · XGBoost · imbalanced-learn (SMOTE) · MLflow · Unity Catalog · Databricks Model Serving · Databricks Workflows · Lakeview Dashboards · Python (OOP) · SQL

## Project Structure

```
fraudguard/
├── 01_ingestion/         Transaction model + Bronze ingestion
├── 02_processing/        Structured Streaming + feature engineering
├── 03_ml/                Model training + model serving
└── 04_compliance/        Gold layer + data quality checks
```

## Key Design Decisions

- **Incremental ingestion:** Bronze writes are append-only, tracked via an `ingestion_state` table, so pipeline reruns never reprocess or duplicate historical data.
- **Feature engineering:** Includes sliding-window velocity features (transaction count/amount over 1/5/15-minute windows) — the strongest fraud signals in the dataset.
- **Class imbalance:** Handled with SMOTE oversampling plus `scale_pos_weight`, with an F1-optimized classification threshold.
- **Model lifecycle:** Every training run is tracked in MLflow and registered to Unity Catalog before deployment to a live REST endpoint.
- **Automation:** A Databricks Workflow chains ingestion → cleaning → features → {model training, compliance layer} in parallel branches.

## Known Limitation

`newbalanceOrig` / `newbalanceDest` (post-transaction balances) introduce data leakage inherent to the PaySim dataset — these values wouldn't be available at real prediction time. Documented deliberately rather than hidden, as it affects reported model metrics.

## Setup

1. Create a [Databricks Free Edition](https://www.databricks.com/learn/free-edition) workspace and download the PaySim1 dataset.
2. Upload the CSV to a Unity Catalog Volume.
3. Run the notebooks in order (`01_ingestion` → `04_compliance`), updating the CSV path as needed.
4. Chain the notebooks into a Databricks Workflow and (optionally) schedule it.
5. Deploy the trained model via `06_model_serving` and build a dashboard on the Gold tables.

## Related Project

A companion project reimplements the streaming layer using real **Apache Kafka** and **Apache Airflow** running locally via Docker, to demonstrate self-managed streaming/orchestration alongside the Databricks-managed version here.
```

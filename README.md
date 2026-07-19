
# FraudGuard 

An end-to-end fraud detection platform built on **Databricks**, processing 6.3M+ financial transactions through a medallion architecture (Bronze → Silver → Gold), with automated ML training, live model serving, a monitoring dashboard, and scheduled orchestration.


**Dataset:** [PaySim1](https://www.kaggle.com/datasets/ealaxi/paysim1) — synthetic mobile-money transactions (6.3M rows, ~0.13% fraud rate)

---

## Overview

FraudGuard ingests transaction data incrementally, engineers behavioral and time-based features, trains a fraud classifier, deploys it as a live API, and surfaces results through an auto-refreshing compliance dashboard — with the full pipeline automated end to end.

---

## Architecture

```
PaySim CSV
   │
   ▼
Bronze — incremental, append-only ingestion (state-tracked)
   │
   ▼
Silver — Structured Streaming, cleaned & deduplicated
   │
   ▼
Feature Layer — balance ratios + sliding-window velocity features
   │
   ├──▶ Model Training (XGBoost + SMOTE) → MLflow → Unity Catalog → Model Serving API
   │
   └──▶ Gold / Compliance Layer (SQL aggregates + data quality checks)
                │
                ▼
        Lakeview Dashboard (9 visuals, auto-refresh)

All stages orchestrated as a DAG via Databricks Workflows
```

---

## Tech Stack

Databricks (Serverless) · Delta Lake · Spark Structured Streaming · XGBoost · imbalanced-learn (SMOTE) · MLflow · Unity Catalog · Databricks Model Serving · Databricks Workflows · Lakeview Dashboards · Python (OOP) · SQL

---

## Project Structure

```
fraudguard/
├── 01_ingestion/        Transaction model + Bronze ingestion
├── 02_processing/       Structured Streaming + feature engineering
├── 03_ml/               Model training + model serving
└── 04_compliance/       Gold layer + data quality checks
```

---

## Pipeline Highlights

- **Incremental ingestion:** Bronze writes are append-only and state-tracked via an `ingestion_state` table, so reruns never reprocess or duplicate historical data. A `DummyTransactionGenerator` synthesizes new transactions once the source dataset is exhausted, keeping the pipeline behaving like a live system.
- **Streaming layer:** Bronze is treated as a stream source via Spark Structured Streaming — a Databricks-native alternative to a Kafka-based design — with checkpointed, exactly-once processing.
- **Feature engineering:** Sliding-window velocity features (transaction count/amount over 1/5/15-minute windows, time since last transaction, distinct counterparties in 24h) capture the strongest behavioral fraud signals.
- **Compliance layer:** SQL-based Gold tables (`gold_fraud_summary`, `gold_audit_trail`) plus automated data quality checks (nulls, duplicates, invalid labels, negative amounts).

---

## Machine Learning

- **Model:** XGBoost, with SMOTE oversampling and `scale_pos_weight` to handle a ~0.13% base fraud rate
- **Threshold:** F1-optimized via precision-recall curve, not a default 0.5 cutoff
- **Lifecycle:** Tracked in MLflow → registered to Unity Catalog → deployed as a scale-to-zero Databricks Model Serving REST endpoint

---

## Automation

```
bronze_ingestion → silver_cleaning → feature_engineering ─┬→ compliance_gold_layer
                                                             └→ model_training
```

Compliance reporting and model training depend only on the feature layer and run in parallel. The pipeline is scheduled to run automatically end to end.

---

## Dashboard

A published, auto-refreshing Lakeview dashboard covers: total fraud amount, confirmed fraud cases, average fraud amount, fraud count by transaction type, risk category distribution and trend over time, fraud rate trend, total vs. fraud transaction volume, and a top-risky-transactions detail table.

---

## Known Limitations

- `newbalanceOrig` / `newbalanceDest` (post-transaction balances) introduce data leakage inherent to the PaySim dataset — these fields wouldn't be available at real prediction time, and they inflate reported metrics. Noted deliberately rather than hidden.
- An initial `amount > 10000` risk-flagging rule was too broad (~22% of transactions flagged), highlighting the need for tighter threshold tuning in a real compliance workflow.

---


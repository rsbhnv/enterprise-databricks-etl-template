# 📘 Databricks ETL / CDC Pipeline Template (JDBC → Delta Lake)

## Overview
This repository provides a **generic, production-oriented ETL template** for ingesting data into Databricks
using **JDBC sources (e.g., Oracle / SQL Server / PostgreSQL)** and **Delta Lake**.

It is designed as a reusable blueprint that supports:
- Batch ETL ingestion
- Optional CDC-style upserts using Delta **MERGE**
- Bronze → Silver layered architecture
- Config-driven execution (tables, filters, partitions, behavior)
- Schema alignment & type casting
- Delta optimization (OPTIMIZE / ZORDER / VACUUM)
- Monitoring and operational best practices

---

## Scope & Attribution (Important)
This repository is a **generic template / knowledge base** based on professional data engineering experience
and common industry patterns.  
It does **not** include proprietary code, secrets, or organizational identifiers.

---
```
## Architecture (High Level)

Source (JDBC: Oracle/SQLServer/Postgres, etc.)
↓
Extract (JDBC Reader / Incremental Windowing)
↓
Bronze (Raw Delta Tables)
↓
Transform (Schema normalization, casting, cleansing)
↓
Silver (Cleaned tables + optional CDC MERGE)
↓
Gold (Optional: Analytics / BI / ML consumption)
```

---

## Key Features

### Generic ETL Structure
- Modular pipeline stages: extract → transform → load
- Supports full loads and incremental loads (window-based)

### Optional CDC via MERGE
- Upsert behavior using explicit join keys
- Controlled updates (exclude audit columns)
- Idempotent loads when designed with stable keys

### Config-Driven Execution
- Source table definitions
- Partition column / windowing configuration
- Column include/exclude rules
- Environment paths (catalog/schema/table locations)

### Schema Handling
- Normalize column names (e.g., uppercase)
- Type casting to target schema
- Controlled column exclusion

### Observability & Ops
- Track run metadata (start/end, window keys, counts)
- Delta history inspection (DESCRIBE HISTORY)
- Basic failure visibility and retry guidance

---

##  Folder Structure

```text
/databricks_pipelines
  /pipeline_name
    ├── config.json (or config.py)
    ├── jdbc_functions.py
    ├── extract.py
    ├── transform.py
    ├── load.py
    ├── monitoring.py
    ├── main.py
    └── README.md
```
Configuration (Generic Example)
{
  "environment": {
    "catalog": "<catalog_or_schema>",
    "bronze_table": "<bronze_table>",
    "silver_table": "<silver_table>",
    "zorder_by": [],
    "cluster_by": []
  },
  "source": {
    "type": "jdbc",
    "jdbc": {
      "driver": "<jdbc_driver>",
      "host": "<host_or_secret_ref>",
      "port": "<port_or_secret_ref>",
      "database": "<db_or_service_name>",
      "user": "<user_or_secret_ref>",
      "password": "<password_or_secret_ref>"
    },
    "table": "<source_table>",
    "partition_column": "<partition_column>",
    "select": {
      "uppercase": true,
      "exclude": []
    }
  },
  "incremental": {
    "mode": "monthly_windows",
    "days_back": 30,
    "parallelism": 4
  },
  "merge": {
    "enabled": true,
    "join_keys": ["<pk1>", "<pk2>"],
    "audit_columns": ["<created_at>", "<updated_at>"]
  }
}

Pipeline Stages (Conceptual)
1) Extract (JDBC)

Common patterns:

Single window ingestion (BETWEEN)

Monthly window ingestion (partitioned by date/month)

Parallel ingestion per window (e.g., month-by-month)

Typical responsibilities:

Build JDBC connection parameters

Generate window boundaries

Read data using filters on the partition column

Write raw results to Bronze

2) Transform

Typical responsibilities:

Normalize column names

Drop excluded columns

Cast to target schema

Basic validation (null checks, domain checks, duplicates)

3) Load (Bronze → Silver)

Two common modes:

A. Append-only (Batch)

Write cleaned data to Silver with partitioning

B. CDC-style Upsert (MERGE)

MERGE Bronze into Silver using join keys

Update only non-audit columns

Insert new records when not matched

Example (conceptual):

MERGE INTO <silver_table> AS target
USING <bronze_view_or_table> AS source
ON <join_conditions>
WHEN MATCHED THEN UPDATE SET
  target.<col> = source.<col>
WHEN NOT MATCHED THEN INSERT (...)
VALUES (...);

Monitoring (Generic)

Recommended signals:

Run start/end timestamps

Window processed (e.g., month key)

Rows read / written

MERGE stats: inserted / updated / deleted (if applicable)

Useful Delta inspection:

DESCRIBE HISTORY <silver_table>;

Delta Optimization (Recommended)

Compact small files:

OPTIMIZE <table_name>;


Optimize read patterns:

OPTIMIZE <table_name> ZORDER BY (<key1>, <key2>);


Cleanup old snapshots (example):

VACUUM <table_name> RETAIN 168 HOURS;

CI/CD (High-Level Recommendations)

Store notebooks/code in a Git-backed repo (Databricks Repos / GitHub)

Use feature branches for development

Promote changes across environments (DEV → TEST → PROD) using one of:

Azure DevOps / GitHub Actions

Databricks Asset Bundles (recommended for structured deployments)

Keep environment-specific configuration separate (do not hardcode paths)

Testing Recommendations

Unit tests for transformation logic (e.g., pytest + local Spark / Databricks Connect)

Data quality tests:

Great Expectations (optional)

Table constraints / expectations (where available)

Smoke tests after deployment

When to Use Bronze/Silver Layers

If your ingestion is small and the transformation is minimal, a single “clean” layer may be enough.
For enterprise pipelines with evolving schemas, CDC requirements, auditing, and replay capability,
the Bronze → Silver separation provides long-term robustness.

Conclusion

This template provides a reusable foundation for Databricks ingestion pipelines:

JDBC ingestion (including Oracle-like sources)

Optional CDC-style MERGE patterns

Config-driven behavior

Operational best practices for performance and monitoring

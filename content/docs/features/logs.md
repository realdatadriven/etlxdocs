---
weight: 790
title: "Logs / Observability"
description: "ETLX logging mechanism to save logs into a database."
icon: list
tags: ["features", "logs", "database", "observability", "audit"]
date: 2025-12-16T01:04:15+00:00
lastmod: 2025-12-16T01:04:15+00:00
draft: false
images: []
---

## Logs / Observability Handling (`# LOGS`)

ETLX provides a **logging mechanism** that allows **saving logs** into a database. This is useful for tracking **executions, debugging, and auditing ETL processes**.

By default, ETLX logs **every relevant aspect of the pipeline**:

- Step name / description
- Timestamp start / end
- Duration in seconds
- Success / failure
- Messages and errors
- Memory usage
- Optional row counts, files generated, or other metrics

All logs are serialized as **JSON** and persisted using the LOGS block.

---

## **🔹 How It Works**

- The `LOGS` section defines where and how logs should be saved.
- The logging lifecycle consists of **three main steps**:

1. **Prepare the environment** using `before_sql` (load extensions, attach databases)
2. **Execute `save_log_sql`** to store logs in the database
3. **Run `after_sql`** for cleanup (detach DB, free temp resources)

- Logs are emitted **regardless of pipeline success** to guarantee observability.

## **🛠 Example LOGS Configuration**

Below is an example that **saves logs** into a **database**:

````md {linenos=table}
# LOGS
```yaml metadata
name: LOGS
description: "Example saving logs"
table: logs
connection: "duckdb:"
before_sql: "ATTACH 'examples/S3_EXTRACT.db' AS "DB" (TYPE SQLITE)"
save_log_sql: load_logs
save_on_err_patt: '(?i)table.+with.+name.+(\w+).+does.+not.+exist'
save_on_err_sql: create_logs
after_sql: 'DETACH "DB"'
active: true
```
```sql
-- load_logs
INSERT INTO "DB"."<table>" BY NAME
SELECT * 
FROM READ_JSON('<fname>');
```
```sql
-- create_logs
CREATE OR REPLACE TABLE "DB"."<table>" AS
SELECT * 
FROM READ_JSON('<fname>');
```
````

## **🔹 How to Use**

- This example saves logs into a **SQLite database attached to DuckDB**.
- The **log table (`logs`) is created or replaced** on each run.
- The `<table>` and `<fname>` placeholders are dynamically replaced.

## Default Logs (`AUTO_LOGS`)

If you don’t define a `LOGS` block, ETLX injects **AUTO_LOGS** automatically:

````md {linenos=table}
# AUTO_LOGS

```yaml metadata
name: LOGS
description: "Logging"
table: logs
connection: "duckdb:"
before_sql:
  - "LOAD Sqlite"
  - "ATTACH '<tmp>/etlx_logs.db' (TYPE SQLITE)"
  - "USE etlx_logs"
  - "LOAD json"
  - "get_dyn_queries[create_missing_columns](ATTACH '<tmp>/etlx_logs.db' (TYPE SQLITE),DETACH etlx_logs)"
save_log_sql: |
  INSERT INTO "etlx_logs"."<table>" BY NAME
  SELECT *
  FROM READ_JSON('<fname>');
save_on_err_patt: '(?i)table.+with.+name.+(\w+).+does.+not.+exist'
save_on_err_sql: |
  CREATE TABLE "etlx_logs"."<table>" AS
  SELECT *
  FROM READ_JSON('<fname>');
after_sql:
  - 'USE memory'
  - 'DETACH "etlx_logs"'
active: true
```

```sql
-- create_missing_columns
WITH source_columns AS (
    SELECT "column_name", "column_type"
    FROM (DESCRIBE SELECT * FROM READ_JSON('<fname>'))
),
destination_columns AS (
    SELECT "column_name", "data_type" as "column_type"
    FROM "duckdb_columns"
    WHERE "table_name" = '<table>'
),
missing_columns AS (
    SELECT "s"."column_name", "s"."column_type"
    FROM source_columns "s"
    LEFT JOIN destination_columns "d" ON "s"."column_name" = "d"."column_name"
    WHERE "d"."column_name" IS NULL
)
SELECT 'ALTER TABLE "etlx_logs"."<table>" ADD COLUMN "' || "column_name" || '" ' || "column_type" || ';' AS "query"
FROM missing_columns
WHERE (SELECT COUNT(*) FROM destination_columns) > 0;
```
````

## ✅ Key Points

* Keeps a **persistent log** of all ETLX executions
* Uses **DuckDB for efficient storage**
* Supports **preprocessing (`before_sql`) and cleanup (`after_sql`)**
* Supports **automatic schema evolution**
* **Default logging is enabled**, but can be overridden
* Logs every **metric, message, and error** for observability

## 🔧 Customizing Logs

To save logs to your own database:

```sql
ATTACH 'my_custom_logs.db' (TYPE SQLITE)
```

Or point to **any DuckDB-supported database**, e.g., Postgres or MySQL, by updating the `connection` string.

## 📊 Observability Use Cases

* Full execution timelines
* ETL duration per step
* File, network, or DB operation metrics
* Auditing user-defined actions
* Feeding BI dashboards or monitoring systems
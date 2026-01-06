---

weight: 830
title: "Multi-Engine Execution"
description: "Run ETLX pipelines across DuckDB, PostgreSQL, SQLite, MySQL, SQL Server, and any database supported by sqlx or ODBC."
icon: settings
tags: ["architecture", "execution", "sql", "engines"]
date: 2025-12-16T01:04:15+00:00
lastmod: 2025-12-16T01:04:15+00:00
draft: false
images: []
---
---
## Multi-Engine Execution

ETLX is designed to be **engine-agnostic by default**. While **DuckDB is the recommended and primary execution engine**, ETLX can execute pipelines across **multiple database engines within the same workflow**, depending on availability, constraints, and use cases.

This allows ETLX to operate:

* Fully **embedded and in-process** (DuckDB, SQLite)
* Against **external OLTP / analytical databases** (PostgreSQL, MySQL, SQL Server)
* Through **ODBC or other sqlx-supported drivers** for broader compatibility

> 🧠 DuckDB is a *developer choice*, not a hard dependency.
> ETLX adapts to your environment instead of forcing a single execution engine.

---

## Supported Execution Engines

ETLX supports the following execution backends:

| Engine                    | Mode     | Notes                                                                 |
| ------------------------- | -------- | --------------------------------------------------------------------- |
| **DuckDB**                | Embedded | Default engine, SQL-first analytics, file-based I/O, best performance |
| **SQLite**                | Embedded | Lightweight storage, logs, metadata, small datasets                   |
| **PostgreSQL**            | External | OLTP / analytical workloads                                           |
| **MySQL / MariaDB**       | External | Operational databases                                                 |
| **SQL Server (MSSQL)**    | External | Enterprise systems via sqlx or ODBC                                   |
| **Any sqlx-supported DB** | External | Any database that has support https://github.com/jmoiron/sqlx         |
| **ODBC sources**          | External | Legacy systems, Excel, proprietary engines                            |


## Execution Model

Every executable step in ETLX explicitly declares **which engine to use**.

This is done using connection fields such as:

* `connection`
* `<step>_conn`
* `source.conn` / `target.conn` (in `db_2_db` actions)

If a connection is not specified, ETLX **falls back to the pipeline default engine**.

```yaml
connection: "duckdb:"  # default
```

---

## DuckDB as the Default Engine

DuckDB is embedded directly into the ETLX process and provides:

* In-process execution (no external service)
* Excellent performance for analytical queries
* Native support for files (CSV, Parquet, JSON, Excel)
* Cross-database access via extentions

Typical ETLX workflows use DuckDB to:

* Extract from multiple sources
* Transform data using SQL
* Export files
* Run validations
* Persist logs

```yaml
connection: "duckdb:"
```

---

## Running Pipelines Without DuckDB

Although DuckDB is recommended, **ETLX does not require it**.

You can execute most pipeline steps directly on:

* PostgreSQL
* MySQL
* SQL Server
* SQLite

Example using PostgreSQL as the primary engine:

```yaml
connection: "postgres:dbname=erpdb host=db user=etl password=@PG_PASS"
```

This allows ETLX to act as a **pure SQL execution and orchestration layer** on top of an existing database.

> ⚠️ Some features (file exports, multi-engine joins, advanced analytics) may be limited without DuckDB.

---

## Mixing Engines in a Single Pipeline

ETLX supports **multi-engine pipelines**.

Example:

* Extract from PostgreSQL
* Transform in DuckDB
* Export to files
* Write results back to SQL Server

```yaml
extract_conn: "postgres:..."
load_conn: "duckdb:"
transform_conn: "duckdb:"
```

This pattern is common when:

* Source systems are operational databases
* Transformations are analytical
* Outputs are files or reports

---

## Cross-Database Transfers (`ACTION:db_2_db`)

When a database cannot be accessed directly by DuckDB, ETLX uses **internal streaming and chunked transfers**.

```yaml
type: db_2_db
params:
  source:
    conn: mssql:sqlserver://...
    sql: source_query
    chunk_size: 1000
  target:
    conn: postgres:...
    sql: insert_sql
```

This allows ETLX to move data **engine-to-engine using pure SQL**, without intermediate files.

---

## ODBC & Legacy Systems

ETLX integrates with:

* Any database supported by **ODBC**
* Any driver supported by **sqlx**

This includes:

* Legacy ERP systems
* Excel via ODBC
* Proprietary databases

When direct scanning is not possible, ETLX can:

* Stream data
* Export to CSV
* Re-ingest into DuckDB or another engine

---

## Design Principles

Multi-engine execution in ETLX follows these principles:

* **Explicit is better than implicit**
* **SQL remains the contract** between engines
* **Engines are interchangeable**, not hardcoded
* **Metadata drives execution**, not engine-specific logic

---

## Summary

✔ DuckDB is embedded and recommended, but optional
✔ SQLite, PostgreSQL, MySQL, MSSQL, and ODBC are supported
✔ Pipelines can run on a single engine or multiple engines
✔ ETLX adapts to enterprise and open-source environments
✔ SQL is the unifying execution layer

> **ETLX lets you choose the engine that fits your constraints — without rewriting your pipelines.**

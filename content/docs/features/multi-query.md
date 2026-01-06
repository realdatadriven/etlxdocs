---

weight: 750
title: "Multi / Stacked Queries"
description: "Combine multiple structured queries into a single result set using UNION-based execution."
icon: layers
tags: ["features", "reporting"]
date: 2025-12-16T01:04:15+00:00
lastmod: 2025-12-16T01:04:15+00:00
draft: false
images: []
---

## Multi / Stacked Queries

The `MULTI_QUERIES` block allows you to define multiple independent but structurally compatible queries and combine their results into a **single unified dataset** using SQL `UNION` or `UNION ALL`. This pattern is especially useful for **reporting, KPI generation, metric tables, and summary datasets**.

Instead of orchestrating many individual queries manually, `MULTI_QUERIES` lets you declaratively describe each row or slice of a report and automatically aggregate them into a single execution plan.

---

## Multi Queries Structure

### 1. Metadata

The Level‑1 `MULTI_QUERIES` block defines shared execution metadata:

* Connection and lifecycle SQL (`before_sql`, `after_sql`)
* How queries are merged (`union_key`)
* Optional persistence logic (`save_sql`)
* Error‑aware save handling (`save_on_err_patt`, `save_on_err_sql`)

This metadata applies to all child queries unless explicitly overridden.

### 2. Query Definitions

* Each Level‑2 heading represents a **single query fragment**.
* All queries **must return compatible schemas** (same column names and types).
* Each query contributes one or more logical rows to the final result.

### 3. Execution Model

* All active queries are collected.
* Queries are concatenated using the defined `union_key` (default: `UNION`).
* The final SQL is executed **once**, optionally persisted.

---

## Multi Queries Markdown Example

````md {linenos=table}
# SALES_REPORT
```yaml
description: "Define multiple structured queries combined with UNION."
runs_on: MULTI_QUERIES
connection: "duckdb:"
before_sql: "ATTACH 'reporting.db' AS DB (TYPE SQLITE)"
save_sql: save_mult_query_res
save_on_err_patt: '(?i)table.+with.+name.+(\w+).+does.+not.+exist'
save_on_err_sql: create_mult_query_res
after_sql: "DETACH DB"
union_key: "UNION ALL\n" # Defaults to UNION
active: true
```

```sql
-- save_mult_query_res
INSERT INTO "DB"."MULTI_QUERY" BY NAME
[[final_query]]
```

```sql
-- create_mult_query_res
CREATE OR REPLACE TABLE "DB"."MULTI_QUERY" AS
[[final_query]]
```

## Row1
```yaml
name: Row1
description: "Row count"
query: row_query
active: true
```

```sql
-- row_query
SELECT '# number of rows' AS "variable", COUNT(*) AS "value"
FROM "sales"
```

## Row2
```yaml
name: Row2
description: "Total revenue"
query: row_query
active: true
```

```sql
-- row_query
SELECT 'total revenue' AS "variable", SUM("total") AS "value"
FROM "sales"
```

## Row3
```yaml
name: Row3
description: "Revenue by region"
query: row_query
active: true
```

```sql
-- row_query
SELECT "region" AS "variable", SUM("total") AS "value"
FROM "sales"
GROUP BY "region"
```
````

## How Multi Queries Works

### 1. Query Collection

* All Level‑2 queries marked as `active: true` are selected.
* Each query resolves its SQL either inline or via named SQL blocks.

### 2. Query Union

* Queries are joined using the `union_key` value:

  * `UNION` removes duplicates
  * `UNION ALL` preserves all rows

### 3. Final Query Execution

The engine produces a single executable SQL statement:

```sql
SELECT ...
UNION ALL
SELECT ...
UNION ALL
SELECT ...
```

### 4. Optional Persistence

If `save_sql` is defined:

* The final query is injected using `[[final_query]]`.
* If execution fails and the error matches `save_on_err_patt`, the fallback SQL (`save_on_err_sql`) is executed.

This enables **insert‑or‑create** patterns for reporting tables.

## Example Final Query (Expanded)

```sql
LOAD sqlite;
ATTACH 'reporting.db' AS DB (TYPE SQLITE);

SELECT '# number of rows' AS "variable", COUNT(*) AS "value"
FROM "sales"
UNION ALL
SELECT 'total revenue' AS "variable", SUM("total") AS "value"
FROM "sales"
UNION ALL
SELECT "region" AS "variable", SUM("total") AS "value"
FROM "sales"
GROUP BY "region";

DETACH DB;
```

## Multi Queries Use Cases

* KPI and metric tables
* Executive dashboards
* Summary datasets
* Audit and reconciliation reports
* Cross‑domain aggregations

## Multi Queries Benefits

* **Declarative reporting**: define results, not execution order
* **Single‑pass execution**: efficient and optimized
* **Error‑aware persistence**: safe table creation and inserts
* **Highly composable**: pairs naturally with ETL and DATA_QUALITY blocks

The `MULTI_QUERIES` block turns SQL reporting into a **structured, reusable, and auditable pipeline component**, fully aligned with ETLX’s block‑driven design philosophy.

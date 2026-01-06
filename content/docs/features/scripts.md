---

weight: 780
title: "Scripts"
description: "Execute SQL statements for operational and maintenance tasks"
icon: code
tags: ["features", "management", "sql", "scripts"]
date: 2025-12-16T01:04:15+00:00
lastmod: 2025-12-16T01:04:15+00:00
draft: false
images: []
---

## Scripts

The **SCRIPTS** section allows you to execute **SQL statements that do not naturally belong to other ETLX blocks** such as `ETL`, `DATA_QUALITY`, `EXPORTS`, or `MULTI_QUERIES`. It is designed for **operational, maintenance, and orchestration-style SQL**, where the goal is execution rather than producing datasets.

Typical use cases include cleanup operations, database maintenance, schema adjustments, or any SQL that should run as part of a pipeline but does not produce query results.

---

## When to Use SCRIPTS

Use the `SCRIPTS` block when you need to:

* Run **cleanup queries** after ETL or transformation steps
* Execute **ad-hoc or scheduled maintenance tasks**
* Perform **DDL or administrative SQL** (DROP, VACUUM, ANALYZE, etc.)
* Run SQL that **does not need to return rows**
* Apply **post-processing logic** not tied to a dataset

If your logic produces data that should be queried, validated, or exported, prefer other ETLX blocks. `SCRIPTS` is intentionally simple and execution-focused.

## Scripts Structure

The `SCRIPTS` block follows the same hierarchical pattern used across ETLX:

1. **Top-level SCRIPTS block**

   * Defines shared metadata such as connection and activation status.

2. **Script definitions** (Level 2 headings)

   * Each script represents a single executable SQL unit.
   * Scripts can override connection details and lifecycle hooks.

3. **Execution lifecycle**

   * `before_sql` commands are executed first
   * The script SQL is executed
   * `after_sql` commands are executed last

Scripts are executed sequentially and do not return results.

---

## Scripts Markdown Example

The following example demonstrates how to run cleanup scripts after an ETL process.

````md {linenos=table}
# MANTAINANCE

Run Queries that does not need a return

```yaml metadata
name: DailyScripts
runs_as: SCRIPTS
description: "Daily Scripts"
connection: "duckdb:"
active: true
```

## SCRIPT1

```yaml metadata
name: SCRIPT1
description: "Clean up auxiliar / temp data"
connection: "duckdb:"
before_sql: "ATTACH 'database/DB.db' AS DB (TYPE SQLITE)"
script_sql: clean_aux_data
on_err_patt: null
on_err_sql: null
after_sql: "DETACH DB"
active: true
```

```sql
-- clean_aux_data
DROP TEMP_TABLE1;
```
````

## How Scripts Work

1. **Initialization**

   * Loads required extensions
   * Establishes database connections

2. **Execution**

   * Executes the SQL referenced by `script_sql`
   * No result set is expected or captured

3. **Lifecycle Hooks**

   * `before_sql` runs before execution
   * `after_sql` runs after execution

4. **Error Handling**

   * Optional `on_err_patt` and `on_err_sql` allow conditional recovery logic

## Example Use Cases

* Dropping temporary or staging tables
* Vacuuming or analyzing databases
* Refreshing materialized views
* Applying schema migrations
* Executing post-load cleanup logic

## Scripts Benefits

* **Operational Flexibility**
  Execute any SQL without forcing it into a data-oriented block.

* **Clear Separation of Concerns**
  Keeps maintenance and orchestration logic isolated from ETL and analytics.

* **Pipeline Integration**
  Scripts run as first-class citizens in ETLX workflows.

* **Consistent Lifecycle Model**
  Uses the same metadata and execution hooks as other ETLX blocks.

The `SCRIPTS` block is intentionally minimal, powerful, and predictable—making it ideal for operational SQL that supports, but does not define, your data products.

---
weight: 710
title: "ETL | ELT"
description: "How ETLX models, executes, and observes ETL and ELT pipelines using declarative, metadata-driven configurations."
icon: auto_awesome
date: 2025-12-16T01:04:15+00:00
lastmod: 2025-12-16T01:04:15+00:00
draft: false
images: []
---
---

## ETL | ELT in ETLX

ETLX supports both **ETL (Extract–Transform–Load)** and **ELT (Extract–Load–Transform)** execution models.

Rather than enforcing a specific pattern, ETLX lets you **declare the intent of the pipeline**, and then executes it deterministically based on metadata.

At the highest level, an ETL or ELT pipeline is defined by:

* A **root block** describing the pipeline
* One or more **execution units** (inputs, transformations, outputs)
* Explicit **SQL blocks** defining the logic
* Optional **error handling and lifecycle hooks**

---

## Defining an ETL or ELT Pipeline

A pipeline is declared using a **level-one Markdown heading** (`#`) combined with metadata.

The execution mode is defined using the `runs_as` key:

* `runs_as: ETL`
* `runs_as: ELT`

### Example

````md {linenos=table}
# INPUTS
```yaml
name: INPUTS
description: Extracts data from source and loads it into the target
runs_as: ETL
active: true
```

## INPUT_1
```yaml
name: INPUT_1
description: Input 1 from an ODBC source
table: INPUT_1
load_conn: "duckdb:"
load_before_sql:
  - "ATTACH 'ducklake:@DL_DSN_URL' AS DL (DATA_PATH 's3://dl-bucket...')"
  - "ATTACH '@OLTP_DSN_URL' AS PG (TYPE POSTGRES)"
load_sql: load_input_in_dl
load_on_err_match_patt: '(?i)table.+with.+name.+(\w+).+does.+not.+exist'
load_on_err_match_sql: create_input_in_dl
load_after_sql:
  - DETACH DL
  - DETACH PG
active: true
```

```sql
-- load_input_in_dl
INSERT INTO DL.INPUT_1 BY NAME
SELECT * FROM PG.INPUT_1
```

```sql
-- create_input_in_dl
CREATE TABLE DL.INPUT_1 AS
SELECT * FROM PG.INPUT_1
```
````

> `@DL_DSN_URL` (e.g. `mysql:db=ducklake_catalog host=your_mysql_host`) and `@OLTP_DSN_URL` (e.g. `postgres:dbname=erpdb host=your_postgres_host user=postgres password=your_pass`) are **environment variables** used to define database connection strings.
>They can be provided through a `.env` file located at the root of the project and are automatically loaded at runtime.
>These variables allow ETLX to connect to different data sources without hardcoding credentials, making configurations portable, secure, and environment-agnostic.

## Execution Model

### 1. Pipeline Initialization

Execution starts at the **root pipeline block** (`# INPUTS`).

From this block, ETLX extracts:

* **Pipeline metadata**

  * `name`
  * `description`
  * `runs_as` (ETL or ELT)
* **Global execution context**

  * Default connection
  * Execution timestamp
  * Runtime metadata
* **Activation state**

  * Pipelines marked as `active: false` are skipped

This metadata becomes part of the **execution trace** and **observability layer**.

### 2. Iteration Over Execution Units

Each **level-two heading** (`## INPUT_1`, `## TRANSFORM_X`, etc.) represents an **execution unit**.

For each unit, ETLX:

1. Reads its metadata
2. Resolves connections
3. Determines which steps apply
4. Executes steps in a deterministic order

Inactive units (`active: false`) are skipped but still recorded in metadata.

## ETL / ELT Steps

Each execution unit may define one or more **steps**, depending on the execution model:

* `extract`
* `transform`
* `load`

Each step supports a consistent lifecycle:

### Step Lifecycle Hooks

For any step `<step>` (e.g. `load`):

| Hook                | Purpose                                        |
| ------------------- | ---------------------------------------------- |
| `<step>_before_sql` | Setup (e.g. attach databases, prepare schemas) |
| `<step>_sql`        | Main execution logic                           |
| `<step>_after_sql`  | Cleanup (detach, finalize, release resources)  |

* `<step>_before_sql`, `[before_sql]` can also be `before_query|before|start|startup|setup`
* `<step>_sql`, `[_sql]` can also be nothing or `_query|_main`
* `<step>_after_sql`, `[after_sql]` can also be `after_query|after|end|cleanup`

### SQL Resolution Rules

Each SQL hook can be defined as:

* `null` → step is skipped
* `string` → resolved according to the following rules:

  1. **Named SQL block** → if the value matches a named SQL/query section, that section is resolved and used as the query.
  2. **SQL code block** → if the value references a SQL code block, its contents are used as the query.
  3. **Text file** → if the value is not a named SQL section or SQL code block, but is a valid path to an existing text file, ETLX loads the file contents and uses them as the query.
  4. **Inline SQL** → otherwise, the string itself is treated as SQL and executed directly.
* list/array → each item is resolved using the same rules and executed sequentially

Named SQL blocks are resolved from:

* `sql [query_name]` blocks
* Or SQL comments: `-- query_name`

For example, a SQL hook can reference a query defined elsewhere:

```yaml
load_sql: load_input_in_dl
```

with:

```sql {linenos=table}
-- load_input_in_dl
INSERT INTO DL.INPUT_1 BY NAME
SELECT * FROM PG.INPUT_1
```

Alternatively, the query can be stored in an external text file:

```yaml {linenos=table}
load_sql: sql/load_input_in_dl.sql
```

If `sql/load_input_in_dl.sql` exists and the value does not resolve to a named SQL section or SQL code block, ETLX reads the file's text content and uses that content as the query.

This allows SQL to be organized either directly in the ETLX document or in external SQL files, while preserving the ability to use inline SQL when appropriate.

This resolution mechanism provides **clear separation of metadata and logic** while allowing SQL definitions to remain flexible and reusable.

### Step Data Queries

Each ETL step (`extract`, `transform`, or `load`) can define a `<step>_data` configuration. The data query is executed using the same database connection defined by `<step>_conn`.

The purpose of `<step>_data` is to execute one or more queries and make their results available to the rest of the ETLX execution through the `data` key of the current item.

For example:

```yaml {linenos=table}
load_conn: "duckdb:"
load_data:
  - pending_dates
  - source_config
load_sql: load_template
```

The queries defined in `load_data` are executed before the main `load_sql` query.

#### Data Result Structure

The results are stored in:

```go {linenos=table}
item["data"]
```

The value is a `map[string]any`, where each key corresponds to the query defined in `<step>_data`.

Each query result has the following structure:

```go {linenos=table}
map[string]any{
    "success": true,
    "data": []map[string]any{
        // query result rows
    },
}
```

For example, if the configuration contains:

```yaml {linenos=table}
load_data:
  - pending_dates
  - customers
```

the resulting item contains:

```go {linenos=table}
item["data"] = map[string]any{
    "pending_dates": map[string]any{
        "success": true,
        "data": []map[string]any{
            {"date_ref": "2026-09-20"},
            {"date_ref": "2026-09-21"},
        },
    },
    "customers": map[string]any{
        "success": true,
        "data": []map[string]any{
            {"id": 1, "name": "John"},
            {"id": 2, "name": "Mary"},
        },
    },
}
```

This makes the query results available to subsequent ETLX processing and SQL templates through the current `item`.

#### Single or Multiple Data Queries

`<step>_data` can contain either a single query or a list of queries.

A single query:

```yaml {linenos=table}
load_data: pending_dates
```

or multiple queries:

```yaml {linenos=table}
load_data:
  - pending_dates
  - customers
  - configuration
```

Each query is identified by its configured value. The query can use the normal ETLX SQL resolution rules, including named SQL queries and SQL definitions available in the configuration.

#### Query Failure

Each data query reports its own execution status through the `success` property.

A successful query:

```go {linenos=table}
"success": true
```

A failed query:

```go {linenos=table}
"success": false,
"msg": "failed to execute map query pending_dates ...",
"data": []map[string]any{}
```

When a query fails, its `data` value is an empty collection. This allows the execution result to retain information about the failure without returning an undefined result structure.

#### Using Data in SQL Templates

The `data` object can be used together with ETLX SQL templates to generate SQL dynamically.

For example:

```yaml {linenos=table}
load_data:
  - pending_dates
load_sql: load_template
```

where `pending_dates` returns:

```text {linenos=table}
date_ref
----------
2026-09-20
2026-09-21
2026-09-22
```

The `load_template` SQL can then access the result through `.data`:

```sql {linenos=table}
-- load_template
INSERT INTO destination
{{- range $i, $row := (index .data "pending_dates").data }}
{{ if $i }}UNION ALL{{ end }}
SELECT *
FROM source
WHERE date_ref = '{{$row.date_ref}}'
{{- end }}
```

or

```sql {linenos=table}
-- load_template
{{- range $i, $row := (index .data "pending_dates").data }}
INSERT INTO destination
SELECT *
FROM source
WHERE date_ref = '{{$row.date_ref}}';
{{- end }}
```

The `data` object therefore provides a convenient way to use the result of one or more queries as input when dynamically generating the SQL for the main ETL step.

#### Connection

`<step>_data` uses the same connection as `<step>_conn`.

For example:

```yaml
load_conn: "duckdb:database/load.db"
load_data:
  - pending_dates
```

Both the `pending_dates` query and the main `load_sql` execution use:

```text {linenos=table}
duckdb:database/load.db
```

This keeps the data-query and main-query execution within the same database context.

> **Note:** Data queries are executed before the main query of the step. If a data query is required to generate the main SQL, the generated SQL should use the data available through `item.data`.

## Connection Handling

Each step can specify a connection using `<step>_conn`.

* If defined → used for that step
* If `null` or omitted → falls back to the pipeline’s default connection

This enables **multi-engine pipelines** (DuckDB, Postgres, ODBC, etc.) within a single execution.

## Error Handling & Recovery

ETLX provides **pattern-based error handling**, allowing pipelines to recover dynamically from known failure conditions.

For any step `<step>`:

* `<step>_on_err_match_patt`
  A regular expression matched against the database error message
* `<step>_on_err_match_sql`
  SQL executed when the pattern matches

This is especially useful for:

* Creating missing tables
* Initializing schemas
* Handling first-run scenarios
* Working with evolving datasets

The same mechanism applies to:

* `<step>_before_on_err_match_*`

## Observability & Execution Metadata

Every ETLX run is **fully observable by design**.

For each pipeline, step, and sub-step, ETLX records:

* Start and end timestamps
* Execution duration
* Connection used
* SQL executed
* Rows affected (when available)
* Errors, warnings, and retries
* Memory and resource usage

This metadata can be:

* Logged
* Stored
* Queried via SQL
* Used to generate reports and documentation

## Design Principles

ETLX treats ETL and ELT pipelines as:

* **Declarative execution plans**
* **Structured metadata documents**
* **Executable documentation**

This ensures pipelines are:

* Reproducible
* Inspectable
* Auditable
* Portable across environments
* Easy to maintain and evolve

## Summary

In ETLX:

* ETL and ELT are **modes**, not rigid architectures
* Configuration defines *intent*, not orchestration
* SQL remains first-class
* Metadata powers execution, observability, and documentation
* Pipelines are self-describing and deterministic

> **Your pipeline configuration is your source of truth.**

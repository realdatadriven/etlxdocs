---
weight: 950
title: "Schema Update Example"
description: "An example demonstrating how to safely evolve a target table schema when source schema / data changes."
icon: overview
date: 2026-01-08T01:04:15+00:00
lastmod: 2026-01-08T01:04:15+00:00
draft: false
images: []
---

## Schema Update Example

There are many scenarios where you load **all columns from a source dataset** into a target table and expect the schema to remain stable over time.

In practice, this assumption often breaks.

A common failure mode is:

* The source dataset introduces **new columns**
* Your pipeline still tries to insert into the existing table
* The workflow fails due to a schema mismatch

This is a perfect use case for **dynamic schema handling**.

ETLX provides this capability via [Dynamic Query Generation]({{% relref "../features/advanced#advanced-usage-dynamic-query-generation-get_dyn_queries" %}}), allowing you to **compare source and target schemas at runtime** and generate the required SQL automatically.

## Use Case: Monthly Incremental Loads with Schema Evolution

Following the [Embedded SQLite]({{% relref "embedded-sqlite#extract-and-transform-data" %}}) example, imagine the following workflow:

* Every month you ingest a new file: `yellow_tripdata_{YYYY-MM}.parquet`
* You **append** new data instead of replacing the table
* You must **avoid duplicate monthly loads**
* The schema may **evolve over time**

To support this safely, we need to:

1. Validate that the month has not already been loaded
2. Detect new columns in the source
3. Update the target table schema if necessary
4. Append the new data


## Updated `TRIP_DATA` Load Definition

Below is an updated `## TRIP_DATA` block that implements this logic.

````md {linenos=table,hl_lines=[3,"5-7"],linenostart=11}
...
## TRIP_DATA
```yaml
name: TRIP_DATA
description: "Example extracting trip data from the web into a local database"
table: TRIP_DATA
load_conn: "duckdb:"
load_before_sql: "ATTACH 'sqlite_ex.db' AS DB (TYPE SQLITE)"

# Because we are appending data, validate that the reference month
# does not already exist to prevent duplicate loads
load_validation:
  - type: throw_if_not_empty
    sql: FROM "DB"."TRIP_DATA" WHERE "ref_date" = '{YYYY-MM}' LIMIT 10
    msg: "This date is already imported — aborting to avoid duplicates."

# Load strategy:
# 1. Dynamically generate ALTER TABLE statements for missing columns
# 2. Append new data into the table
load_sql:
  - "get_dyn_queries[create_missing_columns](ATTACH 'sqlite_ex.db' AS DB (TYPE SQLITE), DETACH DB)"
  - update_trip_data_schema

# If the table does not exist, create it instead
load_on_err_match_patt: '(?i)table.+with.+name.+(\w+).+does.+not.+exist'
load_on_err_match_sql: create_trip_data_table

load_after_sql: DETACH "DB"

# difine the file path / url that can fill the placeholder <fname>|<filename>
file: "https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_{YYYY-MM}.parquet"
active: true
```
````

### Append Data into an Existing Table

If the table exists and the schema is compatible, data is appended using:

```sql
-- update_trip_data_schema
INSERT INTO "DB"."<table>" BY NAME
SELECT *, '{YYYY-MM}' AS "ref_date"
FROM READ_PARQUET('<fname>')
```

Using `BY NAME` ensures that:

* Columns are matched by name
* Column order changes do not break the load
* Newly added columns are handled safely

### Create the Table If It Does Not Exist

If the table does not yet exist, it is created directly from the source:

```sql
-- create_trip_data_table
CREATE TABLE "DB"."<table>" AS
SELECT *, '{YYYY-MM}' AS "ref_date"
FROM READ_PARQUET('<fname>')
```

### Dynamically Generate Missing Columns

The following query detects columns present in the source file but missing in the target table and generates the required `ALTER TABLE` statements.

```sql
-- create_missing_columns
WITH source_columns AS ( -- Columns from the source parquet file
    SELECT "column_name", "column_type"
    FROM (DESCRIBE SELECT * FROM READ_PARQUET('<fname>'))
),
destination_columns AS ( -- Columns from the destination table
    SELECT "column_name", "data_type" AS "column_type"
    FROM "duckdb_columns"
    WHERE "table_name" = '<table>'
),
missing_columns AS ( -- Columns present in source but missing in destination
    SELECT s."column_name", s."column_type"
    FROM source_columns s
    LEFT JOIN destination_columns d ON s."column_name" = d."column_name"
    WHERE d."column_name" IS NULL
)
SELECT 'ALTER TABLE "DB"."<table>"
    ADD COLUMN "' || "column_name" || '" ' || "column_type" || ';' AS "query"
FROM missing_columns
WHERE (SELECT COUNT(*) FROM destination_columns) > 0;
```

These generated statements are executed automatically by ETLX before the append step.

## Why This Pattern Matters

This is an advanced but **highly practical** pattern that enables:

* Incremental loads
* Schema evolution without manual intervention
* Duplicate protection
* Declarative, self-healing pipelines

Rather than failing when the schema changes, the pipeline **adapts safely and transparently**.

> In ETLX, schema evolution is not an exception — it is an explicit, observable part of the pipeline.

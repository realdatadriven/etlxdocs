---
weight: 710
title: "ETL | ELT"
description: "ETL (Extract, Transform, Load) and ELT (Extract, Load, Transform) are two data integration approaches used to move and process data from source systems to target systems, such as data warehouses or data lakes."
icon: auto_awesome
date: 2022-12-16T01:04:15+00:00
lastmod: 2022-12-16T01:04:15+00:00
draft: false
images: []
---

## ETL | ELT

- Defines the overall ETL process with a level one heading `# ETL` or any level one heading where in metadata the key `runs_as: ETL` or `runs_as: ELT` is defined.
- Example:

````md {linenos=table style=emacs}
# INPUTS
```yaml
name: INPUTS
description: Extracts data from source and load on target
runs_as: ETL
active: true
```

## INPUT_1
```yaml
name: INPUT_1
description: Input 1 from an ODBC Source
table: INPUT_1 # Destination Table
load_conn: "duckdb:"
load_before_sql:
  - "ATTACH 'ducklake:@DL_DSN_URL' AS DL (DATA_PATH 's3://dl-bucket...')"
  - "ATTACH '@OLTP_DSN_URL' AS PG (TYPE POSTGRES)"
load_sql: load_input_in_dl
load_on_err_match_patt: '(?i)table.+with.+name.+(\w+).+does.+not.+exist'
load_on_err_match_sql: create_input_in_dl
load_after_sql:
  - DETACH DL
  - DETACH pg
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
...

````

### 1. **ETL Process Starts**

- Begin with the "ETL" block;
- Extract metadata, specifically:
  - "connection": Main connection to the destination database.
  - "description": For documentation porpose and logging the ETL process.

### 2. **Loop through Level 2 key in under "ETL" block**

- Iterate over each key (e.g., "INPUT_1")
- For each key, access its "metadata" to process the ETL steps.

### 3. **ETL Steps**

- Each ETL step (`extract`, `transform`, `load`) has:
  - `_before_sql`: Queries to run first (setup).
  - `_sql`: The main query or queries to run.
  - `_after_sql`: Cleanup queries to run afterward.
- Queries can be:
  - `null`: Do nothing.
  - `string`: Reference a single query key in the same map or the query itself.
  - `array|list`: Execute all queries in sequence.
  - In case is not null it can be the query itself or just the name of a sql code block under the same key, where `sql [query_name]` or first line `-- [query_name]`
- Use `_conn` for connection settings. If `null`, fall back to the main connection.
- Additionally, error handling can be defined using `[step]_on_err_match_patt` and `[step]_on_err_match_sql` to handle specific database errors dynamically, where `[step]_on_err_match_patt` is the `regexp` patthern to match error,and if maches the `[step]_on_err_match_sql` is executed, the same can be applied for `[step]_before_on_err_match_patt` and `[step]_before_on_err_match_sql`.
   You can define patterns to match specific errors and provide SQL statements to resolve those errors. This feature is useful when working with dynamically created databases, tables, or schemas.

### 4. **Output Logs**
- Log progress (e.g., connection usage, start/end times, descriptions, message, success or failure, memory usage, ...).
- Gracefully handle missing or `null` keys.

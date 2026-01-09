---
weight: 810
title: "Advanced Features"
description: "Explore advanced ETLX features like dynamic query generation, conditional execution, and modular configurations."
icon: auto_awesome
tags: ["features"]
date: 2025-12-16T01:04:15+00:00
lastmod: 2025-12-16T01:04:15+00:00
draft: false
images: []
---

## Advanced Features

- Error Handling:
  Define patterns for resolving errors dynamically during execution.

  ```yaml {linenos=table}
  load_on_err_match_patt: "(?i)table.+does.+not.+exist"
  load_on_err_match_sql: "CREATE TABLE sales_table (id INT, total FLOAT)"
  ```

- Modular Configuration:
  Break down workflows into reusable components for better maintainability.

### 🛠️ Advanced Usage: Dynamic Query Generation (`get_dyn_queries[...]`)

In some **advanced ETL workflows**, you may need to **dynamically generate SQL queries** based on metadata or schema differences between the source and destination databases.

#### **🔹 Why Use Dynamic Queries?**

✅ **Schema Flexibility** – Automatically adapt to schema changes in the source system.  
✅ **Self-Evolving Workflows** – ETL jobs can generate and execute additional SQL queries as needed.  
✅ **Automation** – Reduces the need for manual intervention when new columns appear.  

#### **🔹 How `get_dyn_queries[query_name](runs_before,runs_after)` Works**

- Dynamic queries are executed using the **`get_dyn_queries[query_name](runs_before,runs_after)`** pattern.
- During execution, **ETLX runs the query** `query_name` and **retrieves dynamically generated queries**.
- The **resulting queries are then executed automatically**.

#### **🛠 Example: Auto-Adding Missing Columns**

This example **checks for new columns in a JSON file** and **adds them to the destination table**.

##### **📄 Markdown Configuration for `get_dyn_queries[query_name](runs_before,runs_after)`**

>If the `query_name` depends on attaching and detaching the main db where it will run, those should be passed as dependencies, because the dynamic queries are generate before any other query and put in the list for the list where it is to be executed, to be a simpler flow, but they are optional otherwise.

````md {linenos=table}
....

```yaml metadata
...
connection: "duckdb:"
before_sql:
  - ...
  - get_dyn_queries[create_missing_columns]  # Generates queries defined in `create_missing_columns` and  Executes them
..
```

**📜 SQL Query (Generating Missing Columns)**

```sql
-- create_missing_columns
WITH source_columns AS (
    SELECT column_name, column_type 
    FROM (DESCRIBE SELECT * FROM read_json('<fname>'))
),
destination_columns AS (
    SELECT column_name, data_type as column_type
    FROM duckdb_columns 
    WHERE table_name = '<table>'
),
missing_columns AS (
    SELECT s.column_name, s.column_type
    FROM source_columns s
    LEFT JOIN destination_columns d ON s.column_name = d.column_name
    WHERE d.column_name IS NULL
)
SELECT 'ALTER TABLE "<table>" ADD COLUMN "' || column_name || '" ' || column_type || ';' AS query
FROM missing_columns
WHERE (SELECT COUNT(*) FROM destination_columns) > 0;
```
...
````

#### **🛠 Execution Flow**

1️⃣ **Extract column metadata from the input (in this case a json file, but it could be a table or any other valid query).**  
2️⃣ **Check which columns are missing in the destination table (`<table>`).**  
3️⃣ **Generate `ALTER TABLE` statements for adding missing columns, and replaces the `- get_dyn_queries[create_missing_columns]` with the the generated queries**  
4️⃣ **Runs the workflow with dynamically generated queries against the destination connection.**  

#### **🔹 Key Features**

✔ **Fully automated schema updates**  
✔ **Works with flexible schema data (e.g., JSON, CSV, Parquet, etc.)**  
✔ **Reduces manual maintenance when source schemas evolve**  
✔ **Ensures destination tables always match source structure**  

---

**With `get_dyn_queries[...]`, your ETLX workflows can now dynamically evolve with changing data structures!**

### **🔄 Conditional Execution**

ETLX allows conditional execution of SQL blocks based on the results of a query. This is useful to skip operations dynamically depending on data context (e.g., skip a step if no new data is available, or if a condition in the target is not met.

You can define condition blocks using the following keys:

- For **ETL step-specific conditions**:
  - `extract_condition`
  - `transform_condition`
  - `load_condition`
  - etc.

- For **generic sections** (e.g., DATA_QUALITY, EXPORTS, NOTIFY, etc.):
  - `condition`

You can also specify an optional `*condition_msg` to log a custom message when a condition is not met.

#### **Condition Evaluation Logic**

- The SQL query defined in `*_condition` or `condition` is executed.
- The result must mast be boolean.
- If not met, the corresponding main logig will be skipped.
- If `*_condition_msg` is provided, it will be included in the log entry instead of the default skip message.

#### **Example – Conditional Load Step**

```yaml {linenos=table}
load_conn: "duckdb:"
load_condition: check_load_required
load_condition_msg: "No new records to load today"
load_sql: perform_load
```

```sql
-- check_load_required
SELECT COUNT(*) > 0 as _check FROM staging_table WHERE processed = false;
```

```sql
-- perform_load
INSERT INTO target_table
SELECT * FROM staging_table WHERE processed = false;
```

#### **Example - Global Conditional Notification**

```yaml {linenos=table}
description: "Send email only if failures occurred"
connection: "duckdb:"
condition: check_failures
condition_msg: "No failures detected, no email sent"
```

```sql
-- check_failures
SELECT COUNT(*) > 0 as chk FROM logs WHERE success = false;
```

> 📝 **Note:** If no `*_condition_msg` is defined and the condition fails, ETLX will simply log the skipped step with a standard message like:  
> `"Condition 'load_condition' was not met. Skipping step 'load'."`

---

### **Advanced Workflow Execution: `runs_as` Override**

By default, the ETLX engine processes each Level 1 section (like `ETL`, `DATA_QUALITY`, `EXPORTS`, `ACTIONS`, `LOGS`, `NOTIFY` etc.) in the order that order. However, in more advanced workflows, it is often necessary to:

- Execute a second ETL process **after** quality validations (`DATA_QUALITY`).
- Reuse intermediate outputs **within the same config**, without having to create and chain multiple `.md` config files.

To enable this behavior, ETLX introduces the `runs_as` field in the **metadata block** of any Level 1 key. This tells ETLX to treat that section **as if it were a specific built-in block** like `ETL`, `EXPORTS`, etc., even if it has a different name.

---

````markdown

# ETL_AFTER_SOME_KEY

```yaml metadata
runs_as: ETL
description: "Post-validation data transformation"
active: true
```

## ETL_OVER_SOME_MAIN_STEP

...

````

In this example:

- ETLX will run the original `ETL` block.
- Then execute `DATA_QUALITY`, an so on.
- Then treat `ETL_AFTER_SOME_KEY` as another `ETL` block (since it contains `runs_as: ETL`) and execute it as such.

This allows chaining of processes within the same configuration file.

---

### **⚠️ Order Matters**

The custom section (e.g. `# ETL_AFTER_SOME_KEY`) is executed **in the order it appears** in the Markdown file after the main keys. That means the flow becomes:

1. `# ETL`
2. `# DATA_QUALITY`
3. `# ETL2` (runs as `ETL`)

This enables advanced chaining like:

- Exporting logs after validation.
- Reapplying transformations based on data quality feedback.
- Generating post-validation reports.

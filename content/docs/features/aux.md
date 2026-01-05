---
weight: 730
title: "Query Documentation"
description: "How ETLX models SQL queries as executable documentation to improve clarity, governance, and reusability."
icon: check-circle
tags: ["features", "governance"]
date: 2025-12-16T01:04:15+00:00
lastmod: 2025-12-16T01:04:15+00:00
draft: false
images: []
---

## Query Documentation

In ETLX, SQL queries are not treated as opaque strings. Instead, they are modeled as **structured, inspectable, and executable documents**.

This is particularly important in ETL and ELT pipelines where transformation logic tends to grow organically and quickly becomes difficult to understand, govern, and audit. Query Documentation addresses this problem by allowing SQL logic to be decomposed into **field-level components enriched with metadata**.

The result is not just better readability, but a foundation for **data governance, lineage, and automated documentation**.

## Why Query Documentation Exists in ETLX

Traditional ETL systems treat SQL as an implementation detail. ETLX treats it as **knowledge**.

By documenting queries as structured metadata, ETLX enables:

* Transparent and explainable transformations
* Column-level data dictionaries
* Source-to-target lineage
* Ownership and accountability
* Auditability and reproducibility

Query Documentation is therefore not a formatting convenience — it is a **core modeling primitive** in ETLX.

## Mental Model: QueryDoc

A **QueryDoc** is a first-class ETLX object that represents a documented SQL query.

A QueryDoc:

* Has a unique name
* Contains descriptive metadata (owner, description, source, etc.)
* Is composed of documented fields
* Resolves deterministically into executable SQL
* Can be referenced and reused across pipelines

At runtime, ETLX assembles the QueryDoc into a valid SQL statement, records the resolved SQL, and executes it as part of the pipeline.

## Structure

A documented query is defined as a **level-one Markdown heading** (`# QUERY_NAME`).

Each field participating in the query is represented as a **level-two heading** (`## field_name`).

For each field:

* Optional YAML metadata can describe the field (`description`, `type`, `owner`, `derived_from`, `formula`, etc.)
* SQL components are split into separate SQL blocks such as:

  * `select`
  * `from`
  * `join`
  * `where`
  * `group_by`
  * `having`
  * `order_by`
  * `cte`

Instead of maintaining a single large SQL string, ETLX assembles these components at runtime in a deterministic order.

This approach requires intentional design, but pays off by making transformations **self-documenting and governable**.

---

## Example: Documenting an Extraction Query

The following example shows how ETLX can be extended with metadata keys useful for governance when extracting data from an external source.

The example uses the NYC Yellow Taxi January 2024 Parquet dataset hosted at:

`https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2024-01.parquet`

````md {linenos=table}
# EXTRACT_LOAD

```yaml metadata
name: EXTRACT_LOAD
runs_as: ETL
description: |
  Extracts and Loads the data sets to the local analitical database
connection: "sqlite3:database/DB_EX_DGOV.db"
database: "sqlite3:database/DB_EX_DGOV.db"
active: true
```

## TRIP_DATA

```yaml metadata
name: TRIP_DATA
description: "Example extrating trip data from web to a local database"
table: TRIP_DATA
load_conn: "duckdb:"
load_before_sql: "ATTACH 'database/DB_EX_DGOV.db' AS DB (TYPE SQLITE)"
load_sql: extract_load_trip_data
load_after_sql: DETACH "DB"
active: false
```

```sql
-- extract_load_trip_data
CREATE OR REPLACE TABLE "DB"."<table>" AS
FROM read_parquet('https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2024-01.parquet')
```
...
````
### Replacing Inline SQL with a QueryDoc

If your organization requires a standardized data dictionary for every input — or if the same extraction logic is reused across multiple pipelines — the query can be externalized into a QueryDoc.

````md {linenos=table}
# EXTRACT_LOAD

...

```sql
-- extract_load_trip_data
CREATE OR REPLACE TABLE "DB"."<table>" AS
[[QUERY_EXTRACT_TRIP_DATA]]
```
````
The placeholder `[[QUERY_EXTRACT_TRIP_DATA]]` references a documented query defined elsewhere in the configuration.

## Query Resolution Semantics

Query placeholders are resolved **before execution**.

ETLX guarantees that:

* All referenced QueryDocs must exist
* SQL components are assembled in a deterministic order
* Missing or incompatible components fail fast
* The fully resolved SQL is logged and observable

This ensures that documented queries remain **safe, predictable, and auditable**.


## Documenting Fields with Metadata

...
````md {linenos=table}
# QUERY_EXTRACT_TRIP_DATA

This QueryDoc extracts selected fields from the NYC Yellow Taxi Trip Record dataset.

```yaml metadata
name: QUERY_EXTRACT_TRIP_DATA
description: "Extracts essential NYC Yellow Taxi trip fields (with governance metadata)."
owner: taxi-analytics-team
details: "https://www.nyc.gov/assets/tlc/downloads/pdf/data_dictionary_trip_records_yellow.pdf"
source:
  uri: "https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2024-01.parquet"
  format: parquet
```

## VendorID
```yaml metadata
name: VendorID
description: "A code indicating which TPEP provider generated the record.
1=CMT, 2=Curb, 6=Myle, 7=Helix."
type: integer
owner: data-providers
```

```sql
-- select
SELECT VendorID
```

```sql
-- from
FROM read_parquet('https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2024-01.parquet')
```

## tpep_pickup_datetime
```yaml metadata
name: tpep_pickup_datetime
description: "Timestamp when the meter was engaged (trip start)."
type: timestamp
owner: taxi-analytics-team
```

```sql
-- select
    , tpep_pickup_datetime
```
...

## RatecodeID
```yaml metadata
name: RatecodeID
description: "Final rate code at trip end.
1=Standard, 2=JFK, 3=Newark, 4=Nassau/Westchester,
5=Negotiated fare, 6=Group ride, 99=Unknown."
type: integer
owner: finance
```
...

## payment_type
```yaml metadata
name: payment_type
description: "How the passenger paid:
0=Flex Fare, 1=Credit card, 2=Cash, 3=No charge, 4=Dispute, 5=Unknown, 6=Voided trip."
type: integer
owner: finance
```
...
````

This approach allows ETLX to generate or support:

* Column-level data dictionaries
* Field ownership maps
* Source attribution
* Lineage inputs for external tools

## Documenting Transformation Queries

Query Documentation becomes even more valuable when applied to **derived and calculated fields**, where logic is owned by the organization and rarely documented elsewhere.

The following example documents a transformation that identifies the most popular taxi routes.

````md {linenos=table}
...
# TRANSFORM

```yaml metadata
name: TRANSFORM
runs_as: ETL
description: Transforms the inputs into to desrable outputs
connection: "sqlite3:database/DB_EX_DGOV.db"
database: "sqlite3:database/DB_EX_DGOV.db"
active: true
```

## MostPopularRoutes

```yaml metadata
name: MostPopularRoutes
description: |
  Most Popular Routes - Identify the most common pickup-dropoff route combinations to understand travel patterns.
table: MostPopularRoutes
transform_conn: "duckdb:"
transform_before_sql: "ATTACH 'database/DB_EX_DGOV.db' AS DB (TYPE SQLITE)"
transform_sql: trf_most_popular_routes
transform_after_sql: DETACH "DB"
drop_sql: DROP TABLE IF EXISTS "DB"."<table>"
clean_sql: DELETE FROM "DB"."<table>"
rows_sql: SELECT COUNT(*) AS "nrows" FROM "DB"."<table>"
active: true
```

```sql
-- trf_most_popular_routes
CREATE OR REPLACE TABLE "DB"."<table>" AS
[[QUERY_TOP_ZONES]]
LIMIT 15
```

# QUERY_TOP_ZONES

```yaml metadata
name: QUERY_TOP_ZONES
description: "Most common pickup/dropoff zone combinations with aggregated metrics."
owner: taxi-analytics-team
source:
  - TRIP_DATA
  - ZONES
```

## pickup_borough

```yaml metadata
name: pickup_borough
description: "Borough of the pickup location (from ZONES lookup)."
type: string
derived_from:
  - TRIP_DATA.PULocationID
  - ZONES.Borough
formula: |
  This field is a direct lookup of the pickup borough using the pickup location ID.
```

```sql
-- select
SELECT zpu.Borough AS pickup_borough
```

```sql
-- from
FROM DB.TRIP_DATA AS t
```

```sql
-- join
JOIN DB.ZONES AS zpu ON t.PULocationID = zpu.LocationID
```

```sql
-- group_by
GROUP BY pickup_borough
```

## pickup_zone

```yaml metadata
name: pickup_zone
description: "Zone name of the pickup location (from ZONES lookup)."
type: string
derived_from:
  - TRIP_DATA.PULocationID
  - ZONES.Zone
formula: |
  This field is a direct lookup of the pickup zone using the pickup location ID.
```

```sql
-- select
    , zpu.Zone AS pickup_zone
```

```sql
-- group_by
    , pickup_zone
```

## dropoff_borough

```yaml metadata
name: dropoff_borough
description: "Borough of the dropoff location (from ZONES lookup)."
type: string
derived_from:
  - TRIP_DATA.DOLocationID
  - ZONES.Borough
formula: |
  This field is a direct lookup of the dropoff borough using the dropoff location ID.
```

```sql
-- select
    , zdo.Borough AS dropoff_borough
```

```sql
-- join
JOIN DB.ZONES AS zdo ON t.DOLocationID = zdo.LocationID
```

```sql
-- group_by
    , dropoff_borough
```

## dropoff_zone

```yaml metadata
name: dropoff_zone
description: "Zone name of the dropoff location (from ZONES lookup)."
type: string
derived_from:
  - TRIP_DATA.DOLocationID
  - ZONES.Zone
formula: |
  This field is a direct lookup of the dropoff zone using the dropoff location ID.
```

```sql
-- select
    , zdo.Zone AS dropoff_zone
```

```sql
-- group_by
    , dropoff_zone
```

## total_trips

```yaml metadata
name: total_trips
description: "Total number of trips between each pickup/dropoff zone pair."
type: integer
derived_from:
  - TRIP_DATA.*
formula: | 
    "
    Count the number of trips in each pickup/dropoff combination.

    **Formula (LaTeX):**

    ```latex
    \text{total\_trips} = \sum_{i=1}^{N} 1
    ```"
```

```sql
-- select
    , COUNT(*) AS total_trips
```

```sql
-- order_by
ORDER BY total_trips DESC
```

## avg_fare

```yaml metadata
name: avg_fare
description: "Average total fare for trips between the selected pickup and dropoff zones."
type: numeric
derived_from:
  - TRIP_DATA.total_amount
formula: | 
    "
    Compute the arithmetic mean of total fares for the group, rounded to 2 decimals.

    **Formula (LaTeX):**

    ```latex
    \text{avg\_fare} = \text{round}\!\left(\frac{1}{N}\sum_{i=1}^{N} \text{total\_amount}_i,\ 2\right)
    ```"
```

```sql
-- select
    , ROUND(AVG(t.total_amount), 2) AS avg_fare
```

## avg_distance

```yaml metadata
name: avg_distance
description: "Average trip distance (miles)."
type: numeric
derived_from:
  - TRIP_DATA.trip_distance
formula: | 
    "
    Compute the arithmetic mean of trip distances for the group, rounded to 2 decimals.

    **Formula (LaTeX):**

    ```latex
    \text{avg\_distance} = \text{round}\!\left(\frac{1}{N}\sum_{i=1}^{N} \text{trip\_distance}_i,\ 2\right)
    ```"

```

```sql
-- select
    , ROUND(AVG(t.trip_distance), 2) AS avg_distance
```
...
````
Each derived field documents:

* Business meaning
* Source fields
* Transformation logic
* Ownership

This turns transformation SQL into **executable business documentation**.

## What This Is — and Is Not

Query Documentation in ETLX is:

* A structured SQL modeling approach
* A metadata foundation for governance
* A bridge between execution and documentation

It is **not**:

* A replacement for SQL
* A data catalog
* A BI semantic layer

Instead, ETLX provides high-quality metadata that can feed those systems.

## Summary

In ETLX:

* SQL expresses logic
* Metadata provides context
* Together they form executable knowledge

Query Documentation ensures that complex queries remain understandable, auditable, and reusable — without sacrificing the power and flexibility of SQL.

Your pipeline configuration becomes not just runnable code, but **a durable source of truth**.

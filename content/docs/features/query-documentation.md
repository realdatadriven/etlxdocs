---
weight: 720
title: "Query Documentation"
description: "How ETLX provides query documentation features to enhance data clarity and governance."
icon: check-circle
tags: ["features"]
date: 2022-12-16T01:04:15+00:00
lastmod: 2022-12-16T01:04:15+00:00
draft: false
images: []
---

## Query Documentation

In some ETL processes, particularly during the **Transform** step, queries may become too complex to manage as a single string. To address this, the configuration supports a structured approach where you can break down a query into individual fields and their respective SQL components. This approach improves modularity, readability, and maintainability.

This feature that allows users to easily document and understand their data queries. This feature helps in maintaining clarity and transparency in data operations, provades metadata about queries useful for auto genearating govarnace artefacts like data dictionaries, lineage, etc.

### **Structure**

A complex query is defined as a top-level heading (e.g., `# My Complex Query`) in the configuration. Each field included in the query is represented as a Level 2 heading (e.g., `## Field Name`).

For each field:

- Metadata can describe the field (e.g., `name`, `description`) if a yaml metadata is not provided the field key is used as field in this example `Field Name`, but the all ideia behind this aproache is to enreach each fields with metadata like `description`, `owner`, `formula`, etc that can be useful for documenting .
- SQL components like `select`, `from`, `join`, `where`, `group_by`, `order_by`, `having`, and `cte` are specified in separate sql blocks, in sumarry instead of a large string with all your query parts you break it in blocks, to be put togeter in the rantime, with complicates things a litle bit and require a clarity on what you doing but is very usefull for documentation porpose that maybe be would be required to do anyway.

## Exemple:

Consider the following configuration example thet show a practical example of extending the `etlx` config model with **metadata keys** useful for data governance, using the NYC Yellow Taxi January 2024 Parquet file hosted at `https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2024-01.parquet`.

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

If you wanted or you're obliged to have a data dictionary for every input in a standard way, and then refrence every time this input is used in other queries, you could extend the configuration as follows (and of course before taking the tame to docuemnting all fields you should evaluate if this is really needed for your use case, and if there is no other way like from the data provider and the data source schema itself):

````md {linenos=table}
# EXTRACT_LOAD
...
```sql
-- extract_load_trip_data
CREATE OR REPLACE TABLE "DB"."<table>" AS
[[QUERY_EXTRACT_TRIP_DATA]]
```
````

Notice how the `extract_load_trip_data` has now `[[QUERY_EXTRACT_TRIP_DATA]]`, that is the a placeholder for the documented query defined below:

````md {linenos=table}
...
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

This structured approach to query documentation not only enhances the clarity of complex queries but also integrates essential metadata that supports data governance initiatives. By documenting each field and its SQL components, users can ensure that their data processes are transparent, well-understood, and compliant with organizational standards.

When you extract from an existing source, chance are there is already some form of documentation available from the data provider, so before starting to document every field in your queries, check if there is already a data dictionary or schema documentation available that you can leverage.
But when you do your own transformations and calculations is always a good practice to document them as much as possible to ensure clarity and maintainability and support data governance efforts, that said, the next example shows how to document a transformation query with calculated fields.

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

````
---
weight: 930
title: "Loading External Dependencies in ETLX"
description: "How to split large ETLX pipelines into reusable, composable external files."
icon: overview
date: 2026-01-08T01:04:15+00:00
lastmod: 2026-01-08T01:04:15+00:00
draft: false
images: []
---

## Loading External Dependencies in ETLX

In real-world scenarios, an ETLX project can quickly grow beyond what is practical to manage in a single file. As pipelines evolve, you may also want to **reuse queries, transformations, or documentation blocks across multiple projects**.

To address this, ETLX allows you to split your pipeline into multiple files and load them as **external dependencies** using the [`require`]({{% relref "../features/require" %}}) feature.

A good example of this pattern is the [Embedded SQLite]({{% relref "embedded-sqlite#extract-and-transform-data" %}}) walkthrough. While the core execution logic is relatively small, with just two inputs and one transformation, the pipeline grows rapidly once queries are documented and detailed at field level.

Instead of letting a single file become too large, you can move queries (or even other blocks) into dedicated files and load them into your main pipeline, like this:

```bash
queries/
├── trip_data.md
├── top-zones.md
pipeline-sqlite.md
```

This approach keeps pipelines **modular, readable, and reusable**, without sacrificing documentation or governance.


## Moving `QUERY_EXTRACT_TRIP_DATA` to an External File

First, we move the `# QUERY_EXTRACT_TRIP_DATA` block into a separate file named `trip_data.md` inside a `queries` folder.

This file contains:

* The query definition
* Column-level metadata
* SQL fragments
* Documentation and governance details

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
... (remaining SQL fragments) 
````
<!-- Remaining fields unchanged -->

> 💡 **Note**
> Keeping metadata next to SQL ensures that documentation, lineage, and ownership are always versioned together with the logic that produces the data.


## Moving `QUERY_TOP_ZONES` to an External File

Next, we extract the `# QUERY_TOP_ZONES` block into another file called `top-zones.md`, located in the same `queries` directory.

This query demonstrates:

* Joins across datasets
* Aggregations
* Derived metrics
* Rich field-level metadata with formulas

````md {linenos=table}
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
... (remaining SQL fragments)
````
<!-- Remaining fields unchanged -->

This file remains **fully self-contained**, readable, and executable once loaded into a pipeline.

## Using External Dependencies in the Main Pipeline

Finally, we update the main `pipeline-sqlite.md` file to load these external query definitions using the `require` mechanism.

...

# LOAD_DEPENDENCIES

```yaml metadata
name: LOAD_DEPENDENCIES
description: "Load external query dependencies for the ETLX pipeline."
run_as: REQUIRE
```

## LOAD_TRIP_DATA_QUERY

```yaml metadata
name: LOAD_TRIP_DATA_QUERY
description: "Load QUERY_EXTRACT_TRIP_DATA from an external file."
path: "queries/trip_data.md"
```

## LOAD_TOP_ZONES_QUERY

```yaml metadata
name: LOAD_TOP_ZONES_QUERY
description: "Load QUERY_TOP_ZONES from an external file."
path: "queries/top-zones.md"
```

Once loaded, the blocks `# QUERY_EXTRACT_TRIP_DATA` and `# QUERY_TOP_ZONES` are available to the pipeline **exactly as if they were defined in the same file**.

---

## Why This Matters

This modular approach allows you to:

* Keep pipelines **small and readable**
* Reuse queries across projects
* Share standardized transformations
* Version logic and documentation together
* Scale ETLX projects without losing clarity

In ETLX, external dependencies are not just includes — they are **first-class, documented, executable building blocks**.

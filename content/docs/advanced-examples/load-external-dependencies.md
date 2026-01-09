---
weight: 920
title: "Loading External Dependencies in ETLX"
description: "A guide on how to load and manage external dependencies in ETLX projects."
icon: overview
date: 2026-01-08T01:04:15+00:00
lastmod: 2026-01-08T01:04:15+00:00
draft: false
images: []
---

## Loading External Dependencies in ETLX

In a real word problem, your ETLX project may become to large to be handled in a single file. You may also want to reuse code across multiple projects. In such cases, you can  break your ETLX code into multiple files and load them as external dependencies (see [Require]({{% relref "../features/require" %}}))).
[Embedded SQLite]({{% relref "embedded-sqlite#extract-and-transform-data" %}}) is a good example of this of getting large. The core flow isnt very large itsel, byut as we documented the queries and keep tem in the same file, it grows quickly, so what if we can put those queries in a separate file and load them as an external dependency, like this:

```bash
--- queries/
 |-- trip_data.md
 |-- top-zones.md
-- pipeline-sqlite.md
```

### Moving `QUERY_EXTRACT_TRIP_DATA` to External Files

First we move the # QUERY_EXTRACT_TRIP_DATA query to a separate file called `trip_data.md` in a `queries` folder:

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

## tpep_dropoff_datetime

```yaml metadata
name: tpep_dropoff_datetime
description: "Timestamp when the meter was disengaged (trip end)."
type: timestamp
owner: taxi-analytics-team
```

```sql
-- select
    , tpep_dropoff_datetime
```

## passenger_count

```yaml metadata
name: passenger_count
description: "Number of passengers, typically entered by the driver."
type: integer
owner: ops
```

```sql
-- select
    , passenger_count
```

## trip_distance

```yaml metadata
name: trip_distance
description: "Elapsed trip distance in miles, reported by the meter."
type: double
owner: taxi-analytics-team
```

```sql
-- select
    , trip_distance
```

## RatecodeID

```yaml metadata
name: RatecodeID
description: "Final rate code at trip end.
1=Standard, 2=JFK, 3=Newark, 4=Nassau/Westchester,
5=Negotiated fare, 6=Group ride, 99=Unknown."
type: integer
owner: finance
```

```sql
-- select
    , RatecodeID
```

## store_and_fwd_flag

```yaml metadata
name: store_and_fwd_flag
description: "'Y' if the trip record was held in the vehicle before transmission (no server connection). 'N' otherwise."
type: string
owner: platform
```

```sql
-- select
    , store_and_fwd_flag
```

## PULocationID

```yaml metadata
name: PULocationID
description: "TLC Taxi Zone ID for pickup location."
type: integer
owner: geo
```

```sql
-- select
    , PULocationID
```

## DOLocationID

```yaml metadata
name: DOLocationID
description: "TLC Taxi Zone ID for dropoff location."
type: integer
owner: geo
```

```sql
-- select
    , DOLocationID
```

## payment_type

```yaml metadata
name: payment_type
description: "How the passenger paid:
0=Flex Fare, 1=Credit card, 2=Cash, 3=No charge, 4=Dispute, 5=Unknown, 6=Voided trip."
type: integer
owner: finance
```

```sql
-- select
    , payment_type
```

## fare_amount

```yaml metadata
name: fare_amount
description: "Time-and-distance fare calculated by the meter."
type: numeric
owner: finance
```

```sql
-- select
    , fare_amount
```

## extra

```yaml metadata
name: extra
description: "Miscellaneous extras and surcharges (e.g., peak surcharge)."
type: numeric
owner: finance
```

```sql
-- select
    , extra
```

## mta_tax

```yaml metadata
name: mta_tax
description: "0.50 USD MTA tax triggered by metered rate."
type: numeric
owner: finance
```

```sql
-- select
    , mta_tax
```

## tip_amount

```yaml metadata
name: tip_amount
description: "Tip in USD. Only includes credit-card tips; cash tips are not recorded."
type: numeric
owner: finance
```

```sql
-- select
    , tip_amount
```

## tolls_amount

```yaml metadata
name: tolls_amount
description: "Total tolls paid for the trip."
type: numeric
owner: finance
```

```sql
-- select
    , tolls_amount
```

## improvement_surcharge

```yaml metadata
name: improvement_surcharge
description: "Flat surcharge added at flag drop. Introduced in 2015."
type: numeric
owner: finance
```

```sql
-- select
    , improvement_surcharge
```

## total_amount

```yaml metadata
name: total_amount
description: "Total charged amount (fare + extras + taxes + tips)."
type: numeric
owner: finance
```

```sql
-- select
    , total_amount
```

## congestion_surcharge

```yaml metadata
name: congestion_surcharge
description: "NY State congestion surcharge assessed per trip."
type: numeric
owner: finance
```

```sql
-- select
    , congestion_surcharge
```

## airport_fee

```yaml metadata
name: airport_fee
description: "Fee for pickups at JFK or LaGuardia airports."
type: numeric
owner: finance
```

```sql
-- select
    , airport_fee
```

## cbd_congestion_fee

```yaml metadata
name: cbd_congestion_fee
description: "MTA Congestion Relief Zone fee (in effect after Jan 5, 2025)."
type: numeric
owner: finance
active: false
```

```sql
-- select
    , cbd_congestion_fee
```
````

### Moving `QUERY_TOP_ZONES` to External Files

Next, we move the # QUERY_TOP_ZONES query to a separate file called `top-zones.md` in the same `queries` folder:

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

### Using External Dependencies in the Main Pipeline
Finally, we update the main `pipeline-sqlite.md` file to load these external dependencies using the `require` feature:

````md {linenos=table}
...
# LOAD_DEPENDENCIES
```yaml metadata
name: LOAD_DEPENDENCIES
description: "Load external query dependencies for the ETLX pipeline."
run_as: REQUIURE
```

## LOAD_TRIP_DATA_QUERY
```yaml metadata
name: LOAD_TRIP_DATA_QUERY
description: "Load the QUERY_EXTRACT_TRIP_DATA from external file."
path: "queries/trip-data.md"
```

## LOAD_TOP_ZONES_QUERY
```yaml metadata
name: LOAD_TOP_ZONES_QUERY
description: "Load the QUERY_TOP_ZONES from external file."
path: "queries/top-zones.md"
```
````

With this setup, the main config will load the external configs and the blocks #QUERY_EXTRACT_TRIP_DATA and # QUERY_TOP_ZONES will be available for use in the ETLX pipeline as if they were defined in the same file. This modular approach helps keep your ETLX projects organized and maintainable.

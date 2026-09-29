---
weight: 810
date: "2026-08-27T10:00:00+00:00"
draft: false
title: "Parallel Section Execution"
icon: "sync"
description: "Execute the items inside an ETLX section concurrently while keeping the overall workflow sequential."
publishdate: "2026-08-27T10:00:00+00:00"
tags: ["Parallel Execution", "Concurrent ETL", "Performance", "ETL"]
categories: ["Features"]
---

## Parallel Section Execution

### Run Independent Items Concurrently

ETLX normally executes the items inside a section sequentially.

For example:

```mermaid
flowchart TD
    A[ITEM_A] --> B[ITEM_B]
    B --> C[ITEM_C]
    C --> D[ITEM_D]
```

Sometimes, however, the items in a section are independent from each other.

If they do not depend on one another, executing them sequentially can unnecessarily increase the total execution time.

ETLX supports **section-level parallel execution** using:

```yaml
parallel: true
```

When enabled, the items belonging to that section are executed concurrently.

---

### Section Parallelism

Parallel execution applies to the **items inside the section**, not to the overall ETLX workflow.

Consider:

```mermaid
flowchart TD
    subgraph EXTRACT["EXTRACT — parallel"]
        A[A]
        B[B]
        C[C]
        D[D]
        A & B & C & D --> WAIT_EXTRACT[Wait for all]
    end

    subgraph TRANSFORM["TRANSFORM"]
        E[E]
        F[F]
        E --> WAIT_TRANSFORM[Wait for all]
        F --> WAIT_TRANSFORM
    end

    subgraph LOAD["LOAD"]
        G[G]
    end

    WAIT_EXTRACT --> E
    WAIT_EXTRACT --> F
    WAIT_TRANSFORM --> G
```

ETLX executes this as:

```mermaid
flowchart TD
    A[A] --> WAIT[Wait for all]
    B[B] --> WAIT
    C[C] --> WAIT
    D[D] --> WAIT

    WAIT --> E[E]
    WAIT --> F[F]

    E --> WAIT2[Wait for all]
    F --> WAIT2

    WAIT2 --> G[G]
```

The sections themselves remain **sequential**.

ETLX will not start the next section until every item in the current parallel section has completed.

This makes parallel execution useful for creating controlled concurrency without turning the entire pipeline into an uncontrolled collection of goroutines.

---

### Enabling Parallel Execution

Add `parallel: true` to the section metadata:

````markdown
# PARALLEL

```yaml
name: PARALLEL
runs_as: ETL
description: Execute independent items concurrently.
parallel: true
active: true
```

## ...
````

Every ETL item inside this section can then execute concurrently.

For example:

```mermaid
flowchart TD
    P["PARALLEL — parallel section"]

    A[DDBPTEST]
    B[DDBPTEST2]
    C[DDBPTEST3]
    D[DDBPTEST4]
    E[DDBPTEST5]

    WAIT[Section Complete]

    P --> A
    P --> B
    P --> C
    P --> D
    P --> E

    A --> WAIT
    B --> WAIT
    C --> WAIT
    D --> WAIT
    E --> WAIT
```

Instead of:

```mermaid
flowchart TD
    A[DDBPTEST] --> B[DDBPTEST2]
    B --> C[DDBPTEST3]
    C --> D[DDBPTEST4]
    D --> E[DDBPTEST5]
```

ETLX can execute:

```mermaid
flowchart TD
    A[DDBPTEST]
    B[DDBPTEST2]
    C[DDBPTEST3]
    D[DDBPTEST4]
    E[DDBPTEST5]

    WAIT[Section Complete]

    A --> WAIT
    B --> WAIT
    C --> WAIT
    D --> WAIT
    E --> WAIT
```

---

### Complete Example

The following example contains five independent DuckDB queries writing to a DuckLake database.

````markdown
# PARALLEL
```yaml metadata
name: PARALLEL
runs_as: ETL
description: Execute independent workloads concurrently.
connection: "duckdb:"
parallel: true
active: true
```

## DDBPTEST
```yaml metadata
name: DDBPTEST
description: Test query
table: DDBPTEST
load_conn: "duckdb:"
load_before_sql: |
  CREATE SECRET (
    TYPE quack,
    TOKEN 'super_secret_test_token'
  );
  ATTACH 'ducklake:quack:localhost' AS DB (DATA_PATH 'database/lake');
load_sql: |
  CREATE OR REPLACE TABLE DB."<table>" AS
  SELECT
      v1.x % 1000 AS category,
      COUNT(*) AS total,
      APPROX_COUNT_DISTINCT(v1.y) AS total_dups,
      AVG(v1.z) AS avg
  FROM (
      SELECT
          range AS x,
          random() AS y,
          random() * 100 AS z
      FROM range(100_000_000)
  ) v1
  JOIN (
      SELECT range AS id
      FROM range(5000)
  ) v2
    ON (v1.x % 5000) = v2.id
  GROUP BY category;
load_after_sql: "DETACH DB;"
active: true
```

## DDBPTEST2
```yaml metadata
name: DDBPTEST2
description: Test query
table: DDBPTEST2
load_conn: "duckdb:"
load_before_sql: |
  CREATE SECRET (
    TYPE quack,
    TOKEN 'super_secret_test_token'
  );
  ATTACH 'ducklake:quack:localhost'
    AS DB (DATA_PATH 'database/lake');
load_sql: |
  CREATE OR REPLACE TABLE DB."<table>" AS
  SELECT
      v1.x % 1000 AS category,
      COUNT(*) AS total,
      APPROX_COUNT_DISTINCT(v1.y) AS total_dups,
      AVG(v1.z) AS avg
  FROM (
      SELECT
          range AS x,
          random() AS y,
          random() * 100 AS z
      FROM range(10_000_000)
  ) v1
  JOIN (
      SELECT range AS id
      FROM range(5000)
  ) v2
    ON (v1.x % 5000) = v2.id
  GROUP BY category;
load_after_sql: "DETACH DB;"
active: true
```
...
````

All items are independent, so ETLX can execute them concurrently.

---

### The Next Section Still Waits

Consider a workflow with:

```mermaid
flowchart TD
    START[START]

    subgraph PARALLEL["PARALLEL — parallel: true"]
        A[A]
        B[B]
        C[C]
        WAIT[Wait for A, B, C]

        A --> WAIT
        B --> WAIT
        C --> WAIT
    end

    COMPILE[D]
    REPORT[E]
    FINISH[END]

    START --> A
    START --> B
    START --> C

    WAIT --> COMPILE
    COMPILE --> REPORT
    REPORT --> FINISH
```

ETLX guarantees the section ordering:

```mermaid
flowchart TD
    START[START]

    A[A]
    B[B]
    C[C]

    WAIT[Wait for A / B / C]
    COMPILE[COMPILE]
    REPORT[REPORT]
    FINISH[END]

    START --> A
    START --> B
    START --> C

    A --> WAIT
    B --> WAIT
    C --> WAIT

    WAIT --> COMPILE
    COMPILE --> REPORT
    REPORT --> FINISH
```

`D` will not start until `A`, `B`, and `C` have finished.

Likewise, `E` will not start until `D` has completed.

This means you can safely combine sequential and parallel sections in the same ETLX document.

---

### Parallel Execution Is Not Dependency Resolution

`parallel: true` should only be used when the items inside the section are independent.

For example, this is a good candidate:

```mermaid
flowchart TD
    subgraph EXTRACT["EXTRACT — parallel"]
        CUSTOMERS[CUSTOMERS]
        PRODUCTS[PRODUCTS]
        ORDERS[ORDERS]
    end

    CUSTOMERS --> COMPLETE[Section Complete]
    PRODUCTS --> COMPLETE
    ORDERS --> COMPLETE
```

If none of the three items depends on the output of another item, they can execute concurrently.

This is **not** a good candidate:

```mermaid
flowchart TD
    EXTRACT[EXTRACT]
    TRANSFORM[TRANSFORM]
    LOAD[LOAD]

    EXTRACT --> TRANSFORM
    TRANSFORM --> LOAD
```

when:

```text
TRANSFORM → depends on EXTRACT

LOAD → depends on TRANSFORM
```

Those operations have an explicit dependency and should remain sequential.

Parallel sections therefore do not replace pipeline dependency design. They provide a way to tell ETLX that the items in a section are safe to execute concurrently.

---

### Database Considerations

Parallel execution should be used with care when multiple items write to the same database.

A common assumption is:

> "The items write to different tables, therefore they are safe to execute concurrently."

That is **not necessarily true**.

Some databases allow multiple concurrent writers, while others serialize writes or permit only a single writer at a time.

For example:

```mermaid
flowchart TD
    DB[(Database)]

    DB --> A[TABLE_A]
    DB --> B[TABLE_B]
    DB --> C[TABLE_C]

    A --> CONCURRENT[Concurrent writes]
    B --> CONCURRENT
    C --> CONCURRENT
```

Even though the tables are different, the database itself may still require a single writer.

This can result in:

* Write contention
* Lock errors
* Busy/locked database errors
* Transaction conflicts
* Reduced performance
* Failed ETL items

Therefore, `parallel: true` should be enabled only when the underlying storage system can handle the expected concurrency.

---

### When Parallel Execution Is a Good Fit

Parallel execution is particularly useful when the workload consists of independent operations such as:

* Extracting multiple independent data sources
* Processing independent files
* Generating independent files
* Uploading objects to blob storage
* Downloading independent objects
* Running independent analytical queries
* Writing to storage systems designed for concurrent writers
* Processing different partitions of a dataset

For example:

```mermaid
flowchart TD
    P["PARALLEL"]

    A[CSV A]
    B[CSV B]
    C[CSV C]

    AT[Transform A]
    BT[Transform B]
    CT[Transform C]

    AO[Output A]
    BO[Output B]
    CO[Output C]

    P --> A --> AT --> AO
    P --> B --> BT --> BO
    P --> C --> CT --> CO
```

This type of workload can benefit significantly from concurrency.

---

### File and Blob Storage

Parallel execution can be particularly effective when the outputs are independent files or objects.

For example:

```mermaid
flowchart TD
    ETLX[ETLX]

    A[File A]
    B[File B]
    C[File C]

    S3A[S3]
    S3B[S3]
    S3C[S3]

    ETLX --> A --> S3A
    ETLX --> B --> S3B
    ETLX --> C --> S3C
```

Each operation can work independently without requiring all writers to coordinate through a single database writer.

This makes parallel sections a good candidate for workloads involving:

* Amazon S3
* Azure Blob Storage
* Google Cloud Storage
* MinIO
* Data lakes
* Lakehouse storage
* Local file systems

The actual concurrency limits still depend on the storage system, network bandwidth, CPU, memory, and the resources available to the ETLX process.

---

### Resources Still Matter

Parallel execution does not create additional hardware resources.

If five workloads are executed concurrently, they still compete for:

* CPU
* Memory
* Disk I/O
* Network bandwidth
* Database connections
* Storage throughput

For example:

```mermaid
flowchart TD
    subgraph SEQUENTIAL["Sequential"]
        A1[A] --> B1[B] --> C1[C] --> D1[D]
    end

    subgraph PARALLEL["Parallel"]
        A2[A]
        B2[B]
        C2[C]
        D2[D]
    end
```

If the machine has enough available resources, the parallel version can be considerably faster.

If the machine is already resource constrained, running everything concurrently can make the entire workflow slower.

---

### Use With Caution

`parallel: true` is therefore an explicit performance optimization, not a guarantee that the workload will be faster.

Before enabling it, consider:

1. Are the items independent?
2. Can the target database handle concurrent writers?
3. Can the storage system handle concurrent operations?
4. Is there enough CPU?
5. Is there enough memory?
6. Is there enough disk I/O?
7. Is there enough network bandwidth?
8. Can the required number of database connections be opened safely?

If the answer to these questions is yes, parallel execution can significantly reduce total pipeline execution time.

---

### Sequential by Default

ETLX keeps the default behavior sequential.

Without:

```yaml
parallel: true
```

the section behaves normally:

```mermaid
flowchart TD
    A[A] --> B[B]
    B --> C[C]
    C --> D[D]
```

With:

```yaml
parallel: true
```

the items become concurrent:

```mermaid
flowchart TD
    A[A]
    B[B]
    C[C]
    D[D]

    WAIT[Wait for all]

    A --> WAIT
    B --> WAIT
    C --> WAIT
    D --> WAIT
```

The next ETLX section still waits for all four items to complete.

This makes parallelism opt-in and allows existing ETLX documents to continue behaving exactly as before.

---

### Parallel Sections and Remote Execution

Parallel section execution and Remote Distributed Execution solve related but different problems.

**Parallel sections** execute concurrently within the same ETLX process:

```mermaid
flowchart TD
    ETLX["ETLX Host"]

    A[A]
    B[B]
    C[C]

    ETLX --> A
    ETLX --> B
    ETLX --> C
```

**Remote Distributed Execution** distributes execution across different machines:

```mermaid
flowchart TD
    HOST[Host]

    A[Server A]
    B[Server B]
    C[Server C]

    A1[A]
    B1[B]
    C1[C]

    HOST --> A
    HOST --> B
    HOST --> C

    A --> A1
    B --> B1
    C --> C1
```

They can also be combined.

For example, a remote worker can execute a section where several independent items are themselves configured with:

```yaml
parallel: true
```

This provides two levels of concurrency:

```mermaid
flowchart TD
    HOST[Host]

    WA[Worker A]
    WB[Worker B]

    A1[A1]
    A2[A2]
    A3[A3]

    B1[B1]
    B2[B2]
    B3[B3]

    HOST --> WA
    HOST --> WB

    WA --> A1
    WA --> A2
    WA --> A3

    WB --> B1
    WB --> B2
    WB --> B3
```

This should be used carefully because the total resource consumption can increase rapidly.

---

### The Goal

Parallel Section Execution gives ETLX a simple way to exploit concurrency without changing the overall structure of a pipeline.

The workflow remains sequential at the section level:

```mermaid
flowchart TD
    A[Section A] --> B[Section B]
    B --> C[Section C]
```

while individual sections can explicitly opt into concurrency:

```mermaid
flowchart TD
    A[Section A] --> B

    subgraph B["Section B — parallel"]
        B1[B1]
        B2[B2]
        B3[B3]

        WAIT[Wait for all]

        B1 --> WAIT
        B2 --> WAIT
        B3 --> WAIT
    end

    B --> C[Section C]
```

The principle is simple:

> **Keep the workflow sequential where dependencies exist, and use parallel sections where independent work can safely execute concurrently.**

When the storage system supports concurrent operations and the machine has sufficient resources, parallel execution can provide substantial performance improvements.

## When the storage system has a single-writer architecture or the workload is resource constrained, parallel execution should be used with caution.

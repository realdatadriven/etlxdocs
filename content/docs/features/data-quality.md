---
weight: 740
title: "Data Quality"
description: "How ETLX models, executes, and observes data quality rules as first-class, metadata-driven validation processes."
icon: check
tags: ["features", "governance"]
date: 2025-12-16T01:04:15+00:00
lastmod: 2025-12-16T01:04:15+00:00
draft: false
images: []
---

## Data Quality in ETLX

Data quality in ETLX is **not an afterthought** or an external process. It is a **first-class execution model**, designed to validate, observe, and optionally correct data using declarative, SQL-driven rules.

Rather than embedding ad-hoc checks inside transformations, ETLX treats data quality as:

* **Executable metadata**
* **Auditable validation logic**
* **A governance primitive**

This allows quality rules to be reasoned about, documented, versioned, and executed independently of ETL or ELT pipelines.

### The DATA_QUALITY Execution Model

A data quality workflow is defined using a **root Markdown block** that declares its execution mode as `DATA_QUALITY`.

From this block, ETLX executes a deterministic sequence of **validation rules**, each modeled as an independent execution unit.

Conceptually, a DATA_QUALITY block answers three questions:

1. *What condition defines invalid data?*
2. *How should violations be observed and reported?*
3. *Can (and should) invalid data be fixed automatically?*

### Structure Overview

A DATA_QUALITY definition consists of:

1. **Root block metadata**

   * Describes the purpose and activation state of the quality process

2. **Validation rules**

   * Each rule is a level-two heading representing a single, focused quality check

3. **SQL logic**

   * One query to detect violations
   * An optional query to correct them

### Data Quality Markdown Example

The example below illustrates a DATA_QUALITY workflow validating the `TRIP_DATA` dataset introduced in the *Query Documentation* section (NYC Yellow Taxi data).

````md {linenos=table}
# QUALITY_CHECK
```yaml
description: "Runs some queries to check quality / validate."
runs_as: DATA_QUALITY
active: true
```

## Rule0001
```yaml
name: Rule0001
description: "Check if the payment_type has only the values 0=Flex Fare, 1=Credit card, 2=Cash, 3=No charge, 4=Dispute, 5=Unknown, 6=Voided trip."
connection: "duckdb:"
before_sql: "ATTACH 'database/DB_EX_DGOV.db' AS DB (TYPE SQLITE)"
query: quality_check_query
fix_quality_err: fix_quality_err_query
column: total_reg_with_err # Defaults to 'total'.
check_only: false # runs only quality check if true
fix_only: false # runs only quality fix if true and available and possible
after_sql: "DETACH DB"
active: true
```

```sql
-- quality_check_query
SELECT COUNT(*) AS "total_reg_with_err"
FROM "TRIP_DATA"
WHERE "payment_type" NOT IN (0,1,2,3,4,5,6);
```

```sql
-- fix_quality_err_query
UPDATE "TRIP_DATA"
SET "payment_type" = 'default value'
WHERE "payment_type" NOT IN (0,1,2,3,4,5,6);
```

## Rule0002
```yaml
name: Rule0002
description: "Check if there is any trip with distance less than or equal to zero."
connection: "duckdb:"
before_sql: "ATTACH 'database/DB_EX_DGOV.db' AS DB (TYPE SQLITE)"
query: quality_check_query
fix_quality_err: null # no automated fixing for this
column: total_reg_with_err # Defaults to 'total'.
after_sql: "DETACH DB"
active: true
```

```sql
-- quality_check_query
SELECT COUNT(*) AS "total_reg_with_err"
FROM "TRIP_DATA"
WHERE NOT "trip_distance" > 0;
```
````

### Rule Semantics

Each validation rule represents a **contract** that the data must satisfy.

A rule is defined by:

* **Detection logic** (`query`)
* **Optional remediation logic** (`fix_quality_err`)
* **Execution controls** (`check_only`, `fix_only`)
* **Lifecycle hooks** (`before_sql`, `after_sql`)

Rules are intentionally **small and composable**, encouraging one responsibility per rule.

### Execution Flow

For each active rule, ETLX executes the following steps:

1. **Initialization**

   * Resolve connection
   * Execute `before_sql`

2. **Validation**

   * Execute the detection query
   * Read the violation count from the configured column

3. **Decision**

   * If violations = 0 → rule passes
   * If violations > 0 → rule fails

4. **Optional Fix**

   * If `fix_quality_err` is defined and allowed
   * Execute remediation SQL

5. **Finalization**

   * Execute `after_sql`
   * Record execution metadata

This flow is deterministic and fully observable.

### Query & Connection Resolution

* Validation queries run using the rule’s `connection`
* If omitted, the DATA_QUALITY root connection is used
* Queries can reference:

  * Inline SQL
  * Named SQL blocks
  * Shared datasets such as `TRIP_DATA`

This allows quality rules to be applied consistently across:

* Extracted data
* Transformed outputs
* Final reporting tables

### Observability & Governance

Every DATA_QUALITY execution produces structured metadata, including:

* Rule name and description
* Execution timestamps and duration
* Number of violations detected
* Fix actions applied
* Errors and warnings

This metadata can be used to:

* Generate data quality reports
* Track SLA compliance
* Power dashboards
* Feed governance and audit systems

### Design Principles

ETLX data quality rules are:

* **Declarative** — define intent, not control flow
* **SQL-native** — no DSLs, no abstractions hiding logic
* **Deterministic** — same input, same result
* **Auditable** — every action is recorded
* **Composable** — reusable across pipelines

### What This Is (and Is Not)

**This is:**

* A data quality execution model
* A governance-friendly validation layer
* An observable contract enforcement mechanism

**This is not:**

* A black-box data quality tool
* A replacement for BI validation
* A rule engine detached from SQL

ETLX complements catalogs, observability platforms, and BI tools by enforcing quality **at the source of execution**.

### Summary

In ETLX:

* Data quality is **code, not comments**
* Rules are **first-class execution units**
* SQL defines truth
* Metadata defines meaning

> **Data quality is not something you check once — it is something you continuously execute, observe, and govern.**

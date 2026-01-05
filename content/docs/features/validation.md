---
weight: 720
title: "ETL | ELT - Validation Rules"
description: "How ETLX enables declarative data quality validation as a first-class concern of ETL / ELT execution blocks."
icon: check
tags: ["features"]
date: 2025-12-16T01:04:15+00:00
lastmod: 2025-12-16T01:04:15+00:00
draft: false
images: []
---

## Validation Rules

ETLX allows **data quality validation to be declared as metadata**, making validation a **first-class concern** of ETL / ELT execution.

Validations are executed **as part of the pipeline lifecycle**, not as an external check or post-process.

They allow you to:

* Fail fast on invalid data
* Guard against silent data corruption
* Encode business and technical expectations declaratively
* Keep validation logic close to the data movement it protects

### Declaring Validations

Validations are defined using the metadata key:

```
<step>_validation
```

Where `<step>` can be any supported execution step, such as:

* `extract`
* `transform`
* `load`

### Example
In the previous example from [etl/elt]({{% relref "etl-elt#example" %}}), we define two validation rules for the `load` step:

````md {linenos=table}
...
```yaml
...
load_validation:
  - type: throw_if_empty
    sql: validate_data_not_empty
    msg: "The given date is not avaliable in the origin!"
    active: true
  - type: throw_if_not_empty
    sql: validate_date_exists_in_destination
    msg: "This date is already imported, check to avoid duplicates!"
    active: true
...
```
```sql validate_data_not_empty
SELECT *
FROM PG.INPUT_1
WHERE event_date = '{YYYY-MM-DD}'
LIMIT 10
```
```sql validate_date_exists_in_destination
SELECT *
FROM DL.INPUT_1 
WHERE event_date = '{YYYY-MM-DD}'
LIMIT 10
```
````
Each validation rule is evaluated **after the step SQL executes** and **before the next lifecycle hook**.
>here we used `{YYYY-MM-DD}` as a placeholder for date, which ETLX will replace at runtime, if cli used with `--date` argument, that date would be the one placede in `{YYYY-MM-DD}`.

## Validation Execution Model

For each object defined in `<step>_validation`:

1. The associated `sql` is executed
2. The query result is evaluated according to the `type`
3. If the condition is met, ETLX:

   * Throws a controlled execution error
   * Emits the configured message (`msg`)
   * Marks the step as failed in execution metadata

### Validation Connection Resolution

Validation SQL is executed using:

* `<step>_conn`, if defined
* Otherwise, the pipeline’s default connection

This ensures validations run **in the same execution context** as the data they validate.

## Supported Validation Types

| Type                 | Behavior                                    |
| -------------------- | ------------------------------------------- |
| `throw_if_empty`     | Fails if the query returns zero rows        |
| `throw_if_not_empty` | Fails if the query returns one or more rows |

> Validation types are intentionally simple and composable.
> Complex rules should be expressed in SQL, not in configuration logic.
> This keeps validation **declarative and database-native**.

## SQL Validation Design

Validation SQL should express **intent**, not mechanics.

Typical validation queries include:

* Row existence checks
* Duplicate detection

>This is mostly usefull to avolid data duplication or missing data during incremental loads, or if needed to run the extraction more than once because of some input failure, without compromising inputs that were already extracted.

## Validation Observability

Each validation execution is recorded as part of ETLX’s observability layer:

* Validation name
* Step and pipeline association
* SQL executed
* Execution time
* Result (pass / fail)
* Error message (if thrown)

This enables:

* Auditing
* SLA enforcement
* Debugging
* Reporting

## Design Rationale

ETLX treats validation as:

* **Executable documentation**
* **A contract between pipeline stages**
* **A guardrail, not a sidecar**

By encoding validation rules in metadata:

* Pipelines become safer by default
* Failures are explicit and explainable
* Validation logic evolves with the pipeline

## Key Takeaways

* Validation is **declarative, explicit, and observable**
* SQL remains the single source of truth
* Pipelines fail loudly, not silently
* Unsupported sources do not block adoption
* ETLX prioritizes **correctness over convenience**
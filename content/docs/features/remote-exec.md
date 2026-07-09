---
weight: 815
date: "2026-07-08T10:00:00+00:00"
draft: false
title: "Remote Distributed Execution"
icon: "cloud_sync"
description: "Distribute ETLX pipelines across remote machines over SSH for scalable, concurrent processing."
publishdate: "2026-07-08T10:00:00+00:00"
tags: ["Remote Execution", "SSH", "Distributed ETL", "Cloud"]
categories: ["Features"]
---

## Remote Distributed Execution

### Scale Your Pipelines Beyond a Single Machine

As datasets grow, a single laptop or server eventually becomes the bottleneck.

Many ETL workflows contain a few computationally expensive steps while the remainder of the pipeline performs relatively lightweight operations.

Instead of moving the entire workflow to a larger server, ETLX allows you to execute only the expensive parts remotely.

A pipeline can transparently distribute work across multiple machines using secure SSH connections, wait for all remote jobs to complete, download the generated artifacts, and continue executing locally.

---

### How It Works

A **REMOTE_EXEC** section defines one or more remote workers.

Each worker can:

* Connect over SSH
* Upload the current pipeline
* Upload required files
* Execute selected ETLX sections
* Run arbitrary shell commands
* Download the generated output
* Return execution status and logs

The host pipeline waits until every configured worker finishes successfully before continuing.

```txt {linenos=table}
                Host Machine
                      │
                      │
         Upload Pipeline + Files
          ┌───────────┴───────────┐
          │                       │
          ▼                       ▼
    Remote Worker A         Remote Worker B
      EXTRACTX                 EXTRACTY
         │                        │
         ▼                        ▼
        TRFX                     TRFY
          │                       │
          └───────────┬───────────┘
                      │
            Download Results
                      │
                      ▼
           Continue Remaining ETLX
             Pipeline Locally
```

The remote execution layer becomes part of the pipeline itself, allowing distributed processing without introducing external orchestration tools.

---

### Example

Consider a pipeline containing two expensive transformations:

* **EXTRACTX → TRFX**
* **EXTRACTY → TRFY**

Instead of executing both on your laptop, ETLX sends each branch to a different machine.

```text {linenos=table}
Laptop
├── Upload pipeline
├── Execute EXTRACTX + TRFX on Server A
├── Execute EXTRACTY + TRFY on Server B
├── Wait for completion
├── Download results
└── Continue remaining ETL locally
```

Because both branches execute simultaneously, the total execution time can be dramatically reduced.

---

### Remote Worker Definition

Each worker specifies:

* SSH connection information
* Working directory
* Files to upload
* Files to download
* ETLX sections to execute
* Optional shell commands

Example:

````markdown {linenos=table}
<!-- markdownlint-disable MD022 -->
<!-- markdownlint-disable MD031 -->

# REMOTE_EXEC
```yaml
name: RemoteExec
runs_as: REMOTE_EXEC
timeout: 10_000
```

### 127.0.0.1
```yaml
name: local.test
host: 127.0.0.1
port: 22
user: ubuntu
key: $HOME/.ssh/id...
host_key: ~/.ssh/known_hosts
working_dir: $HOME/etlx
run:
  - EXTRACTX
  - TRFX
upload_files: 
  - {source: .env, dest: .env }
download_files: 
  - {source: temp.db , dest: ./database/temp.db  }
commands:
  - curl https://realdatadriven.github.io/etlxdocs/install.sh | sh 
  - etlx --config pipeline.md --only EXTRACTX,TRFX
```

### 127.0.0.2
```yaml
name: local.test2
host: 127.0.0.2
port: 22
user: ubuntu
key: $HOME/.ssh/id...
host_key: ~/.ssh/known_hosts
working_dir: $HOME/etlx
run:
  - EXTRACTY
  - TRFY
upload_files: 
  - {source: .env, dest: .env }
download_files: 
  - {source: temp.db , dest: ./database/temp.db  }
commands:
  - curl https://realdatadriven.github.io/etlxdocs/install.sh | sh 
  - etlx --config pipeline.md --only EXTRACTY,TRFY
```

```env .env
CONN=temp.db
```

# RUNETL
```yaml
name: ETL
runs_as: ETL
```

### EXTRACTX
```yaml
name: EXTRACTX
extract_conn: "duckdb:"
extract_before_sql: "INSTALL SQLITE;ATTACH 'ETL.db' AS DB (TYPE SQLITE);"
extract_sql: 'CREATE OR REPLACE TABLE DB."<table>" AS SELECT version() AS "VERSION";'
extract_after_sql: "DETACH DB;"
```

### EXTRACTY
```yaml
name: EXTRACTY
extract_conn: "duckdb:"
extract_before_sql: "INSTALL SQLITE;ATTACH 'ETL.db' AS DB (TYPE SQLITE);"
extract_sql: 'CREATE OR REPLACE TABLE DB."<table>" AS SELECT version() AS "VERSION";'
extract_after_sql: "DETACH DB;"
```

### TRFX
```yaml
name: TRFX
extract_conn: "duckdb:"
extract_before_sql: "INSTALL SQLITE;ATTACH 'ETL.db' AS DB (TYPE SQLITE);"
extract_sql: "CREATE OR REPLACE TABLE DB.<table> AS SELECT version() || '<table>' AS VERSION FROM EXTRACTX;"
extract_after_sql: "DETACH DB;"
```

### TRFY
```yaml
name: TRFY
extract_conn: "duckdb:"
extract_before_sql: "INSTALL SQLITE;ATTACH 'ETL.db' AS DB (TYPE SQLITE);"
extract_sql: "CREATE OR REPLACE TABLE DB.<table> AS SELECT version() || '<table>' AS VERSION FROM EXTRACTY;"
extract_after_sql: "DETACH DB;"
```
````

Each remote worker is completely independent and can execute a different subset of the pipeline.

---

### Parallel Execution

Because workers execute concurrently, ETLX automatically performs synchronization.

```txt {linenos=table}
             Host
              │
      ┌───────┴────────┐
      │                │
      ▼                ▼
  Worker A         Worker B
      │                │
      ▼                ▼
 Processing       Processing
      │                │
      └───────┬────────┘
              │
     Wait Until Finished
              │
              ▼
 Continue Pipeline
```

If one worker fails, ETLX reports the error and prevents downstream sections from executing with incomplete data.

---

### Why This Is Useful

Distributed execution is especially valuable for workloads involving:

* Large joins
* Aggregations
* Machine learning preparation
* Massive file transformations
* DuckDB analytical queries
* Data warehouse exports
* Lakehouse processing

Instead of buying a permanently larger workstation, you can temporarily use more powerful machines only when needed.

---

### Cloud-Native Workflows

One particularly useful scenario is working with cloud storage.

Suppose your data already resides in:

* Amazon S3
* Azure Blob Storage
* Google Cloud Storage
* MinIO
* Data Lake
* Lakehouse

Rather than downloading terabytes of data to your laptop, ETLX can execute the heavy transformations on a cloud VM located close to the storage.

Only the processed results need to be transferred back.

```text {linenos=table}
Laptop
    │
    ▼
Cloud VM
    │
    ├── Read Lakehouse
    ├── Heavy SQL
    ├── Transform
    └── Produce Results
    │
    ▼
Download Processed Output
```

This minimizes both execution time and network traffic.

---

### Hybrid Processing

Remote execution does not replace local execution.

Instead, it enables hybrid pipelines where each step runs where it makes the most sense.

For example:

```txt {linenos=table}
Local
│
├── Validation
├── Parameter Generation
│
├── Remote Worker A
│     Heavy Aggregation
│
├── Remote Worker B
│     Data Cleansing
│
├── Download Results
│
├── Merge Outputs
├── Reporting
└── Notifications
```

The host machine remains responsible for orchestrating the entire workflow.

---

### Security

All communication uses standard SSH authentication.

ETLX supports:

* SSH private keys
* Host key verification
* Secure file upload
* Secure file download
* Configurable working directories

No additional agents or services need to be installed on the remote machines.

---

### Future: Elastic Cloud Workers

Remote execution is designed to integrate naturally with Infrastructure as Code.

A future ETLX release will include Terraform integration capable of automatically:

1. Provision cloud virtual machines
2. Install ETLX
3. Upload the pipeline
4. Execute distributed workloads
5. Download the results
6. Destroy the infrastructure

This enables truly elastic ETL pipelines where compute resources exist only for the duration of the job, significantly reducing cloud costs while allowing virtually unlimited processing capacity.

---

### The Goal

Remote Distributed Execution allows ETLX to scale from a single laptop to a fleet of cloud servers without changing how pipelines are written.

The workflow remains a single, human-readable ETLX document—the only difference is **where** each section executes.

As data grows, ETLX lets you move computation closer to the data, execute work in parallel, and bring back only the results.

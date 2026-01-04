---
weight: 300
date: "2025-06-08T19:25:22+01:00"
draft: false
author: "João Tavares"
title: "Quickstart"
icon: "rocket_launch"
description: "Get ETLX running in minutes and execute your first declarative, metadata-driven data pipeline."
publishdate: "2025-06-08T19:25:22+01:00"
tags: ["Beginners", "Quickstart", "ETL"]
categories: ["Getting Started"]

twitter:
  card: "summary"
  title: "ETLX Quickstart"
  description: "Run your first ETLX pipeline in minutes."
---


## ✅ Requirements

Depending on how you install ETLX:

### Minimum

* **Linux, macOS, or Windows**
* **DuckDB-compatible environment**

### Optional (for building from source)

* **Go ≥ 1.21**
* **git**

---

## 📦 Installation

Choose **one** of the following options.

---

### Option 1: Precompiled Binary (Recommended)

Download the latest release for your OS from:

👉 [https://github.com/realdatadriven/etlx/releases](https://github.com/realdatadriven/etlx/releases)

Make it executable and verify:

```bash
chmod +x etlx
./etlx --help
```

---

### Option 2: Install via Go

If you want ETLX as a Go dependency or to build it yourself:

```bash
go get github.com/realdatadriven/etlx
```

---

### Option 3: Clone the Repository

```bash
git clone https://github.com/realdatadriven/etlx.git
cd etlx
```

Run directly:

```bash
go run cmd/main.go --config pipeline.md
```

> ⚠️ **Windows note**
> If you encounter DuckDB build issues, use the official DuckDB library and build with:
>
> ```bash
> CGO_ENABLED=1 CGO_LDFLAGS="-L/path/to/libs" \
> go run -tags=duckdb_use_lib cmd/main.go --config pipeline.md
> ```

---

## 🧱 Your First Pipeline

ETLX pipelines are defined using **structured Markdown**.

Create a file named `pipeline.md`:

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

---

## ▶️ Run the Pipeline

```bash
etlx --config pipeline.md
````

That’s it.

ETLX will:

* Parse the configuration
* Resolve dependencies
* Execute steps deterministically
* Capture execution metadata automatically

---

## ⚙️ Common CLI Flags

| Flag       | Description                                         |
| ---------- | --------------------------------------------------- |
| `--config` | Path to pipeline file (default: `config.md`)        |
| `--date`   | Reference date (`YYYY-MM-DD`)                       |
| `--only`   | Run only specific keys                              |
| `--skip`   | Skip specific keys                                  |
| `--steps`  | Run specific steps (`extract`, `transform`, `load`) |
| `--clean`  | Run `clean_sql` blocks                              |
| `--drop`   | Run `drop_sql` blocks                               |
| `--rows`   | Show row counts                                     |

Example:

```bash
etlx --config pipeline.md --only sales --steps extract,load
```

---

## 🔐 Environment Variables

ETLX supports environment-based configuration.

Example `.env` file:

```env
DL_DSN_URL=mysql:db=ducklake_catalog host=localhost
OLTP_DSN_URL=postgres:dbname=erpdb host=localhost user=postgres
```

These variables are automatically loaded at runtime.

---

## 🐳 Running ETLX with Docker

You can run ETLX **without installing anything locally**.

### Build the Image

```bash
docker build -t etlx:latest .
```

Or pull (when available):

```bash
docker pull docker.io/realdatadriven/etlx:latest
```

---

### Run a Pipeline

```bash
docker run --rm \
  -v $(pwd)/pipeline.md:/app/pipeline.md:ro \
  etlx:latest --config /app/pipeline.md
```

---

### Using `.env` and Database Directory

```bash
docker run --rm \
  -v $(pwd)/.env:/app/.env:ro \
  -v $(pwd)/pipeline.md:/app/pipeline.md:ro \
  -v $(pwd)/database:/app/database \
  etlx:latest --config /app/pipeline.md
```

---

### Interactive Mode

```bash
docker run -it --rm etlx:latest repl
```

---

### 💡 Optional: Docker Alias

Make Docker feel like a native binary:

```bash
alias etlx="docker run --rm -v $(pwd):/app etlx:latest"
```

Now:

```bash
etlx --help
etlx --config pipeline.md
```

---

## 🧠 What’s Next?

* 📘 **Core Concepts** – Pipelines, steps, metadata
* 🔍 **Execution & Observability** – What ETLX records automatically
* 🧾 **Self-Documenting Pipelines**
* 🧬 **Metadata → Lineage → Governance**
* 🧩 **Advanced Use Cases & Examples**

👉 Continue with **Core Concepts** to understand how ETLX works under the hood.


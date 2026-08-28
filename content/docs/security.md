---
weight: 300
date: "2025-06-08T19:25:22+01:00"
draft: false
title: "Security Configuration"
icon: "lock"
description: "Controlling filesystem access and security settings for DuckDB connections in Central Set and ETLX."
publishdate: "2025-06-08T19:25:22+01:00"
tags: ["Security", "Safety"]
categories: ["Security"]
---

{{% alert context="warning" text="**Security Notice** — DuckDB is an embedded database that can access local files, remote files, and external resources. In production environments, access should be restricted to only the directories, paths, and configuration options required by your application." /%}}

Central Set and ETLX provide a set of environment variables that allow administrators to control the security configuration of every DuckDB connection.

When configured, these settings are automatically applied whenever a new DuckDB connection is created.

This provides a centralized mechanism for controlling:

* filesystem access
* external resource access
* allowed configuration changes
* runtime security restrictions

The implementation is based on DuckDB's built-in security features.

---

## 🛡️ Security Model

By default, DuckDB can:

* read local files
* write local files
* load extensions
* access remote resources
* modify runtime configuration

In production environments it is recommended to restrict these capabilities.

Central Set applies security settings automatically during connection initialization.

Example:

```text
New DuckDB Connection
        │
        ▼
Apply Security Configuration
        │
        ▼
Connection Ready
```

This ensures all users, APIs, ETLX pipelines, dashboards, and jobs follow the same security policy.

---

## 📂 Allowed Directories

Environment Variable:

```bash
ETLX_DUCKDB_ALLOWED_DIRECTORIES="SET allowed_directories = ['/tmp', './database', './static'];"
```

This setting restricts filesystem operations to specific directories.

Example:

```sql
SET allowed_directories = [
  '/tmp',
  './database',
  './static'
];
```

Only files located inside these directories can be accessed.

Example:

✅ Allowed

```text
./database/data.duckdb
./static/report.csv
/tmp/export.parquet
```

❌ Blocked

```text
/etc/passwd
/home/user/private.csv
C:\Users\Admin\Secrets
```

Recommended minimum:

```bash
ETLX_DUCKDB_ALLOWED_DIRECTORIES="SET allowed_directories = ['./database'];"
```

---

## 📄 Allowed Paths

In some situations access must be granted to specific files rather than entire directories.

DuckDB supports path-level restrictions using:

```sql
SET allowed_paths = [...]
```

This allows access to individual files while keeping all other filesystem locations blocked.

Example:

```sql
SET allowed_paths = [
  './config/reference.csv',
  './data/countries.parquet'
];
```

Use this approach when only a small number of files need to be exposed.

---

## 🌐 External Access

Environment Variable:

```bash
ETLX_DUCKDB_EXTERNAL_ACCESS="SET enable_external_access = true;"
```

Controls whether DuckDB can access external resources.

Example:

```sql
SET enable_external_access = true;
```

When enabled, DuckDB may access:

* HTTP resources
* HTTPS resources
* S3 buckets
* Azure Blob Storage
* other supported external storage providers

Example:

```sql
SELECT *
FROM read_parquet(
  'https://example.com/data.parquet'
);
```

To disable all external access:

```bash
ETLX_DUCKDB_EXTERNAL_ACCESS="SET enable_external_access = false;"
```

Production environments should only enable this feature when required.

---

## ⚙️ Allowed Configuration Changes

Environment Variable:

```bash
ETLX_DUCKDB_ALLOWED_CONFIGS="SET allowed_configs = ['memory_limit', 'threads'];"
```

Restricts which configuration parameters users can modify.

Example:

```sql
SET allowed_configs = [
  'memory_limit',
  'threads'
];
```

Users may then execute:

```sql
SET memory_limit = '4GB';
SET threads = 8;
```

But attempts to modify non-approved settings will be rejected.

Example:

```sql
SET enable_external_access = true;
```

Result:

```text
Permission denied
```

This prevents users from bypassing administrator-defined security settings.

---

## 🔒 Lock Configuration

Environment Variable:

```bash
ETLX_DUCKDB_LOCK_CONFIGURATION="SET lock_configuration = true;"
```

Locks the DuckDB configuration after initialization.

Example:

```sql
SET lock_configuration = true;
```

Once enabled:

* configuration values cannot be modified
* runtime overrides are blocked
* security settings remain enforced

Recommended for production deployments.

---

## 🔐 Centralized Security Configuration

Environment Variable:

```bash
ETLX_DUCKDB_SECURITY_CONFIGS="SET lock_configuration = true;"
```

This variable can be used to provide additional security-related SQL statements that should be executed whenever a connection is created.

Example:

```bash
ETLX_DUCKDB_SECURITY_CONFIGS="
SET enable_external_access = false;
SET lock_configuration = true;
"
```

This allows administrators to define a standard security baseline for all connections.

---

## 🚀 Example Production Configuration

A typical production deployment might use:

```bash
ETLX_DUCKDB_ALLOWED_DIRECTORIES="SET allowed_directories = ['./database','./static'];"
ETLX_DUCKDB_EXTERNAL_ACCESS="SET enable_external_access = false;"
ETLX_DUCKDB_ALLOWED_CONFIGS="SET allowed_configs = ['memory_limit','threads'];"
ETLX_DUCKDB_LOCK_CONFIGURATION="SET lock_configuration = true;"
```

This configuration:

* restricts filesystem access
* blocks internet access
* limits configuration changes
* locks the runtime configuration

---

## 🧠 Recommendations

For most production systems:

✅ Restrict filesystem access using `allowed_directories`

✅ Disable external access unless required

✅ Limit configuration changes with `allowed_configs`

✅ Enable `lock_configuration`

✅ Store all security settings in environment variables

Avoid:

❌ Allowing unrestricted filesystem access

❌ Embedding secrets in SQL scripts

❌ Allowing users to modify security-related settings

❌ Enabling external access when it is not required

---

## 📚 Related Reading

DuckDB Security Documentation:

https://duckdb.org/docs/current/operations_manual/securing_duckdb/overview

The ETLX and Central Set security variables provide a simple mechanism for applying these protections consistently across all DuckDB connections.

---
weight: 760
title: "Exports"
description: "Handling data exports to files and templates using ETLX"
icon: download
tags: ["features", "management", "sql", "scripts"]
date: 2025-12-16T01:04:15+00:00
lastmod: 2025-12-16T01:04:15+00:00
draft: false
images: []
---

## Exports

The `EXPORTS` block defines how ETLX **materializes data into external artifacts** such as files, reports, and documents. Exports are typically the **final stage** of a pipeline, where curated data is delivered to downstream consumers like analysts, regulators, partners, or automated systems.

Unlike `ETL` or `MULTI_QUERIES`, which focus on data movement and computation, `EXPORTS` is concerned with **presentation, distribution, and persistence outside the database**.

ETLX leverages DuckDB’s native `COPY` capabilities and extends them with **template-based rendering** to support both structured and text-based outputs.

---

## Export Structure

An `EXPORTS` block follows the same declarative pattern used throughout ETLX:

1. **Block Metadata**

   * Defines connection, base path, activation, and shared execution context.

2. **Export Units** (Level 2 headings)

   * Each export represents a single file or document to be produced.
   * Exports may write directly via SQL or populate templates.

3. **Execution Lifecycle**

   * Optional `before_sql` and `after_sql` hooks
   * SQL execution and file materialization

## Example Configuration

````md {linenos=table}
# DAYLY_REPORTS
```yaml metadata
name: DailyExports
description: "Daily file exports for various datasets."
runs_as: EXPORTS
database: reporting_db
connection: "duckdb:"
path: "/path/to/Reports/YYYYMMDD"
active: true
```

## Sales Data Export
```yaml metadata
name: SalesExport
description: "Export daily sales data to CSV."
connection: "duckdb:"
export_sql:
  - "LOAD sqlite"
  - "ATTACH 'reporting.db' AS DB (TYPE SQLITE)"
  - export_query
  - "DETACH DB"
active: true
```

```sql
-- export_query
COPY (
    SELECT *
    FROM "DB"."Sales"
    WHERE "sale_date" = '{YYYY-MM-DD}'
) TO '/path/to/Reports/YYYYMMDD/sales_YYYYMMDD.csv' (FORMAT 'csv', HEADER true);
```

## Region Data Export to Excel

```yaml metadata
name: RegionExport
description: "Export region data to an Excel file."
connection: "duckdb:"
export_sql:
  - "LOAD sqlite"
  - "LOAD excel"
  - "ATTACH 'reporting.db' AS DB (TYPE SQLITE)"
  - export
  - "DETACH DB"
active: true
```

```sql
-- export
COPY (
    SELECT *
    FROM "DB"."Regions"
    WHERE "updated_at" >= '{YYYY-MM-DD}'
) TO '/path/to/Reports/YYYYMMDD/regions_YYYYMMDD.xlsx' (FORMAT XLSX, HEADER TRUE);
```
````

## Template-Based Exports

In addition to raw file exports, ETLX supports **template-driven exports**. These are ideal for producing standardized reports where query results must be injected into predefined layouts.

Templates are commonly used with Excel, but the same concept applies to text-based formats such as HTML or Markdown.

### Excel Template Example

````md {linenos=table}
## Sales Report Template

```yaml metadata
name: SalesReport
description: "Generate a sales report from a template."
connection: "duckdb:"
before_sql:
  - "LOAD sqlite"
  - "ATTACH 'reporting.db' AS DB (TYPE SQLITE)"
template: "/path/to/Templates/sales_template.xlsx"
path: "/path/to/Reports/sales_report_YYYYMMDD.xlsx"
mapping:
  - sheet: Summary
    range: B2 # {total_sales: B2, order_counts: C2}
    sql: summary_query
    type: value
    table: SummaryTable
    table_style: TableStyleLight1
    header: true
    if_exists: delete
  - sheet: Details
    range: A1
    sql: details_query
    type: range
    key: total_sales
after_sql: "DETACH DB"
active: true
```

```sql
-- summary_query
SELECT SUM(total_sales) AS total_sales
FROM "DB"."Sales"
WHERE "sale_date" = '{YYYY-MM-DD}'
```

```sql
-- details_query
SELECT *
FROM "DB"."Sales"
WHERE "sale_date" = '{YYYY-MM-DD}';
```
````

### Mapping Notes

* `mapping` can be:

  * A YAML list (inline definition)
  * A string referencing a query
  * Loaded dynamically from a database table
* In real-world scenarios, mappings often grow large and are easier to maintain in spreadsheets or database tables rather than static configuration files.

## Text-Based Template Exports

ETLX also supports exporting reports using **plain-text templates** such as HTML, XML, or Markdown. These exports use Go’s `text/template` engine and receive query results as structured input data.

This mechanism is shared with the `NOTIFY` block and is well-suited for:

* HTML/XML/Markdown reports
* Integration payloads
* Generated documentation
* Email-ready content

### Example

````md {linenos=table}
## TEXT_TMPL

```yaml metadata
name: TEXT_TMPL
description: "Export data to text base template"
connection: "duckdb:"
before_sql:
  - "INSTALL sqlite"
  - "LOAD sqlite"
  - "ATTACH 'database/HTTP_EXTRACT.db' AS DB (TYPE SQLITE)"
data_sql:
  - logs
  - data
after_sql: "DETACH DB"
tmp_prefix: null
text_template: true
template: template # inline codeblock or file path
return_content: false
path: "nyc_taxy_YYYYMMDD.html"
active: true
```
```sql
-- data
SELECT *
FROM "DB"."NYC_TAXI"
WHERE "tpep_pickup_datetime"::DATETIME <= '{YYYY-MM-DD}'
LIMIT 100
```

```sql
-- logs
SELECT *
FROM "DB"."etlx_logs"
--WHERE "ref" = '{YYYY-MM-DD}'
```

```html template
<style>
  table {
    border-collapse: collapse;
    width: 100%;
    font-family: sans-serif;
    font-size: 14px;
  }
  th, td {
    border: 1px solid #ddd;
    padding: 8px;
    text-align: left;
  }
  th {
    background-color: #f2f2f2;
    font-weight: bold;
  }
  tr:nth-child(even) {
    background-color: #f9f9f9;
  }
  tr:hover {
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.2);
    background-color: #eef6ff;
  }
</style>
<b>ETLX Text Template</b><br /><br />
This is gebnerated by ETLX automatically!<br />
{{ with .logs }}
    {{ if eq .success true }}
      <table>
        <tr>
            <th>Name</th>
            <th>Ref</th>
            <th>Start</th>
            <th>End</th>
            <th>Duration</th>
            <th>Success</th>
            <th>Message</th>
        </tr>
        {{ range .data }}
        <tr>
            <td>{{ .name }}</td>
            <td>{{ .ref }}</td>
            <td>{{ .start_at | date "2006-01-02 15:04:05" }}</td>
            <td>{{ .end_at | date "2006-01-02 15:04:05" }}</td>
            <td>{{ divf .duration 1000000000 | printf "%.4fs" }}</td>
            <td>{{ .success }}</td>
            <td><span title="{{ .msg }}">{{ .msg | toString | abbrev 30}}</span></td>
        </tr>
        {{ else }}
        <tr>
          <td colspan="7">No items available</td>
        </tr>
        {{ end }}
      </table>
    {{ else }}
      <p>{{.msg}}</p>
    {{ end }}
{{ else }}
<p>Logs information missing.</p>
{{ end }}
```
````

### Parameters

| Field            | Description                                         |
| ---------------- | --------------------------------------------------- |
| `text_template`  | Enables text-based template rendering               |
| `template`       | SQL block containing the Go template                |
| `data_sql`       | Named SQL blocks whose results feed the template    |
| `path`           | Output file path (supports placeholders)            |
| `return_content` | If true, returns content instead of writing to disk |

I would add a new section immediately after **Text-Based Template Exports**, since PDF generation is a natural extension of that feature.

---

## PDF Export Support

Text templates are not limited to HTML, Markdown, or XML. ETLX can also generate **PDF reports** with minimal configuration.

The output format is determined by the file extension specified in `path`.

For example:

```yaml
path: reports/daily_report.pdf
```

If the output path ends with `.pdf`, ETLX automatically renders the template as a PDF instead of writing plain text.


### HTML to PDF (Default)

By default, ETLX assumes the template is **HTML** and renders it using a Chromium-based engine.

```yaml
text_template: true
template: report.html
path: reports/report.pdf
```

This approach preserves:

* CSS styling
* Tables
* Images
* Charts
* Modern HTML layouts

> **Requirement:** A Chromium-compatible browser (such as Google Chrome or Chromium) must be installed and available on the system.

---

### LaTeX to PDF

For scientific papers, invoices, books, or highly formatted reports, ETLX can render templates using **LaTeX** instead.

Simply add a `pdf` configuration section:

```yaml
text_template: true
template: report.tex
path: reports/report.pdf
pdf:
  latex: true
```

In this mode the template itself should contain valid **LaTeX** rather than HTML.

Example:

```latex
\documentclass{article}

\begin{document}

Hello {{ .customer }}

Total: {{ .total }}

\end{document}
```

> **Requirement:** The `pdflatex` executable must be installed and available in your system `PATH`.

This allows ETLX to generate publication-quality PDFs suitable for:

* Academic papers
* Financial statements
* Regulatory reports
* Contracts
* Books
* Technical documentation

### PDF Configuration

The `pdf` object controls how PDF rendering is performed.

```yaml
pdf:
  latex: true
```

Additional rendering options may be added in future releases.


### Native DuckDB PDF Generation

For simple reports, you may not need a template engine at all.

DuckDB's community **PDF extension** can generate PDFs directly from SQL.

```sql
INSTALL pdf;
LOAD pdf;

SELECT write_pdf(
    'Hello from DuckDB!',
    '/tmp/hello.pdf'
);
```

or export an entire query:

```sql
COPY (
    SELECT *
    FROM sales
)
TO '/tmp/sales.pdf'
(FORMAT PDF);
```

This approach is powered by **libharu** and requires **no external rendering engine**, making it an excellent option for lightweight document generation directly from SQL.

It can also be combined with ETLX workflows—for example, preparing data with ETLX and using DuckDB's native PDF capabilities to generate simple tabular reports.

### Choosing the Right Approach

| Method               | Best For                                           | External Dependencies |
| -------------------- | -------------------------------------------------- | --------------------- |
| HTML → PDF           | Dashboards, styled reports, invoices, rich layouts | Chromium              |
| LaTeX → PDF          | Scientific documents, books, regulatory reports    | `pdflatex`            |
| DuckDB PDF Extension | Simple SQL-generated PDFs and tables               | None                  |

This flexibility allows ETLX to support everything from lightweight SQL-generated documents to fully designed enterprise reports, all within the same declarative workflow.


#### 🧰 Advanced Template Functions (Sprig)

ETLX integrates the [`Sprig`](https://github.com/Masterminds/sprig) The Sprig library provides over 70 template functions for Go’s template language, such as:

* String manipulation: `upper`, `lower`, `trim`, `contains`, `replace`
* Math: `add`, `mul`, `round`
* Date formatting: `date`, `now`, `dateModify`
* List operations: `append`, `uniq`, `join`

You can use these helpers directly in your templates:

```gotmpl
{{ .ref | upper }}
{{ .start_at | date "2006-01-02" }}
```

This enables powerful report generation and custom formatting out-of-the-box.

## How Exports Work

1. **Parse Configuration**

   * Each export is treated as an independent execution unit.

2. **Prepare Environment**

   * Executes `before_sql` hooks and loads required extensions.

3. **Materialize Output**

   * Executes export SQL or applies template mappings.

4. **Finalize**

   * Runs `after_sql` hooks and closes resources.

---

## Benefits of EXPORTS

* **Format Flexibility**: CSV, Excel, Parquet, HTML, Markdown, and more
* **Separation of Concerns**: Clean split between computation and delivery
* **Repeatability**: Deterministic, parameterized file generation
* **Governance-Friendly**: Outputs are traceable, reproducible, and auditable

The `EXPORTS` block completes the ETLX pipeline by turning validated, curated data into consumable artifacts.

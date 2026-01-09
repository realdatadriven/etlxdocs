---
weight: 920
title: "Governance Artifacts"
description: "Generating governance artifacts like data dictionaries and data quality rules from ETLX pipelines."
icon: overview
date: 2026-01-08T01:04:15+00:00
lastmod: 2026-01-08T01:04:15+00:00
draft: false
images: []
---

## Generate Governance Artifacts

To generate governance artifacts for the example above, you can add an `EXPORTS` block that defines a text template. This template can traverse all the metadata already embedded in the pipeline to generate artifacts such as a data dictionary, data quality rules, or compliance documentation.

This is possible because an export template in ETLX is not limited to query results. It can generate text from *any* data passed through it via `data_sql`, including the pipeline configuration itself, which is available through the `.conf` variable.

As mentioned earlier, the Markdown specification is parsed into a nested Go `map[string]any`, where every structural and semantic aspect of the document—queries, fields, metadata, ordering, and relationships—is preserved as data. This means the specification is not just documentation, but a first-class data structure that can be queried, transformed, and rendered.

In practice, this allows the pipeline to produce its own governance artifacts directly from the same source of truth, ensuring that documentation, execution, and governance always remain aligned.

Following the example [Embedded SQLite]({{% relref "embedded-sqlite#extract-and-transform-data" %}}), here is how you could define an `EXPORTS` block to generate a data dictionary as an HTML file:

````md {linenos=table}
<!-- markdownlint-disable MD025 -->
# GOVERNANCE_ARTIFACTS
```yaml metadata    
name: GOVERNANCE_ARTIFACTS  
description: "Generates governance artifacts like data dictionary and data quality rules."
runs_as: EXPORTS    
active: true
``` 
## DATA_DICTIONARY

```yaml metadata    
name: DATA_DICTIONARY  
description: "Generates a data dictionary from the metadata of the queries."    
connection: "duckdb:"
before_sql: "ATTACH 'sqlite_ex.db' AS DB (TYPE SQLITE)"
data_sql: null
after_sql: "DETACH DB"
tmp_prefix: null
text_template: true
template: data_dictionary_template
return_content: false
path: "data_dictionary.html"
active: true
``` 

```html data_dictionary_template
<!-- governance_template -->
<html encoding="UTF-8" lang="en">
<head>
    <title>Governance Artifacts</title>
    <style>
        body {
            font-family: Arial, sans-serif;
        }
        table {
            width: 100%;
            border-collapse: collapse;
            margin-bottom: 32px;
        }
        th, td {
            border: 1px solid black;
            padding: 8px;
            text-align: left;
            vertical-align: top;
        }
        th {
            background-color: #f2f2f2;
        }
        h1, h2, h3 {
            margin-top: 32px;
        }
    </style>
</head>
<body>
<h1>Governance Artifacts</h1>
{{- range $blockName, $block := .conf }}
    {{- if and
        (ne $blockName "__order")
        (ne $blockName "metadata")
        (kindIs "map" $block)
    }}
    {{- $meta := index $block "metadata" }}
    <!-- ===================================================== -->
    <!-- DATA DICTIONARY (no run_as at level 1) -->
    <!-- ===================================================== -->
    {{- if or (not $meta) (not (hasKey $meta "run_as")) }}
        <h2>
            {{ $blockName }}
            {{- with $meta }} — {{ index . "description" }}{{ end }}
        </h2>
        {{- $order := index $block "__order" }}
        {{- if kindIs "slice" $order }}
        <h3>Data Dictionary</h3>
        <table>
            <tr>
                <th>Field Name</th>
                <th>Description</th>
                <th>Type</th>
                <th>Owner</th>
                <th>Derived From</th>
                <th>Formula</th>
            </tr>
            {{- range $i, $fieldName := $order }}
                {{- $field := index $block $fieldName }}
                {{- if kindIs "map" $field }}
                {{- $fmeta := index $field "metadata" }}
                <tr>
                    <td>{{ $fieldName }}</td>
                    <td>{{ index $fmeta "description" | default "N/A" }}</td>
                    <td>{{ index $fmeta "type" | default "N/A" }}</td>
                    <td>{{ index $fmeta "owner" | default "N/A" }}</td>
                    <td>
                        {{- with index $fmeta "derived_from" }}
                            {{- range $j, $src := . }}
                                {{- if $j }}, {{ end }}{{ $src }}
                            {{- end }}
                        {{- else }}
                            N/A
                        {{- end }}
                    </td>
                    <td>{{ index $fmeta "formula" | default "N/A" }}</td>
                </tr>
                {{- end }}
            {{- end }}
        </table>
        {{- end }}
    {{- end }}
    <!-- ===================================================== -->
    <!-- DATA QUALITY RULES (run_as = DATA_QUALITY) -->
    <!-- ===================================================== -->
    {{- if and
        $meta
        (eq (index $meta "run_as") "DATA_QUALITY")
    }}
        <h2>
            {{ $blockName }}
            {{- with $meta }} — {{ index . "description" }}{{ end }}
        </h2>
        <h3>Data Quality Rules</h3>
        <table>
            <tr>
                <th>Rule Name</th>
                <th>Description</th>
                <th>Active</th>
            </tr>
            {{- range $ruleName, $rule := $block }}
                {{- if and
                    (ne $ruleName "metadata")
                    (ne $ruleName "__order")
                    (kindIs "map" $rule)
                }}
                {{- $rmeta := index $rule "metadata" }}
                <tr>
                    <td>{{ $ruleName }}</td>
                    <td>{{ index $rmeta "description" | default "N/A" }}</td>
                    <td>
                        {{- if hasKey $rule "active" -}}
                            {{ index $rule "active" }}
                        {{- else -}}
                            true
                        {{- end -}}
                    </td>
                </tr>
                {{- end }}
            {{- end }}
        </table>
    {{- end }}
    {{- end }}
{{- end }}
</body>
</html>
```
````
---

Here’s a **concrete layout example** of what your template would generate when rendered, using a fictional ETLX pipeline. This is **not the template**, but the **resulting page structure/content** a user would see in the browser.

# Data Dictionary

## sales_orders — Cleaned and enriched sales orders dataset

| Field Name    | Description                     | Type    | Owner        | Derived From                  | Formula                  |
| ------------- | ------------------------------- | ------- | ------------ | ----------------------------- | ------------------------ |
| order_id      | Unique identifier for the order | STRING  | sales_team   | raw_orders.order_id           | N/A                      |
| customer_id   | Identifier of the customer      | STRING  | crm_team     | raw_orders.customer_ref       | N/A                      |
| order_date    | Date when the order was placed  | DATE    | sales_team   | raw_orders.created_at         | CAST(created_at AS DATE) |
| total_amount  | Total order value after tax     | DECIMAL | finance_team | line_items.amount, tax.amount | SUM(amount) + SUM(tax)   |
| is_high_value | Flags high value orders         | BOOLEAN | analytics    | total_amount                  | total_amount > 1000      |


## daily_sales_summary — Aggregated daily sales metrics

| Field Name      | Description                  | Type    | Owner        | Derived From                | Formula                      |
| --------------- | ---------------------------- | ------- | ------------ | --------------------------- | ---------------------------- |
| sales_date      | Business date                | DATE    | analytics    | sales_orders.order_date     | N/A                          |
| total_orders    | Number of orders per day     | INTEGER | analytics    | sales_orders.order_id       | COUNT(order_id)              |
| gross_revenue   | Total revenue before refunds | DECIMAL | finance_team | sales_orders.total_amount   | SUM(total_amount)            |
| avg_order_value | Average value per order      | DECIMAL | analytics    | gross_revenue, total_orders | gross_revenue / total_orders |


> Notes on how this maps to your template

>* Only pipelines **without `metadata.run_as`** are shown ✔️
>* Fields are rendered **in `__order` sequence** ✔️
>* Keys like `__order` are **ignored as fields** ✔️
>* Only **map-based field definitions** are rendered (slices ignored) ✔️
>* Missing metadata gracefully falls back to **`N/A`** ✔️
>* `derived_from` supports **multiple sources** ✔️

---

## Customizing the Data Dictionary Output

ETLX allows you to fully customize how a Data Dictionary is generated and rendered. The example shown above is rendered as an **HTML table**, but the same metadata can be presented in many different ways depending on your needs.

### Conditional Rendering

You can apply conditions to control **what is included or excluded** in the output.

Typical use cases include:

* Rendering only pipelines **without `metadata.run_as`**
* Skipping technical or internal fields
* Showing optional columns only when metadata exists
* Applying different layouts for different environments (e.g. dev vs prod)

Example conditions:

* Include a field only if metadata exists
* Exclude system keys such as `__order`
* Render only map-based field definitions (ignore slices or arrays)

This makes the documentation **data-driven**, consistent, and safe to auto-generate.

### Styling and Presentation

Because the rendering layer is template-based, you can control **visual styling independently from logic**.

With HTML templates you can:

* Apply custom CSS (fonts, colors, spacing)
* Highlight derived or calculated fields
* Emphasize business-critical columns
* Add section headers, notes, or warnings

For example:

* Calculated fields could be highlighted
* Fields owned by Finance could have a different style
* Deprecated fields could be visually marked

### Alternative Layouts

While the example uses a **tabular layout**, ETLX does not enforce a specific structure.

You may render the same metadata as:

#### Bullet-point documentation

```md
• order_id (STRING)
  - Description: Unique identifier for the order
  - Owner: sales_team
  - Source: raw_orders.order_id

• total_amount (DECIMAL)
  - Derived from: line_items.amount, tax.amount
  - Formula: SUM(amount) + SUM(tax)
```

This format is often easier to read in Markdown or plain-text documentation.

### Multiple Output Formats

Although this example uses **HTML**, the same metadata can be rendered into other formats using different templates:

* **HTML** → Web documentation, internal portals
* **CSV** → Audits, governance, Excel exports
* **XML** → System integrations, legacy tools
* **Markdown** → GitHub, GitLab, static docs
* **JSON** → APIs, automation, lineage tools

The data dictionary generation logic remains the same — only the **template changes**.


### Why This Matters in ETLX

This approach ensures that:

* Documentation stays **in sync with pipelines**
* Business and technical users share a **single source of truth**
* Governance, lineage, and ownership are **first-class citizens**
* Output can evolve without touching the ETL logic

In ETLX, documentation is not an afterthought — it is **generated, versioned, and reproducible**.

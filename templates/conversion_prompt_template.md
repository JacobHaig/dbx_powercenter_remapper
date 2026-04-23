# Conversion Prompt Template

Use this template to construct the prompt sent to the Databricks conversion agent. Replace every `{{PLACEHOLDER}}` with the actual value before sending. Do not leave placeholders unfilled.

---

## Template

```
You are the Databricks conversion agent described in AGENT.md.

Before starting, re-read the following documents:
- prompt/mega_prompt.md
- docs/powercenter_reference.md
- docs/transformation_mappings.md
- docs/conversion_standards.md
- docs/workflow.md

Then perform the conversion described below.

---

## Conversion Request

**Mapping Name:** {{MAPPING_NAME}}
  The name of the <MAPPING> element to convert. If the XML contains multiple mappings,
  convert only this one unless instructed otherwise.

**Source System:** {{SOURCE_SYSTEM}}
  The originating system for this mapping's data (e.g., "Oracle EBS R12", "SQL Server 2019",
  "Flat file — pipe-delimited CSV", "Teradata EDW").

**Source Connection Variable:** {{SOURCE_CONNECTION_VARIABLE}}
  The PowerCenter connection variable name (e.g., "$DBConnection_Orders"). Map this to:
  Unity Catalog path: {{UNITY_CATALOG_SOURCE_PATH}}
  OR JDBC secret scope/key: {{JDBC_SECRET_SCOPE}} / {{JDBC_SECRET_KEY}}
  (delete whichever does not apply)

**Target Catalog:** {{TARGET_CATALOG}}
  The Unity Catalog catalog name where output tables reside (e.g., "prod_catalog", "dev_catalog").

**Target Schema:** {{TARGET_SCHEMA}}
  The schema/database within the catalog (e.g., "orders_dw", "finance").

**Target Table(s):** {{TARGET_TABLE_NAMES}}
  Comma-separated list of target table names as they should appear in the notebook
  (e.g., "fact_orders, dim_customer_snapshot").

**Environment:** {{ENVIRONMENT}}
  The intended deployment environment: dev / staging / prod.
  Used as the default value for the `catalog` widget.

**Run Date Handling:** {{RUN_DATE_HANDLING}}
  How incremental dates should be handled:
  - "widget" — accept a run_date widget parameter, default to current date
  - "parameter: $$LAST_EXTRACTION_DATE" — map PowerCenter parameter to widget
  - "full reload" — no date filtering, always process full dataset

**Special Instructions:** {{SPECIAL_INSTRUCTIONS}}
  Any mapping-specific notes, known issues, or deviations from standard behavior.
  Example: "The Aggregator uses a conditional SUM — verify the filter condition maps correctly."
  Leave blank if none.

---

## Input XML

{{XML_CONTENT}}

---

## Expected Output

Return:

1. The complete Databricks notebook as a `.py` file with `# COMMAND ----------` cell delimiters.
   File name convention: `nb_<mapping_name_lowercase>.py`

2. A Conversion Summary Report in this format:

   ```
   Mapping:          <name>
   Folder:           <folder>
   Source Count:     <n> (<source types>)
   Target Count:     <n> (<table names>)
   Transformations:  <total> (<translated> translated, <review_count> flagged for REVIEW)
   Complexity:       Low / Medium / High
   REVIEW Items:
     1. <description> (cell: <cell_comment>)
     2. ...
   ```
```

---

## Placeholder Reference

| Placeholder | Description | Example |
|---|---|---|
| `{{MAPPING_NAME}}` | Exact `NAME` attribute of the `<MAPPING>` element | `M_LOAD_FACT_ORDERS` |
| `{{SOURCE_SYSTEM}}` | Human-readable source system name | `Oracle EBS R12` |
| `{{SOURCE_CONNECTION_VARIABLE}}` | PowerCenter connection variable | `$DBConnection_Orders` |
| `{{UNITY_CATALOG_SOURCE_PATH}}` | Unity Catalog path for the source | `source_catalog.ebs.orders` |
| `{{JDBC_SECRET_SCOPE}}` | Databricks secret scope for JDBC | `prod_connections` |
| `{{JDBC_SECRET_KEY}}` | Secret key within the scope | `oracle_orders_url` |
| `{{TARGET_CATALOG}}` | Destination Unity Catalog catalog | `prod_catalog` |
| `{{TARGET_SCHEMA}}` | Destination schema | `orders_dw` |
| `{{TARGET_TABLE_NAMES}}` | Target table name(s) | `fact_orders` |
| `{{ENVIRONMENT}}` | Deployment environment | `dev` |
| `{{RUN_DATE_HANDLING}}` | How incremental dates are handled | `widget` |
| `{{SPECIAL_INSTRUCTIONS}}` | Freeform notes | _(blank)_ |
| `{{XML_CONTENT}}` | Full content of the PowerCenter XML export | _(paste XML here)_ |

---

## Batch Conversion Template

When converting an entire folder, use this wrapper:

```
You are the Databricks conversion agent described in AGENT.md.

Convert all ISVALID="YES" mappings in the attached XML export.
Process them in dependency order (mapplets first).

For each mapping, produce:
1. A notebook file named nb_<mapping_name_lowercase>.py
2. A per-mapping Conversion Summary Report

After all mappings are processed, produce a Folder Summary:
  - Total mappings attempted
  - Total mappings successfully converted
  - Total mappings escalated (with reason)
  - Combined REVIEW item count

Global settings for all mappings in this batch:
  Target Catalog: {{TARGET_CATALOG}}
  Target Schema: {{TARGET_SCHEMA}}
  Environment: {{ENVIRONMENT}}
  Source System: {{SOURCE_SYSTEM}}
  Source Connection Variable: {{SOURCE_CONNECTION_VARIABLE}}
  Unity Catalog Source Path: {{UNITY_CATALOG_SOURCE_PATH}}

XML Export:
{{XML_CONTENT}}
```

---

## Example: Filled Template

```
You are the Databricks conversion agent described in AGENT.md.
...

## Conversion Request

**Mapping Name:** M_LOAD_FACT_ORDERS
**Source System:** Oracle EBS R12
**Source Connection Variable:** $DBConnection_EBS
  Unity Catalog path: prod_catalog.ebs_source.orders
**Target Catalog:** prod_catalog
**Target Schema:** orders_dw
**Target Table(s):** fact_orders
**Environment:** dev
**Run Date Handling:** widget
**Special Instructions:** The SQ_ORDERS has a Sql Query override — preserve it exactly.

## Input XML

<?xml version="1.0" encoding="UTF-8"?>
<POWERMART ...>
  ...
</POWERMART>
```

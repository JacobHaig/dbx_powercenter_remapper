# Conversion Prompt Template

Use this template to construct the prompt sent to the Databricks conversion agent. Replace every `{{PLACEHOLDER}}` with the actual value before sending. Do not leave placeholders unfilled.

> **Prefer `templates/mega_prompt.md` for Genie Code sessions** — it is the interactive entry point with built-in agent instructions. Use this template when constructing prompts programmatically or outside of Genie Code.

---

## Template

```
You are the Databricks conversion agent described in AGENT.md.

Before starting, re-read the following documents in full:
- templates/mega_prompt.md
- docs/powercenter_reference.md
- docs/transformation_mappings.md
- docs/xml_to_pyspark_examples.md
- docs/conversion_standards.md

Then perform the conversion described below, following the 4-phase execution model
(Review & Analysis → Extraction → Notebook Creation → Cell Review) defined in
templates/mega_prompt.md.

---

## Conversion Request

**Mapping file:** {{MAPPING_FILENAME}}
  The XML file placed in the `input/` folder (e.g., `M_LOAD_FACT_ORDERS.xml`).
  The agent reads from `input/{{MAPPING_FILENAME}}`.

**Mapplet files (if any):** {{MAPPLET_FILENAMES}}
  Comma-separated list of mapplet XML filenames placed in `input/` using the `mlt_` prefix.
  Leave blank if the mapping has no Mapplet references.
  Example: `mlt_address_cleanse.xml, mlt_date_normalize.xml`

**Connection Mapping:**
  For each PowerCenter connection name found in the XML, the Unity Catalog equivalent:

  | PowerCenter Connection Name | Unity Catalog Path (`catalog.schema`) |
  |---|---|
  {{CONNECTION_MAPPING_ROWS}}

  Example row: `| \`PROD_DW_READ\` | \`prod_catalog.dw\` |`

**Target:**

  | Field | Value |
  |---|---|
  | Catalog | {{TARGET_CATALOG}} |
  | Schema | {{TARGET_SCHEMA}} |
  | Table(s) | {{TARGET_TABLE_NAMES}} |

**Merge Keys (optional):**
  For targets with an upstream Update Strategy, the match key columns:

  | Target Table | Merge Key Column(s) |
  |---|---|
  {{MERGE_KEY_ROWS}}

  Leave blank if unknown — the agent will flag these as REVIEW.

**Run Date Handling:** {{RUN_DATE_HANDLING}}
  Choose one:
  - `widget` — create a run_date widget, default to today's date
  - `full reload` — no date filtering, process full dataset every run
  - `parameter: $$<PARAM_NAME>` — map PowerCenter parameter to a widget named run_date

**Special Instructions:** {{SPECIAL_INSTRUCTIONS}}
  Any mapping-specific notes, known issues, or deviations from standard behavior.
  Leave blank if none.

---

## Expected Output

The agent produces output in this order:

1. **Phase 1 — Analysis Summary**
   ```
   Phase 1 — Analysis Summary
   Input files:     <filenames found in input/>
   Mapping:         <name>
   Folder:          <folder>
   Sources:         <n> (<types>)
   Targets:         <n> (<names>)
   Transformations: <total> (<type x count, ...>)
   Unsupported:     <name> (<type>) — will flag as REVIEW   [omit if none]
   Complexity:      Low / Medium / High
   ```

2. **Phase 2 — Extraction Complete manifest**
   ```
   Phase 2 — Extraction Complete
   Sources extracted:          <n> (<total columns> columns)
   Targets extracted:          <n> (<total columns> columns)
   Transformations extracted:  <n> (all fields and attributes)
   Connectors mapped:          <n> edges — DAG validated, no cycles
   Connection variables:       <$Var> → <catalog.schema>
   Parameters found:           <$$PARAM> → <widget_name>   [omit if none]
   ```

3. **Phase 4 — Review Summary**
   ```
   Phase 4 — Review Summary
   Cells reviewed:  <n>
   Issues found:    <n> (<brief description>)   [or "none"]
   REVIEW flags:    <n> (<brief description>)   [or "none"]
   Status:          Ready
   ```

4. **The notebook** — written to `notebooks/nb_<mapping_name_lowercase>.py`.
   First line: `# Databricks notebook source`. Cells separated by `# COMMAND ----------`.

5. **Conversion Summary**
   ```
   Mapping:          <MAPPING NAME>
   Folder:           <FOLDER NAME>
   Sources:          <n> (<types>)
   Targets:          <n> (<table names>)
   Transformations:  <total> (<n> translated, <n> flagged for REVIEW)
   Complexity:       Low / Medium / High
   REVIEW Items:
     1. <description> (cell: <cell comment>)
   ```
```

---

## Placeholder Reference

| Placeholder | Description | Example |
|---|---|---|
| `{{MAPPING_FILENAME}}` | XML filename in `input/` | `M_LOAD_FACT_ORDERS.xml` |
| `{{MAPPLET_FILENAMES}}` | Mapplet XML filenames in `input/` (comma-separated) | `mlt_address_cleanse.xml` |
| `{{CONNECTION_MAPPING_ROWS}}` | One table row per connection: `\| \`NAME\` \| \`catalog.schema\` \|` | `\| \`PROD_DW_READ\` \| \`prod_catalog.dw\` \|` |
| `{{TARGET_CATALOG}}` | Unity Catalog catalog name | `prod_catalog` |
| `{{TARGET_SCHEMA}}` | Target schema | `orders_dw` |
| `{{TARGET_TABLE_NAMES}}` | Target table name(s) | `fact_orders` |
| `{{MERGE_KEY_ROWS}}` | One table row per Update Strategy target: `\| \`table\` \| \`KEY_COL\` \|` | `\| \`fact_orders\` \| \`ORDER_ID\` \|` |
| `{{RUN_DATE_HANDLING}}` | Date handling selection | `widget` |
| `{{SPECIAL_INSTRUCTIONS}}` | Freeform notes | _(blank)_ |

---

## Batch Conversion Template

When converting an entire folder, use this wrapper. Place all mapping XML files in `input/` before running.

```
You are the Databricks conversion agent described in AGENT.md.

Convert all ISVALID="YES" mappings found in the `input/` folder.
Process them in dependency order (mapplets first, then mappings that reference them).

For each mapping, follow the 4-phase execution model and produce:
1. A notebook written to `notebooks/nb_<mapping_name_lowercase>.py`
2. A per-mapping Conversion Summary

After all mappings are processed, produce a Folder Summary:
  - Total mappings attempted
  - Total mappings successfully converted
  - Total mappings with REVIEW items (with item count)
  - Combined REVIEW item count

Global settings for all mappings in this batch:
  Target Catalog: {{TARGET_CATALOG}}
  Target Schema: {{TARGET_SCHEMA}}
  Run Date Handling: {{RUN_DATE_HANDLING}}
  Special Instructions: {{SPECIAL_INSTRUCTIONS}}

Connection Mapping (applies to all mappings):
  | PowerCenter Connection Name | Unity Catalog Path |
  |---|---|
  {{CONNECTION_MAPPING_ROWS}}
```

---

## Example: Filled Template

```
You are the Databricks conversion agent described in AGENT.md.

Before starting, re-read: templates/mega_prompt.md, docs/powercenter_reference.md,
docs/transformation_mappings.md, docs/xml_to_pyspark_examples.md, docs/conversion_standards.md.

## Conversion Request

**Mapping file:** M_LOAD_FACT_ORDERS.xml
**Mapplet files (if any):** mlt_address_cleanse.xml

**Connection Mapping:**
  | PowerCenter Connection Name | Unity Catalog Path (`catalog.schema`) |
  |---|---|
  | `PROD_DW_READ` | `prod_catalog.ebs_source` |
  | `$DBConnection_EBS` | `prod_catalog.ebs_source` |

**Target:**
  | Field | Value |
  | Catalog | prod_catalog |
  | Schema | orders_dw |
  | Table(s) | fact_orders |

**Merge Keys:**
  | Target Table | Merge Key Column(s) |
  |---|---|
  | `fact_orders` | `ORDER_ID` |

**Run Date Handling:** widget
**Special Instructions:** The SQ_ORDERS Source Qualifier has a Sql Query override — preserve it exactly.
```

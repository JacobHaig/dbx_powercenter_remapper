# AGENT.md — Databricks Conversion Agent Instructions

Read this file at the start of every conversion session. It is your operating manual.

## Your Role

You convert Informatica PowerCenter mapping XML into Databricks PySpark Notebooks. Each `<MAPPING>` in the XML becomes one notebook. Your output must be valid, runnable Python that a Databricks engineer could deploy without modification.

## The Fundamental Rule

**One `<MAPPING>` = one Databricks notebook.** Every transformation, source read, and target write from that mapping lives in a single `.py` notebook file. Nothing is split across files. Nothing is left to the caller to wire up. All environment-specific values (catalog, schema, paths, connection info) are parameterized as `dbutils.widgets` at the top of the notebook so it can run in any environment without code changes.

## Before You Start a Conversion

1. Read `docs/powercenter_reference.md` to understand the XML schema you are parsing.
2. Read `docs/transformation_mappings.md` to know how to translate each transformation type.
3. Read `docs/xml_to_pyspark_examples.md` for concrete side-by-side XML → PySpark examples.
4. Read `docs/conversion_standards.md` to know how to structure the output notebook.
5. Read `docs/workflow.md` for the end-to-end process you must follow.

## Execution Phases

Every conversion follows these four phases in order. Each phase has a required output before the next begins.

### Phase 1 — Review & Analysis
Validate `ISVALID="YES"`. Build the DAG from `<CONNECTOR>` elements. Topological sort. Inventory all transformation types. Flag unsupported types. Score complexity. Output the **Analysis Summary** before proceeding.

### Phase 2 — Extraction
Extract everything from the XML: all `<SOURCE>`, `<TARGET>`, and `<TRANSFORMATION>` definitions (every `<TRANSFORMFIELD>` and `<TABLEATTRIBUTE>`), full connector adjacency list, `<TARGETLOADORDER>`, connection variable → Unity Catalog mappings, parameter → widget mappings, all expression strings. Output the **Extraction Complete manifest** before proceeding.

### Phase 3 — Notebook Creation
Write the complete native Databricks notebook using patterns from `docs/transformation_mappings.md` and `docs/conversion_standards.md`.

Required first line: `# Databricks notebook source`
Cell separator: `# COMMAND ----------`

Cell order: header markdown → widgets → imports → source reads (one per SQ) → transformations (DAG order) → target writes (TARGETLOADORDER sequence) → validation cell (if applicable).

Use `docs/transformation_mappings.md` for every translation decision. Never leave PowerCenter syntax in output. Proceed directly to Phase 4 with no pause.

### Phase 4 — Cell Review
Walk every cell. For each: check no PowerCenter syntax remains, all ports accounted for, all values from widgets, Update Strategy → MERGE, small Lookups → broadcast, REVIEW comments in header. Correct inline. Output the **Review Summary**, then deliver the notebook and conversion summary.

## When You Are Uncertain

If a transformation type is not in `docs/transformation_mappings.md`:
1. Note it as `# REVIEW: UNSUPPORTED TRANSFORMATION — <type>` in the cell.
2. Provide a best-effort PySpark approximation.
3. List it prominently in the header cell.

If an expression function is not in the function mapping table:
1. Use Python/PySpark built-ins if a direct equivalent exists.
2. Otherwise stub it as `# REVIEW: UNMAPPED FUNCTION — <function_name>(<args>)`.

## Session Inputs

XML files are read from the `input/` folder at the root of this repository:
- `input/<mapping_filename>.xml` — the main mapping XML; filename is specified in `### Mapping File` in the conversion request
- `input/mlt_<name>.xml` — mapplet XML files; read automatically when mapplet references are discovered in the mapping

All other inputs come from the filled-in fields in `### Connection Mapping`, `### Target`, `### Merge Keys`, `### Run Date Handling`, and `### Special Instructions` in `templates/mega_prompt.md`.

## Session Outputs

Return in this order:
1. **Phase 1 — Analysis Summary** (before any code is written)
2. **Phase 2 — Extraction Complete manifest** (before notebook is written)
3. **Phase 4 — Review Summary** (after cell review)
4. **The notebook** — written to `notebooks/nb_<mapping_name_lowercase>.py` in this repo; first line `# Databricks notebook source`; cells separated by `# COMMAND ----------`
5. **Conversion Summary** — mapping name, source/target counts, transformation count, REVIEW items, complexity score

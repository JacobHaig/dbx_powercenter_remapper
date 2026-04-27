# AGENT.md — Databricks Conversion Agent Instructions

Read this file at the start of every conversion session. It is your operating manual.

## Your Role

You convert Informatica PowerCenter mapping XML into Databricks PySpark Notebooks. Each `<MAPPING>` in the XML becomes one notebook. Your output must be valid, runnable Python that a Databricks engineer could deploy without modification.

## The Fundamental Rule

**One `<MAPPING>` = one Databricks notebook asset.** Every transformation, source read, and target write from that mapping lives in a single notebook. Nothing is split across files. Nothing is left to the caller to wire up. All environment-specific values (catalog, schema, paths, connection info) are parameterized as `dbutils.widgets` at the top of the notebook so it can run in any environment without code changes. Create notebooks using the SDK pattern in `docs/databricks_notebook_creation.md` — never by writing a `.py` text file.

## Before You Start a Conversion

1. Read `docs/powercenter_reference.md` to understand the XML schema you are parsing.
2. Read `docs/transformation_mappings.md` to know how to translate each transformation type.
3. Read `docs/xml_to_pyspark_examples.md` for concrete side-by-side XML → PySpark examples.
4. Read `docs/conversion_standards.md` to know how to structure a regular notebook output.
5. Read `docs/workflow.md` for the end-to-end process you must follow.
6. Check the `### Output Format` field in the conversion request. If it is `pipeline`, also read `docs/lakeflow_pipeline_standards.md` before Phase 3.

## Execution Phases

Every conversion follows these four phases in order. Each phase has a required output before the next begins.

### Phase 1 — Review & Analysis
Validate `ISVALID="YES"`. Build the DAG from `<CONNECTOR>` elements. Topological sort. Inventory all transformation types. Flag unsupported types. Score complexity. Output the **Analysis Summary** before proceeding.

### Phase 2 — Extraction
Extract everything from the XML: all `<SOURCE>`, `<TARGET>`, and `<TRANSFORMATION>` definitions (every `<TRANSFORMFIELD>` and `<TABLEATTRIBUTE>`), full connector adjacency list, `<TARGETLOADORDER>`, connection variable → Unity Catalog mappings, parameter → widget mappings, all expression strings. Output the **Extraction Complete manifest** before proceeding.

### Phase 3 — Notebook Creation

Check the `### Output Format` field in the conversion request **before writing any code.**

- If `notebook` (or no selection): follow `docs/conversion_standards.md`. See below.
- If `pipeline`: read `docs/lakeflow_pipeline_standards.md` in full, then follow it instead.
- If neither is selected: stop and ask the user to mark a selection before proceeding.

Required first line of content: `# Databricks notebook source`
Cell separator: `# COMMAND ----------`
Create as a workspace notebook asset using `docs/databricks_notebook_creation.md` — never a `.py` file.

**notebook format cell order:** header markdown → widgets → imports → source reads (one per SQ) → transformations (DAG order) → target writes (TARGETLOADORDER sequence) → validation cell (if applicable).

**pipeline format cell order:** header markdown → imports (include `from pyspark import pipelines as dp`) → pipeline parameters → source `@dp.table` functions (one per SQ) → transformation `@dp.table` / `@dp.temporary_view` / `@dp.materialized_view` functions (DAG order) → target `@dp.table` or `dp.create_auto_cdc_flow` (TARGETLOADORDER sequence).

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
4. **The notebook** — created as a Databricks workspace notebook asset at `notebooks/nb_<mapping_name_lowercase>` (no `.py` extension); first line of content `# Databricks notebook source`; cells separated by `# COMMAND ----------`; use the SDK pattern in `docs/databricks_notebook_creation.md`
5. **Conversion Summary** — mapping name, source/target counts, transformation count, REVIEW items, complexity score

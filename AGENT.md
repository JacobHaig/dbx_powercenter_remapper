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

## Conversion Rules (Non-Negotiable)

### Parsing
- Parse the full `<MAPPING>` before writing any output. Understand the complete DAG before translating individual nodes.
- Walk the `<CONNECTOR>` list to reconstruct the data flow graph. Transformation order in the XML is not reliable; connector topology is.
- Respect `<TARGETLOADORDER>` when it exists.

### Translation
- Use `docs/transformation_mappings.md` as the authoritative translation reference. Do not invent patterns.
- When a transformation has no direct PySpark equivalent, translate to the closest semantic equivalent and add a `# REVIEW:` comment explaining the gap.
- PowerCenter expression language functions must be translated using the function mapping table in `docs/transformation_mappings.md`. Never leave PowerCenter syntax in the output.

### Output Notebook Structure (in cell order)
1. **Header cell (Markdown):** Mapping name, source system, target system, original mapping description, conversion timestamp, any `# REVIEW:` flags.
2. **Configuration cell:** Widget definitions (`dbutils.widgets`) for environment-dependent values (catalog, schema, source paths).
3. **Imports cell:** All `from pyspark.sql import ...` and library imports.
4. **Source reads:** One cell per source, using the pattern in `docs/conversion_standards.md`.
5. **Transformation cells:** One cell per logical transformation group, preserving the PowerCenter DAG order.
6. **Target writes:** One cell per target, using the Delta write pattern in `docs/conversion_standards.md`.
7. **Validation cell (optional):** Row count assertions if the original mapping had count checks.

### Quality Gates
- Every generated notebook must pass these checks before you return it:
  - [ ] No PowerCenter-specific syntax remains in any cell
  - [ ] All `<TRANSFORMFIELD>` output ports are accounted for downstream
  - [ ] All source/target names are parameterized (no hardcoded paths)
  - [ ] All `UPDATE_STRATEGY` transformations result in a Delta `MERGE` statement
  - [ ] Lookup transformations use broadcast joins when the lookup source is small (< 1 GB heuristic)
  - [ ] All `# REVIEW:` comments are listed in the header cell

## When You Are Uncertain

If a transformation type is not in `docs/transformation_mappings.md`:
1. Note it as `# REVIEW: UNSUPPORTED TRANSFORMATION — <type>` in the cell.
2. Provide a best-effort PySpark approximation.
3. List it prominently in the header cell.

If an expression function is not in the function mapping table:
1. Use Python/PySpark built-ins if a direct equivalent exists.
2. Otherwise stub it as `# REVIEW: UNMAPPED FUNCTION — <function_name>(<args>)`.

## Session Inputs

You will receive:
- `{{XML_CONTENT}}` — the raw PowerCenter XML export
- `{{TARGET_CATALOG}}` — Unity Catalog catalog name
- `{{TARGET_SCHEMA}}` — target schema/database
- `{{SOURCE_TYPE}}` — source system type (e.g., Oracle, SQL Server, flat file)
- `{{ENVIRONMENT}}` — dev / staging / prod

See `templates/conversion_prompt_template.md` for the full prompt structure.

## Session Outputs

Return:
1. The complete notebook as a single Python file (`.py`) with Databricks cell delimiters (`# COMMAND ----------`).
2. A short conversion summary listing: mapping name, transformation count, any `# REVIEW:` items, and estimated complexity (Low / Medium / High).

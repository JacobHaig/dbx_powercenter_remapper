# Validation Prompt Design

## Goal

Create `templates/validation_prompt.md` — a separate Genie Code prompt that teaches the Databricks agent how to build a test notebook for a previously converted PowerCenter notebook. The agent reads the conversion notebook, maps sample data files to source tables, builds one wholistic Databricks native notebook asset that loads sample data, runs the original notebook against it, validates the output, and tears down all temp tables it created.

## Context

- `templates/mega_prompt.md` handles conversion (PowerCenter XML → Databricks notebook)
- `templates/validation_prompt.md` handles validation — run AFTER conversion
- Conversion notebooks already parameterize everything via `dbutils.widgets`
- All generated notebooks live in `notebooks/` as Databricks workspace notebook assets (no `.py` extension)
- Same SDK creation pattern as conversion notebooks (`docs/databricks_notebook_creation.md`)

---

## User Inputs (filled in before submitting to Genie Code)

| Field | Purpose |
|---|---|
| Notebook to test | Name of the conversion notebook (e.g. `nb_m_load_fact_orders`) |
| Test catalog | Unity Catalog catalog where temp Delta tables will be written |
| Test schema | Schema within that catalog for temp Delta tables |
| Run date | Date value to pass to the original notebook's `run_date` widget |
| Expected row count (optional) | If known from source system — used for row count assertion |

The user also drops sample data files into `tests/data/` before running the prompt. File naming convention: `<source_table_name>.csv` or `<source_table_name>.parquet` (case-insensitive). The agent maps files to source tables by matching filenames to the table names it extracts from the conversion notebook.

---

## Execution Model

The test notebook uses `dbutils.notebook.run()` to execute the original conversion notebook as a black-box subprocess. All widget values in the original notebook are overridden via the `arguments` dict to point at the test catalog/schema. This is the only approach that works correctly — temp views and global temp views do not cross the execution context boundary created by `dbutils.notebook.run()`.

```python
result = dbutils.notebook.run(
    f"{repo_root}/notebooks/nb_m_load_fact_orders",
    timeout_seconds=600,
    arguments={
        "source_catalog": test_catalog,
        "source_schema": test_schema,
        "target_catalog": test_catalog,
        "target_schema": test_schema,
        "run_date": run_date
    }
)
```

All widget names passed in the `arguments` dict are extracted from the conversion notebook in Phase 1 — nothing is hardcoded.

---

## Four-Phase Execution Model

### Phase 1: Read & Analyze

The agent reads the conversion notebook at `notebooks/<notebook_name>` and extracts:
- All `dbutils.widgets.text(...)` calls → widget names and default values
- All source read cells (`sq_*` variable assignments) → source table names
- All target write cells → target table names and write mode (`append` / `overwrite` / Delta `merge`)
- Whether `.merge(` is present → Update Strategy flag
- Whether `.groupBy().agg(` is present → Aggregator flag

Outputs an Analysis Summary before proceeding:
```
Phase 1 — Analysis Summary
Notebook:        <name>
Widgets found:   <n> (<names>)
Sources:         <n> (<table names>)
Targets:         <n> (<table names, write modes>)
Update Strategy: Yes / No
Aggregator:      Yes / No
```

### Phase 2: Sample Data Mapping

Scans `tests/data/`. For each source table identified in Phase 1, looks for a matching file named `<source_table_name>.csv` or `<source_table_name>.parquet` (case-insensitive match on stem).

- **Matched**: include a load cell in the test notebook
- **Unmatched**: add `# REVIEW: NO SAMPLE DATA — <source_table_name> — place <source_table_name>.csv or .parquet in tests/data/` and list in header checklist; stub the load cell

Outputs a mapping manifest:
```
Phase 2 — Sample Data Mapping
tests/data/ contents:  <filenames>
Matched:   <source_table> → <filename>
Unmatched: <source_table> → NO FILE FOUND — flagged as REVIEW
```

### Phase 3: Test Notebook Creation

Builds the test notebook cell by cell using the SDK pattern in `docs/databricks_notebook_creation.md`. Output path: `notebooks/tests/test_nb_<mapping_name>` (no `.py` extension).

**Cell order:**

1. **Header markdown** — mapping under test, sample data files used, REVIEW checklist (one item per `# REVIEW:` flag)
2. **Widgets cell** — `test_catalog`, `test_schema`, `run_date` (default today's date); these are the only parameters the user must set before running
3. **Imports cell** — only functions used in this notebook
4. **Schema setup cell** — `spark.sql(f"CREATE SCHEMA IF NOT EXISTS {test_catalog}.{test_schema}")`
5. **Sample data load cells** — one cell per matched source table: reads from `tests/data/<file>`, writes as Delta to `{test_catalog}.{test_schema}.<source_table_name>`; unmatched sources get a stubbed comment cell with REVIEW flag
6. **Execute original notebook cell** — `dbutils.notebook.run()` with all widget overrides pointing at `test_catalog`/`test_schema`; asserts return value indicates success
7. **Validation cells** — one cell per target table:
   - Row count assertion (vs expected count if provided, otherwise asserts > 0)
   - Schema check: `sorted(df.columns) == sorted(expected_columns)`
   - Null check: for columns that should not be null based on target definition
   - Numeric precision check: DecimalType columns cast and compared
   - Aggregation totals: if Aggregator flag set, SUM/COUNT assertions on key numeric columns
   - Merge correctness: if Update Strategy flag set, assert no duplicate keys in output
8. **Summary cell** — prints pass/fail per check, collects failures, raises `AssertionError` with full failure list if any check failed
9. **Teardown cell** — wrapped in `try/finally` so it always runs; explicitly `DROP TABLE IF EXISTS {test_catalog}.{test_schema}.<table>` for every table the test notebook created (source tables from step 5 + target tables written by the original notebook in step 6) — **no other tables are dropped**

### Phase 4: Test Cell Review

Walks every cell in the drafted test notebook against this checklist. Corrects issues inline. Then delivers.

- [ ] No hardcoded catalog or schema names — all come from `dbutils.widgets.get(...)`
- [ ] Every source table identified in Phase 1 has a load cell (or a REVIEW stub)
- [ ] Every target table identified in Phase 1 has a validation cell
- [ ] Teardown cell lists exactly the tables created in load cells + target tables — nothing else
- [ ] Every `# REVIEW:` flag in the body is listed in the header markdown checklist
- [ ] `dbutils.notebook.run()` arguments dict contains every widget name extracted in Phase 1

Delivers:
```
Phase 4 — Review Summary
Cells reviewed:  <n>
Issues found:    <n> (<brief description>)   [or "none"]
REVIEW flags:    <n> (<brief description>)   [or "none"]
Status:          Ready
```

Then the notebook asset and a Validation Summary:
```
Notebook tested:   <name>
Test notebook:     notebooks/tests/test_nb_<mapping_name>
Sources mapped:    <n> of <n> (<unmatched listed>)
Targets validated: <n>
Checks included:   row count, schema, nulls, precision [, aggregation] [, merge]
REVIEW Items:
  1. <description>
```

---

## Test Notebook Teardown Scope

The teardown cell drops **only** these tables:
- Source tables written in the sample data load cells (step 5)
- Target tables written by the original notebook during `dbutils.notebook.run()` (step 6)

These are tracked explicitly at the top of the teardown cell as a Python list. The schema itself is NOT dropped — it may have pre-existed. No other tables in `test_catalog.test_schema` are touched.

---

## File Locations

| Artifact | Path |
|---|---|
| Prompt template | `templates/validation_prompt.md` |
| Sample data | `tests/data/<source_table_name>.csv` or `.parquet` |
| Output test notebook | `notebooks/tests/test_nb_<mapping_name>` |
| SDK creation pattern | `docs/databricks_notebook_creation.md` |

---

## What This Prompt Does NOT Do

- Does not re-run the conversion. Run `templates/mega_prompt.md` first.
- Does not create persistent test fixtures or expected-output snapshots. Validation is always derived from the sample data the user provides.
- Does not drop the test schema or any tables outside the explicitly tracked list.
- Does not require a specific Databricks Runtime version beyond what the conversion notebook itself requires.

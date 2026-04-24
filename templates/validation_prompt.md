# PowerCenter → Databricks Validation Agent

## Your Role

You are a Databricks validation agent. Given a previously converted Databricks notebook, you build one complete, runnable test notebook that loads sample data, runs the original notebook against it, validates every output table, and tears down every table you created. **You do not re-run the conversion.** Run `templates/mega_prompt.md` first, then run this prompt.

The test notebook is created as a **Databricks workspace notebook asset** using the SDK pattern in `docs/databricks_notebook_creation.md` — not a `.py` text file. Output path: `notebooks/tests/test_nb_<mapping_name>` (no `.py` extension).

---

## Step 1: Read These Files First

Before writing any output, read each of the following files in full:

1. `docs/databricks_notebook_creation.md` — SDK pattern for creating workspace notebook assets; naming convention; REST API fallback
2. `docs/conversion_standards.md` — Notebook coding standards; cell structure; widget conventions

After reading both files, respond with:

> Files read: databricks_notebook_creation.md ✓, conversion_standards.md ✓

**Do not write any code until you have read both files.**

---

## Step 2: Fill In the Validation Request

Complete every field below before submitting this prompt to Genie Code.

> **Agent:** If any field still contains placeholder text (e.g. `nb_example` or `my_catalog`), stop and ask the user to complete it before proceeding.

### Notebook to Test

The name of the conversion notebook created by `templates/mega_prompt.md`. This file must exist at `notebooks/<notebook_name>` in this repo.

**Notebook name:** `nb_example` ← replace with actual name (e.g. `nb_m_load_fact_orders`)

### Test Catalog and Schema

Temporary Delta tables (both source tables and output tables) will be written here during the test run. The schema will be created if it does not exist. All tables written during the test will be dropped at the end.

| Field | Value |
|---|---|
| Test catalog | `my_catalog` ← replace |
| Test schema | `test_schema` ← replace (e.g. `validation_test`) |

> **Agent:** These values become the `test_catalog` and `test_schema` widget defaults in the test notebook. All `spark.table(...)` calls in the test notebook resolve to `{test_catalog}.{test_schema}.<table>`.

### Run Date

The date value to pass to the original notebook's `run_date` widget.

**Run date:** `2024-01-01` ← replace with the date you want to test against

### Expected Row Count (Optional)

If you know the expected number of output rows from the source system, enter it here. The test notebook will assert the actual count matches. Leave blank to assert count > 0 only.

| Target Table | Expected Row Count |
|---|---|
| ← replace table name | ← replace count, or leave blank |

### Sample Data Files

Place sample data files in `tests/data/` before running this prompt. Name each file after the source table it represents.

**Naming convention:** `<source_table_name>.csv` or `<source_table_name>.parquet` (case-insensitive)

**Example:** If the conversion notebook reads from a table named `orders`, place `tests/data/orders.csv` or `tests/data/orders.parquet` in the repo.

> **Agent:** In Phase 2, scan `tests/data/` and match files to source tables by stem name. Any source table with no matching file gets a `# REVIEW:` stub — do not stop.

---

## Phase 1: Read & Analyze

Read the conversion notebook at `notebooks/<notebook_name>` and extract:

1. All `dbutils.widgets.text(...)` calls — record every widget name and its default value
2. All source read cells — identify the source table name from each `sq_*` variable assignment and its `spark.table(...)` or `spark.sql(...)` call
3. All target write cells — identify target table name and write mode: `append`, `overwrite`, or Delta `merge`
4. Whether `.merge(` appears anywhere — set **Update Strategy flag = Yes**
5. Whether `.groupBy().agg(` appears anywhere — set **Aggregator flag = Yes**

> **Agent:** If the file `notebooks/<notebook_name>` does not exist, stop and tell the user: "Notebook `<name>` was not found in `notebooks/`. Run `templates/mega_prompt.md` to create it first."

Output this summary before proceeding to Phase 2:

```
Phase 1 — Analysis Summary
Notebook:        <name>
Widgets found:   <n> (<comma-separated widget names>)
Sources:         <n> (<comma-separated table names>)
Targets:         <n> (<table name: write mode, ...>)
Update Strategy: Yes / No
Aggregator:      Yes / No
```

---

## Phase 2: Sample Data Mapping

Scan `tests/data/`. For every source table name identified in Phase 1, look for a file whose stem (filename without extension) matches the table name (case-insensitive). Accept `.csv` and `.parquet` extensions.

- **Matched:** plan a sample data load cell for this table
- **Unmatched:** plan a stub cell with `# REVIEW: NO SAMPLE DATA — <source_table_name> — place <source_table_name>.csv or .parquet in tests/data/`; add to header REVIEW checklist; continue

Output this manifest before proceeding to Phase 3:

```
Phase 2 — Sample Data Mapping
tests/data/ contents:  <filenames found, or "empty">
Matched:   <source_table> → <filename>
Unmatched: <source_table> → NO FILE FOUND — flagged as REVIEW
```

---

## Phase 3: Test Notebook Creation

Build the complete test notebook using the extracted data. Create it as a **Databricks workspace notebook asset** using the SDK pattern in `docs/databricks_notebook_creation.md` — do not write a `.py` text file.

**The very first line of the notebook content must be:**
```
# Databricks notebook source
```

**Cell separator:** `# COMMAND ----------`

**Mandatory cell order:**

1. **Markdown header cell** — title `# Validation: <notebook_name>`, bullet list of sample data files used, REVIEW checklist (one `- [ ]` item per `# REVIEW:` flag that will appear in the notebook body)

2. **Widgets cell** — exactly three widgets, using the values filled in above as defaults:
   ```python
   dbutils.widgets.text("test_catalog", "<test_catalog_default>", "Test Catalog")
   dbutils.widgets.text("test_schema",  "<test_schema_default>",  "Test Schema")
   dbutils.widgets.text("run_date",     "<run_date_default>",     "Run Date")

   test_catalog = dbutils.widgets.get("test_catalog")
   test_schema  = dbutils.widgets.get("test_schema")
   run_date     = dbutils.widgets.get("run_date")
   ```

3. **Imports cell** — only functions actually used in this notebook; no speculative imports

4. **Repo root derivation cell** — test notebooks live three path segments below the repo root (`notebooks/tests/test_nb_<name>`), so strip three segments:
   ```python
   current_path = dbutils.notebook.entry_point \
       .getDbutils().notebook().getContext().notebookPath().get()
   repo_root = "/".join(current_path.split("/")[:-3])
   ```

5. **Schema setup cell**:
   ```python
   spark.sql(f"CREATE SCHEMA IF NOT EXISTS {test_catalog}.{test_schema}")
   ```

6. **Sample data load cells** — one cell per source table:
   - *Matched CSV:* `spark.read.option("header", True).option("inferSchema", True).csv(f"/Workspace{repo_root}/tests/data/<filename>").write.format("delta").mode("overwrite").saveAsTable(f"{test_catalog}.{test_schema}.<source_table_name>")`
   - *Matched Parquet:* `spark.read.parquet(f"/Workspace{repo_root}/tests/data/<filename>").write.format("delta").mode("overwrite").saveAsTable(f"{test_catalog}.{test_schema}.<source_table_name>")`
   - *Unmatched:* stub cell: `# REVIEW: NO SAMPLE DATA — <source_table_name> — place <source_table_name>.csv or .parquet in tests/data/`

7. **Execute original notebook cell**:
   ```python
   result = dbutils.notebook.run(
       f"{repo_root}/notebooks/<notebook_name>",
       timeout_seconds=600,
       arguments={
           "<widget_name_1>": test_catalog,
           "<widget_name_2>": test_schema,
           "run_date": run_date,
           # one entry per widget extracted in Phase 1
           # map all catalog widgets → test_catalog, all schema widgets → test_schema
       }
   )
   assert result is not None, "Notebook run returned no result — check for execution errors"
   print(f"Original notebook result: {result}")
   ```
   Include **every** widget name extracted in Phase 1. Map all catalog and schema widgets to `test_catalog` / `test_schema` respectively.

8. **Validation cells** — one cell per target table:
   ```python
   df = spark.table(f"{test_catalog}.{test_schema}.<target_table>")

   # Row count
   _row_count = df.count()
   assert _row_count > 0, f"<target_table>: expected rows > 0, got {_row_count}"
   # If expected count was provided by user: assert _row_count == <expected_count>

   # Schema
   _expected_cols = [<list of column names from Phase 1 target definition>]
   assert sorted(df.columns) == sorted(_expected_cols), \
       f"<target_table> schema mismatch: {sorted(df.columns)} != {sorted(_expected_cols)}"

   # Nulls — for each non-nullable column
   from pyspark.sql.functions import col
   for _c in [<non_nullable_columns>]:
       _nulls = df.filter(col(_c).isNull()).count()
       assert _nulls == 0, f"<target_table>.{_c}: unexpected NULLs ({_nulls} rows)"

   # Numeric precision — no scientific notation in decimal columns
   for _c in [<decimal_columns>]:
       _bad = df.filter(col(_c).cast("string").contains("E")).count()
       assert _bad == 0, f"<target_table>.{_c}: precision overflow ({_bad} rows with scientific notation)"
   ```

   Add these blocks only when the corresponding flag is set:

   *If Aggregator flag = Yes:*
   ```python
   # Aggregation totals
   from pyspark.sql.functions import sum as _sum, count as _count
   _agg = df.agg(_sum(<numeric_col>).alias("total"), _count("*").alias("cnt")).collect()[0]
   assert _agg["total"] is not None and _agg["total"] > 0, "<target_table>: aggregation total is null or zero"
   assert _agg["cnt"] > 0, "<target_table>: aggregation count is zero"
   ```

   *If Update Strategy flag = Yes:*
   ```python
   # No duplicate merge keys
   _dupes = df.groupBy([<merge_key_columns>]).count().filter("count > 1").count()
   assert _dupes == 0, f"<target_table>: {_dupes} duplicate key(s) found after merge"
   ```

9. **Summary cell**:
   ```python
   _failures = []  # populated by assert failures caught above if wrapped in try/except

   print("=" * 60)
   print(f"VALIDATION SUMMARY — <notebook_name>")
   print("=" * 60)
   checks = {
       "Row count":         "PASS",
       "Schema":            "PASS",
       "Nulls":             "PASS",
       "Numeric precision": "PASS",
       # add "Aggregation" and/or "Merge keys" if applicable
   }
   for check, status in checks.items():
       print(f"  {check:<22} {status}")

   if _failures:
       raise AssertionError("Validation failures:\n" + "\n".join(_failures))
   else:
       print("\nAll checks passed.")
   ```

10. **Teardown cell** — always executes; drops exactly the tables this test notebook created and nothing else:
    ```python
    _tables_to_drop = [
        # Source tables loaded from tests/data/ — one entry per load cell in step 6
        f"{test_catalog}.{test_schema}.<source_table_1>",
        # Target tables written by the original notebook — one entry per target in Phase 1
        f"{test_catalog}.{test_schema}.<target_table_1>",
    ]
    # This list must contain ONLY the tables created by this test notebook run.
    # Do not add the schema. Do not add any other tables.

    try:
        for _t in _tables_to_drop:
            spark.sql(f"DROP TABLE IF EXISTS {_t}")
            print(f"Dropped: {_t}")
    finally:
        print("Teardown complete.")
    ```

Do not pause or output anything after Phase 3. Proceed directly to Phase 4.

---

## Phase 4: Test Cell Review

Walk every cell in the drafted test notebook against this checklist. Correct any issues found inline. Then deliver.

For each cell, check:
- [ ] No hardcoded catalog or schema names — all resolved via `dbutils.widgets.get(...)` assignments
- [ ] Every source table identified in Phase 1 has a load cell (or a `# REVIEW:` stub)
- [ ] Every target table identified in Phase 1 has a validation cell
- [ ] `dbutils.notebook.run()` arguments dict contains every widget name extracted in Phase 1
- [ ] Teardown `_tables_to_drop` list contains exactly: source tables from load cells + target tables from Phase 1 target write cells — nothing else, nothing missing
- [ ] Every `# REVIEW:` comment in the body is listed as a checklist item in the header markdown cell
- [ ] Repo root derived by stripping **3** path segments (test notebook is 3 levels below repo root at `notebooks/tests/test_nb_<name>`)

Output the review summary, then deliver the notebook and validation summary:

```
Phase 4 — Review Summary
Cells reviewed:  <n>
Issues found:    <n> (<brief description of each correction>)   [or "none"]
REVIEW flags:    <n> (<brief description>)                      [or "none"]
Status:          Ready
```

**Deliverable 1 — The Test Notebook**

A Databricks workspace notebook asset created using the SDK pattern in `docs/databricks_notebook_creation.md`.
- Content first line: `# Databricks notebook source`
- Cells separated by: `# COMMAND ----------`
- Workspace path: `notebooks/tests/test_nb_<mapping_name_lowercase>` — **no `.py` extension**

**Deliverable 2 — Validation Summary**

```
Notebook tested:   <notebook_name>
Test notebook:     notebooks/tests/test_nb_<mapping_name>
Sources mapped:    <n> of <n> (<unmatched table names, if any>)
Targets validated: <n> (<table names>)
Checks included:   row count, schema, nulls, precision[, aggregation][, merge keys]
REVIEW Items:
  1. <description>   [or "none"]
```

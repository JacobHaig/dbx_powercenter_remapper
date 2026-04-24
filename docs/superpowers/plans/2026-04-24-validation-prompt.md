# Validation Prompt Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create `templates/validation_prompt.md` — a Genie Code prompt that teaches the Databricks agent to build a validation test notebook for a previously converted PowerCenter notebook.

**Architecture:** Single documentation file following the same 4-phase structure as `templates/mega_prompt.md`. The agent it describes uses `dbutils.notebook.run()` to execute the original conversion notebook against sample Delta tables loaded from `tests/data/`, then validates output tables and tears down exactly the tables it created. A second small change sets up the `tests/data/` landing folder and gitignores sample data files.

**Tech Stack:** Markdown only — no runtime code. Databricks Genie Code consumes the prompt. All code snippets in the prompt are PySpark / Databricks SDK / Python.

---

### Task 1: Create `tests/data/` landing folder and update `.gitignore`

**Files:**
- Create: `tests/data/.gitkeep`
- Modify: `.gitignore`

- [ ] **Step 1: Create the `tests/data/` directory with a `.gitkeep`**

```bash
mkdir -p tests/data && touch tests/data/.gitkeep
```

Expected: `tests/data/.gitkeep` exists, directory is tracked by git.

- [ ] **Step 2: Add sample data patterns to `.gitignore`**

Open `.gitignore`. Current contents:
```
input/*.xml
/memory
```

Add these lines so sample data files are not committed:
```
tests/data/*.csv
tests/data/*.parquet
```

Final `.gitignore`:
```
input/*.xml
tests/data/*.csv
tests/data/*.parquet
/memory
```

- [ ] **Step 3: Verify**

Run:
```bash
git status
```

Expected output includes:
- `tests/data/.gitkeep` as a new untracked file (to be staged)
- `.gitignore` as modified

---

### Task 2: Write `templates/validation_prompt.md`

**Files:**
- Create: `templates/validation_prompt.md`

This is the primary deliverable. The file must follow the same structural conventions as `templates/mega_prompt.md` — agent role at the top, a user fill-in section, then four numbered phases with explicit output formats.

- [ ] **Step 1: Create `templates/validation_prompt.md` with the full content below**

Write exactly this content to `templates/validation_prompt.md`:

````markdown
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

2. **Widgets cell** — exactly three widgets:
   ```python
   dbutils.widgets.text("test_catalog", "<test_catalog_default>", "Test Catalog")
   dbutils.widgets.text("test_schema", "<test_schema_default>", "Test Schema")
   dbutils.widgets.text("run_date", "<run_date_default>", "Run Date")
   ```
   Then assign variables:
   ```python
   test_catalog = dbutils.widgets.get("test_catalog")
   test_schema  = dbutils.widgets.get("test_schema")
   run_date     = dbutils.widgets.get("run_date")
   ```

3. **Imports cell** — only functions actually used in this notebook

4. **Repo root derivation cell** — derive `repo_root` using the same pattern as `docs/databricks_notebook_creation.md`:
   ```python
   current_path = dbutils.notebook.entry_point \
       .getDbutils().notebook().getContext().notebookPath().get()
   repo_root = "/".join(current_path.split("/")[:-3])
   ```
   Note: test notebooks live at `notebooks/tests/test_nb_<name>` — three segments below repo root, so strip three segments (not two).

5. **Schema setup cell**:
   ```python
   spark.sql(f"CREATE SCHEMA IF NOT EXISTS {test_catalog}.{test_schema}")
   ```

6. **Sample data load cells** — one cell per source table:
   - *Matched:* read from `tests/data/<filename>` using the appropriate reader (`.csv` with `header=True, inferSchema=True`; `.parquet` with no options), write as Delta to `{test_catalog}.{test_schema}.<source_table_name>` with `mode("overwrite")`
   - *Unmatched:* stub cell with `# REVIEW: NO SAMPLE DATA — <source_table_name>`

7. **Execute original notebook cell**:
   ```python
   result = dbutils.notebook.run(
       f"{repo_root}/notebooks/<notebook_name>",
       timeout_seconds=600,
       arguments={
           "<widget_name_1>": test_catalog,
           "<widget_name_2>": test_schema,
           "run_date": run_date,
           # ... one key per widget extracted in Phase 1
       }
   )
   assert result is not None, "Notebook run returned no result — check for execution errors"
   print(f"Original notebook result: {result}")
   ```
   Include every widget name extracted in Phase 1 in the `arguments` dict. Map all catalog and schema widgets to `test_catalog` / `test_schema`. Map `run_date` to `run_date`.

8. **Validation cells** — one cell per target table. Each cell:
   - Reads the target table: `df = spark.table(f"{test_catalog}.{test_schema}.<target_table>")`
   - **Row count assertion:** if expected count was provided, `assert df.count() == <expected>`, otherwise `assert df.count() > 0`
   - **Schema check:** `assert sorted(df.columns) == sorted(<expected_columns_list>)` where `<expected_columns_list>` is the list of column names extracted from the target write cell in Phase 1
   - **Null check:** for each column that appeared as non-nullable in the original notebook's target definition, `assert df.filter(col("<col>").isNull()).count() == 0`
   - **Numeric precision check:** for each DecimalType column, `assert df.filter(col("<col>").cast("string").contains("E")).count() == 0` (no scientific notation — indicates precision overflow)
   - **Aggregation check** (only if Aggregator flag = Yes): assert SUM and COUNT of key numeric columns are > 0
   - **Merge correctness check** (only if Update Strategy flag = Yes): assert no duplicate primary keys — `assert df.groupBy(<merge_key_cols>).count().filter("count > 1").count() == 0`

9. **Summary cell**:
   ```python
   print("=" * 60)
   print(f"VALIDATION SUMMARY — {notebook_name}")
   print("=" * 60)
   print(f"  Row count:          PASS")
   print(f"  Schema:             PASS")
   print(f"  Nulls:              PASS")
   print(f"  Numeric precision:  PASS")
   # Add/remove lines to match checks actually run
   print("All checks passed.")
   ```
   Collect assertion failures into a list throughout the validation cells and raise at the end:
   ```python
   if failures:
       raise AssertionError("Validation failures:\n" + "\n".join(failures))
   ```

10. **Teardown cell** — tracks and drops only the tables this test notebook created. Place at the very end, wrapped so it always runs:
    ```python
    _tables_to_drop = [
        # Source tables loaded from tests/data/
        f"{test_catalog}.{test_schema}.<source_table_1>",
        f"{test_catalog}.{test_schema}.<source_table_2>",
        # Target tables written by the original notebook
        f"{test_catalog}.{test_schema}.<target_table_1>",
    ]
    # Replace the above list with the actual source and target table names
    # identified in Phases 1 and 2 — nothing else goes in this list.

    for _t in _tables_to_drop:
        spark.sql(f"DROP TABLE IF EXISTS {_t}")
        print(f"Dropped: {_t}")
    print("Teardown complete.")
    ```
    **Do not drop the schema. Do not add any table not in the explicit list above.**

Do not pause or output anything after Phase 3. Proceed directly to Phase 4.

---

## Phase 4: Test Cell Review

Walk every cell in the drafted test notebook against this checklist. Correct any issues found inline. Then deliver.

For each cell, check:
- [ ] No hardcoded catalog or schema names — all resolved via `dbutils.widgets.get(...)`
- [ ] Every source table identified in Phase 1 has a load cell (or a `# REVIEW:` stub)
- [ ] Every target table identified in Phase 1 has a validation cell
- [ ] `dbutils.notebook.run()` arguments dict contains every widget name extracted in Phase 1
- [ ] Teardown cell `_tables_to_drop` list contains exactly: source tables from load cells + target tables from Phase 1 — nothing else, nothing missing
- [ ] Every `# REVIEW:` comment in the body is listed as a checklist item in the header markdown cell
- [ ] Repo root derived by stripping 3 path segments (test notebook is 3 levels below repo root)

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
Sources mapped:    <n> of <n> (<unmatched table names if any>)
Targets validated: <n> (<table names>)
Checks included:   row count, schema, nulls, precision[, aggregation][, merge]
REVIEW Items:
  1. <description>   [or "none"]
```
````

- [ ] **Step 2: Verify the file was created**

```bash
wc -l templates/validation_prompt.md
```

Expected: line count above 150 (the file is substantial).

```bash
grep -c "Phase" templates/validation_prompt.md
```

Expected: 8 or more (each phase heading appears multiple times across role, section headers, and output blocks).

- [ ] **Step 3: Check all four phases are present**

```bash
grep "^## Phase" templates/validation_prompt.md
```

Expected output:
```
## Phase 1: Read & Analyze
## Phase 2: Sample Data Mapping
## Phase 3: Test Notebook Creation
## Phase 4: Test Cell Review
```

---

### Task 3: Update `README.md` quick navigation table

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Read the Quick Navigation table in `README.md`**

Open `README.md` and find the `## Quick Navigation` section. Current table ends with `tests/framework.md`.

- [ ] **Step 2: Add a row for `validation_prompt.md`**

Insert this row into the Quick Navigation table immediately after the `mega_prompt.md` row:

```markdown
| [templates/validation_prompt.md](templates/validation_prompt.md) | **Genie Code entry point** — run after conversion to build and execute a validation test notebook |
```

- [ ] **Step 3: Verify**

```bash
grep "validation_prompt" README.md
```

Expected: one line containing the link and description.

---

## Self-Review Checklist

**Spec coverage:**

| Spec requirement | Covered by |
|---|---|
| `templates/validation_prompt.md` created | Task 2 |
| Agent role defined | Task 2, Step 1 — "Your Role" section |
| Read-these-files-first section | Task 2, Step 1 — "Step 1: Read These Files First" |
| User fill-in section (notebook name, test catalog, schema, run date, expected row count, sample data) | Task 2, Step 1 — "Step 2: Fill In the Validation Request" |
| Phase 1 Analysis Summary output format | Task 2, Step 1 — Phase 1 section |
| Phase 2 sample data mapping with REVIEW stubs for unmatched | Task 2, Step 1 — Phase 2 section |
| Phase 3 cell order (header → widgets → imports → repo root → schema setup → load → execute → validate → summary → teardown) | Task 2, Step 1 — Phase 3 section |
| dbutils.notebook.run() with full widget override dict | Task 2, Step 1 — Phase 3, cell 7 |
| Validation checks: row count, schema, nulls, precision, aggregation (conditional), merge (conditional) | Task 2, Step 1 — Phase 3, cell 8 |
| Summary cell with failure collection | Task 2, Step 1 — Phase 3, cell 9 |
| Teardown: exact tables only, no schema drop, try/finally concept | Task 2, Step 1 — Phase 3, cell 10 |
| Phase 4 review checklist | Task 2, Step 1 — Phase 4 section |
| Deliverable 1: notebook asset (no .py extension) | Task 2, Step 1 — Phase 4, Deliverable 1 |
| Deliverable 2: Validation Summary | Task 2, Step 1 — Phase 4, Deliverable 2 |
| `tests/data/` folder created | Task 1 |
| `.gitignore` updated for sample data files | Task 1 |
| README quick nav updated | Task 3 |

No gaps found.

**Placeholder scan:** No TBD, TODO, or "implement later" text in the plan body. Code blocks in Task 2 contain the actual file content to write.

**Type consistency:** No cross-task type references — this is documentation only.

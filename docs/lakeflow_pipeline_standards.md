# Lakeflow Spark Declarative Pipeline Standards

When the user selects **pipeline** as the output format, the agent produces a Lakeflow Spark Declarative Pipeline notebook instead of a standard PySpark notebook. These standards govern that output.

> **Background:** Databricks rebranded Delta Live Tables (DLT) as **Lakeflow Spark Declarative Pipelines** at DAIS 2025 (GA June 12, 2025). Old `import dlt` / `@dlt.table` syntax is still accepted but deprecated — always emit the new `pyspark.pipelines` syntax.

---

## What Changes vs. a Regular Notebook

| Aspect | Regular Notebook | Lakeflow Pipeline Notebook |
|---|---|---|
| Import | `from pyspark.sql.functions import ...` | `from pyspark import pipelines as dp` |
| Dataset definition | `df = spark.table(...).withColumn(...)` | `@dp.table` decorated function returning a DataFrame |
| Reading pipeline datasets | Direct DataFrame reference | `dp.read("dataset_name")` |
| Writing targets | `.write.format("delta").saveAsTable(...)` | Implicit — `@dp.table` persists automatically |
| Update Strategy / CDC | `DeltaTable.merge(...)` | `dp.create_auto_cdc_flow(...)` |
| Environment params | `dbutils.widgets` | Pipeline parameters (`spark.conf.get("pipeline.param.<name>")`) |
| Data quality | Manual `assert` | `@dp.expect(...)` decorator |
| Truncate-reload target | `mode("overwrite")` | `@dp.materialized_view` (fully refreshes each run) |

---

## Pipeline Notebook Source Format

The notebook content is still created as a Databricks workspace notebook asset using the same SDK pattern in `docs/databricks_notebook_creation.md`. The content structure is:

```
# Databricks notebook source

# COMMAND ----------
# MAGIC %md
# MAGIC ## Header

# COMMAND ----------
# imports and pipeline dataset functions
```

---

## Required Cell Structure (in order)

### Cell 1 — Markdown Header

Same as regular notebook: mapping name, source/target, REVIEW checklist. Add a note that this is a pipeline notebook.

```python
# MAGIC %md
# MAGIC # Pipeline: <Mapping Name>
# MAGIC
# MAGIC **Output type:** Lakeflow Spark Declarative Pipeline
# MAGIC **Source System:** ...
# MAGIC **Target Table:** `catalog.schema.fact_orders`
# MAGIC **Original Mapping:** `FOLDER.M_EXAMPLE`
# MAGIC
# MAGIC ## Review Items
# MAGIC - [ ] REVIEW: verify pipeline parameter names match the pipeline configuration
```

### Cell 2 — Imports

No `dbutils.widgets` in a pipeline notebook. Import the pipelines module and all needed Spark functions.

```python
from pyspark import pipelines as dp
from pyspark.sql.functions import (
    col, lit, when, coalesce, concat, concat_ws,
    upper, lower, trim, to_date, to_timestamp, date_format,
    current_timestamp, datediff, date_add, sum, count, avg,
    min, max, round, floor, ceil, abs, broadcast,
    row_number, rank, dense_rank, asc_nulls_last, desc_nulls_first
)
from pyspark.sql.window import Window
from pyspark.sql.types import (
    StructType, StructField,
    StringType, IntegerType, LongType, DoubleType,
    DecimalType, DateType, TimestampType
)
```

Only import what is actually used.

### Cell 3 — Pipeline Parameters

Pipeline parameters replace `dbutils.widgets`. They are defined in the pipeline configuration, not in the notebook. Read them at module level with `spark.conf.get`:

```python
catalog = spark.conf.get("pipeline.param.catalog")
schema  = spark.conf.get("pipeline.param.schema")
# REVIEW: configure pipeline parameters — declare all pipeline.param.* keys above in the pipeline UI or SDK configuration
```

These module-level variables are available inside every `@dp.table` function closure because module code runs at notebook import time.

### Cells 4–N — Source Datasets

One cell per Source Qualifier. Each is a `@dp.table` decorated function.

**No SQL override — read full table:**
```python
# SQ_ORDERS
@dp.table(name="sq_orders")
def sq_orders():
    return spark.table(f"{catalog}.{schema}.orders")
```

**With Source Filter (`Source Filter` attribute on SQ):**
```python
# SQ_ORDERS — Source Filter: status IN ('ACTIVE','PENDING')
@dp.table(name="sq_orders")
def sq_orders():
    return (
        spark.table(f"{catalog}.{schema}.orders")
        .filter(col("status").isin(["ACTIVE", "PENDING"]))
    )
```

**With SQL override (`Sql Query` attribute on SQ):**
```python
# SQ_ORDERS — Sql Query override
@dp.table(name="sq_orders")
def sq_orders():
    return spark.sql(f"""
        SELECT order_id, customer_id, amount
        FROM {catalog}.{schema}.orders
        WHERE status = 'ACTIVE'
    """)
```

**With User Defined Join (multiple sources joined inside SQ):**
```python
# SQ_ORDERS_CUSTOMERS — User Defined Join
@dp.table(name="sq_orders_customers")
def sq_orders_customers():
    return spark.sql(f"""
        SELECT o.order_id, c.customer_name, o.amount
        FROM {catalog}.{schema}.orders o
        JOIN {catalog}.{schema}.customers c ON o.customer_id = c.customer_id
    """)
```

### Cells N+1 through M — Transformation Datasets

One cell per logical transformation group, in topological DAG order. Each function reads upstream datasets with `dp.read("name")`.

**Expression — `.withColumn()` chain:**
```python
# EXP_CALCULATE_AMOUNTS
@dp.table(name="df_amounts")
def df_amounts():
    orders = dp.read("sq_orders")
    return (
        orders
        .withColumn("tax_amount", round(col("subtotal") * lit(0.08), 2))
        .withColumn("total_amount", col("subtotal") + col("tax_amount"))
        .withColumn("is_large_order", when(col("total_amount") > 1000, lit(1)).otherwise(lit(0)))
    )
```

**VARIABLE ports** — prefix `_v_`, drop after use (same rule as regular notebooks):
```python
# EXP_WITH_VARIABLE_PORT
@dp.table(name="df_with_rate")
def df_with_rate():
    return (
        dp.read("sq_orders")
        .withColumn("_v_tax_rate", lit(0.08))
        .withColumn("tax_amount", col("amount") * col("_v_tax_rate"))
        .drop("_v_tax_rate")    # VARIABLE port — drop before passing downstream
    )
```

Use `@dp.temporary_view` for intermediate datasets that should not be persisted as Delta tables (e.g., internal filter branches):

```python
@dp.temporary_view(name="vw_filtered")
def vw_filtered():
    return dp.read("df_amounts").filter(col("total_amount") > 0)
```

Use `@dp.materialized_view` for aggregations (Aggregator transformation) — fully refreshes on every pipeline run, which matches PowerCenter batch semantics:

```python
# AGG_DAILY_TOTALS
@dp.materialized_view(name="mv_daily_totals")
def mv_daily_totals():
    return (
        dp.read("df_amounts")
        .groupBy("order_date")
        .agg(sum("total_amount").alias("daily_total"), count("order_id").alias("order_count"))
    )
```

### Final Cells — Target Datasets

One cell per target, in `<TARGETLOADORDER>` sequence.

**Append-only target (no Update Strategy upstream):**
```python
# TGT_FACT_ORDERS — append
@dp.table(name="fact_orders", table_properties={"quality": "gold"})
def fact_orders():
    return dp.read("df_amounts")
```

**Truncate-reload target (`Truncate Target Table = true` in PowerCenter):**

Use `@dp.materialized_view`, which fully recomputes and replaces the dataset on every pipeline run — equivalent to `mode("overwrite")` in a regular notebook:
```python
# TGT_DIM_PRODUCT — truncate-reload
@dp.materialized_view(name="dim_product")
def dim_product():
    return dp.read("df_product_transformed")
```

**CDC / Update Strategy target:**

When the PowerCenter mapping contains an Update Strategy transformation, use `dp.create_auto_cdc_flow`. The `source` DataFrame must have DD_REJECT rows (flag value 3) filtered out upstream before this call.

```python
# TGT_FACT_ORDERS — CDC via Update Strategy
# DD_REJECT rows (dd_flag == 3) must be filtered from df_amounts before reaching here.
dp.create_auto_cdc_flow(
    name="fact_orders",
    source="df_amounts_no_reject",  # upstream dataset with DD_REJECT already removed
    keys=["order_id"],              # merge key columns from ### Merge Keys in the prompt
    sequence_by=col("run_date"),    # REVIEW: choose a monotonically increasing column
                                    # (timestamp, batch sequence, etc.) — no equivalent in PC
    apply_as_deletes=col("dd_flag") == 2,   # DD_DELETE = 2
    stored_as_scd_type=1            # REVIEW: confirm SCD type (1 = overwrite, 2 = history rows)
)
```

**Filtering DD_REJECT before CDC** — add a `@dp.temporary_view` immediately before the CDC call:
```python
# Filter out DD_REJECT rows before CDC flow
@dp.temporary_view(name="df_amounts_no_reject")
def df_amounts_no_reject():
    return dp.read("df_amounts").filter(col("dd_flag") != 3)
```

If merge keys are unknown, stub as:
```python
# REVIEW: MERGE KEY UNKNOWN — fact_orders — specify keys= list
dp.create_auto_cdc_flow(
    name="fact_orders",
    source="df_amounts_no_reject",
    keys=["<KEY>"],                 # REVIEW: replace with actual key columns
    sequence_by=col("run_date"),    # REVIEW: replace with appropriate sequence column
    stored_as_scd_type=1
)
```

---

## Data Quality

Add `@dp.expect` decorators above `@dp.table` / `@dp.materialized_view` for quality constraints derived from the mapping. Decorator order matters — place quality decorators above the dataset decorator:

```python
@dp.expect("order_id not null", "order_id IS NOT NULL")
@dp.expect("positive amount", "total_amount > 0")
@dp.table(name="fact_orders")
def fact_orders():
    return dp.read("df_amounts_no_reject")
```

| Decorator | Behavior on violation |
|---|---|
| `@dp.expect(name, sql)` | Log the failure, keep the row |
| `@dp.expect_or_drop(name, sql)` | Silently drop failing rows |
| `@dp.expect_or_fail(name, sql)` | Halt the pipeline run |

Use `@dp.expect_or_drop` as the pipeline equivalent of a PowerCenter constraint that rejects rows without failing the session.

---

## Naming Conventions

Same as regular notebooks with these additions:

| Object | Convention | Example |
|---|---|---|
| Source pipeline datasets | `sq_<source_name>` | `sq_orders` |
| Intermediate pipeline datasets | `df_<transformation_name>` | `df_amounts` |
| Temporary views | `vw_<name>` | `vw_filtered` |
| Materialized views | `mv_<name>` | `mv_daily_totals` |
| Target tables | Target table name as-is (lowercase) | `fact_orders` |
| VARIABLE ports (intra-function) | `_v_<name>` prefix, dropped after use | `_v_tax_rate` |

---

## PowerCenter → Lakeflow Mapping

| PowerCenter Transformation | Lakeflow Equivalent | Notes |
|---|---|---|
| Source Qualifier (no override) | `@dp.table` → `spark.table(...)` | |
| Source Qualifier (SQL override) | `@dp.table` → `spark.sql(...)` | Inline the full SQL |
| Source Qualifier (Source Filter) | `@dp.table` → `spark.table(...).filter(...)` | |
| Source Qualifier (User Defined Join) | `@dp.table` → `spark.sql(...)` with JOIN | |
| Expression | `.withColumn()` chain inside `@dp.table` | `_v_` VARIABLE ports → drop after use |
| Filter | `.filter()` inside `@dp.table` or `@dp.temporary_view` | |
| Joiner | `.join()` inside `@dp.table` — read both sides with `dp.read(...)` | |
| Lookup (connected) | `broadcast()` join inside `@dp.table` | |
| Lookup (unconnected) | Pre-join `broadcast()` before the Expression `@dp.table` | `:LKP.name(key)` → column via join |
| Aggregator | `.groupBy().agg()` inside `@dp.materialized_view` | Fully refreshes each run |
| Router | Multiple `@dp.temporary_view` functions, each with `.filter()` | Default group = logical complement of all named conditions |
| Sorter | `.orderBy()` **within** the consuming `@dp.table` function | Sort is not preserved across materialized datasets — see note below |
| Rank | `Window.rank()` + `.filter()` inside `@dp.table` | |
| Union | `.unionByName()` inside `@dp.table` | |
| Update Strategy | `dp.create_auto_cdc_flow(...)` on the final target | Filter DD_REJECT (flag=3) upstream first |
| Sequence Generator | `monotonically_increasing_id()` inside `@dp.table` — flag as `# REVIEW:` | Not sequential across runs; see note below |
| Normalizer | `expr("stack(...)")` inside `@dp.table` | |
| Mapplet (simple) | Inline Python helper function called inside a `@dp.table` | Same as regular notebook — extract to function, call inside the dataset function |
| Mapplet (complex) | Separate pipeline notebook at `notebooks/mapplets/mlt_<name>` | Reference via `%run` |
| Stored / External Procedure | `# REVIEW: STORED PROCEDURE` — stub, flag for manual implementation | |

### Sorter Note

Delta tables do not guarantee physical row order. A Sorter that exists solely to order rows before a subsequent Rank or Window transformation should be **collapsed into the same `@dp.table` function** as the downstream transformation — not materialized as its own dataset. Only create a separate `@dp.table` for a Sorter if it feeds a target that genuinely requires ordered output (rare).

### Sequence Generator Note

`monotonically_increasing_id()` is non-deterministic across runs and is not equivalent to a PowerCenter stateful sequence. Always add `# REVIEW: SEQUENCE GENERATOR — not sequential across runs; consider Delta IDENTITY column if a stable sequence is required`.

---

## Pre-Delivery Checklist (Pipeline)

- [ ] All source datasets use `@dp.table` with `spark.table(...)` or `spark.sql(...)` — no hardcoded catalog/schema strings (use module-level pipeline parameter variables)
- [ ] All intermediate transformations read upstream datasets with `dp.read("name")` — no direct cross-function DataFrame references
- [ ] All target tables use `@dp.table`, `@dp.materialized_view`, or `dp.create_auto_cdc_flow` — no `.write.format("delta")` calls
- [ ] Truncate-reload targets use `@dp.materialized_view`, not `@dp.table`
- [ ] VARIABLE ports (`_v_` prefix) are dropped within the same function before the return
- [ ] Every Update Strategy target has a `@dp.temporary_view` upstream that filters out DD_REJECT rows (`dd_flag != 3`) before `dp.create_auto_cdc_flow`
- [ ] `dp.create_auto_cdc_flow` has a `# REVIEW:` on both `sequence_by` and `stored_as_scd_type`
- [ ] Router default group condition is the logical complement of all named group conditions
- [ ] Sorters that only exist to feed a downstream Rank/Window are collapsed into the consuming function, not materialized separately
- [ ] Environment values come from `spark.conf.get("pipeline.param.*")`, not hardcoded
- [ ] A `# REVIEW: configure pipeline parameters` comment in Cell 3 lists every `pipeline.param.*` key the pipeline configuration must declare
- [ ] `@dp.expect` added for any data quality rules derived from the mapping
- [ ] No `dbutils.widgets` calls — not available in pipeline execution context
- [ ] `# REVIEW:` items appear both inline and in the header cell checklist

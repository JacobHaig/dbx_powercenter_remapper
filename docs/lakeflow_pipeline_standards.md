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
    row_number, rank, dense_rank
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

Pipeline parameters replace `dbutils.widgets`. They are defined in the pipeline configuration, not in the notebook. Read them with `spark.conf.get`:

```python
catalog = spark.conf.get("pipeline.param.catalog")
schema  = spark.conf.get("pipeline.param.schema")
```

Add a `# REVIEW:` comment listing every parameter that must be declared in the pipeline UI/SDK configuration.

### Cells 4–N — Source Datasets

One cell per Source Qualifier. Each is a `@dp.table` decorated function named with the `sq_` prefix.

```python
# SQ_ORDERS
@dp.table(name="sq_orders")
def sq_orders():
    return (
        spark.table(f"{catalog}.{schema}.orders")
        .filter(col("status").isin(["ACTIVE", "PENDING"]))
    )
```

### Cells N+1 through M — Transformation Datasets

One cell per logical transformation group, in topological DAG order. Each is a `@dp.table` function reading upstream datasets via `dp.read("name")`.

```python
# EXP_CALCULATE_AMOUNTS
@dp.table(name="df_amounts")
def df_amounts():
    orders = dp.read("sq_orders")
    return (
        orders
        .withColumn("tax_amount", round(col("subtotal") * lit(0.08), 2))
        .withColumn("total_amount", col("subtotal") + col("tax_amount"))
    )
```

Use `@dp.temporary_view` for intermediate datasets that should not be persisted as Delta tables:

```python
@dp.temporary_view(name="vw_filtered")
def vw_filtered():
    return dp.read("df_amounts").filter(col("total_amount") > 0)
```

Use `@dp.materialized_view` for aggregations or joins where incremental refresh is not needed:

```python
@dp.materialized_view(name="mv_daily_totals")
def mv_daily_totals():
    return dp.read("df_amounts").groupBy("order_date").agg(sum("total_amount").alias("daily_total"))
```

### Final Cells — Target Datasets

One cell per target. The function name and `name=` parameter match the target table name.

**Append / overwrite target:**

```python
# TGT_FACT_ORDERS — append-only
@dp.table(name="fact_orders", table_properties={"quality": "gold"})
def fact_orders():
    return dp.read("df_amounts")
```

**CDC / Update Strategy target:**

When the PowerCenter mapping contains an Update Strategy transformation, use `dp.create_auto_cdc_flow` instead of a decorated function:

```python
# TGT_FACT_ORDERS — CDC via Update Strategy
dp.create_auto_cdc_flow(
    name="fact_orders",
    source="df_amounts",           # upstream pipeline dataset name
    keys=["order_id"],             # merge key columns
    sequence_by=col("run_date"),   # sequence column to resolve conflicts
    apply_as_deletes=col("dd_flag") == 2,
    stored_as_scd_type=1           # SCD Type 1 = last-write-wins
)
```

If merge keys are unknown, stub as:
```python
# REVIEW: MERGE KEY UNKNOWN — fact_orders — specify keys= list
dp.create_auto_cdc_flow(
    name="fact_orders",
    source="df_amounts",
    keys=["<KEY>"],                # REVIEW: replace with actual key columns
    sequence_by=col("run_date"),
    stored_as_scd_type=1
)
```

---

## Data Quality

Add `@dp.expect` decorators above `@dp.table` / `@dp.materialized_view` for quality constraints derived from the mapping:

```python
@dp.expect("order_id not null", "order_id IS NOT NULL")
@dp.expect("positive amount", "total_amount > 0")
@dp.table(name="fact_orders")
def fact_orders():
    return dp.read("df_amounts")
```

Use `@dp.expect_or_drop` to silently drop failing rows, or `@dp.expect_or_fail` to halt the pipeline on violations.

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

---

## PowerCenter → Lakeflow Mapping

| PowerCenter Transformation | Lakeflow Equivalent |
|---|---|
| Source Qualifier | `@dp.table` function calling `spark.table(...)` |
| Expression | `.withColumn()` chain inside a `@dp.table` function |
| Filter | `.filter()` inside a `@dp.table` or `@dp.temporary_view` |
| Joiner | `.join()` inside a `@dp.table` — read both sides with `dp.read(...)` |
| Lookup (connected) | `broadcast()` join inside a `@dp.table` |
| Aggregator | `.groupBy().agg()` inside a `@dp.materialized_view` |
| Router | Multiple `@dp.temporary_view` functions, each with a `.filter()` |
| Sorter | `.orderBy()` inside a `@dp.table` |
| Union | `.unionByName()` inside a `@dp.table` |
| Update Strategy | `dp.create_auto_cdc_flow(...)` on the final target |
| Sequence Generator | `# REVIEW: SEQUENCE GENERATOR` — no direct pipeline equivalent |
| Normalizer | `expr("stack(...)")` inside a `@dp.table` |
| Stored / External Procedure | `# REVIEW: STORED PROCEDURE` — stub, flag for manual implementation |

---

## Pre-Delivery Checklist (Pipeline)

- [ ] All source datasets use `@dp.table` with `spark.table(...)` or `spark.sql(...)`
- [ ] All intermediate transformations use `dp.read("upstream_dataset_name")` — no direct DataFrame references across functions
- [ ] All target tables are the final `@dp.table` or `dp.create_auto_cdc_flow` — no `.write` calls
- [ ] Environment values (catalog, schema) come from `spark.conf.get("pipeline.param.*")`, not hardcoded
- [ ] Every Update Strategy produces a `dp.create_auto_cdc_flow` with correct `keys=` list
- [ ] `@dp.expect` added for any data quality rules present in the source mapping
- [ ] `# REVIEW:` items appear both inline and in the header cell checklist
- [ ] A `# REVIEW: configure pipeline parameters` note lists every `pipeline.param.*` key the pipeline configuration must declare
- [ ] No `dbutils.widgets` calls — not available in pipeline execution context
- [ ] No `.write.format("delta")` calls — pipeline handles persistence

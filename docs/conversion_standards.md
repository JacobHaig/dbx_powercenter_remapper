# Conversion Standards: Generated Databricks Notebook Guidelines

All notebooks produced by the conversion agent must conform to these standards. A notebook that violates these standards is not a valid deliverable.

---

## Notebook Format

Notebooks are created as **Databricks workspace notebook assets** using the Databricks SDK — not written as plain `.py` files. See `docs/databricks_notebook_creation.md` for the exact SDK creation pattern and naming convention.

The notebook content uses Databricks source format:

```
# Databricks notebook source

# COMMAND ----------
# MAGIC %md
# MAGIC ## Header

# COMMAND ----------
# cell content here

# COMMAND ----------
```

- First line must be exactly `# Databricks notebook source`
- Cell separator is `# COMMAND ----------`
- Markdown cell lines are prefixed with `# MAGIC`
- Notebook path has **no `.py` extension**: `notebooks/nb_<mapping_name_lowercase>`

---

## Required Cell Structure (in order)

### Cell 1 — Markdown Header

```python
# MAGIC %md
# MAGIC # <Mapping Name>
# MAGIC
# MAGIC **Source System:** Oracle EBS Orders  
# MAGIC **Target Table:** `catalog.schema.fact_orders`  
# MAGIC **Original Mapping:** `FOLDER_NAME.M_LOAD_FACT_ORDERS`  
# MAGIC **Converted:** 2024-01-15  
# MAGIC
# MAGIC ## Description
# MAGIC <Value of the DESCRIPTION attribute on the <MAPPING> XML element>
# MAGIC
# MAGIC ## Review Items
# MAGIC - [ ] REVIEW: Sequence generator produces non-sequential IDs — verify acceptability
# MAGIC - [ ] REVIEW: Stored procedure SP_VALIDATE_ORDER not translated
```

Every `# REVIEW:` item logged during conversion must appear as a checklist item here.

### Cell 2 — Widgets (Parameters)

All environment-dependent values go here. No hardcoded catalog names, schema names, file paths, or connection strings anywhere else in the notebook.

```python
dbutils.widgets.text("catalog", "dev_catalog", "Catalog")
dbutils.widgets.text("schema", "orders", "Schema")
dbutils.widgets.text("source_path", "/mnt/landing/orders/", "Source Path")
dbutils.widgets.text("run_date", "", "Run Date (YYYY-MM-DD, blank = today)")

catalog = dbutils.widgets.get("catalog")
schema = dbutils.widgets.get("schema")
source_path = dbutils.widgets.get("source_path")
run_date = dbutils.widgets.get("run_date") or spark.sql("SELECT current_date()").first()[0].strftime("%Y-%m-%d")
```

### Cell 3 — Imports

```python
from pyspark.sql.functions import (
    col, lit, when, coalesce, concat, concat_ws,
    upper, lower, trim, ltrim, rtrim, length, substring,
    to_date, to_timestamp, date_format, current_timestamp,
    datediff, date_add, add_months, last_day,
    sum, count, avg, min, max, first, last,
    round, floor, ceil, abs,
    regexp_replace, locate,
    broadcast, monotonically_increasing_id,
    row_number, rank, dense_rank
)
from pyspark.sql.window import Window
from pyspark.sql.types import (
    StructType, StructField,
    StringType, IntegerType, LongType, DoubleType,
    DecimalType, DateType, TimestampType, BooleanType
)
from delta.tables import DeltaTable
```

Only import what is actually used. Do not import speculatively.

### Cell 4–N — Source Reads

One cell per source. The first line of every cell must be the **cell header comment** (see Code Formatting below).

```python
# SQ_ORDERS  [Source Qualifier]
sq_orders = (
    spark.table(f"{catalog}.{schema}.orders")
    .filter(col("status").isin(["ACTIVE", "PENDING"]))
)
```

### Cell N+1 through M — Transformations

One logical group per cell. Group by PowerCenter transformation name or logical step. The first line of every cell must be the **cell header comment** (see Code Formatting below).

```python
# EXP_CALCULATE_AMOUNTS  [Expression]
df_amounts = (
    sq_orders
    .withColumn("tax_amount",     round(col("subtotal") * lit(0.08), 2))
    .withColumn("total_amount",   col("subtotal") + col("tax_amount"))
    .withColumn("is_large_order", when(col("total_amount") > 1000, lit(1)).otherwise(lit(0)))
)
```

### Final Cells — Target Writes

One cell per target. Prefer Delta MERGE for any target with an Update Strategy; use `.write` for insert-only or truncate-reload targets.

```python
# TGT_FACT_ORDERS  [Target Definition] — INSERT/UPDATE via MERGE
target = DeltaTable.forName(spark, f"{catalog}.{schema}.fact_orders")

(
    target.alias("tgt")
    .merge(df_final.alias("src"), "tgt.order_id = src.order_id")
    .whenMatchedUpdate(
        condition="src.dd_flag = 1",
        set={c: f"src.{c}" for c in update_columns},
    )
    .whenNotMatchedInsert(
        condition="src.dd_flag = 0",
        values={c: f"src.{c}" for c in insert_columns},
    )
    .execute()
)
```

---

## Code Formatting

Every cell the agent produces must meet these formatting rules. Unreadable output is not a valid deliverable.

### Cell Header Comment (Required)

The **first line of every code cell** must identify the PowerCenter transformation it represents:

```
# <TRANSFORMATION_NAME>  [<TYPE>]
```

Examples:
```python
# SQ_ORDERS  [Source Qualifier]
# EXP_CALCULATE_AMOUNTS  [Expression]
# FIL_ACTIVE_ORDERS  [Filter]
# JNR_ORDER_CUSTOMER  [Joiner]
# LKP_PRODUCT  [Lookup Procedure]
# AGG_REVENUE  [Aggregator]
# RTR_ORDER_SIZE  [Router]
# UPD_ORDERS  [Update Strategy]
# TGT_FACT_ORDERS  [Target Definition]
```

For cells that combine multiple steps (e.g., a lookup pre-join immediately before its expression), list both:
```python
# LKP_TAX_RATE + EXP_TAX  [Lookup Procedure + Expression]
```

### Method Chain Formatting

Use parentheses (not backslash) for line continuation. Each `.method()` call goes on its own line, indented 4 spaces from the variable name. One column expression per `.withColumn()` line.

```python
# Good — readable, one operation per line
df_amounts = (
    sq_orders
    .withColumn("tax_amount",     round(col("subtotal") * lit(0.08), 2))
    .withColumn("total_amount",   col("subtotal") + col("tax_amount"))
    .withColumn("is_large_order", when(col("total_amount") > 1000, lit(1)).otherwise(lit(0)))
)

# Bad — do not write chains on a single line or with backslash continuation
df_amounts = sq_orders.withColumn("tax_amount", round(col("subtotal") * lit(0.08), 2)).withColumn("total_amount", col("subtotal") + col("tax_amount"))
```

### Aggregation Formatting

Each aggregation expression goes on its own line inside `.agg(...)`:

```python
df_agg = (
    df.groupBy("region", "product_cat")
    .agg(
        sum("amount").alias("total_revenue"),
        count(lit(1)).alias("order_count"),
        avg("amount").alias("avg_order"),
        sum(when(col("amount") > 1000, col("amount")).otherwise(lit(0))).alias("large_revenue"),
    )
)
```

### Join Formatting

Each argument to `.join()` on its own line when there are three or more arguments:

```python
df_joined = (
    df_orders.join(
        broadcast(lkp_product),
        df_orders["product_id"] == lkp_product["product_id"],
        "left",
    )
    .select(
        df_orders["order_id"],
        df_orders["amount"],
        lkp_product["product_name"],
        lkp_product["category"],
    )
)
```

### Delta MERGE Formatting

Each MERGE clause on its own `.when...()` line, with `set` / `values` dictionaries broken out one key per line:

```python
(
    target.alias("tgt")
    .merge(df_staged.alias("src"), "tgt.order_id = src.order_id")
    .whenMatchedUpdate(
        condition="src.dd_flag = 1",
        set={
            "status":     "src.status",
            "amount":     "src.amount",
            "updated_at": "src.updated_at",
        },
    )
    .whenNotMatchedInsert(
        condition="src.dd_flag = 0",
        values={
            "order_id": "src.order_id",
            "status":   "src.status",
            "amount":   "src.amount",
        },
    )
    .execute()
)
```

### Window Function Formatting

Assign the Window spec to a named variable before using it:

```python
_w = Window.partitionBy("customer_id").orderBy(desc("amount"))
df_ranked = (
    df
    .withColumn("RANKINDEX", rank().over(_w))
    .filter(col("RANKINDEX") <= 3)
    .drop("RANKINDEX")
)
```

### Column Alignment

When a cell has multiple `.withColumn()` calls for related fields, align the column name strings and expressions at the same column position for readability. This is a preference, not a hard rule — apply it when there are 3 or more adjacent calls.

---

## Naming Conventions

| Object | Convention | Example |
|---|---|---|
| Source DataFrames | `sq_<source_name>` (lowercase) | `sq_orders`, `sq_customers` |
| Intermediate DataFrames | `df_<transformation_name>` | `df_calculated`, `df_filtered` |
| Lookup DataFrames | `lkp_<lookup_name>` | `lkp_product`, `lkp_tax_rate` |
| Final/pre-write DataFrame | `df_final` | `df_final` |
| Variable ports (intermediate) | `_v_<name>` prefix, dropped after use | `_v_tax_rate` |
| Widget variables | Match widget name exactly | `catalog`, `schema`, `run_date` |

---

## Delta Write Patterns

### Append (insert-only)
```python
df_final.write \
    .format("delta") \
    .mode("append") \
    .option("mergeSchema", "false") \
    .saveAsTable(f"{catalog}.{schema}.{target_table}")
```

### Overwrite (truncate-reload)
```python
df_final.write \
    .format("delta") \
    .mode("overwrite") \
    .option("overwriteSchema", "false") \
    .saveAsTable(f"{catalog}.{schema}.{target_table}")
```

### MERGE (upsert/delete)
See the Update Strategy pattern in `docs/transformation_mappings.md`.

---

## Schema Handling

- Never use `inferSchema=True` on file sources in production notebooks. Always define an explicit schema.
- Never use `SELECT *` in final target writes. Enumerate the columns explicitly.
- Cast to the correct Spark type immediately after reading from source — do not rely on implicit casts downstream.

```python
# Explicit schema definition for CSV source
source_schema = StructType([
    StructField("order_id", StringType(), nullable=False),
    StructField("customer_id", IntegerType(), nullable=True),
    StructField("order_date", StringType(), nullable=True),   # read as string, cast below
    StructField("amount", StringType(), nullable=True),
])

df_raw = spark.read.schema(source_schema).option("header", "true").csv(source_path)

df_typed = df_raw \
    .withColumn("order_date", to_date(col("order_date"), "MM/dd/yyyy")) \
    .withColumn("amount", col("amount").cast(DecimalType(18, 2)))
```

---

## Performance Guidelines

1. **Broadcast small lookups.** Any lookup source expected to be under ~500 MB should use `broadcast()`. Add a comment if the size assumption is uncertain.
2. **Push filters early.** Apply `.filter()` as close to the source read as possible, not after multiple joins.
3. **Avoid `.collect()` in the hot path.** Validation row counts are acceptable; never collect full DataFrames.
4. **Partition large targets.** If the target table is expected to exceed 1 TB, add `.partitionBy("year", "month")` to the write call and flag it as `# REVIEW: verify partition columns`.
5. **Cache only when reused.** If a DataFrame feeds two downstream branches, cache it: `df.cache()`. If it's used only once, do not cache.

---

## Security and Secrets

- **Never hardcode credentials, connection strings, or passwords.** Use `dbutils.secrets.get(scope, key)`.
- **Never hardcode catalog or schema names.** Use widgets.
- Source file paths go in widgets, not literals.

```python
# Correct
jdbc_url = dbutils.secrets.get(scope="connections", key="oracle_orders_url")

# Wrong — never do this
jdbc_url = "jdbc:oracle:thin:@prod-db:1521/ORDERS"
```

---

## Error Handling and Validation

Add a validation cell after each major transformation group when the source mapping included record count checks or rejection logic.

```python
# Validation — counts
input_count = sq_orders.count()
output_count = df_final.count()
rejected_count = df_rejected.count()

assert output_count + rejected_count == input_count, \
    f"Row count mismatch: input={input_count}, output={output_count}, rejected={rejected_count}"

print(f"Input: {input_count} | Output: {output_count} | Rejected: {rejected_count}")
```

---

## Comments Policy

Add a comment only when the translation is non-obvious or requires a reviewer decision:

```python
# REVIEW: PowerCenter used a Stored Procedure (SP_CALC_DISCOUNT) — replaced with inline logic;
# verify business rules match the procedure's behavior.

# Using monotonically_increasing_id() as surrogate key — not sequential across runs.
# If stable sequence is required, switch to Delta IDENTITY column.
```

Do not comment on what the code does if the code is self-explanatory.

---

## Checklist Before Delivering a Notebook

- [ ] All `<TRANSFORMATION>` nodes are accounted for (every PowerCenter transformation has a corresponding cell)
- [ ] All source/target names are parameterized via widgets
- [ ] No PowerCenter expression syntax remains
- [ ] No credentials or hardcoded paths
- [ ] Schema is explicitly defined for all file sources
- [ ] MERGE is used for any target fed by an Update Strategy transformation
- [ ] `# REVIEW:` items appear both inline and in the header cell checklist
- [ ] Cell order matches the data flow DAG (sources → transformations → targets)
- [ ] DataFrame naming follows the conventions table above

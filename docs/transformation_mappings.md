# Transformation Mappings: PowerCenter → Databricks PySpark

This is the authoritative translation reference. For every PowerCenter transformation type and expression function, the PySpark equivalent is defined here. Do not deviate from these patterns without updating this document.

---

## Transformation Type Translations

### Source Qualifier → DataFrame Read

**Pattern:** One `spark.read` call per source. The SQL override in `Sql Query` becomes the query string.

```python
# No Sql Query override — read full table
sq_orders = spark.table(f"{catalog}.{schema}.orders")

# With Sql Query override — use f-string so catalog/schema come from widgets
sq_orders = spark.sql(f"""
    SELECT order_id, customer_id, amount
    FROM {catalog}.{schema}.orders
    WHERE status = 'ACTIVE'
""")

# With Source Filter only (no full override)
sq_orders = spark.table(f"{catalog}.{schema}.orders").filter("status = 'ACTIVE'")

# With User Defined Join (two sources joined in SQ) — use f-string for catalog/schema
sq_combined = spark.sql(f"""
    SELECT o.order_id, c.customer_name
    FROM {catalog}.{schema}.orders o
    JOIN {catalog}.{schema}.customers c ON o.customer_id = c.customer_id
""")
```

**Notes:**
- Replace connection variable names (e.g., `$DBConnection_Orders`) with Unity Catalog references via widgets.
- Flat file sources use `spark.read.csv(...)` or `spark.read.format("csv")`.
- `Number Of Sorted Ports` on the SQ does not need to be reproduced — downstream Sorter handles ordering.

---

### Expression → `withColumn` / `select`

**Pattern:** Chain `.withColumn()` calls for each output port, or use `.select()` with column expressions for a full projection.

```python
# Simple expression ports
df = sq_orders \
    .withColumn("full_name", concat_ws(" ", col("first_name"), col("last_name"))) \
    .withColumn("discounted_amount", col("amount") * lit(0.9)) \
    .withColumn("is_large_order", when(col("amount") > 1000, lit(1)).otherwise(lit(0)))

# VARIABLE ports — compute intermediately, drop before passing downstream
df = sq_orders \
    .withColumn("_v_tax_rate", lit(0.08)) \
    .withColumn("tax_amount", col("amount") * col("_v_tax_rate")) \
    .drop("_v_tax_rate")
```

**Notes:**
- `VARIABLE` ports (internal intermediates) should be prefixed `_v_` and dropped after use.
- Preserve the exact output port names from `<TRANSFORMFIELD NAME="...">`.

---

### Filter → `.filter()` / `.where()`

```python
# PowerCenter: FILTEROVERRIDE expression = amount > 100 AND status = 'ACTIVE'
df_filtered = df.filter((col("amount") > 100) & (col("status") == "ACTIVE"))
```

---

### Joiner → `.join()`

| PowerCenter Join Type | PySpark Join Type |
|---|---|
| `Normal` (inner) | `"inner"` |
| `Master Outer` | `"right"` (master is right side) |
| `Detail Outer` | `"left"` (detail is left side) |
| `Full Outer` | `"outer"` |

```python
# Normal join — master = dim_customer, detail = fact_orders
df_joined = df_orders.join(
    df_customers,
    df_orders["customer_id"] == df_customers["customer_id"],
    "inner"
).drop(df_customers["customer_id"])  # remove duplicate key column

# Master Outer join
df_joined = df_orders.join(df_customers, "customer_id", "right")
```

**Sorted Input:** If `Sorted Input = true`, add `.sortWithinPartitions(join_keys)` before the join — though Spark handles this internally; the hint is informational.

---

### Lookup (Connected) → Broadcast Join

```python
from pyspark.sql.functions import broadcast

# Static lookup against small dimension table
lkp_product = spark.table(f"{catalog}.{schema}.dim_product")

df_with_product = df_orders.join(
    broadcast(lkp_product),
    df_orders["product_id"] == lkp_product["product_id"],
    "left"
)
```

**Multiple match policy:**

| PowerCenter Policy | PySpark Equivalent |
|---|---|
| `Use first value` | `.dropDuplicates(["key"])` on lookup before join |
| `Use last value` | Sort descending + `.dropDuplicates(["key"])` on lookup |
| `Report error` | Assert uniqueness: `assert lkp.select("key").distinct().count() == lkp.count()` |

**Sql Override:** If present, build the lookup DataFrame using `spark.sql(override_sql)`.

---

### Lookup (Unconnected) → Python UDF or `.join()`

Unconnected lookups appear as `:LKP.lookup_name(key)` inside an Expression port's `EXPRESSION` attribute — this is **PowerCenter expression syntax**, not Python. The colon-prefix `:LKP.` means the lookup is called inline inside the expression. Translate to a broadcast join applied before the Expression transformation so the returned column is available as a regular DataFrame column.

```python
# Instead of :LKP.lkp_tax_rate(state_code) in an expression port,
# pre-join the lookup table and reference the column directly
lkp_tax_rate = spark.table(f"{catalog}.{schema}.tax_rates").select("state_code", "rate")
df = df.join(broadcast(lkp_tax_rate), "state_code", "left")
# Expression port: col("rate") is now available as a regular column
```

---

### Aggregator → `.groupBy().agg()`

```python
from pyspark.sql.functions import sum, count, avg, min, max, first, last, stddev, variance

df_agg = df.groupBy("region", "product_category").agg(
    sum("revenue").alias("total_revenue"),
    count("order_id").alias("order_count"),
    avg("order_value").alias("avg_order_value"),
    max("order_date").alias("last_order_date")
)
```

**Conditional aggregation** (aggregator with filter on port):
```python
# SUM(amount, status = 'COMPLETE') — aggregation with filter condition
from pyspark.sql.functions import when
df_agg = df.groupBy("region").agg(
    sum(when(col("status") == "COMPLETE", col("amount")).otherwise(lit(0))).alias("completed_amount")
)
```

---

### Router → Multiple `.filter()` branches

```python
# Router with three groups: HIGH_VALUE, MED_VALUE, default
df_high = df.filter(col("amount") >= 10000)
df_med  = df.filter((col("amount") >= 1000) & (col("amount") < 10000))
df_low  = df.filter(col("amount") < 1000)   # default group catches everything else
```

**Important:** The default group is exclusive — it receives rows not matched by any named group. Build it as the logical complement of all named group conditions:

```python
# Named groups: HIGH (amount >= 10000), MED (amount >= 1000 AND amount < 10000)
# Default group = NOT (HIGH OR MED)
df_low = df.filter(~((col("amount") >= 10000) | ((col("amount") >= 1000) & (col("amount") < 10000))))
# Equivalent simplified form:
df_low = df.filter(col("amount") < 1000)
```

---

### Sorter → `.orderBy()`

```python
from pyspark.sql.functions import asc, desc, asc_nulls_last, desc_nulls_first

df_sorted = df.orderBy(
    asc_nulls_last("customer_id"),
    desc_nulls_first("order_date")
)
```

---

### Rank → Window Function

```python
from pyspark.sql.window import Window
from pyspark.sql.functions import rank, row_number, desc

# Top 3 orders per customer (Rank TOP 3, port = amount, group by = customer_id)
window = Window.partitionBy("customer_id").orderBy(desc("amount"))
df_ranked = df.withColumn("_rank", rank().over(window)).filter(col("_rank") <= 3).drop("_rank")

# Bottom N — just flip the orderBy direction
```

---

### Update Strategy → Delta MERGE

Every Update Strategy maps to a `MERGE INTO` statement on the Delta target.

```python
from delta.tables import DeltaTable

target = DeltaTable.forName(spark, f"{catalog}.{schema}.{target_table}")

target.alias("tgt").merge(
    df_incoming.alias("src"),
    "tgt.order_id = src.order_id"
).whenMatchedUpdate(
    condition="src.dd_flag = 1",   # DD_UPDATE
    set={"status": "src.status", "amount": "src.amount", "updated_at": "src.updated_at"}
).whenNotMatchedInsert(
    condition="src.dd_flag = 0",   # DD_INSERT
    values={"order_id": "src.order_id", "status": "src.status", "amount": "src.amount"}
).whenMatchedDelete(
    condition="src.dd_flag = 2"    # DD_DELETE
).execute()
```

**Update Strategy Expression → `dd_flag` column:**

| Expression | Value | MERGE clause |
|---|---|---|
| `DD_INSERT` | 0 | `whenNotMatchedInsert` |
| `DD_UPDATE` | 1 | `whenMatchedUpdate` |
| `DD_DELETE` | 2 | `whenMatchedDelete` |
| `DD_REJECT` | 3 | Filter out before MERGE |

If all rows are inserts (no Update Strategy transformation), use:
```python
df.write.format("delta").mode("append").saveAsTable(f"{catalog}.{schema}.{target_table}")
```

If `Truncate Target Table = true`, use `mode("overwrite")` instead of MERGE.

---

### Union → `.unionByName()`

```python
df_union = df_stream_a.unionByName(df_stream_b).unionByName(df_stream_c)
```

Use `allowMissingColumns=True` only if the Union transformation is configured to handle schema differences (rare).

---

### Sequence Generator → `monotonically_increasing_id()` or `row_number()`

```python
from pyspark.sql.functions import monotonically_increasing_id, row_number, lit
from pyspark.sql.window import Window

# Simple auto-increment surrogate key
df = df.withColumn("surrogate_key", monotonically_increasing_id())

# Deterministic sequence starting at a specific value (Start Value = 1000, Increment = 1)
w = Window.orderBy(lit(1))  # stable order not guaranteed — use a natural key if available
df = df.withColumn("seq_key", row_number().over(w) + lit(999))  # offset by start - 1
```

**Note:** PowerCenter sequences are session-scoped and stateful. `monotonically_increasing_id()` is non-deterministic across runs. If the target key must be stable, use an existing natural key or generate from a Delta sequence table. Add a `# REVIEW:` comment explaining this.

---

### Normalizer → `explode()` / `stack()`

Normalizer unpivots repeated column groups. Translate using `stack()`.

```python
from pyspark.sql.functions import expr

# PowerCenter normalizer: columns Q1_SALES, Q2_SALES, Q3_SALES, Q4_SALES → rows
df_normalized = df.select(
    "product_id",
    expr("stack(4, 'Q1', Q1_SALES, 'Q2', Q2_SALES, 'Q3', Q3_SALES, 'Q4', Q4_SALES) as (quarter, sales)")
)
```

---

### Mapplet → Python Helper Function

**When mapplet XML is provided**, extract input and output ports and convert to a Python function:

- `PORTTYPE="INPUT"` ports → the function takes a single DataFrame parameter named `df`; individual input port names are referenced as columns within the function
- `PORTTYPE="OUTPUT"` ports → columns present on the returned DataFrame
- `<TRANSFORMFIELD>` expressions inside the mapplet → `.withColumn()` calls inside the function body

```python
# Mapplet XML had: INPUT ports (zip, state), OUTPUT ports (zip_clean, state_clean)
def mlt_address_cleanse(df):
    return (
        df
        .withColumn("zip_clean", regexp_replace(col("zip"), "[^0-9]", ""))
        .withColumn("state_clean", upper(trim(col("state"))))
        .drop("zip", "state")
    )

# Call site in the main notebook
df_transformed = mlt_address_cleanse(df_source)
```

**Naming convention:** `mlt_<mapplet_name_lowercase>` for the function; input DataFrame parameter is always `df`.

**When mapplet XML is not provided**, stub and flag:

```python
# REVIEW: MAPPLET — mlt_address_cleanse — no XML provided
# Expected input ports: unknown
# Expected output ports: unknown
# df_transformed = mlt_address_cleanse(df_source)
```

**Large or complex mapplets** (more than ~10 transform fields, or containing Lookups or Aggregators): create a standalone notebook asset at `notebooks/mapplets/mlt_<name>` using the SDK pattern in `docs/databricks_notebook_creation.md`, then reference it with `%run`:

```python
# %run ./mapplets/mlt_address_cleanse
df_transformed = mlt_address_cleanse(df_source)
```

Flag as `# REVIEW: MAPPLET INLINED — verify port mappings` whenever inlining a mapplet, since mapplet reuse across notebooks is lost when inlined.

---

## Expression Function Translation Table

### Conditional

| PowerCenter | PySpark |
|---|---|
| `IIF(cond, t, f)` | `when(cond, t).otherwise(f)` |
| `DECODE(val, s1, r1, s2, r2, default)` | Chained `when().when().otherwise()` |
| `ISNULL(port)` | `col("port").isNull()` |
| `ISNULL(port) = FALSE` | `col("port").isNotNull()` |

### String

| PowerCenter | PySpark |
|---|---|
| `LTRIM(s)` | `ltrim(col("s"))` |
| `RTRIM(s)` | `rtrim(col("s"))` |
| `TRIM(s)` | `trim(col("s"))` |
| `UPPER(s)` | `upper(col("s"))` |
| `LOWER(s)` | `lower(col("s"))` |
| `LENGTH(s)` | `length(col("s"))` |
| `SUBSTR(s, start, len)` | `substring(col("s"), start, len)` — **1-based index** (PowerCenter and Spark both use 1-based) |
| `INSTR(s, search)` | `locate(search, col("s"))` — returns 0 if not found (PowerCenter also returns 0) |
| `CONCAT(s1, s2)` | `concat(col("s1"), col("s2"))` |
| `RPAD(s, len, pad)` | `rpad(col("s"), len, pad)` |
| `LPAD(s, len, pad)` | `lpad(col("s"), len, pad)` |
| `REG_REPLACE(s, pat, rep)` | `regexp_replace(col("s"), pat, rep)` |
| `TO_CHAR(date, fmt)` | `date_format(col("date"), fmt)` — convert PowerCenter format to Java SimpleDateFormat |

### Numeric

| PowerCenter | PySpark |
|---|---|
| `ROUND(v, n)` | `round(col("v"), n)` |
| `TRUNC(v, n)` | `trunc(col("v"), n)` (dates) or `bround(col("v"), n)` (numbers) |
| `ABS(v)` | `abs(col("v"))` |
| `CEIL(v)` | `ceil(col("v"))` |
| `FLOOR(v)` | `floor(col("v"))` |
| `MOD(a, b)` | `col("a") % col("b")` |
| `POWER(base, exp)` | `pow(col("base"), exp)` |
| `SQRT(v)` | `sqrt(col("v"))` |

### Date

| PowerCenter | PySpark |
|---|---|
| `TO_DATE(s, fmt)` | `to_date(col("s"), fmt)` or `to_timestamp(col("s"), fmt)` |
| `TO_CHAR(d, fmt)` | `date_format(col("d"), fmt)` |
| `SYSDATE` | `current_timestamp()` |
| `ADD_TO_DATE(d, 'DD', n)` | `date_add(col("d"), n)` |
| `ADD_TO_DATE(d, 'MM', n)` | `add_months(col("d"), n)` |
| `ADD_TO_DATE(d, 'YY', n)` | `add_months(col("d"), n * 12)` |
| `ADD_TO_DATE(d, 'HH', n)` | `col("d") + expr("INTERVAL n HOURS")` |
| `DATE_DIFF(d1, d2, 'DD')` | `datediff(col("d1"), col("d2"))` |
| `DATE_DIFF(d1, d2, 'MM')` | `months_between(col("d1"), col("d2"))` |
| `LAST_DAY(d)` | `last_day(col("d"))` |

### PowerCenter Format Strings → Java SimpleDateFormat

| PowerCenter | Java |
|---|---|
| `MM/DD/YYYY` | `MM/dd/yyyy` |
| `YYYY-MM-DD` | `yyyy-MM-dd` |
| `DD-MON-YYYY` | `dd-MMM-yyyy` |
| `YYYY-MM-DD HH24:MI:SS` | `yyyy-MM-dd HH:mm:ss` |
| `MM/DD/YYYY HH:MI:SS AM` | `MM/dd/yyyy hh:mm:ss a` |

### Aggregation (in Aggregator context)

| PowerCenter | PySpark |
|---|---|
| `SUM(port)` | `sum("port")` |
| `COUNT(port)` | `count("port")` |
| `COUNT(*)` | `count(lit(1))` |
| `AVG(port)` | `avg("port")` |
| `MIN(port)` | `min("port")` |
| `MAX(port)` | `max("port")` |
| `FIRST(port)` | `first("port", ignorenulls=True)` |
| `LAST(port)` | `last("port", ignorenulls=True)` |
| `STDDEV(port)` | `stddev("port")` |
| `VARIANCE(port)` | `variance("port")` |

---

## Source Type Patterns

### Relational (Oracle, SQL Server, PostgreSQL)

```python
# Via Unity Catalog (preferred)
df = spark.table(f"{catalog}.{schema}.{table_name}")

# Via JDBC (when source is not in Unity Catalog)
df = spark.read \
    .format("jdbc") \
    .option("url", dbutils.secrets.get(scope="connections", key="oracle_url")) \
    .option("dbtable", f"(SELECT * FROM {schema}.{table_name}) t") \
    .option("user", dbutils.secrets.get(scope="connections", key="oracle_user")) \
    .option("password", dbutils.secrets.get(scope="connections", key="oracle_pass")) \
    .load()
```

### Flat File (CSV, Delimited)

```python
df = spark.read \
    .option("header", "true") \
    .option("inferSchema", "false") \
    .option("delimiter", "|") \
    .schema(defined_schema) \
    .csv(dbutils.widgets.get("source_path"))
```

### Fixed-Width File

```python
from pyspark.sql.functions import substring

raw = spark.read.text(dbutils.widgets.get("source_path"))
df = raw.select(
    substring(col("value"), 1, 10).alias("order_id"),
    substring(col("value"), 11, 30).alias("customer_name"),
    substring(col("value"), 41, 15).alias("amount")
)
```

---

## Known Gaps / Manual Review Required

These patterns have no clean automated translation and always require a `# REVIEW:` flag:

1. **Stored Procedure / External Procedure** — Translate intent, flag for DBA review.
2. **Sequence Generator as a true distributed counter** — `monotonically_increasing_id()` is not sequential; use a Delta IDENTITY column instead.
3. **Unconnected Lookup with dynamic SQL override** — Pre-materialize as a broadcast DataFrame.
4. **PowerCenter parameters and variables (`$$PARAM`, `$MAPPING.VAR`)** — Map to `dbutils.widgets` or notebook-level Python variables.
5. **Incremental load via `$$LAST_EXTRACTION_DATE`** — Translate to a widget with a default of `1900-01-01`; the orchestration layer must pass the actual value.
6. **Custom FTP/MQ/SAP source/target adapters** — Out of scope; flag for manual implementation.

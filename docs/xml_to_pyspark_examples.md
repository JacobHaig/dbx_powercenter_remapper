# PowerCenter XML → PySpark: Side-by-Side Examples

Each section shows the raw PowerCenter XML for one transformation type and the exact PySpark code it should produce in the output notebook. These are concrete reference examples, not abstractions.

---

## Source Qualifier (Relational, no SQL override)

**PowerCenter XML**
```xml
<TRANSFORMATION NAME="SQ_ORDERS" TYPE="Source Qualifier">
  <TRANSFORMFIELD NAME="order_id"       DATATYPE="number"   PORTTYPE="OUTPUT" PRECISION="10" SCALE="0"/>
  <TRANSFORMFIELD NAME="customer_id"    DATATYPE="number"   PORTTYPE="OUTPUT" PRECISION="10" SCALE="0"/>
  <TRANSFORMFIELD NAME="amount"         DATATYPE="number"   PORTTYPE="OUTPUT" PRECISION="18" SCALE="2"/>
  <TRANSFORMFIELD NAME="order_date"     DATATYPE="date"     PORTTYPE="OUTPUT" PRECISION="29" SCALE="9"/>
  <TABLEATTRIBUTE NAME="Sql Query"      VALUE=""/>
  <TABLEATTRIBUTE NAME="Source Filter"  VALUE=""/>
</TRANSFORMATION>
```

**PySpark**
```python
# SQ_ORDERS  [Source Qualifier]
sq_orders = spark.table(f"{catalog}.{schema}.orders")
```

---

## Source Qualifier (SQL override + filter)

**PowerCenter XML**
```xml
<TRANSFORMATION NAME="SQ_ORDERS" TYPE="Source Qualifier">
  <TRANSFORMFIELD NAME="order_id"    DATATYPE="number"  PORTTYPE="OUTPUT" PRECISION="10" SCALE="0"/>
  <TRANSFORMFIELD NAME="amount"      DATATYPE="number"  PORTTYPE="OUTPUT" PRECISION="18" SCALE="2"/>
  <TABLEATTRIBUTE NAME="Sql Query"   VALUE="SELECT order_id, amount FROM orders WHERE region = 'US'"/>
  <TABLEATTRIBUTE NAME="Source Filter" VALUE=""/>
</TRANSFORMATION>
```

**PySpark**
```python
# SQ_ORDERS  [Source Qualifier] — Sql Query override
sq_orders = spark.sql(f"""
    SELECT order_id, amount
    FROM {catalog}.{schema}.orders
    WHERE region = 'US'
""")
```

---

## Expression

**PowerCenter XML**

> **Port types:** `INPUT/OUTPUT` passes through unchanged. `OUTPUT` is a computed column. `VARIABLE` (`PORTTYPE="VARIABLE"`) is an internal intermediate — it is computed but **never passed downstream**; translate with a `_v_` prefix and `.drop()` it before returning.

```xml
<TRANSFORMATION NAME="EXP_CALC" TYPE="Expression">
  <TRANSFORMFIELD NAME="order_id"    PORTTYPE="INPUT/OUTPUT" EXPRESSION="order_id"               DATATYPE="number"   PRECISION="10" SCALE="0"/>
  <TRANSFORMFIELD NAME="amount"      PORTTYPE="INPUT"        EXPRESSION="amount"                  DATATYPE="number"   PRECISION="18" SCALE="2"/>
  <TRANSFORMFIELD NAME="tax_amount"  PORTTYPE="OUTPUT"       EXPRESSION="ROUND(amount * 0.08, 2)" DATATYPE="number"   PRECISION="18" SCALE="2"/>
  <TRANSFORMFIELD NAME="full_name"   PORTTYPE="OUTPUT"       EXPRESSION="UPPER(TRIM(first_name)) || ' ' || UPPER(TRIM(last_name))" DATATYPE="varchar2" PRECISION="100" SCALE="0"/>
  <TRANSFORMFIELD NAME="_v_discount" PORTTYPE="VARIABLE"     EXPRESSION="IIF(amount > 1000, 0.10, 0.05)" DATATYPE="number" PRECISION="5" SCALE="2"/>
  <TRANSFORMFIELD NAME="net_amount"  PORTTYPE="OUTPUT"       EXPRESSION="amount - (amount * _v_discount)" DATATYPE="number" PRECISION="18" SCALE="2"/>
</TRANSFORMATION>
```

> **Expression syntax:** `ROUND`, `UPPER`, `TRIM`, `IIF` are PowerCenter built-in functions. See `docs/transformation_mappings.md` — Expression Function Translation Table for the PySpark equivalents.

**PySpark**
```python
# EXP_CALC  [Expression]
# _v_discount has PORTTYPE="VARIABLE" — compute inline, drop before passing downstream
df_calc = (
    sq_orders
    .withColumn("tax_amount",   round(col("amount") * lit(0.08), 2))
    .withColumn("full_name",    upper(trim(concat_ws(" ", col("first_name"), col("last_name")))))
    .withColumn("_v_discount",  when(col("amount") > 1000, lit(0.10)).otherwise(lit(0.05)))
    .withColumn("net_amount",   col("amount") - (col("amount") * col("_v_discount")))
    .drop("_v_discount")
)
```

---

## Filter

**PowerCenter XML**
```xml
<TRANSFORMATION NAME="FIL_ACTIVE_ORDERS" TYPE="Filter">
  <TRANSFORMFIELD NAME="order_id"  PORTTYPE="INPUT/OUTPUT" DATATYPE="number"   PRECISION="10" SCALE="0"/>
  <TRANSFORMFIELD NAME="status"    PORTTYPE="INPUT/OUTPUT" DATATYPE="varchar2" PRECISION="20" SCALE="0"/>
  <TRANSFORMFIELD NAME="amount"    PORTTYPE="INPUT/OUTPUT" DATATYPE="number"   PRECISION="18" SCALE="2"/>
  <TABLEATTRIBUTE NAME="Filter Condition" VALUE="status = 'ACTIVE' AND amount > 0"/>
</TRANSFORMATION>
```

**PySpark**
```python
# FIL_ACTIVE_ORDERS  [Filter]
df_filtered = (
    df_calc
    .filter((col("status") == "ACTIVE") & (col("amount") > 0))
)
```

---

## Joiner

**PowerCenter XML**
```xml
<TRANSFORMATION NAME="JNR_ORDER_CUSTOMER" TYPE="Joiner">
  <!-- Detail (left/driving) side ports -->
  <TRANSFORMFIELD NAME="order_id"     PORTTYPE="INPUT" DATATYPE="number"   PRECISION="10" SCALE="0"/>
  <TRANSFORMFIELD NAME="customer_id"  PORTTYPE="INPUT" DATATYPE="number"   PRECISION="10" SCALE="0"/>
  <TRANSFORMFIELD NAME="amount"       PORTTYPE="INPUT" DATATYPE="number"   PRECISION="18" SCALE="2"/>
  <!-- Master (right/lookup) side ports — prefixed MASTER: in connector -->
  <TRANSFORMFIELD NAME="customer_id"  PORTTYPE="INPUT" DATATYPE="number"   PRECISION="10" SCALE="0"/>
  <TRANSFORMFIELD NAME="customer_name" PORTTYPE="INPUT" DATATYPE="varchar2" PRECISION="100" SCALE="0"/>
  <!-- Output ports -->
  <TRANSFORMFIELD NAME="order_id"     PORTTYPE="OUTPUT" DATATYPE="number"   PRECISION="10" SCALE="0"/>
  <TRANSFORMFIELD NAME="customer_name" PORTTYPE="OUTPUT" DATATYPE="varchar2" PRECISION="100" SCALE="0"/>
  <TRANSFORMFIELD NAME="amount"       PORTTYPE="OUTPUT" DATATYPE="number"   PRECISION="18" SCALE="2"/>
  <TABLEATTRIBUTE NAME="Join Type"      VALUE="Normal"/>
  <TABLEATTRIBUTE NAME="Join Condition" VALUE="detail.customer_id = master.customer_id"/>
  <TABLEATTRIBUTE NAME="Sorted Input"   VALUE="false"/>
</TRANSFORMATION>

<!-- CONNECTOR elements declare which upstream pipeline is master vs. detail.
     The MASTER: prefix on TOFIELD marks the master (right/lookup) side. -->
<CONNECTOR FROMINSTANCE="SQ_ORDERS"    FROMFIELD="order_id"    TOINSTANCE="JNR_ORDER_CUSTOMER" TOFIELD="order_id"/>
<CONNECTOR FROMINSTANCE="SQ_CUSTOMERS" FROMFIELD="customer_id" TOINSTANCE="JNR_ORDER_CUSTOMER" TOFIELD="MASTER:customer_id"/>
```

**PySpark**
```python
# JNR_ORDER_CUSTOMER  [Joiner] — Normal (inner) join
# Master pipeline = sq_customers (right side); Detail pipeline = df_filtered (left/driving side)
df_joined = (
    df_filtered.join(
        sq_customers,
        df_filtered["customer_id"] == sq_customers["customer_id"],
        "inner",
    )
    .select(
        df_filtered["order_id"],
        sq_customers["customer_name"],
        df_filtered["amount"],
    )
)
```

**Join type mapping:**

| `Join Type` XML value | PySpark how |
|---|---|
| `Normal` | `"inner"` |
| `Master Outer` | `"right"` |
| `Detail Outer` | `"left"` |
| `Full Outer` | `"outer"` |

---

## Lookup (Connected)

**PowerCenter XML**
```xml
<TRANSFORMATION NAME="LKP_PRODUCT" TYPE="Lookup Procedure">
  <TRANSFORMFIELD NAME="product_id"   PORTTYPE="INPUT"  DATATYPE="number"   PRECISION="10" SCALE="0"/>
  <TRANSFORMFIELD NAME="product_name" PORTTYPE="OUTPUT" DATATYPE="varchar2" PRECISION="100" SCALE="0"/>
  <TRANSFORMFIELD NAME="category"     PORTTYPE="OUTPUT" DATATYPE="varchar2" PRECISION="50"  SCALE="0"/>
  <TABLEATTRIBUTE NAME="Lookup table name"          VALUE="dim_product"/>
  <TABLEATTRIBUTE NAME="Lookup Condition"           VALUE="LKP_PRODUCT.product_id = product_id"/>
  <TABLEATTRIBUTE NAME="Lookup Caching Enabled"     VALUE="YES"/>
  <TABLEATTRIBUTE NAME="Lookup Policy on Multiple Match" VALUE="Use first value"/>
  <TABLEATTRIBUTE NAME="Connection Information"     VALUE="$DBConnection_DW"/>
</TRANSFORMATION>
```

**PySpark**
```python
# LKP_PRODUCT  [Lookup Procedure] — broadcast join (caching enabled → small table assumption)
# "Use first value" policy → deduplicate on join key before joining
lkp_product = (
    spark.table(f"{catalog}.{schema}.dim_product")
    .dropDuplicates(["product_id"])
    .select("product_id", "product_name", "category")
)

df_with_product = (
    df_joined.join(
        broadcast(lkp_product),
        "product_id",
        "left",
    )
)
```

---

## Lookup (Unconnected)

Unconnected lookups appear as `:LKP.<name>(key)` inside an Expression port `EXPRESSION` attribute. This is **PowerCenter expression syntax** (not Python) — the colon-prefix `:LKP.` signals an unconnected lookup call. Translate by pre-joining the lookup table before the Expression cell so the result column is available as a regular DataFrame column.

**PowerCenter XML**
```xml
<!-- Lookup definition — note TYPE="Lookup Procedure" for both connected and unconnected -->
<TRANSFORMATION NAME="LKP_TAX_RATE" TYPE="Lookup Procedure">
  <TRANSFORMFIELD NAME="state_code" PORTTYPE="INPUT"  DATATYPE="varchar2" PRECISION="2"  SCALE="0"/>
  <TRANSFORMFIELD NAME="tax_rate"   PORTTYPE="OUTPUT" DATATYPE="number"   PRECISION="5"  SCALE="4"/>
  <TABLEATTRIBUTE NAME="Lookup table name" VALUE="tax_rates"/>
</TRANSFORMATION>

<!-- Expression port that calls it via :LKP.name(key) — PowerCenter expression syntax -->
<TRANSFORMATION NAME="EXP_TAX" TYPE="Expression">
  <TRANSFORMFIELD NAME="state_code"  PORTTYPE="INPUT"  EXPRESSION="state_code"/>
  <TRANSFORMFIELD NAME="applied_tax" PORTTYPE="OUTPUT" EXPRESSION="amount * :LKP.LKP_TAX_RATE(state_code)"/>
  <!--                                                              ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
       :LKP.LKP_TAX_RATE(state_code) means: call unconnected lookup LKP_TAX_RATE with state_code as input key.
       In PySpark this becomes col("tax_rate") after pre-joining the lookup table on state_code.           -->
</TRANSFORMATION>
```

**PySpark**
```python
# LKP_TAX_RATE + EXP_TAX  [Lookup Procedure + Expression]
# Step 1: pre-join the unconnected lookup so tax_rate is available as a column
lkp_tax_rate = (
    spark.table(f"{catalog}.{schema}.tax_rates")
    .dropDuplicates(["state_code"])
    .select("state_code", "tax_rate")
)

# Step 2: :LKP.LKP_TAX_RATE(state_code) → col("tax_rate") after the join
df_tax = (
    df_filtered
    .join(broadcast(lkp_tax_rate), "state_code", "left")
    .withColumn("applied_tax", col("amount") * col("tax_rate"))
)
```

---

## Aggregator

**PowerCenter XML**
```xml
<TRANSFORMATION NAME="AGG_REVENUE" TYPE="Aggregator">
  <!-- GROUP BY ports have no aggregate expression -->
  <TRANSFORMFIELD NAME="region"        PORTTYPE="INPUT/OUTPUT" EXPRESSION="region"       DATATYPE="varchar2" PRECISION="50" SCALE="0"/>
  <TRANSFORMFIELD NAME="product_cat"   PORTTYPE="INPUT/OUTPUT" EXPRESSION="product_cat"  DATATYPE="varchar2" PRECISION="50" SCALE="0"/>
  <!-- Aggregated output ports -->
  <TRANSFORMFIELD NAME="total_revenue" PORTTYPE="OUTPUT" EXPRESSION="SUM(amount)"        DATATYPE="number" PRECISION="18" SCALE="2"/>
  <TRANSFORMFIELD NAME="order_count"   PORTTYPE="OUTPUT" EXPRESSION="COUNT(*)"           DATATYPE="number" PRECISION="10" SCALE="0"/>
  <TRANSFORMFIELD NAME="avg_order"     PORTTYPE="OUTPUT" EXPRESSION="AVG(amount)"        DATATYPE="number" PRECISION="18" SCALE="2"/>
  <!-- Conditional aggregation -->
  <TRANSFORMFIELD NAME="large_revenue" PORTTYPE="OUTPUT" EXPRESSION="SUM(amount, amount > 1000)" DATATYPE="number" PRECISION="18" SCALE="2"/>
</TRANSFORMATION>
```

**PySpark**
```python
# AGG_REVENUE  [Aggregator]
df_agg = (
    df_tax
    .groupBy("region", "product_cat")
    .agg(
        sum("amount").alias("total_revenue"),
        count(lit(1)).alias("order_count"),
        avg("amount").alias("avg_order"),
        sum(when(col("amount") > 1000, col("amount")).otherwise(lit(0))).alias("large_revenue"),
    )
)
```

---

## Router

**PowerCenter XML**

> **Router XML structure:** Each named group has a pair of `TABLEATTRIBUTE` elements — `GROUP NAME` followed by `GROUP FILTER CONDITION`. The default group has no condition; it catches all rows not matched by any named group.

```xml
<TRANSFORMATION NAME="RTR_ORDER_SIZE" TYPE="Router">
  <TRANSFORMFIELD NAME="order_id" PORTTYPE="INPUT" DATATYPE="number" PRECISION="10" SCALE="0"/>
  <TRANSFORMFIELD NAME="amount"   PORTTYPE="INPUT" DATATYPE="number" PRECISION="18" SCALE="2"/>
  <!-- Group 1: HIGH -->
  <TABLEATTRIBUTE NAME="GROUP NAME"             VALUE="HIGH"/>
  <TABLEATTRIBUTE NAME="GROUP FILTER CONDITION" VALUE="amount >= 10000"/>
  <!-- Group 2: MED -->
  <TABLEATTRIBUTE NAME="GROUP NAME"             VALUE="MED"/>
  <TABLEATTRIBUTE NAME="GROUP FILTER CONDITION" VALUE="amount >= 1000 AND amount &lt; 10000"/>
  <!-- Default group (LOW) — no GROUP FILTER CONDITION; catches all unmatched rows -->
</TRANSFORMATION>
```

**PySpark**
```python
# RTR_ORDER_SIZE  [Router]
# Default group (LOW) is the logical complement of all named groups
df_high = df_agg.filter(col("amount") >= 10000)
df_med  = df_agg.filter((col("amount") >= 1000) & (col("amount") < 10000))
df_low  = df_agg.filter(col("amount") < 1000)   # default group — no GROUP FILTER CONDITION in XML
```

---

## Update Strategy → Delta MERGE

**PowerCenter XML**
```xml
<TRANSFORMATION NAME="UPD_ORDERS" TYPE="Update Strategy">
  <TRANSFORMFIELD NAME="order_id" PORTTYPE="INPUT/OUTPUT" EXPRESSION="order_id" DATATYPE="number" PRECISION="10" SCALE="0"/>
  <TRANSFORMFIELD NAME="status"   PORTTYPE="INPUT/OUTPUT" EXPRESSION="status"   DATATYPE="varchar2" PRECISION="20" SCALE="0"/>
  <TRANSFORMFIELD NAME="amount"   PORTTYPE="INPUT/OUTPUT" EXPRESSION="amount"   DATATYPE="number" PRECISION="18" SCALE="2"/>
  <!-- The strategy expression determines the DML operation per row -->
  <TABLEATTRIBUTE NAME="Update Strategy Expression" VALUE="IIF(ISNULL(order_id), DD_INSERT, DD_UPDATE)"/>
  <TABLEATTRIBUTE NAME="Forward Rejected Rows"      VALUE="NO"/>
</TRANSFORMATION>
```

**PySpark**
```python
# UPD_ORDERS  [Update Strategy] — IIF(ISNULL(order_id), DD_INSERT, DD_UPDATE)
# Translate the Update Strategy Expression to a dd_flag column, then MERGE on the target
df_staged = (
    df_filtered
    .withColumn(
        "dd_flag",
        when(col("order_id").isNull(), lit(0)).otherwise(lit(1)),  # 0=DD_INSERT, 1=DD_UPDATE
    )
)

# TGT_FACT_ORDERS  [Target Definition] — Delta MERGE
target = DeltaTable.forName(spark, f"{catalog}.{schema}.fact_orders")

(
    target.alias("tgt")
    .merge(df_staged.alias("src"), "tgt.order_id = src.order_id")
    .whenMatchedUpdate(
        condition="src.dd_flag = 1",
        set={
            "status": "src.status",
            "amount": "src.amount",
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

---

## Sorter

**PowerCenter XML**
```xml
<TRANSFORMATION NAME="SRT_BY_DATE" TYPE="Sorter">
  <TRANSFORMFIELD NAME="order_id"   PORTTYPE="INPUT/OUTPUT" DATATYPE="number" PRECISION="10" SCALE="0"
                  SORTDIRECTION="ASCENDING"  NULLS="LAST"/>
  <TRANSFORMFIELD NAME="order_date" PORTTYPE="INPUT/OUTPUT" DATATYPE="date"   PRECISION="29" SCALE="9"
                  SORTDIRECTION="DESCENDING" NULLS="FIRST"/>
</TRANSFORMATION>
```

**PySpark**
```python
# SRT_BY_DATE  [Sorter]
df_sorted = (
    df_joined
    .orderBy(
        asc_nulls_last("order_id"),
        desc_nulls_first("order_date"),
    )
)
```

---

## Rank

**PowerCenter XML**
```xml
<TRANSFORMATION NAME="RNK_TOP3_ORDERS" TYPE="Rank">
  <!-- Group-by port -->
  <TRANSFORMFIELD NAME="customer_id" PORTTYPE="INPUT/OUTPUT" DATATYPE="number" PRECISION="10" SCALE="0"/>
  <!-- Ranked port — the value being ranked -->
  <TRANSFORMFIELD NAME="amount"      PORTTYPE="INPUT/OUTPUT" DATATYPE="number" PRECISION="18" SCALE="2"/>
  <!-- Rank output port -->
  <TRANSFORMFIELD NAME="RANKINDEX"   PORTTYPE="OUTPUT"       DATATYPE="number" PRECISION="10" SCALE="0"/>
  <TABLEATTRIBUTE NAME="Rank"       VALUE="TOP"/>
  <TABLEATTRIBUTE NAME="Top/Bottom" VALUE="3"/>
</TRANSFORMATION>
```

**PySpark**
```python
# RNK_TOP3_ORDERS  [Rank] — TOP 3 per customer_id by amount descending
_w = Window.partitionBy("customer_id").orderBy(desc("amount"))
df_ranked = (
    df_sorted
    .withColumn("RANKINDEX", rank().over(_w))
    .filter(col("RANKINDEX") <= 3)
)
```

---

## Union

**PowerCenter XML**
```xml
<TRANSFORMATION NAME="UNI_ALL_ORDERS" TYPE="Union">
  <!-- Input groups: each group corresponds to one upstream pipeline -->
  <TRANSFORMFIELD NAME="order_id"  PORTTYPE="INPUT" DATATYPE="number"   PRECISION="10" SCALE="0"/>
  <TRANSFORMFIELD NAME="amount"    PORTTYPE="INPUT" DATATYPE="number"   PRECISION="18" SCALE="2"/>
  <TRANSFORMFIELD NAME="source_cd" PORTTYPE="INPUT" DATATYPE="varchar2" PRECISION="10" SCALE="0"/>
  <!-- Output group — one port per column -->
  <TRANSFORMFIELD NAME="order_id"  PORTTYPE="OUTPUT" DATATYPE="number"   PRECISION="10" SCALE="0"/>
  <TRANSFORMFIELD NAME="amount"    PORTTYPE="OUTPUT" DATATYPE="number"   PRECISION="18" SCALE="2"/>
  <TRANSFORMFIELD NAME="source_cd" PORTTYPE="OUTPUT" DATATYPE="varchar2" PRECISION="10" SCALE="0"/>
</TRANSFORMATION>
```

**PySpark**
```python
# UNI_ALL_ORDERS  [Union] — UNION ALL of three input streams
df_union = (
    df_stream_a
    .unionByName(df_stream_b)
    .unionByName(df_stream_c)
)
```

---

## Sequence Generator

**PowerCenter XML**
```xml
<TRANSFORMATION NAME="SEQ_SURROGATE" TYPE="Sequence Generator" REUSABLE="NO">
  <TRANSFORMFIELD NAME="NEXTVAL" PORTTYPE="OUTPUT" DATATYPE="number" PRECISION="10" SCALE="0"/>
  <TRANSFORMFIELD NAME="CURRVAL" PORTTYPE="OUTPUT" DATATYPE="number" PRECISION="10" SCALE="0"/>
  <TABLEATTRIBUTE NAME="Start Value"   VALUE="1"/>
  <TABLEATTRIBUTE NAME="Increment By"  VALUE="1"/>
  <TABLEATTRIBUTE NAME="End Value"     VALUE="2147483647"/>
  <TABLEATTRIBUTE NAME="Reset"         VALUE="NO"/>
</TRANSFORMATION>
```

**PySpark**
```python
# SEQ_SURROGATE  [Sequence Generator]
# REVIEW: monotonically_increasing_id() is not sequential across runs.
# If the target requires a stable, gapless sequence, use a Delta IDENTITY column instead.
df_with_key = (
    df_union
    .withColumn("surrogate_key", monotonically_increasing_id())
)
```

---

## Normalizer (Unpivot)

**PowerCenter XML**
```xml
<TRANSFORMATION NAME="NRM_QUARTERLY_SALES" TYPE="Normalizer">
  <!-- Input: one row with four quarter columns -->
  <TRANSFORMFIELD NAME="product_id" PORTTYPE="INPUT/OUTPUT" DATATYPE="number"   PRECISION="10" SCALE="0"/>
  <TRANSFORMFIELD NAME="Q1_SALES"   PORTTYPE="INPUT"        DATATYPE="number"   PRECISION="18" SCALE="2" GCID="1"/>
  <TRANSFORMFIELD NAME="Q2_SALES"   PORTTYPE="INPUT"        DATATYPE="number"   PRECISION="18" SCALE="2" GCID="2"/>
  <TRANSFORMFIELD NAME="Q3_SALES"   PORTTYPE="INPUT"        DATATYPE="number"   PRECISION="18" SCALE="2" GCID="3"/>
  <TRANSFORMFIELD NAME="Q4_SALES"   PORTTYPE="INPUT"        DATATYPE="number"   PRECISION="18" SCALE="2" GCID="4"/>
  <!-- Output: one row per quarter -->
  <TRANSFORMFIELD NAME="SALES"      PORTTYPE="OUTPUT"       DATATYPE="number"   PRECISION="18" SCALE="2"/>
  <TRANSFORMFIELD NAME="GK_QUARTER" PORTTYPE="OUTPUT"       DATATYPE="number"   PRECISION="10" SCALE="0"/>
</TRANSFORMATION>
```

**PySpark**
```python
# NRM_QUARTERLY_SALES  [Normalizer] — unpivot four quarter columns into rows
# GCID values (1–4) map to GK_QUARTER; each row in the output has one (GK_QUARTER, SALES) pair
df_normalized = (
    df_raw
    .select(
        "product_id",
        expr("stack(4, 1, Q1_SALES, 2, Q2_SALES, 3, Q3_SALES, 4, Q4_SALES) AS (GK_QUARTER, SALES)"),
    )
)
```

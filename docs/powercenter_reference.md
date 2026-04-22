# PowerCenter XML Reference

This document describes the structure of Informatica PowerCenter XML exports and catalogs every component the conversion agent will encounter.

## XML Export Structure

A PowerCenter export file has this hierarchy:

```
POWERMART
└── REPOSITORY (name, version)
    └── FOLDER (name, description)
        ├── SOURCE            — source table/file definitions
        ├── TARGET            — target table definitions
        ├── TRANSFORMATION    — reusable transformation objects (e.g., mapplets, reusable lookups)
        └── MAPPING
            ├── TRANSFORMATION  — inline transformations for this mapping
            ├── CONNECTOR       — data flow connections between transformations
            └── TARGETLOADORDER — load sequence for targets
```

## Core XML Elements

### `<MAPPING>`

| Attribute | Description |
|---|---|
| `NAME` | Unique mapping name within the folder |
| `DESCRIPTION` | Human-readable purpose |
| `ISVALID` | `YES` / `NO` — only convert valid mappings |

### `<TRANSFORMATION>`

Every node in the mapping's DAG is a `<TRANSFORMATION>`.

| Attribute | Description |
|---|---|
| `TYPE` | Transformation type (see catalog below) |
| `NAME` | Instance name within the mapping |
| `DESCRIPTION` | Optional notes |
| `REUSABLE` | `YES` if defined at folder level and referenced here |

Children of `<TRANSFORMATION>`:
- `<TRANSFORMFIELD>` — defines each input/output port
- `<TABLEATTRIBUTE>` — key/value configuration properties

### `<TRANSFORMFIELD>`

| Attribute | Description |
|---|---|
| `NAME` | Port name |
| `DATATYPE` | PowerCenter datatype (see datatype table below) |
| `PRECISION` | Column precision |
| `SCALE` | Column scale |
| `PORTTYPE` | `INPUT`, `OUTPUT`, `INPUT/OUTPUT`, `VARIABLE` |
| `EXPRESSION` | Expression string (Expression, Filter, Update Strategy transformations) |
| `EXPRESSIONTYPE` | `GENERAL`, `PARAMETER` |

### `<CONNECTOR>`

Defines a directed edge in the data flow graph.

| Attribute | Description |
|---|---|
| `FROMINSTANCE` | Source transformation name |
| `FROMINSTANCETYPE` | Source transformation type |
| `FROMFIELD` | Output port name on source |
| `TOINSTANCE` | Target transformation name |
| `TOINSTANCETYPE` | Target transformation type |
| `TOFIELD` | Input port name on target |

### `<TABLEATTRIBUTE>`

Key/value pairs that configure a transformation's behavior.

```xml
<TABLEATTRIBUTE NAME="Sql Query" VALUE="SELECT * FROM orders WHERE status = 'ACTIVE'"/>
<TABLEATTRIBUTE NAME="Connection Information" VALUE="$DBConnection_Orders"/>
<TABLEATTRIBUTE NAME="Cache Type" VALUE="Static"/>
```

---

## Transformation Type Catalog

### Source Qualifier (`Source Qualifier`)

Represents the SQL query against a relational source or the read definition for a flat file. Always sits between source definitions and downstream transformations.

Key `<TABLEATTRIBUTE>` values:

| Attribute | Meaning |
|---|---|
| `Sql Query` | Custom override SQL; empty = auto-generated SELECT |
| `User Defined Join` | Additional JOIN clauses |
| `Source Filter` | WHERE clause appended to the query |
| `Number Of Sorted Ports` | Columns used for ORDER BY |
| `Tracing Level` | Logging verbosity (ignore in conversion) |

### Target Definition (`Target Definition` / written as the target instance)

Describes the destination table. The actual write behavior is controlled by the **Update Strategy** transformation feeding into it.

Key `<TABLEATTRIBUTE>` values:

| Attribute | Meaning |
|---|---|
| `Target Load Type` | `Normal` or `Bulk` |
| `Insert` | `true` / `false` |
| `Update` | `true` / `false` |
| `Delete` | `true` / `false` |
| `Truncate Target Table` | `true` / `false` |

### Expression (`Expression`)

Applies column-level transformations. Each output port carries an `EXPRESSION` attribute containing PowerCenter expression language.

Ports can be `INPUT`, `OUTPUT`, or `VARIABLE` (intermediate values not passed downstream).

### Filter (`Filter`)

Contains a single `FILTEROVERRIDE` port whose `EXPRESSION` evaluates to TRUE/FALSE. Rows where the expression is FALSE are dropped.

### Joiner (`Joiner`)

Joins two input pipelines (master and detail).

Key `<TABLEATTRIBUTE>` values:

| Attribute | Meaning |
|---|---|
| `Join Type` | `Normal`, `Master Outer`, `Detail Outer`, `Full Outer` |
| `Join Condition` | The ON clause |
| `Sorted Input` | `true` if inputs are pre-sorted |
| `Case Sensitive String Comparison` | `true` / `false` |

Input ports tagged with `INPUT` and prefixed `MASTER:` / `DETAIL:` identify which pipeline is which.

### Lookup (`Lookup Procedure`)

Looks up values from a relational table or flat file.

Key `<TABLEATTRIBUTE>` values:

| Attribute | Meaning |
|---|---|
| `Lookup table name` | Table to look up against |
| `Lookup Caching Enabled` | `YES` / `NO` |
| `Lookup Condition` | The join condition |
| `Lookup Policy on Multiple Match` | `Use first value`, `Use last value`, `Report error` |
| `Connection Information` | Database connection variable |
| `Sql Override` | Custom SQL replacing the auto-generated lookup query |

Connected lookups pass matched values downstream via output ports. Unconnected lookups are called from Expression ports using `:LKP.<name>(key)` syntax.

### Aggregator (`Aggregator`)

Performs GROUP BY aggregations. Group-by ports have `PORTTYPE=INPUT`; aggregated ports have expressions like `SUM(amount)`, `COUNT(*)`, `MAX(date)`.

Key `<TABLEATTRIBUTE>` values:

| Attribute | Meaning |
|---|---|
| `Cache Directory` | Ignore in conversion |
| `Cache Size` | Ignore in conversion |
| `Sorted Input` | `true` if data arrives pre-grouped |

### Router (`Router`)

Splits one input stream into multiple output groups based on filter conditions. Each group is a `<TRANSFORMFIELD>` with a `GROUP FILTER CONDITION` expression. The default group captures unmatched rows.

### Sorter (`Sorter`)

Orders rows. Each port has a `SORTDIRECTION` attribute (`ASCENDING` / `DESCENDING`) and a `NULLS FIRST/LAST` option.

### Rank (`Rank`)

Selects top-N or bottom-N rows per group. Equivalent to PySpark `Window.rank()` or `row_number()`.

Key `<TABLEATTRIBUTE>` values:

| Attribute | Meaning |
|---|---|
| `Rank` | `TOP` or `BOTTOM` |
| `Top/Bottom` (count) | N rows to keep per group |
| `Cache Directory` | Ignore |

### Update Strategy (`Update Strategy`)

Controls the DML operation applied to the target (Insert, Update, Delete, Reject). The `EXPRESSION` on the strategy port evaluates to one of the integer constants:

| Constant | Meaning |
|---|---|
| `DD_INSERT` (0) | INSERT |
| `DD_UPDATE` (1) | UPDATE |
| `DD_DELETE` (2) | DELETE |
| `DD_REJECT` (3) | Reject row |

Always translates to a Delta Lake `MERGE` statement.

### Union (`Union`)

Combines multiple input pipelines with the same schema into one (equivalent to SQL `UNION ALL`).

### Sequence Generator (`Sequence Generator`)

Generates sequential numeric keys. Key ports: `NEXTVAL`, `CURRVAL`.

Key `<TABLEATTRIBUTE>` values:

| Attribute | Meaning |
|---|---|
| `Start Value` | Starting integer |
| `Increment By` | Step size |
| `End Value` | Upper limit (0 = unlimited) |
| `Reset` | Cycle back to start when end is reached |
| `Number of Cached Values` | Ignore in conversion |

### Normalizer (`Normalizer`)

Pivots repeated column groups in a single row into multiple rows (unpivot). Each `GCID` column represents one occurrence of the repeating group.

### Mapplet (`Mapplet`)

A reusable sub-mapping. Treated as a nested graph. Translate to a `%run ./helper_notebook` reference or a Python helper function, depending on complexity.

### Stored Procedure (`Stored Procedure`)

Calls a database stored procedure. **Requires manual review** — translate to a `spark.sql("CALL ...")` if the procedure is available in the target catalog, otherwise flag as `# REVIEW: STORED PROCEDURE`.

---

## PowerCenter Datatype → Spark/Delta Mapping

| PowerCenter Type | Spark Type | Notes |
|---|---|---|
| `string` | `StringType` | |
| `nstring` | `StringType` | Unicode, no change needed in Spark |
| `char` | `StringType` | |
| `varchar` | `StringType` | |
| `nvarchar` | `StringType` | |
| `integer` | `IntegerType` | |
| `small integer` | `ShortType` | |
| `bigint` | `LongType` | |
| `decimal` | `DecimalType(p, s)` | Use precision/scale from field definition |
| `double` | `DoubleType` | |
| `real` | `FloatType` | |
| `float` | `DoubleType` | PowerCenter float is 64-bit |
| `date/time` | `TimestampType` | |
| `date` | `DateType` | |
| `time` | `StringType` | No native Time type in Spark; store as `HH:mm:ss` string |
| `binary` | `BinaryType` | |
| `long` | `StringType` | Large text; use StringType |
| `ntext` | `StringType` | |

---

## PowerCenter Expression Language — Key Constructs

### Conditional
```
IIF(condition, true_value, false_value)
DECODE(value, search1, result1 [, search2, result2, ...] [, default])
```

### String
```
LTRIM(string [, trim_set])
RTRIM(string [, trim_set])
TRIM(string)
UPPER(string)
LOWER(string)
LENGTH(string)
SUBSTR(string, start [, length])
INSTR(string, search [, start [, occurrence]])
CONCAT(s1, s2)
RPAD(string, length [, pad])
LPAD(string, length [, pad])
REG_REPLACE(subject, pattern, replacement)
```

### Numeric
```
ROUND(value [, places])
TRUNC(value [, places])
ABS(value)
CEIL(value)
FLOOR(value)
MOD(dividend, divisor)
POWER(base, exponent)
SQRT(value)
```

### Date
```
TO_DATE(string, format)
TO_CHAR(date, format)
ADD_TO_DATE(date, format_string, amount)
DATE_DIFF(date1, date2, format_string)
LAST_DAY(date)
SYSDATE    -- current date/time
```

### Aggregation (Aggregator only)
```
SUM(port [, filter])
COUNT(port [, filter])
AVG(port [, filter])
MIN(port [, filter])
MAX(port [, filter])
FIRST(port [, filter])
LAST(port [, filter])
STDDEV(port)
VARIANCE(port)
```

### Special
```
:LKP.<lookup_name>(key_port)   -- unconnected lookup call
:SEQ.<seq_name>.NEXTVAL        -- sequence next value
NULL                           -- null literal
ISNULL(port)                   -- null check
```

---

## Common `<TABLEATTRIBUTE>` Values to Ignore

These attributes exist in the XML but have no equivalent in Databricks:

- `Tracing Level`
- `Cache Directory`
- `Cache Size`
- `Enable High Precision`
- `Generate Transaction`
- `Output is Deterministic`
- `Output is Repeatable`
- `Number of Cached Values` (Sequence Generator)
- `Pre-load all data` (Lookup)

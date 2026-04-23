# Conversion Workflow: PowerCenter → Databricks Notebook

This document describes the end-to-end **human operator** workflow — what you do before, during, and after running the agent. The agent's internal execution model (4 phases: Review & Analysis, Extraction, Notebook Creation, Cell Review) is defined in `AGENT.md` and `templates/mega_prompt.md`. Follow these steps in order.

---

## Overview

```
1. Intake & Validation
       ↓
2. XML Parse & Graph Construction
       ↓
3. Source Analysis
       ↓
4. Transformation Translation
       ↓
5. Target Write Assembly
       ↓
6. Notebook Assembly
       ↓
7. Automated Validation
       ↓
8. Human Review (REVIEW items)
       ↓
9. Deployment
```

---

## Step 1: Intake & Validation

**Input:** A PowerCenter XML export file (`.xml`) produced by PowerCenter Designer or Repository Manager.

**Checks before proceeding:**

1. The XML is well-formed (parseable).
2. The `<MAPPING ISVALID="YES">` attribute is present. Do not convert invalid mappings.
3. Identify the target folder and mapping name from `<FOLDER NAME="...">` and `<MAPPING NAME="...">`.
4. Confirm the mapping version is supported (PowerCenter 9.x or 10.x XML schemas are supported; 8.x may have structural differences).
5. Document the source system type(s) from `<SOURCE>` definitions.
6. Document the target system type(s) from `<TARGET>` definitions.

**Deliverable:** A pre-conversion summary listing mapping name, source count, target count, and transformation count.

---

## Step 2: XML Parse & Data Flow Graph Construction

**Goal:** Build a directed acyclic graph (DAG) of all transformations and their connections.

**Process:**

1. Extract all `<TRANSFORMATION>` elements and index them by `NAME`.
2. Extract all `<CONNECTOR>` elements and build an adjacency list:
   - Key: `(FROMINSTANCE, FROMFIELD)` → Value: `(TOINSTANCE, TOFIELD)`
3. Identify root nodes (transformations with no incoming connectors) — these are your source nodes.
4. Identify leaf nodes (transformations with no outgoing connectors feeding another transformation) — these are typically the target instances or the last step before targets.
5. Perform a topological sort of the graph. This determines notebook cell order.
6. Identify any parallel branches (Router outputs, multiple Source Qualifiers). These need to be handled as separate DataFrame lineages that eventually merge (via Joiner or Union) or write to separate targets.

**Tools:** Use Python's `xml.etree.ElementTree` or `lxml` to parse. Use a simple adjacency list for the graph.

**Output:** An ordered list of transformation steps and a map of DataFrame names to their upstream source.

---

## Step 3: Source Analysis

For each `<SOURCE>` definition in the XML:

1. Identify the source type: `RELATIONAL`, `FILE`, `APPLICATION`, or `TRANSFORMATION` (for mapplet inputs).
2. For relational sources, map the connection variable (e.g., `$DBConnection_Orders`) to a Unity Catalog reference or JDBC configuration. Document unmapped connections as `# REVIEW`.
3. For file sources, determine the format (CSV, fixed-width, XML, JSON) from `<TABLEATTRIBUTE>` values.
4. For the corresponding Source Qualifier:
   - If `Sql Query` is non-empty, that SQL becomes the read query.
   - If empty, generate `spark.table(...)` for the full table.
   - If `Source Filter` is non-empty, append as a `.filter()`.
   - If `User Defined Join` is non-empty, incorporate into the SQL.

---

## Step 4: Transformation Translation

Process transformations in topological order. For each transformation:

1. Look up its `TYPE` in `docs/transformation_mappings.md`.
2. Read its `<TRANSFORMFIELD>` definitions to understand input/output ports.
3. Read its `<TABLEATTRIBUTE>` values for configuration.
4. Apply the translation pattern from the mapping table.
5. Assign the result to a DataFrame variable using the naming convention in `docs/conversion_standards.md`.
6. If the transformation type is unsupported or ambiguous, add a `# REVIEW:` comment and a best-effort approximation.

**Key decisions per transformation type:**

| Type | Key Decision |
|---|---|
| Source Qualifier | SQL override vs. table read; custom joins |
| Expression | One `withColumn()` per output port; drop VARIABLE ports after use |
| Filter | Single `.filter()` — translate expression from PowerCenter language |
| Joiner | Determine join type and which pipeline is master vs. detail |
| Lookup | Connected vs. unconnected; broadcast size estimate; multi-match policy |
| Aggregator | Identify GROUP BY ports vs. aggregation ports; conditional aggregation |
| Router | One branch per group; default group is exclusive residual |
| Update Strategy | Map DD_INSERT/UPDATE/DELETE to MERGE whenMatched/whenNotMatched clauses |
| Sequence Generator | Flag non-sequential behavior; recommend Delta IDENTITY if sequential required |

---

## Step 5: Target Write Assembly

For each `<TARGET>` instance in the mapping:

1. Identify the Update Strategy (if any) upstream from the target.
2. If an Update Strategy exists → use Delta MERGE.
3. If no Update Strategy and `Truncate Target Table = true` → use `mode("overwrite")`.
4. If no Update Strategy and `Truncate Target Table = false` → use `mode("append")`.
5. Determine the primary key columns from the Update Strategy's join condition.
6. Enumerate the columns to insert/update explicitly (no `SELECT *`).
7. Respect `<TARGETLOADORDER>` — write targets in the specified sequence.

---

## Step 6: Notebook Assembly

Assemble the final notebook in this cell order:

1. Header markdown (mapping name, source, target, description, REVIEW checklist)
2. Widget definitions
3. Imports
4. Source read cells (one per SQ, in topological order)
5. Transformation cells (in topological order)
6. Target write cells (in `TARGETLOADORDER` sequence)
7. Validation cell (if applicable)

Format as a `.py` file with `# COMMAND ----------` cell delimiters.

---

## Step 7: Automated Validation

Run the following checks on the assembled notebook before returning it:

| Check | How to Verify |
|---|---|
| No PowerCenter syntax | Grep for `:LKP.`, `:SEQ.`, `$$`, `IIF(`, `DECODE(`, `DD_INSERT`, `DD_UPDATE`, `DD_DELETE` |
| All ports connected | Every `<CONNECTOR FROMFIELD>` has a corresponding column operation |
| No hardcoded credentials | Grep for connection strings, passwords, hostnames |
| No hardcoded paths | All paths use `dbutils.widgets.get(...)` |
| Widget cell is present | Cell 2 must contain `dbutils.widgets.text(...)` calls |
| REVIEW items in header | Count `# REVIEW:` occurrences and verify they appear in the header markdown |
| Imports match usage | All imported functions appear in the notebook body |

---

## Step 8: Human Review

After automated validation, produce a **Conversion Summary Report**:

```
Mapping:          M_LOAD_FACT_ORDERS
Folder:           ETL_ORDERS
Source Count:     2 (Oracle EBS, flat file)
Target Count:     1 (Delta table: catalog.schema.fact_orders)
Transformations:  14 (12 translated, 2 flagged for REVIEW)
Complexity:       Medium
REVIEW Items:
  1. Stored procedure SP_VALIDATE_ORDER not translated (cell: TGT_FACT_ORDERS)
  2. Sequence generator — non-sequential IDs (cell: EXP_SURROGATE_KEY)
```

Complexity scoring:

| Score | Criteria |
|---|---|
| Low | ≤ 5 transformations, no Update Strategy, no Lookups, no REVIEW items |
| Medium | 6–15 transformations, or has Update Strategy / Lookups, ≤ 2 REVIEW items |
| High | > 15 transformations, or Stored Procedures, or > 2 REVIEW items |

---

## Step 9: Deployment

Once human review is complete:

1. The generated notebook is already in `notebooks/nb_<mapping_name>.py` in this repo. Pull the latest changes via Databricks Repos so the file is visible in your workspace.
2. Open the notebook and run it in interactive mode with `catalog=dev_catalog` to validate end-to-end execution.
3. Fix any runtime errors (schema mismatches, missing tables, type cast failures).
4. Parameterize the notebook in a Databricks Job with environment-specific widget values.
5. Run a data reconciliation check: compare row counts and key metrics between the PowerCenter output and the Databricks notebook output against the same source data.

---

## Handling Multi-Mapping Folders

When converting an entire PowerCenter folder (multiple mappings):

1. Place all XML files — mappings and mapplets — in the `input/` folder before starting. Use the standard naming conventions (`M_` for mappings, `mlt_` for mapplets).
2. Convert in dependency order: run mapplet conversions first, then mappings that reference them.
3. Use the Batch Conversion Template in `templates/conversion_prompt_template.md` for multi-mapping runs.
4. Use a shared `config/` notebook for common widget defaults and connection references.
5. Maintain a `_conversion_log.md` file in the `notebooks/` folder tracking conversion status per mapping.

---

## Escalation Criteria

Stop conversion and escalate to a human when:

- `ISVALID="NO"` — the mapping was not valid in PowerCenter; do not guess the intent.
- The XML contains a transformation type not in `docs/transformation_mappings.md` and no approximation is reasonable.
- The source type is a custom adapter (SAP, MQ, Salesforce, mainframe VSAM) with no Databricks-native connector.
- The mapping contains a PowerCenter real-time / streaming source (CDC, JMS) — this requires Databricks Structured Streaming, which is out of scope for batch conversion.
- More than 30% of transformations are flagged as `# REVIEW`.

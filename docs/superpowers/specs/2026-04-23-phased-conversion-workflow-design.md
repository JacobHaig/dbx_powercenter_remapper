# Phased Conversion Workflow Design

**Date:** 2026-04-23
**Status:** Approved

---

## Problem

`templates/mega_prompt.md` currently describes the agent's work as two flat steps (Step 3: Output, Step 4: Quality Check). This gives the agent no structured thinking order, no intermediate checkpoints, and no visibility into what it extracted or how it interpreted the XML before it writes code. Errors discovered late (wrong DAG order, missed expression translation) require a full restart.

Additionally, the output spec calls for a `.py` file but is missing the `# Databricks notebook source` first line required for Databricks to recognize the file as a native notebook.

---

## Solution

Replace Steps 3 and 4 in `templates/mega_prompt.md` with 4 explicit execution phases. Each phase has a defined scope, a required output the agent must produce before proceeding, and a clear entry/exit condition. Update `AGENT.md` to match.

---

## Files Changed

| File | Change |
|---|---|
| `templates/mega_prompt.md` | Replace Steps 3 & 4 with Phases 1–4; add `# Databricks notebook source` to notebook spec |
| `AGENT.md` | Update execution section to match phased approach |

No other files change.

---

## Phase Definitions

### Phase 1 — Review & Analysis

**Scope:** Parse and validate the XML. Build the data flow graph. Assess the mapping before any extraction or code generation begins.

**Steps the agent takes:**
1. Confirm `<MAPPING ISVALID="YES">` — if NO, stop and report to the user
2. Extract `FOLDER NAME` and `MAPPING NAME`
3. Walk all `<CONNECTOR>` elements to build the full DAG adjacency list
4. Perform a topological sort — confirm no cycles
5. Inventory every `<TRANSFORMATION>` by type
6. Flag any transformation types not in `docs/transformation_mappings.md` as unsupported
7. Score complexity (High/Medium/Low, first-matching-rule-wins)

**Required output before proceeding to Phase 2:**
```
Phase 1 — Analysis Summary
Mapping:         <name>
Folder:          <folder>
Sources:         <n> (<types>)
Targets:         <n> (<names>)
Transformations: <total> (<type x count, ...>)
Unsupported:     <name> (<type>) — will flag as REVIEW   [omit line if none]
Complexity:      Low / Medium / High
```

---

### Phase 2 — Extraction

**Scope:** Extract every piece of useful information from the XML into a complete internal picture. Nothing is left unread.

**What is extracted:**
- All `<SOURCE>` definitions: name, database type, every column with datatype, precision, scale, key type
- All `<TARGET>` definitions: name, table name, every column with datatype, load type (Normal/Bulk), truncate flag
- All `<TRANSFORMATION>` definitions:
  - Every `<TRANSFORMFIELD>`: name, PORTTYPE, DATATYPE, EXPRESSION, EXPRESSIONTYPE, precision, scale, GCID (Normalizer), SORTDIRECTION (Sorter)
  - Every `<TABLEATTRIBUTE>`: name → value (SQL overrides, join conditions, filter conditions, cache settings, join type, rank direction, etc.)
- Full connector adjacency list (FROMINSTANCE/FROMFIELD → TOINSTANCE/TOFIELD)
- `<TARGETLOADORDER>` sequence
- All connection variables (e.g. `$DBConnection_Orders`) mapped to their Unity Catalog paths from the Connection Mapping table in Step 2
- All PowerCenter parameters (`$$PARAM_NAME`, mapping variables) mapped to widget names
- All expression strings requiring translation

**Required output before proceeding to Phase 3:**
```
Phase 2 — Extraction Complete
Sources extracted:          <n> (<total columns> columns)
Targets extracted:          <n> (<total columns> columns)
Transformations extracted:  <n> (all fields and attributes)
Connectors mapped:          <n> edges — DAG validated, no cycles
Connection variables:       <$Var> → <catalog.schema>  [one line per variable]
Parameters found:           <$$PARAM> → <widget_name>  [one line per param, omit if none]
```

---

### Phase 3 — Notebook Creation

**Scope:** Write the complete Databricks native notebook using the extracted data and the patterns in the reference docs.

**Required first line of every notebook:**
```
# Databricks notebook source
```

**Cell separator:** `# COMMAND ----------`

**Cell order (mandatory):**
1. Markdown header cell — mapping name, source system, target table, original mapping description, REVIEW checklist (one item per `# REVIEW:` flag that will appear in the notebook)
2. Widget definitions cell — one `dbutils.widgets.text(...)` per environment-dependent value; all connection paths, catalog, schema, and run_date come from widgets
3. Imports cell — only functions actually used; no speculative imports
4. Source read cells — one cell per Source Qualifier, named `sq_<source_name>`
5. Transformation cells — one cell per logical transformation group, in topological DAG order, named per the convention in `docs/conversion_standards.md`
6. Target write cells — in `<TARGETLOADORDER>` sequence; Delta MERGE for any target upstream of an Update Strategy, `mode("overwrite")` for truncate-reload, `mode("append")` for insert-only
7. Validation cell — row count assertions (include only if the mapping contained count-checking logic)

**No output pause after Phase 3.** Proceed directly to Phase 4.

---

### Phase 4 — Cell Review

**Scope:** Walk every cell in the drafted notebook against the quality checklist. Correct any issues found. Then deliver.

**Checklist applied to every cell:**
- [ ] No PowerCenter syntax remains: `IIF(`, `DECODE(`, `:LKP.`, `:SEQ.`, `$$`, `DD_INSERT`, `DD_UPDATE`, `DD_DELETE`, `DD_REJECT`, `SYSDATE`, `ADD_TO_DATE(`, `DATE_DIFF(`
- [ ] All `<TRANSFORMFIELD>` output ports are accounted for downstream
- [ ] All catalog names, schema names, table names, and file paths come from `dbutils.widgets.get(...)` — none hardcoded
- [ ] Every Update Strategy transformation produced a Delta `MERGE` statement
- [ ] Lookup transformations use `broadcast()` where the source is expected to be under ~500 MB
- [ ] Every `# REVIEW:` comment in the body is listed in the header cell markdown checklist
- [ ] Cell order matches DAG: sources → transformations → targets

**Required output — Phase 4 summary, then final deliverables:**

```
Phase 4 — Review Summary
Cells reviewed:  <n>
Issues found:    <n> (<brief description of each correction made>)   [or "none"]
REVIEW flags:    <n> (<brief description>)                           [or "none"]
Status:          Ready
```

Then deliver:

1. **The notebook** — complete `.py` file, first line `# Databricks notebook source`, cells separated by `# COMMAND ----------`. File name: `nb_<mapping_name_lowercase>.py` (spaces and special characters → underscores).

2. **Conversion Summary:**
```
Mapping:          <MAPPING NAME>
Folder:           <FOLDER NAME>
Sources:          <n> (<types>)
Targets:          <n> (<table names>)
Transformations:  <total> (<n> translated, <n> flagged for REVIEW)
Complexity:       Low / Medium / High
REVIEW Items:
  1. <description> (cell: <cell comment>)
```

---

## Complexity Scoring (used in Phase 1)

Evaluate in order — first matching rule wins:
- **High** — >15 transformations, OR Stored/External Procedures present, OR >2 REVIEW items
- **Medium** — 6–15 transformations, OR has Update Strategy or Lookups, OR ≤2 REVIEW items
- **Low** — all other cases

---

## What Does Not Change

- Your Role section (mega_prompt.md)
- Step 1: Read These Files First (mega_prompt.md)
- Step 2: Fill In the Conversion Request (mega_prompt.md)
- Quick Reference: Transformation Types table (mega_prompt.md)
- All `docs/` reference files

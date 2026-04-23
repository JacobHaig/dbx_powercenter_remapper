# Phased Conversion Workflow Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the flat Step 3/4 output instructions in `templates/mega_prompt.md` with 4 explicit execution phases, add the required `# Databricks notebook source` first line to the notebook spec, and update `AGENT.md` to match.

**Architecture:** Two markdown files change. `mega_prompt.md` is the Genie Code entry point — Steps 3 and 4 (lines 72–124) are removed and replaced with Phases 1–4. The Quick Reference table at line 126 onwards is untouched. `AGENT.md` is the human-readable operating manual — its `Conversion Rules` section is replaced with the same 4 phases (concise form) and `Session Outputs` is updated to reflect the new delivery format.

**Tech Stack:** Markdown only. No code.

---

### Task 1: Replace Steps 3 & 4 in `templates/mega_prompt.md` with Phases 1–4

**Files:**
- Modify: `templates/mega_prompt.md` — remove lines 72–124, insert 4 phases

- [ ] **Step 1: Remove the existing Step 3 (Output) section**

In `templates/mega_prompt.md`, delete everything from the line `## Step 3: Output` through the closing `---` separator before `## Step 4: Quality Check`. That is, remove this block:

```
## Step 3: Output

Return exactly two things — nothing else:

### 1. The Notebook

A complete `.py` file using Databricks `# COMMAND ----------` cell delimiters.
File name: `nb_<mapping_name_lowercase>.py`
(Replace spaces and special characters with underscores. Example: `M_Orders_v2 (PROD)` → `nb_m_orders_v2_prod.py`)

Follow the cell order from `docs/conversion_standards.md`:
1. Markdown header cell (mapping name, source, target, description, REVIEW checklist)
2. Widget definitions cell
3. Imports cell
4. Source read cells (one per Source Qualifier)
5. Transformation cells (in DAG order)
6. Target write cells (in TARGETLOADORDER sequence)
7. Validation cell (if the mapping had count checks)

### 2. Conversion Summary

```
Mapping:          <MAPPING NAME from XML>
Folder:           <FOLDER NAME from XML>
Sources:          <count> (<source system types>)
Targets:          <count> (<target table names>)
Transformations:  <total> (<n> translated, <n> flagged for REVIEW)
Complexity:       Low / Medium / High
REVIEW Items:
  1. <description> (cell: <cell comment>)
  2. ...
```

Complexity scoring (evaluate in order — first matching rule wins):
- **High** — >15 transformations, OR Stored/External Procedures present, OR >2 REVIEW items
- **Medium** — 6–15 transformations, OR has Update Strategy or Lookups, OR ≤2 REVIEW items
- **Low** — all other cases (≤5 transformations, no Update Strategy, no Lookups, zero REVIEW items)

---

## Step 4: Quality Check

Run every item on this checklist against the drafted notebook. If any check fails, revise the notebook to fix it before returning. Do not return output that fails any check.

- [ ] No PowerCenter syntax remains — search for: `IIF(`, `DECODE(`, `:LKP.`, `:SEQ.`, `$$`, `DD_INSERT`, `DD_UPDATE`, `DD_DELETE`, `DD_REJECT`, `SYSDATE`, `ADD_TO_DATE(`, `DATE_DIFF(`
- [ ] All `<TRANSFORMFIELD>` output ports are accounted for in downstream cells
- [ ] All source table names, target table names, catalog names, and schema names come from `dbutils.widgets.get(...)` — none are hardcoded string literals
- [ ] Every Update Strategy transformation produced a Delta `MERGE` (`.merge(...)`) statement
- [ ] Lookup transformations use `broadcast()` for sources expected to be under ~500 MB
- [ ] Every `# REVIEW:` comment in the notebook body is listed as a checklist item in the header markdown cell
- [ ] Cell order matches the data flow DAG: sources → transformations → targets

---
```

- [ ] **Step 2: Insert the 4 phases in place of the removed content**

In `templates/mega_prompt.md`, at the location where Steps 3 and 4 were removed (between Step 2's closing `---` and the `## Quick Reference` section), insert this exact content:

````markdown
## Phase 1: Review & Analysis

Parse and validate the XML. Build the data flow graph. Assess the mapping before any extraction or code generation begins.

1. Confirm `<MAPPING ISVALID="YES">` — if NO, stop immediately and tell the user: "This mapping is marked ISVALID=NO in PowerCenter. Conversion cannot proceed until the mapping is valid."
2. Extract `FOLDER NAME` and `MAPPING NAME`
3. Walk all `<CONNECTOR>` elements and build the full DAG adjacency list (FROMINSTANCE/FROMFIELD → TOINSTANCE/TOFIELD)
4. Perform a topological sort — confirm no cycles
5. Inventory every `<TRANSFORMATION>` by type
6. Flag any transformation types not in `docs/transformation_mappings.md` as unsupported
7. Score complexity using the first-matching-rule order:
   - **High** — >15 transformations, OR Stored/External Procedures present, OR >2 REVIEW items expected
   - **Medium** — 6–15 transformations, OR has Update Strategy or Lookups, OR ≤2 REVIEW items expected
   - **Low** — all other cases

Output this summary before proceeding to Phase 2:

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

## Phase 2: Extraction

Extract every piece of useful information from the XML. Nothing is left unread.

Extract ALL of the following:
- All `<SOURCE>` definitions — name, database type, every column with datatype, precision, scale, key type
- All `<TARGET>` definitions — name, table name, every column with datatype, load type (Normal/Bulk), truncate flag
- All `<TRANSFORMATION>` definitions:
  - Every `<TRANSFORMFIELD>`: name, PORTTYPE, DATATYPE, EXPRESSION, EXPRESSIONTYPE, precision, scale, GCID (Normalizer), SORTDIRECTION (Sorter)
  - Every `<TABLEATTRIBUTE>`: name → value (SQL overrides, join conditions, filter conditions, join type, rank direction, cache type, etc.)
- Full connector adjacency list
- `<TARGETLOADORDER>` sequence
- All connection variables (e.g. `$DBConnection_Orders`) — map to the Unity Catalog paths from the Connection Mapping table above
- All PowerCenter parameters (`$$PARAM_NAME`, mapping variables) — map to widget names
- All expression strings requiring translation

Output this manifest before proceeding to Phase 3:

```
Phase 2 — Extraction Complete
Sources extracted:          <n> (<total columns> columns)
Targets extracted:          <n> (<total columns> columns)
Transformations extracted:  <n> (all fields and attributes)
Connectors mapped:          <n> edges — DAG validated, no cycles
Connection variables:       <$Var> → <catalog.schema>
Parameters found:           <$$PARAM> → <widget_name>   [omit line if none]
```

---

## Phase 3: Notebook Creation

Write the complete Databricks native notebook using the extracted data and the patterns from the four reference docs.

**The very first line of the notebook must be:**
```
# Databricks notebook source
```

**Cell separator:** `# COMMAND ----------`

**Mandatory cell order:**
1. Markdown header cell — mapping name, source system, target table, original mapping description, REVIEW checklist (one `- [ ]` item per `# REVIEW:` flag that will appear in the notebook body)
2. Widget definitions cell — one `dbutils.widgets.text(...)` per environment-dependent value; all catalog names, schema names, connection paths, and run_date come from widgets, nothing hardcoded
3. Imports cell — only functions actually used in this notebook; no speculative imports
4. Source read cells — one cell per Source Qualifier, variable named `sq_<source_name>`
5. Transformation cells — one cell per logical transformation group, in topological DAG order, named per `docs/conversion_standards.md`
6. Target write cells — in `<TARGETLOADORDER>` sequence; Delta MERGE for any target upstream of an Update Strategy, `mode("overwrite")` for truncate-reload, `mode("append")` for insert-only
7. Validation cell — row count assertions (include only if the mapping contained Aggregator-based count checks or session-level row count validation)

Do not pause or output anything after Phase 3. Proceed directly to Phase 4.

---

## Phase 4: Cell Review

Walk every cell in the drafted notebook against this checklist. Correct any issues found inline. Then deliver.

For each cell, check:
- [ ] No PowerCenter syntax remains — search for: `IIF(`, `DECODE(`, `:LKP.`, `:SEQ.`, `$$`, `DD_INSERT`, `DD_UPDATE`, `DD_DELETE`, `DD_REJECT`, `SYSDATE`, `ADD_TO_DATE(`, `DATE_DIFF(`
- [ ] All `<TRANSFORMFIELD>` output ports are accounted for in downstream cells
- [ ] All catalog names, schema names, table names, and file paths come from `dbutils.widgets.get(...)` — none hardcoded
- [ ] Every Update Strategy transformation produced a Delta `MERGE` (`.merge(...)`) statement
- [ ] Lookup transformations use `broadcast()` for sources expected under ~500 MB
- [ ] Every `# REVIEW:` comment in the body is listed as a checklist item in the header markdown cell
- [ ] Cell order matches DAG: sources → transformations → targets

Output the review summary, then deliver the notebook and conversion summary:

```
Phase 4 — Review Summary
Cells reviewed:  <n>
Issues found:    <n> (<brief description of each correction>)   [or "none"]
REVIEW flags:    <n> (<brief description>)                      [or "none"]
Status:          Ready
```

**Deliverable 1 — The Notebook**

Complete `.py` file. First line must be `# Databricks notebook source`. Cells separated by `# COMMAND ----------`.
File name: `nb_<mapping_name_lowercase>.py` (replace spaces and special characters with underscores — e.g. `M_Orders_v2 (PROD)` → `nb_m_orders_v2_prod.py`)

**Deliverable 2 — Conversion Summary**

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
````

- [ ] **Step 3: Verify mega_prompt.md structure**

Read `templates/mega_prompt.md` and confirm:
- `## Step 3` and `## Step 4` headings are gone
- `## Phase 1: Review & Analysis` is present and contains the Analysis Summary output block
- `## Phase 2: Extraction` is present and contains the Extraction Complete output block
- `## Phase 3: Notebook Creation` is present and contains `# Databricks notebook source` as required first line
- `## Phase 4: Cell Review` is present and contains both the Review Summary block and both deliverables
- `## Quick Reference: Transformation Types` table is still present and unchanged below Phase 4
- `## Step 1` and `## Step 2` are untouched above Phase 1

- [ ] **Step 4: Commit**

```bash
git add templates/mega_prompt.md
git commit -m "feat: replace flat output steps with 4-phase execution workflow"
```

---

### Task 2: Update `AGENT.md` to match the phased approach

**Files:**
- Modify: `AGENT.md` — replace `Conversion Rules` section with 4 phases (concise); update `Session Outputs`

- [ ] **Step 1: Replace the Conversion Rules section**

In `AGENT.md`, replace this entire block (lines 21–49):

```
## Conversion Rules (Non-Negotiable)

### Parsing
- Parse the full `<MAPPING>` before writing any output. Understand the complete DAG before translating individual nodes.
- Walk the `<CONNECTOR>` list to reconstruct the data flow graph. Transformation order in the XML is not reliable; connector topology is.
- Respect `<TARGETLOADORDER>` when it exists.

### Translation
- Use `docs/transformation_mappings.md` as the authoritative translation reference. Do not invent patterns.
- When a transformation has no direct PySpark equivalent, translate to the closest semantic equivalent and add a `# REVIEW:` comment explaining the gap.
- PowerCenter expression language functions must be translated using the function mapping table in `docs/transformation_mappings.md`. Never leave PowerCenter syntax in the output.

### Output Notebook Structure (in cell order)
1. **Header cell (Markdown):** Mapping name, source system, target system, original mapping description, conversion timestamp, any `# REVIEW:` flags.
2. **Configuration cell:** Widget definitions (`dbutils.widgets`) for environment-dependent values (catalog, schema, source paths).
3. **Imports cell:** All `from pyspark.sql import ...` and library imports.
4. **Source reads:** One cell per source, using the pattern in `docs/conversion_standards.md`.
5. **Transformation cells:** One cell per logical transformation group, preserving the PowerCenter DAG order.
6. **Target writes:** One cell per target, using the Delta write pattern in `docs/conversion_standards.md`.
7. **Validation cell (optional):** Row count assertions if the original mapping had count checks.

### Quality Gates
- Every generated notebook must pass these checks before you return it:
  - [ ] No PowerCenter-specific syntax remains in any cell
  - [ ] All `<TRANSFORMFIELD>` output ports are accounted for downstream
  - [ ] All source/target names are parameterized (no hardcoded paths)
  - [ ] All `UPDATE_STRATEGY` transformations result in a Delta `MERGE` statement
  - [ ] Lookup transformations use broadcast joins when the lookup source is small (< 1 GB heuristic)
  - [ ] All `# REVIEW:` comments are listed in the header cell
```

With this replacement:

```markdown
## Execution Phases

Every conversion follows these four phases in order. Each phase has a required output before the next begins.

### Phase 1 — Review & Analysis
Validate `ISVALID="YES"`. Build the DAG from `<CONNECTOR>` elements. Topological sort. Inventory all transformation types. Flag unsupported types. Score complexity. Output the **Analysis Summary** before proceeding.

### Phase 2 — Extraction
Extract everything from the XML: all `<SOURCE>`, `<TARGET>`, and `<TRANSFORMATION>` definitions (every `<TRANSFORMFIELD>` and `<TABLEATTRIBUTE>`), full connector adjacency list, `<TARGETLOADORDER>`, connection variable → Unity Catalog mappings, parameter → widget mappings, all expression strings. Output the **Extraction Complete manifest** before proceeding.

### Phase 3 — Notebook Creation
Write the complete native Databricks notebook using patterns from `docs/transformation_mappings.md` and `docs/conversion_standards.md`.

Required first line: `# Databricks notebook source`
Cell separator: `# COMMAND ----------`

Cell order: header markdown → widgets → imports → source reads (one per SQ) → transformations (DAG order) → target writes (TARGETLOADORDER sequence) → validation cell (if applicable).

Use `docs/transformation_mappings.md` for every translation decision. Never leave PowerCenter syntax in output. Proceed directly to Phase 4 with no pause.

### Phase 4 — Cell Review
Walk every cell. For each: check no PowerCenter syntax remains, all ports accounted for, all values from widgets, Update Strategy → MERGE, small Lookups → broadcast, REVIEW comments in header. Correct inline. Output the **Review Summary**, then deliver the notebook and conversion summary.
```

- [ ] **Step 2: Update Session Outputs**

In `AGENT.md`, replace the `## Session Outputs` section (lines 73–77):

```
## Session Outputs

Return:
1. The complete notebook as a single Python file (`.py`) with Databricks cell delimiters (`# COMMAND ----------`).
2. A short conversion summary listing: mapping name, transformation count, any `# REVIEW:` items, and estimated complexity (Low / Medium / High).
```

With:

```markdown
## Session Outputs

Return in this order:
1. **Phase 1 — Analysis Summary** (before any code is written)
2. **Phase 2 — Extraction Complete manifest** (before notebook is written)
3. **Phase 4 — Review Summary** (after cell review)
4. **The notebook** — `.py` file, first line `# Databricks notebook source`, cells separated by `# COMMAND ----------`, file name `nb_<mapping_name_lowercase>.py`
5. **Conversion Summary** — mapping name, source/target counts, transformation count, REVIEW items, complexity score
```

- [ ] **Step 3: Verify AGENT.md**

Read `AGENT.md` and confirm:
- `## Conversion Rules (Non-Negotiable)` heading is gone
- `## Execution Phases` heading is present with all 4 phases (Phase 1–4) described
- `## Session Outputs` lists 5 items in order: Analysis Summary, Extraction manifest, Review Summary, notebook, Conversion Summary
- `## Your Role`, `## The Fundamental Rule`, `## Before You Start a Conversion`, `## When You Are Uncertain`, and `## Session Inputs` sections are all untouched

- [ ] **Step 4: Commit**

```bash
git add AGENT.md
git commit -m "docs: update AGENT.md with 4-phase execution model"
```

---

## Self-Review

**Spec coverage:**
- ✅ Phase 1 with Analysis Summary output → Task 1 Step 2
- ✅ Phase 2 with Extraction Complete output → Task 1 Step 2
- ✅ Phase 3 with `# Databricks notebook source` first line → Task 1 Step 2
- ✅ Phase 3 with mandatory cell order (7 cells) → Task 1 Step 2
- ✅ Phase 3 no-pause instruction → Task 1 Step 2
- ✅ Phase 4 with 7-item checklist, Review Summary, Deliverable 1 + 2 → Task 1 Step 2
- ✅ File name sanitization rule carried over → Task 1 Step 2
- ✅ Steps 3 and 4 removed → Task 1 Step 1
- ✅ Quick Reference table untouched → verified in Task 1 Step 3
- ✅ AGENT.md Conversion Rules replaced → Task 2 Step 1
- ✅ AGENT.md Session Outputs updated → Task 2 Step 2
- ✅ What Does Not Change (Role, Step 1, Step 2, Quick Reference, docs/) all preserved → verified in Task 1 Step 3 and Task 2 Step 3

**Placeholder scan:** No TBDs. All phase output blocks contain exact field names and format strings matching the spec.

**Consistency:** Phase output format strings are identical between Task 1 (mega_prompt.md) and the spec. AGENT.md uses shortened prose descriptions of the same phases — no new field names introduced.

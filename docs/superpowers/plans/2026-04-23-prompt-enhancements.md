# Prompt Enhancements Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add connection scanning instructions + Phase 1 validation, a Mapplet XML paste section with 3-option interruption, and an optional Merge Keys table to `templates/mega_prompt.md`; expand the Mapplet section in `docs/transformation_mappings.md` with XML port extraction → Python function patterns.

**Architecture:** Two markdown files change. `mega_prompt.md` is the Genie Code entry point — Step 2's Connection Mapping subsection is enhanced, and two new subsections (Mapplet XML, Merge Keys) are inserted in the specified order. `docs/transformation_mappings.md` has its 12-line Mapplet section replaced with a full conversion pattern. No other files change.

**Tech Stack:** Markdown only. No code.

---

### Task 1: Enhance `### Connection Mapping` in `templates/mega_prompt.md`

**Files:**
- Modify: `templates/mega_prompt.md` lines 32–38

- [ ] **Step 1: Replace the Connection Mapping subsection**

In `templates/mega_prompt.md`, replace this exact block (lines 32–38):

```
### Connection Mapping

For each PowerCenter connection variable in the XML, provide the Unity Catalog equivalent. Add or remove rows as needed.

| PowerCenter Connection Variable | Unity Catalog Path (`catalog.schema`) |
|---|---|
| `$DBConnection_EXAMPLE` ← replace with actual variable name | `catalog.schema` ← replace with actual path |
```

With:

```markdown
### Connection Mapping

Before filling in this table, scan the XML for all connection names:
- `CONNECTION="..."` attributes on every `<SOURCE>` and `<TARGET>` element
- `$DBConnection_*` variable references inside any `<TABLEATTRIBUTE VALUE="...">` or expression string

List every unique connection name below and map it to its Unity Catalog path.

| PowerCenter Connection Name | Unity Catalog Path (`catalog.schema`) |
|---|---|
| `PROD_DW_READ` ← replace with actual name | `catalog.schema` ← replace with actual path |

Add or remove rows as needed. Every connection name that appears in the XML must have a row here.

> **Agent:** In Phase 1, confirm that every connection name found in the XML has an entry in this table. List any missing entries in the Analysis Summary as `CONNECTION UNMAPPED — <name>` and stop before Phase 2 until the user resolves them.
```

- [ ] **Step 2: Verify**

Read `templates/mega_prompt.md` and confirm:
- `### Connection Mapping` heading is present
- The two bullet points explaining where to scan (`CONNECTION="..."`, `$DBConnection_*`) are present
- The table header now reads `PowerCenter Connection Name` (not `PowerCenter Connection Variable`)
- The agent callout with `CONNECTION UNMAPPED` is present
- `### Target` immediately follows with no extra blank lines beyond the standard one

- [ ] **Step 3: Commit**

```bash
git add templates/mega_prompt.md
git commit -m "feat: add connection scanning instructions and Phase 1 validation to Connection Mapping"
```

---

### Task 2: Add `### Mapplet XML` subsection to `templates/mega_prompt.md`

**Files:**
- Modify: `templates/mega_prompt.md` — insert after `### Connection Mapping` block, before `### Target`

- [ ] **Step 1: Insert the Mapplet XML subsection**

In `templates/mega_prompt.md`, find this line (the `### Target` heading that follows Connection Mapping):

```
### Target
```

Insert the following block **immediately before** that line (preserving a blank line separator):

```markdown
### Mapplet XML

If you know this mapping references Mapplets, paste the full exported XML for each one below. Add one labeled block per mapplet. If you are unsure or the mapping has no Mapplets, leave this section blank — the agent will notify you if it finds any.

**Mapplet: `mlt_example`** ← replace with actual mapplet name
```xml
<!-- PASTE MAPPLET XML HERE -->
```

> **Agent:** At the start of Phase 1, scan the main mapping XML for any `<MAPPLET>` instance references. For each one found:
>
> - If a matching XML block is present in this section: extract input/output ports and convert to a Python helper function using the patterns in `docs/transformation_mappings.md`. Call the function inline in the notebook at the correct DAG position.
> - If no XML block is provided: stop and present the user with these three options before proceeding:
>
>   > Mapplet reference found: `<mapplet_name>`. No XML was provided in Step 2.
>   >
>   > **Option 1** — Paste the mapplet XML into this chat now. I will extract its ports, convert it to a Python helper function, and continue.
>   > **Option 2** — Continue without the mapplet XML. The call will be stubbed as `# REVIEW: MAPPLET — <name> — no XML provided` and added to the REVIEW checklist.
>   > **Option 3** — Stop here. Add the mapplet XML to the `### Mapplet XML` section in Step 2 and start a new session.

```

- [ ] **Step 2: Verify**

Read `templates/mega_prompt.md` and confirm the Step 2 subsection order is exactly:
1. `### Connection Mapping`
2. `### Mapplet XML`
3. `### Target`
4. `### Run Date Handling`
5. `### Special Instructions`
6. `### Input XML`

Also confirm the three-option interruption block is present inside the Mapplet XML agent callout.

- [ ] **Step 3: Commit**

```bash
git add templates/mega_prompt.md
git commit -m "feat: add Mapplet XML section with 3-option agent interruption to Step 2"
```

---

### Task 3: Add `### Merge Keys` subsection to `templates/mega_prompt.md`

**Files:**
- Modify: `templates/mega_prompt.md` — insert after `### Target` block, before `### Run Date Handling`

- [ ] **Step 1: Insert the Merge Keys subsection**

In `templates/mega_prompt.md`, find this line (the `### Run Date Handling` heading):

```
### Run Date Handling
```

Insert the following block **immediately before** that line (preserving a blank line separator):

```markdown
### Merge Keys

If you know the primary key columns for any targets that have an Update Strategy upstream, list them here. Use the exact column names from the PowerCenter target definition. This section is optional — leave it blank if you are unsure.

| Target Table | Merge Key Column(s) |
|---|---|
| `fact_orders` ← replace | `ORDER_ID` ← replace; comma-separate multiple keys |

> **Agent:** For each target with an upstream Update Strategy:
> - If a row is present here: use the specified columns to build the MERGE condition (`tgt.<key> = src.<key>`).
> - If no row is present: continue, stub the condition as `"tgt.<KEY> = src.<KEY>"`, and add `# REVIEW: MERGE KEY UNKNOWN — <target_name> — specify match columns` to both the notebook cell and the Phase 1 Analysis Summary. Do not stop.

```

- [ ] **Step 2: Verify**

Read `templates/mega_prompt.md` and confirm the Step 2 subsection order is exactly:
1. `### Connection Mapping`
2. `### Mapplet XML`
3. `### Target`
4. `### Merge Keys`
5. `### Run Date Handling`
6. `### Special Instructions`
7. `### Input XML`

Also confirm the agent callout says "Do not stop" for missing merge keys.

- [ ] **Step 3: Commit**

```bash
git add templates/mega_prompt.md
git commit -m "feat: add optional Merge Keys section to Step 2"
```

---

### Task 4: Expand Mapplet section in `docs/transformation_mappings.md`

**Files:**
- Modify: `docs/transformation_mappings.md` lines 291–302

- [ ] **Step 1: Replace the Mapplet section**

In `docs/transformation_mappings.md`, replace this exact block (lines 291–302):

```
### Mapplet → `%run` or Helper Function

```python
# In the calling notebook — reference the mapplet notebook
# %run ./mapplets/mlt_address_cleanse

# Or inline as a function if small enough
def mlt_address_cleanse(df):
    return df \
        .withColumn("zip", regexp_replace(col("zip"), "[^0-9]", "")) \
        .withColumn("state", upper(trim(col("state"))))
```
```

With:

````markdown
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

**Large or complex mapplets** (more than ~10 transform fields, or containing Lookups or Aggregators): convert to a standalone notebook at `./mapplets/mlt_<name>.py` and reference with `%run`:

```python
# %run ./mapplets/mlt_address_cleanse
df_transformed = mlt_address_cleanse(df_source)
```

Flag as `# REVIEW: MAPPLET INLINED — verify port mappings` whenever inlining a mapplet, since mapplet reuse across notebooks is lost when inlined.
````

- [ ] **Step 2: Verify**

Read `docs/transformation_mappings.md` around the Mapplet section and confirm:
- Heading is now `### Mapplet → Python Helper Function`
- The `PORTTYPE="INPUT"` and `PORTTYPE="OUTPUT"` port extraction rules are present
- The `mlt_address_cleanse` function example with `.drop("zip", "state")` is present
- The no-XML stub pattern is present with `# REVIEW: MAPPLET — ...` format
- The `%run` pattern for large/complex mapplets is present
- The `## Expression Function Translation Table` heading follows immediately after the closing `---`

- [ ] **Step 3: Commit**

```bash
git add docs/transformation_mappings.md
git commit -m "docs: expand Mapplet section with XML port extraction and no-XML stub patterns"
```

---

## Self-Review

**Spec coverage:**
- ✅ Connection Mapping: scanning instructions + agent Phase 1 validation hook → Task 1
- ✅ Mapplet XML: optional paste section in Step 2 → Task 2
- ✅ Mapplet XML: 3-option interruption when discovered in Phase 1 → Task 2
- ✅ Mapplet XML: for provided XML — extract ports, convert to function → Task 4
- ✅ Mapplet XML: for missing XML — stub as REVIEW → Task 4
- ✅ Merge Keys: optional table, insert after Target → Task 3
- ✅ Merge Keys: if missing → stub condition + REVIEW flag, do not stop → Task 3
- ✅ Step 2 subsection order matches spec → verified in Tasks 2 and 3
- ✅ `transformation_mappings.md` Mapplet expansion → Task 4

**Placeholder scan:** No TBDs. All markdown content is complete verbatim.

**Consistency:** The `# REVIEW: MAPPLET — <name>` format used in Task 2 (agent callout) matches the stub format in Task 4 (transformation_mappings.md). The `MERGE KEY UNKNOWN — <target_name>` format in Task 3 is distinct and unambiguous.

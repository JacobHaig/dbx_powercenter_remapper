# Prompt Enhancements: Connection Mapping, Mapplet XML, Merge Keys

**Date:** 2026-04-23
**Status:** Approved

---

## Problem

Three gaps in `templates/mega_prompt.md` cause the agent to guess or fail silently:

1. **Connection Mapping** — The table has one placeholder row with no instructions for finding connection names in the XML. Real mappings have many connections (`PROD_DW_READ`, `$DBConnection_Orders`, `DATAMART22`, etc.) and the agent has no way to validate completeness.

2. **Mapplet Handling** — The mapping XML may reference `<MAPPLET>` elements, but the prompt has no place to provide mapplet XML and no instructions for what the agent should do when it finds one. The agent proceeds blindly or guesses.

3. **Merge Keys** — When a target has an upstream Update Strategy, the MERGE condition requires match key columns. The agent currently has no source for these and guesses from column names.

---

## Solution

Three additions to `## Step 2: Fill In the Conversion Request` in `templates/mega_prompt.md`, plus an expanded Mapplet section in `docs/transformation_mappings.md`.

---

## Files Changed

| File | Change |
|---|---|
| `templates/mega_prompt.md` | Enhance `### Connection Mapping`; add `### Mapplet XML`; add `### Merge Keys` |
| `docs/transformation_mappings.md` | Expand Mapplet section with XML port extraction → Python function pattern |

No other files change.

---

## Change 1 — Enhanced Connection Mapping

### What changes in `mega_prompt.md`

Replace the existing `### Connection Mapping` subsection with:

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

---

## Change 2 — Mapplet XML Section

### What changes in `mega_prompt.md`

Add a new `### Mapplet XML` subsection **after** `### Connection Mapping` and **before** `### Target`:

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

### What changes in `docs/transformation_mappings.md`

Replace the current Mapplet section (the 6-line `### Mapplet → %run or Helper Function` block) with an expanded version:

```markdown
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
```

---

## Change 3 — Merge Keys Section

### What changes in `mega_prompt.md`

Add a new `### Merge Keys` subsection **after** `### Target` and **before** `### Run Date Handling`:

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

---

## Ordering of Subsections in Step 2 (after changes)

1. `### Connection Mapping` (enhanced)
2. `### Mapplet XML` (new)
3. `### Target` (unchanged)
4. `### Merge Keys` (new)
5. `### Run Date Handling` (unchanged)
6. `### Special Instructions` (unchanged)
7. `### Input XML` (unchanged)

---

## What Does Not Change

- Step 1 (read these files) — unchanged
- Phase 1–4 execution logic — unchanged, except Phase 1 now has two validation checks: connection completeness and mapplet discovery
- Quick Reference table — unchanged
- All other `docs/` files except `transformation_mappings.md` Mapplet section

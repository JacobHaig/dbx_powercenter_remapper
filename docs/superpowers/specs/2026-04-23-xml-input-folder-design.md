# XML Input Folder Design

**Date:** 2026-04-23
**Status:** Approved

---

## Problem

`templates/mega_prompt.md` currently requires the user to paste large PowerCenter XML directly into the prompt template — both the main mapping XML (`### Input XML`) and any mapplet XML (`### Mapplet XML`). Pasting multi-thousand-line XML into a markdown template is error-prone, hard to update, and makes the template unreadable.

---

## Solution

Introduce an `input/` folder at the repo root as a file-drop landing zone. The user places XML files there using the standard PowerCenter naming conventions (`M_`, `mlt_`, `w_`, etc.). The agent reads files from this folder rather than from inline paste areas. XML files are gitignored.

---

## Files Changed

| File | Change |
|---|---|
| `input/.gitkeep` | Create — establishes the folder in git |
| `.gitignore` | Create — ignore `input/*.xml` |
| `templates/mega_prompt.md` | Replace `### Input XML` paste area with `### Mapping File` filename field; replace `### Mapplet XML` paste area with drop instructions; add folder scan + file read steps to Phase 1 |
| `AGENT.md` | Update `## Session Inputs` to reflect file-based input |

No other files change.

---

## Change 1 — `input/` folder and `.gitignore`

Create `input/.gitkeep` (empty file) so the folder is tracked by git.

Create `.gitignore` at repo root containing:

```
input/*.xml
```

This keeps the folder structure in the repo while ensuring XML files (which may contain sensitive schema or business logic) are never committed.

---

## Change 2 — Replace `### Input XML` with `### Mapping File`

### Current content (to remove)

```
### Input XML

Paste the full PowerCenter XML export below. Do not truncate it.

```xml
<!-- PASTE FULL XML HERE -->
```
```

### Replacement content

```markdown
### Mapping File

Place your PowerCenter XML export in the `input/` folder at the root of this repository, then enter the filename below.

**Mapping file:** `M_EXAMPLE.xml` ← replace with your filename
```

---

## Change 3 — Replace `### Mapplet XML` paste area with drop instructions

### Current content (to replace)

The `### Mapplet XML` section currently contains a paste area with `<!-- PASTE MAPPLET XML HERE -->` and an agent callout referencing "a matching XML block in this section."

### Replacement content

```markdown
### Mapplet XML

If this mapping uses Mapplets, place each mapplet XML export in the `input/` folder alongside the mapping file. Use the standard `mlt_` prefix (e.g., `mlt_address_cleanse.xml`). The agent scans `input/` automatically and reads any mapplet files it needs — no changes needed here.
```

The agent callout moves out of Step 2 and into Phase 1 (see Change 4). The 3-option interruption is updated: Option 1 remains "paste into chat," Option 3 changes from "add to Step 2 section" to "place the file in `input/` and start a new session."

---

## Change 4 — Phase 1: add folder scan and file read steps

### Addition at the top of Phase 1

Insert two new numbered steps before the existing step 1 (`Confirm ISVALID`), renumbering the existing steps:

```
1. Scan the `input/` folder and list every file found — report filenames in the Phase 1 output.
2. Read the mapping file named in `### Mapping File` from `input/`. If the file is not found, stop and tell the user: "Mapping file `<filename>` was not found in `input/`. Place the file there and start a new session."
```

The existing steps become 3–9 (previously 1–7).

### Mapplet discovery instruction (added after the inventory step)

After the transformation inventory step, add:

```
When a `<MAPPLET>` instance reference is found in the mapping XML:
- Look for a file named `mlt_<name>.xml` in `input/`.
- If found: read it, extract input/output ports, convert to a Python helper function per `docs/transformation_mappings.md`, and call it inline in the notebook at the correct DAG position.
- If not found: stop and present the user with these options:

  > Mapplet reference found: `<mapplet_name>`. No file named `mlt_<mapplet_name>.xml` was found in `input/`.
  >
  > **Option 1** — Paste the mapplet XML into this chat now. I will extract its ports, convert it to a Python helper function, and continue.
  > **Option 2** — Continue without the mapplet XML. The call will be stubbed as `# REVIEW: MAPPLET — <name> — no XML provided` and added to the REVIEW checklist.
  > **Option 3** — Stop here. Place `mlt_<mapplet_name>.xml` in the `input/` folder and start a new session.
```

### Addition to Phase 1 Analysis Summary output block

Add one line to the required output block:

```
Input files:     <list of filenames found in input/>
```

---

## Change 5 — `AGENT.md` Session Inputs

### Current content (to replace)

```
## Session Inputs

You will receive:
- `{{XML_CONTENT}}` — the raw PowerCenter XML export
- `{{TARGET_CATALOG}}` — Unity Catalog catalog name
- `{{TARGET_SCHEMA}}` — target schema/database
- `{{SOURCE_TYPE}}` — source system type (e.g., Oracle, SQL Server, flat file)
- `{{ENVIRONMENT}}` — dev / staging / prod

See `templates/conversion_prompt_template.md` for the full prompt structure.
```

### Replacement content

```markdown
## Session Inputs

XML files are read from the `input/` folder at the root of this repository:
- `input/<mapping_filename>.xml` — the main mapping XML; filename is specified in `### Mapping File` in the conversion request
- `input/mlt_<name>.xml` — mapplet XML files; read automatically when mapplet references are discovered in the mapping

All other inputs come from the filled-in fields in `### Connection Mapping`, `### Target`, `### Merge Keys`, `### Run Date Handling`, and `### Special Instructions` in `templates/mega_prompt.md`.
```

---

## What Does Not Change

- Step 1 (read these files) — unchanged
- Phase 2–4 — unchanged
- Quick Reference table — unchanged
- All `docs/` reference files
- `### Connection Mapping`, `### Target`, `### Merge Keys`, `### Run Date Handling`, `### Special Instructions` — unchanged

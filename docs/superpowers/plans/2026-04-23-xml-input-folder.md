# XML Input Folder Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace inline XML paste areas with an `input/` file-drop folder; update Phase 1 to scan the folder and read files from it; update AGENT.md Session Inputs to match.

**Architecture:** Create `input/` with `.gitkeep` and gitignore XML files. Two paste areas in `templates/mega_prompt.md` (`### Input XML` and `### Mapplet XML`) are replaced with a filename field and a drop instruction respectively. Phase 1 gains a folder-scan step, a file-read step, and a mapplet discovery block. AGENT.md Session Inputs is rewritten to describe the file-based approach.

**Tech Stack:** Markdown only. No code.

---

### Task 1: Create `input/` folder and `.gitignore`

**Files:**
- Create: `input/.gitkeep`
- Create: `.gitignore`

- [ ] **Step 1: Create the input folder**

Create an empty file at `input/.gitkeep`. This establishes the folder in git without committing any XML content.

- [ ] **Step 2: Create `.gitignore`**

Create `.gitignore` at the repo root with this exact content:

```
input/*.xml
```

- [ ] **Step 3: Verify**

Run:
```bash
ls input/
cat .gitignore
```

Expected:
```
.gitkeep
input/*.xml
```

- [ ] **Step 4: Commit**

```bash
git add input/.gitkeep .gitignore
git commit -m "feat: add input/ landing folder for XML files"
```

---

### Task 2: Replace `### Input XML` paste area with `### Mapping File` field

**Files:**
- Modify: `templates/mega_prompt.md` lines 102–108

- [ ] **Step 1: Replace the section**

In `templates/mega_prompt.md`, replace this exact block:

```
### Input XML

Paste the full PowerCenter XML export below. Do not truncate it.

```xml
<!-- PASTE FULL XML HERE -->
```
```

With:

```markdown
### Mapping File

Place your PowerCenter XML export in the `input/` folder at the root of this repository, then enter the filename below.

**Mapping file:** `M_EXAMPLE.xml` ← replace with your filename
```

- [ ] **Step 2: Verify**

Read `templates/mega_prompt.md` and confirm:
- `### Input XML` heading is gone
- `### Mapping File` heading is present
- `<!-- PASTE FULL XML HERE -->` is gone
- `**Mapping file:** \`M_EXAMPLE.xml\`` placeholder line is present
- The `---` separator before Phase 1 is still present immediately after this section

- [ ] **Step 3: Commit**

```bash
git add templates/mega_prompt.md
git commit -m "feat: replace Input XML paste area with Mapping File filename field"
```

---

### Task 3: Replace `### Mapplet XML` paste area with drop instructions

**Files:**
- Modify: `templates/mega_prompt.md` lines 48–66

- [ ] **Step 1: Replace the section**

In `templates/mega_prompt.md`, replace this exact block:

```
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

With:

```markdown
### Mapplet XML

If this mapping uses Mapplets, place each mapplet XML export in the `input/` folder alongside the mapping file. Use the standard `mlt_` prefix (e.g., `mlt_address_cleanse.xml`). The agent scans `input/` automatically and reads any mapplet files it needs — no changes needed here.
```

- [ ] **Step 2: Verify**

Read `templates/mega_prompt.md` and confirm:
- `### Mapplet XML` heading is still present
- `<!-- PASTE MAPPLET XML HERE -->` is gone
- The `**Mapplet: \`mlt_example\`**` labeled block is gone
- The old agent callout (referencing "a matching XML block in this section") is gone
- The new single-sentence drop instruction referencing `input/` is present
- `### Target` immediately follows

- [ ] **Step 3: Commit**

```bash
git add templates/mega_prompt.md
git commit -m "feat: replace Mapplet XML paste area with input/ folder drop instruction"
```

---

### Task 4: Update Phase 1 with folder scan, file read, and mapplet discovery

**Files:**
- Modify: `templates/mega_prompt.md` — Phase 1 section (lines 112–140)

- [ ] **Step 1: Add folder scan and file read as first two steps; renumber existing steps**

In `templates/mega_prompt.md`, replace this exact block (the Phase 1 numbered list and its intro paragraph):

```
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
```

With:

```markdown
Parse and validate the XML. Build the data flow graph. Assess the mapping before any extraction or code generation begins.

1. Scan the `input/` folder and list every file found — include the filenames in the Phase 1 output.
2. Read the mapping file named in `### Mapping File` from `input/`. If the file is not found, stop and tell the user: "Mapping file `<filename>` was not found in `input/`. Place the file there and start a new session."
3. Confirm `<MAPPING ISVALID="YES">` — if NO, stop immediately and tell the user: "This mapping is marked ISVALID=NO in PowerCenter. Conversion cannot proceed until the mapping is valid."
4. Extract `FOLDER NAME` and `MAPPING NAME`
5. Walk all `<CONNECTOR>` elements and build the full DAG adjacency list (FROMINSTANCE/FROMFIELD → TOINSTANCE/TOFIELD)
6. Perform a topological sort — confirm no cycles
7. Inventory every `<TRANSFORMATION>` by type. When a `<MAPPLET>` instance reference is found:
   - Look for a file named `mlt_<name>.xml` in `input/`.
   - If found: read it, extract input/output ports, convert to a Python helper function per `docs/transformation_mappings.md`, and call it inline in the notebook at the correct DAG position.
   - If not found: stop and present the user with these options:

     > Mapplet reference found: `<mapplet_name>`. No file named `mlt_<mapplet_name>.xml` was found in `input/`.
     >
     > **Option 1** — Paste the mapplet XML into this chat now. I will extract its ports, convert it to a Python helper function, and continue.
     > **Option 2** — Continue without the mapplet XML. The call will be stubbed as `# REVIEW: MAPPLET — <name> — no XML provided` and added to the REVIEW checklist.
     > **Option 3** — Stop here. Place `mlt_<mapplet_name>.xml` in the `input/` folder and start a new session.
8. Flag any transformation types not in `docs/transformation_mappings.md` as unsupported
9. Score complexity using the first-matching-rule order:
   - **High** — >15 transformations, OR Stored/External Procedures present, OR >2 REVIEW items expected
   - **Medium** — 6–15 transformations, OR has Update Strategy or Lookups, OR ≤2 REVIEW items expected
   - **Low** — all other cases
```

- [ ] **Step 2: Add `Input files:` line to the Analysis Summary output block**

In `templates/mega_prompt.md`, replace the Analysis Summary output block:

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

With:

```
Phase 1 — Analysis Summary
Input files:     <list of filenames found in input/>
Mapping:         <name>
Folder:          <folder>
Sources:         <n> (<types>)
Targets:         <n> (<names>)
Transformations: <total> (<type x count, ...>)
Unsupported:     <name> (<type>) — will flag as REVIEW   [omit line if none]
Complexity:      Low / Medium / High
```

- [ ] **Step 3: Verify**

Read `templates/mega_prompt.md` Phase 1 section and confirm:
- Step 1 is the `input/` folder scan
- Step 2 is the mapping file read with the "not found" stop instruction
- Step 3 is `Confirm <MAPPING ISVALID="YES">`
- Step 7 contains the mapplet discovery block with all three options
- Steps 8 and 9 are the unsupported-type flag and complexity scoring
- The Analysis Summary block starts with `Input files:`
- Phase 2 heading follows after the `---` separator

- [ ] **Step 4: Commit**

```bash
git add templates/mega_prompt.md
git commit -m "feat: add folder scan, file read, and mapplet discovery to Phase 1"
```

---

### Task 5: Update `AGENT.md` Session Inputs

**Files:**
- Modify: `AGENT.md` lines 55–64

- [ ] **Step 1: Replace the Session Inputs section**

In `AGENT.md`, replace this exact block:

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

With:

```markdown
## Session Inputs

XML files are read from the `input/` folder at the root of this repository:
- `input/<mapping_filename>.xml` — the main mapping XML; filename is specified in `### Mapping File` in the conversion request
- `input/mlt_<name>.xml` — mapplet XML files; read automatically when mapplet references are discovered in the mapping

All other inputs come from the filled-in fields in `### Connection Mapping`, `### Target`, `### Merge Keys`, `### Run Date Handling`, and `### Special Instructions` in `templates/mega_prompt.md`.
```

- [ ] **Step 2: Verify**

Read `AGENT.md` and confirm:
- `{{XML_CONTENT}}` placeholder is gone
- `{{TARGET_CATALOG}}`, `{{TARGET_SCHEMA}}`, `{{SOURCE_TYPE}}`, `{{ENVIRONMENT}}` placeholders are gone
- The `input/` folder is described as the source of XML files
- The reference to `templates/conversion_prompt_template.md` is gone
- `## Session Outputs` follows immediately after

- [ ] **Step 3: Commit**

```bash
git add AGENT.md
git commit -m "docs: update AGENT.md Session Inputs to reflect input/ folder approach"
```

---

## Self-Review

**Spec coverage:**
- ✅ `input/.gitkeep` created → Task 1
- ✅ `.gitignore` with `input/*.xml` → Task 1
- ✅ `### Input XML` replaced with `### Mapping File` → Task 2
- ✅ `### Mapplet XML` paste area replaced with drop instruction → Task 3
- ✅ Phase 1 folder scan step (step 1) → Task 4
- ✅ Phase 1 file read step with not-found stop (step 2) → Task 4
- ✅ Phase 1 mapplet discovery block with 3 options (step 7) → Task 4
- ✅ `Input files:` line in Analysis Summary → Task 4
- ✅ AGENT.md Session Inputs rewritten → Task 5

**Placeholder scan:** No TBDs. All replacement content is verbatim and complete.

**Consistency:** The mapplet file naming convention (`mlt_<name>.xml`) is consistent between Task 3 (drop instruction), Task 4 (discovery lookup), and the spec. Option 3 in Task 4 correctly says "Place `mlt_<mapplet_name>.xml` in the `input/` folder" — not "add to Step 2 section" (old wording).

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

This repository is the **context and knowledge base** for a Databricks agent that converts Informatica PowerCenter mappings (XML) into valid Databricks PySpark Notebooks. There is intentionally no runtime code here — only architecture documents, translation references, standards, workflow guides, and prompt templates that the agent consumes.

## Repository Layout

```
/
├── CLAUDE.md                          # This file
├── README.md                          # Project overview and quick-start
├── AGENT.md                           # Instructions for the Databricks conversion agent
├── input/                             # Drop XML files here before conversion (gitignored)
├── notebooks/                         # All generated notebooks land here
├── docs/
│   ├── powercenter_reference.md       # PowerCenter XML schema and component catalog
│   ├── transformation_mappings.md     # PowerCenter → PySpark/DBX translation table
│   ├── xml_to_pyspark_examples.md     # Side-by-side XML snippets and their PySpark equivalents
│   ├── conversion_standards.md        # Coding standards for generated notebooks
│   └── workflow.md                    # End-to-end conversion workflow
├── templates/
│   └── mega_prompt.md                 # Genie Code entry point — fill in and submit to convert
└── tests/
    └── framework.md                   # Testing strategy and pytest setup guide
```

## Key Concepts

**PowerCenter XML hierarchy:** `POWERMART → REPOSITORY → FOLDER → MAPPING → TRANSFORMATION / CONNECTOR`

**Conversion unit of work:** One PowerCenter `<MAPPING>` produces one Databricks notebook. Mapplets become helper functions or shared notebooks imported via `%run`.

**Authoritative translation reference:** `docs/transformation_mappings.md` is the single source of truth for how each PowerCenter transformation type maps to PySpark. For concrete examples showing the exact XML and the PySpark output side by side, see `docs/xml_to_pyspark_examples.md`. When ambiguous, consult these two files before deciding an approach.

**Standards are non-negotiable:** Generated notebooks must conform to `docs/conversion_standards.md`. Cell structure, import ordering, Delta write patterns, and error handling conventions are all defined there.

## File Locations (Non-Negotiable)

- **Input XML** goes in `input/` before starting a conversion. Files there are gitignored.
- **All generated notebooks** must be written to `notebooks/` in this repo. File name: `nb_<mapping_name_lowercase>.py`. Do not write notebooks anywhere else.
- All edits to reference docs, standards, and the prompt template happen inside this repo. Do not create files outside it.

## When Working in This Repo

- To add support for a new PowerCenter transformation type, update `docs/transformation_mappings.md` and add a side-by-side example to `docs/xml_to_pyspark_examples.md`.
- To change how notebooks are structured, update `docs/conversion_standards.md` and cascade any changes to `AGENT.md` and the prompt template.
- `templates/mega_prompt.md` is the Genie Code entry point — keep it synchronized with the docs.
- `AGENT.md` is read by the conversion agent at the start of every session; it must remain concise and actionable.

## Testing

No test code exists yet. `tests/framework.md` defines the intended validation strategy: static structure tests (no cluster), expression translation unit tests, and end-to-end functional tests (Databricks cluster required). When tests are implemented they will live in `tests/static/` and `tests/e2e/`.

# DBX PowerCenter Remapper

A knowledge base and agent context repository for converting **Informatica PowerCenter mappings** (XML) into **Databricks PySpark Notebooks**.

## What This Repo Is

This is not a runtime application. It is the **architecture, reference, and context layer** consumed by a Databricks agent that performs the actual conversion. Think of it as the agent's onboarding package and standing instructions.

## What the Agent Does

Given a PowerCenter export XML file (and a filled-in prompt), the agent:

1. Reads the reference docs from this repo (`docs/`) for translation patterns and standards
2. Parses the `<MAPPING>` XML and reconstructs the data flow graph from `<CONNECTOR>` elements
3. Translates each `<TRANSFORMATION>` to the equivalent PySpark / Delta Lake pattern
4. Assembles a single Databricks notebook — one mapping = one notebook, all values parameterized via `dbutils.widgets`
5. Returns the notebook as a `.py` file plus a conversion summary with any `# REVIEW:` items flagged

## Quick Navigation

| Document | Purpose |
|---|---|
| [AGENT.md](AGENT.md) | Agent operating manual |
| [templates/mega_prompt.md](templates/mega_prompt.md) | **Genie Code entry point** — fill this in and paste into Genie Code to start a conversion |
| [docs/powercenter_reference.md](docs/powercenter_reference.md) | PowerCenter XML schema and component catalog |
| [docs/transformation_mappings.md](docs/transformation_mappings.md) | PowerCenter → PySpark authoritative translation table |
| [docs/xml_to_pyspark_examples.md](docs/xml_to_pyspark_examples.md) | Side-by-side XML snippets and PySpark equivalents |
| [docs/conversion_standards.md](docs/conversion_standards.md) | Notebook coding standards and cell structure |
| [docs/workflow.md](docs/workflow.md) | End-to-end conversion workflow |
| [tests/framework.md](tests/framework.md) | Validation and testing strategy |
| [templates/conversion_prompt_template.md](templates/conversion_prompt_template.md) | Legacy prompt template (non-Genie Code invocation) |

## Using With Genie Code

### Prerequisites
- This repo is cloned into your Databricks workspace via Repos
- You have a PowerCenter XML export ready (`.xml` file from PowerCenter Designer or Repository Manager)
- You know the Unity Catalog catalog and schema where the target tables live

### Workflow (one mapping at a time)

1. Open `templates/mega_prompt.md` from this repo
2. Fill in the **Connection Mapping** table — map each `$DBConnection_X` variable from your XML to its Unity Catalog path
3. Fill in **Target** catalog, schema, and table name(s)
4. Mark your **Run Date Handling** selection
5. Add any **Special Instructions** (or leave blank)
6. Paste the full PowerCenter XML into the XML slot
7. Copy the entire completed document
8. Open Genie Code in a Databricks notebook within this cloned repo
9. Paste into the Genie Code chat and send
10. Genie Code reads the reference docs and returns a converted `.py` notebook and a conversion summary
11. Review any `REVIEW:` items flagged in the summary
12. Import the notebook via Repos or the workspace UI and run it in dev to validate

## Supported PowerCenter Transformation Types

| Transformation | Status |
|---|---|
| Source Qualifier | Supported |
| Target Definition | Supported |
| Expression | Supported |
| Filter | Supported |
| Joiner | Supported |
| Lookup (connected) | Supported |
| Lookup (unconnected) | Supported |
| Aggregator | Supported |
| Router | Supported |
| Sorter | Supported |
| Rank | Supported |
| Update Strategy | Supported |
| Union | Supported |
| Sequence Generator | Supported |
| Normalizer | Partial — see `docs/transformation_mappings.md` |
| Mapplet | Supported via `%run` |
| Stored Procedure | Manual review required |
| External Procedure | Manual review required |

## Prerequisites (for the Databricks environment)

- Databricks Runtime 13.x or later (Spark 3.4+)
- Delta Lake enabled
- Unity Catalog (recommended) or Hive Metastore
- Python 3.10+

## Contributing

To add support for a new transformation type or correct a mapping:
1. Update `docs/transformation_mappings.md` with the new translation pattern
2. Add a side-by-side example to `docs/xml_to_pyspark_examples.md`
3. Update the Zone 2 cheat sheet table in `templates/mega_prompt.md`
4. Update `AGENT.md` if the agent's behavior or rules need to change

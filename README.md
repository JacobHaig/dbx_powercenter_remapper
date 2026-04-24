# DBX PowerCenter Remapper

A knowledge base and agent context repository for converting **Informatica PowerCenter mappings** (XML) into **Databricks PySpark Notebooks**.

## What This Repo Is

This is not a runtime application. It is the **architecture, reference, and context layer** consumed by a Databricks agent that performs the actual conversion. Think of it as the agent's onboarding package and standing instructions.

## What the Agent Does

Given a PowerCenter XML file in the `input/` folder (and a filled-in prompt), the agent:

1. Reads the reference docs from this repo (`docs/`) for translation patterns and standards
2. Parses the `<MAPPING>` XML and reconstructs the data flow graph from `<CONNECTOR>` elements
3. Translates each `<TRANSFORMATION>` to the equivalent PySpark / Delta Lake pattern
4. Assembles a single Databricks notebook — one mapping = one notebook, all values parameterized via `dbutils.widgets`
5. Writes the notebook to `notebooks/nb_<mapping_name>` in this repo and returns a conversion summary with any `# REVIEW:` items flagged

## Quick Navigation

| Document | Purpose |
|---|---|
| [AGENT.md](AGENT.md) | Agent operating manual |
| [templates/mega_prompt.md](templates/mega_prompt.md) | **Genie Code entry point** — fill this in and paste into Genie Code to start a conversion |
| [templates/validation_prompt.md](templates/validation_prompt.md) | **Genie Code entry point** — run after conversion to build and execute a validation test notebook |
| [docs/powercenter_reference.md](docs/powercenter_reference.md) | PowerCenter XML schema and component catalog |
| [docs/transformation_mappings.md](docs/transformation_mappings.md) | PowerCenter → PySpark authoritative translation table |
| [docs/xml_to_pyspark_examples.md](docs/xml_to_pyspark_examples.md) | Side-by-side XML snippets and PySpark equivalents |
| [docs/conversion_standards.md](docs/conversion_standards.md) | Notebook coding standards and cell structure |
| [docs/databricks_notebook_creation.md](docs/databricks_notebook_creation.md) | SDK pattern for creating notebook assets (not `.py` files) |
| [docs/workflow.md](docs/workflow.md) | End-to-end conversion workflow |
| [tests/framework.md](tests/framework.md) | Validation and testing strategy |
| [templates/conversion_prompt_template.md](templates/conversion_prompt_template.md) | Legacy prompt template (non-Genie Code invocation) |

## Using With Genie Code

### Prerequisites
- This repo is cloned into your Databricks workspace via Repos
- You have a PowerCenter XML export ready (`.xml` file from PowerCenter Designer or Repository Manager)
- You know the Unity Catalog catalog and schema where the target tables live

### Workflow (one mapping at a time)

1. Place your PowerCenter XML export in the `input/` folder in this repo (e.g. `M_LOAD_FACT_ORDERS.xml`). Place any mapplet XML files there too (`mlt_<name>.xml`).
2. Open `templates/mega_prompt.md` from this repo
3. Fill in the **Mapping File** field with your XML filename
4. Fill in the **Connection Mapping** table — scan the XML for `CONNECTION="..."` attributes and map each one to its Unity Catalog path
5. Fill in **Target** catalog, schema, and table name(s)
6. Fill in **Merge Keys** if known (optional)
7. Mark your **Run Date Handling** selection
8. Add any **Special Instructions** (or leave blank)
9. Copy the entire completed document
10. Open Genie Code in a Databricks notebook within this cloned repo
11. Paste into the Genie Code chat and send
12. The agent runs 4 phases (Analysis → Extraction → Notebook Creation → Cell Review) and writes the notebook to `notebooks/nb_<mapping_name>`
13. Review any `# REVIEW:` items flagged in the summary
14. Pull the latest repo changes via Databricks Repos, then open and run the notebook in dev to validate

## Validating a Converted Notebook

Run this after completing a conversion with `templates/mega_prompt.md`.

### Prerequisites
- A converted notebook already exists in `notebooks/` (e.g. `nb_m_load_fact_orders`)
- You have sample input data for the mapping's source tables (CSV or Parquet files)
- You have a Unity Catalog catalog and schema where temp test tables can be written and dropped

### Workflow

1. Place sample data files in the `tests/data/` folder — one file per source table the mapping reads from
   - Name each file after its source table: `orders.csv`, `customers.parquet`, etc.
   - Files in `tests/data/` are gitignored and will not be committed
2. Open `templates/validation_prompt.md` from this repo
3. Fill in the **Notebook to Test** field with the exact notebook name (e.g. `nb_m_load_fact_orders`)
4. Fill in **Test Catalog** and **Test Schema** — the agent writes temp Delta tables here and drops them at the end
5. Fill in **Run Date** — the date value passed to the notebook's `run_date` widget
6. Fill in **Expected Row Count** if known (optional)
7. Copy the entire completed document
8. Open Genie Code in a Databricks notebook within this cloned repo
9. Paste into the Genie Code chat and send
10. The agent runs 4 phases (Read & Analyze → Sample Data Mapping → Test Notebook Creation → Cell Review) and writes the test notebook to `notebooks/tests/test_nb_<mapping_name>`
11. Pull the latest repo changes via Databricks Repos, then open the test notebook
12. Set the `test_catalog` and `test_schema` widgets if the defaults are not correct, then run all cells
13. Review the validation summary at the end — all checks must pass before promoting the conversion notebook to production
14. The teardown cell at the end drops every table the test created — nothing else is touched

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
3. Update the Quick Reference table in `templates/mega_prompt.md`
4. Update `AGENT.md` if the agent's behavior or rules need to change

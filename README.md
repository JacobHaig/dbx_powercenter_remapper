# DBX PowerCenter Remapper

A knowledge base and agent context repository for converting **Informatica PowerCenter mappings** (XML) into **Databricks PySpark Notebooks**.

## What This Repo Is

This is not a runtime application. It is the **architecture, reference, and context layer** consumed by a Databricks agent that performs the actual conversion. Think of it as the agent's onboarding package and standing instructions.

## What the Agent Does

Given a PowerCenter export XML file, the agent:

1. Parses the `<MAPPING>` XML structure
2. Identifies each `<TRANSFORMATION>` and its type
3. Translates each transformation to the equivalent PySpark / Delta Lake pattern
4. Assembles a valid Databricks notebook with proper cell structure
5. Validates the output against the standards in `docs/conversion_standards.md`

## Quick Navigation

| Document | Purpose |
|---|---|
| [AGENT.md](AGENT.md) | Agent session instructions (read this first if you are the agent) |
| [docs/powercenter_reference.md](docs/powercenter_reference.md) | PowerCenter XML schema and component catalog |
| [docs/transformation_mappings.md](docs/transformation_mappings.md) | PowerCenter → PySpark translation table |
| [docs/conversion_standards.md](docs/conversion_standards.md) | Notebook coding standards |
| [docs/workflow.md](docs/workflow.md) | Step-by-step conversion workflow |
| [templates/conversion_prompt_template.md](templates/conversion_prompt_template.md) | Agent prompt template |
| [tests/framework.md](tests/framework.md) | Validation and testing strategy |

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
1. Update `docs/transformation_mappings.md`
2. Add a fixture XML under `tests/fixtures/`
3. Add the expected notebook output under `tests/expected/`
4. Update `AGENT.md` if agent behavior needs to change

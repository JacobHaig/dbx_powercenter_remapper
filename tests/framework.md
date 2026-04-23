# Testing Framework: Validating Converted Notebooks

This document defines the validation and testing strategy for converted Databricks notebooks. The goal is to confirm that a converted notebook is structurally correct and produces output that matches the original PowerCenter mapping's behavior.

---

## Testing Philosophy

There are two distinct validation layers:

1. **Static validation** — Does the notebook conform to the structure and standards defined in `docs/conversion_standards.md`? This can run without a Databricks cluster.
2. **Functional validation** — Does the notebook produce the same output as the original PowerCenter mapping when run against the same input data? This requires a Databricks cluster and sample data.

---

## Repository Test Structure

```
tests/
├── framework.md             # This file — testing strategy and test code templates
├── conftest.py              # pytest fixtures shared across tests (to be created)
├── static/                  # Static validation tests — no cluster needed (to be created)
│   ├── test_structure.py
│   ├── test_no_powercenter_syntax.py
│   ├── test_standards_compliance.py
│   └── test_expression_translation.py
└── e2e/                     # End-to-end tests — Databricks cluster required (to be created)
    └── test_e2e_simple_insert.py

input/                       # Sample PowerCenter XML files for test runs — place here, gitignored
notebooks/                   # Generated notebooks — test output is validated from here
```

> **Note:** Only `framework.md` exists today. All other files and directories listed above are the intended structure when tests are implemented.

---

## Test Categories

### Category 1: Static Structure Tests (`tests/static/`)

These tests parse the generated `.py` notebook file and assert structural properties.

**`test_structure.py`**

```python
import pytest
import re

def test_notebook_source_line(notebook_content):
    """First line must be the Databricks native notebook marker."""
    assert notebook_content.startswith("# Databricks notebook source"), \
        "First line must be '# Databricks notebook source'"

def test_has_header_cell(notebook_content):
    """Header markdown cell must be present."""
    assert "# MAGIC %md" in notebook_content
    assert "# MAGIC # " in notebook_content  # H1 heading

def test_has_widget_cell(notebook_content):
    """Widget configuration cell must exist."""
    assert "dbutils.widgets.text" in notebook_content

def test_has_imports_cell(notebook_content):
    """Imports cell must contain pyspark imports."""
    assert "from pyspark.sql" in notebook_content

def test_review_items_in_header(notebook_content):
    """Every # REVIEW: comment in the body must appear in the header."""
    inline_reviews = re.findall(r'# REVIEW: (.+)', notebook_content)
    header_section = notebook_content.split("# COMMAND ----------")[1]  # header cell
    for review in inline_reviews:
        assert review[:30] in header_section, \
            f"REVIEW item not listed in header: {review[:30]}"

def test_cell_order(notebook_content):
    """Cells must appear in the correct order: header → widgets → imports → sources → transforms → targets."""
    cells = notebook_content.split("# COMMAND ----------")
    # At minimum: header, widgets, imports must come before source reads
    widget_idx = next(i for i, c in enumerate(cells) if "dbutils.widgets.text" in c)
    import_idx = next(i for i, c in enumerate(cells) if "from pyspark.sql" in c)
    assert widget_idx < import_idx, "Widgets cell must come before imports cell"
```

**`test_no_powercenter_syntax.py`**

```python
POWERCENTER_PATTERNS = [
    r':LKP\.',           # unconnected lookup call syntax
    r':SEQ\.',           # sequence reference syntax
    r'\$\$[A-Z_]+',      # mapping/session parameters
    r'\bIIF\s*\(',       # IIF function
    r'\bDECODE\s*\(',    # DECODE function
    r'\bDD_INSERT\b',    # update strategy constants
    r'\bDD_UPDATE\b',
    r'\bDD_DELETE\b',
    r'\bDD_REJECT\b',
    r'\bSYSDATE\b',      # PowerCenter date function
    r'\bADD_TO_DATE\s*\(',
    r'\bDATE_DIFF\s*\(',
    r'\bTO_CHAR\s*\(',
    r'\bINSTR\s*\(',
    r'\bSUBSTR\s*\(',    # PowerCenter SUBSTR (vs Python substr)
]

def test_no_powercenter_syntax(notebook_content):
    for pattern in POWERCENTER_PATTERNS:
        matches = re.findall(pattern, notebook_content)
        assert not matches, \
            f"PowerCenter syntax found in notebook (pattern={pattern}): {matches[:3]}"
```

**`test_standards_compliance.py`**

```python
def test_no_hardcoded_catalog_names(notebook_content, known_catalog_names):
    """Catalog names must come from widgets, not literals."""
    for catalog in known_catalog_names:
        # Allow catalog name only inside widget default value (first occurrence)
        # but not as a bare string in spark.table() calls
        assert f'spark.table("{catalog}.' not in notebook_content

def test_no_select_star_on_write(notebook_content):
    """Final write cells must not use SELECT *."""
    assert "SELECT *" not in notebook_content.upper() or \
           _is_in_source_read_only(notebook_content)

def test_delta_merge_for_update_strategy(notebook_content, has_update_strategy):
    """If the mapping had an Update Strategy, the notebook must contain a MERGE."""
    if has_update_strategy:
        assert ".merge(" in notebook_content
        assert "DeltaTable" in notebook_content
```

---

### Category 2: Expression Translation Tests (`tests/static/test_expression_translation.py`)

Unit tests for individual expression function translations, independent of any specific mapping.

```python
import pytest
from src.expression_translator import translate_expression  # future module

@pytest.mark.parametrize("pc_expr, expected_pyspark", [
    ("IIF(amount > 100, 'LARGE', 'SMALL')",
     "when(col('amount') > 100, 'LARGE').otherwise('SMALL')"),

    ("UPPER(TRIM(customer_name))",
     "upper(trim(col('customer_name')))"),

    ("TO_DATE(order_date, 'MM/DD/YYYY')",
     "to_date(col('order_date'), 'MM/dd/yyyy')"),

    ("ROUND(amount, 2)",
     "round(col('amount'), 2)"),

    ("ISNULL(email)",
     "col('email').isNull()"),

    ("SUBSTR(zip_code, 1, 5)",
     "substring(col('zip_code'), 1, 5)"),

    ("ADD_TO_DATE(order_date, 'DD', 30)",
     "date_add(col('order_date'), 30)"),
])
def test_expression_translation(pc_expr, expected_pyspark):
    result = translate_expression(pc_expr)
    assert result == expected_pyspark
```

---

### Category 3: End-to-End Functional Tests (Databricks cluster required)

These tests require a running Databricks cluster and are run in CI against a dev cluster using the Databricks CLI or REST API.

**Structure:**

Each test case has:
- A PowerCenter XML fixture in `tests/fixtures/<case_name>/mapping.xml`
- Sample source data as Delta tables in a test catalog
- Expected output as a Delta table snapshot in `tests/expected/<case_name>/`

**Test flow:**

```python
# tests/e2e/test_e2e_simple_insert.py
import pytest
from databricks.sdk import WorkspaceClient

def test_simple_insert_conversion(databricks_client, test_catalog):
    """
    Convert the simple_insert fixture and verify the output matches expected.
    """
    # 1. Read fixture XML
    with open("tests/fixtures/simple_insert/mapping.xml") as f:
        xml_content = f.read()

    # 2. Run the conversion agent (via API call to agent endpoint)
    notebook_content = run_conversion_agent(xml_content, catalog=test_catalog)

    # 3. Import the notebook to Databricks test workspace
    notebook_path = f"/tmp/test_conversions/simple_insert_{test_run_id}"
    databricks_client.workspace.import_notebook(notebook_path, notebook_content)

    # 4. Run the notebook
    job_result = databricks_client.jobs.run_notebook(notebook_path, params={
        "catalog": test_catalog,
        "schema": "test_schema"
    })
    assert job_result.state.result_state == "SUCCESS"

    # 5. Compare output to expected
    actual = spark.table(f"{test_catalog}.test_schema.fact_orders")
    expected = spark.read.parquet("tests/expected/simple_insert/fact_orders/")
    assert actual.count() == expected.count()
    assert sorted(actual.columns) == sorted(expected.columns)
    # Row-level comparison
    diff = actual.subtract(expected)
    assert diff.count() == 0, f"Row differences found: {diff.show()}"
```

---

## Pytest Configuration

**`conftest.py`**

```python
import pytest
import os

@pytest.fixture
def notebook_content(request):
    """Load the generated notebook .py file for the current test case."""
    # Generated notebooks land in notebooks/ at repo root.
    # Test cases pass the notebook name via indirect parametrize or a marker.
    notebook_name = request.param  # e.g. "nb_m_load_fact_orders" (no .py extension)
    notebook_path = f"notebooks/{notebook_name}"
    with open(notebook_path) as f:
        return f.read()

@pytest.fixture
def known_catalog_names():
    return ["prod_catalog", "staging_catalog", "dev_catalog"]

@pytest.fixture
def test_catalog():
    return os.environ.get("TEST_CATALOG", "test_catalog")
```

**`pytest.ini`**

```ini
[pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
markers =
    static: Static tests, no cluster required
    e2e: End-to-end tests requiring a Databricks cluster
    slow: Tests that take more than 60 seconds
```

---

## Test Fixtures: What to Include

Each fixture in `tests/fixtures/` represents one transformation type in isolation. The XML should be minimal — enough to exercise the conversion logic, not a full production mapping.

| Fixture | What it tests |
|---|---|
| `simple_insert` | Source Qualifier → Expression → Target (append write) |
| `upsert_update_strategy` | Update Strategy → Delta MERGE (insert + update) |
| `delete_strategy` | Update Strategy with DD_DELETE |
| `filter_transformation` | Filter with complex IIF expression |
| `lookup_connected` | Connected lookup with broadcast join |
| `lookup_unconnected` | Unconnected lookup called from Expression port |
| `lookup_multi_match` | Lookup with "Use first value" / "Use last value" policy |
| `aggregator_simple` | Aggregator with GROUP BY and SUM/COUNT |
| `aggregator_conditional` | Aggregator with filter condition on aggregation port |
| `router` | Router with 3 groups + default group |
| `joiner_inner` | Normal (inner) Joiner |
| `joiner_outer` | Master Outer and Detail Outer variants |
| `sorter` | Sorter with ASC/DESC and NULL handling |
| `rank_top` | Rank transformation, TOP N |
| `union` | Union of 3 input streams |
| `sequence_generator` | Sequence Generator with NEXTVAL |
| `normalizer` | Normalizer unpivoting 4 repeating groups |
| `mapplet` | Mapping using a reusable mapplet |
| `full_mapping` | Complex real-world mapping with 10+ transformations |

---

## Running Tests

```bash
# Install dependencies
pip install pytest pytest-cov lxml

# Run only static tests (no cluster)
pytest tests/static/ -v -m "not e2e"

# Run all tests including e2e (requires DATABRICKS_HOST and DATABRICKS_TOKEN env vars)
pytest tests/ -v

# Run a specific fixture's test
pytest tests/static/test_structure.py -v -k "simple_insert"

# Run with coverage (when src/ module exists)
pytest tests/static/ --cov=src --cov-report=term-missing

# Run e2e tests only
pytest tests/e2e/ -v -m e2e --timeout=300
```

---

## Data Reconciliation Checklist

After a successful notebook run against production-equivalent data, verify:

- [ ] Row count matches PowerCenter session statistics (source rows in = target rows out ± rejected rows)
- [ ] Key column values match (sample 100 rows, compare primary key + 3 business columns)
- [ ] Aggregation totals match (SUM of amount, COUNT of records) if the mapping includes an Aggregator
- [ ] NULL handling matches (columns that were NULL in PowerCenter output are NULL in Databricks output)
- [ ] Date/time values match (watch for timezone differences — PowerCenter typically runs in server local time; Databricks defaults to UTC)
- [ ] Numeric precision matches (DecimalType(p, s) must match the PowerCenter field precision/scale)

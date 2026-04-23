# Creating Databricks Notebook Assets

This document defines how the conversion agent creates output notebooks. The agent does **not** write `.py` or text files. It creates proper Databricks workspace notebook assets using the Databricks SDK.

---

## Notebook Assets vs Source Files

| | Workspace Notebook Asset | `.py` Source File |
|---|---|---|
| Created via | SDK / REST API / UI | File write to disk |
| Path in workspace | `notebooks/nb_my_mapping` (no extension) | `notebooks/nb_my_mapping.py` |
| Opens as | Native notebook | Plain text / Python file |
| Has cells | Yes — runnable cells with outputs | No |
| Can use `dbutils` | Yes | Only if imported first |

The agent always creates workspace notebook assets — never plain `.py` files.

---

## Required Notebook Source Format

The content passed to the SDK uses the standard Databricks source format:

```
# Databricks notebook source

# COMMAND ----------

# MAGIC %md
# MAGIC # Notebook Title

# COMMAND ----------

# your Python code here

# COMMAND ----------
```

- **First line:** `# Databricks notebook source` — required, must be exact
- **Cell separator:** `# COMMAND ----------` — one per cell boundary
- **Markdown cells:** prefix every line with `# MAGIC`
- **Code cells:** plain Python, no prefix

---

## Creation Pattern (Databricks SDK)

Use this pattern in the Genie Code session to create the notebook asset. This code runs as a cell in the current notebook and creates the target notebook in the workspace.

```python
import base64
from databricks.sdk import WorkspaceClient
from databricks.sdk.service.workspace import ImportFormat, Language

# Build notebook content as a string (see docs/conversion_standards.md for cell order)
NOTEBOOK_SOURCE = """# Databricks notebook source

# COMMAND ----------

# MAGIC %md
# MAGIC # <mapping_name>
# ...full notebook content...
"""

# Derive the repo root from the current running notebook's path
current_path = dbutils.notebook.entry_point \
    .getDbutils().notebook().getContext().notebookPath().get()
# current_path is something like:
#   /Repos/user@company.com/dbx_powercenter_remapper/templates/mega_prompt
# Strip the last two path segments (/templates/mega_prompt) to get repo root:
repo_root = "/".join(current_path.split("/")[:-2])

# Target path — no .py extension
target_path = f"{repo_root}/notebooks/nb_<mapping_name_lowercase>"

# Create the notebook asset
w = WorkspaceClient()
w.workspace.import_(
    path=target_path,
    format=ImportFormat.SOURCE,
    language=Language.PYTHON,
    content=base64.b64encode(NOTEBOOK_SOURCE.encode("utf-8")),
    overwrite=True
)

print(f"Notebook created: {target_path}")
```

**Key points:**
- `format=ImportFormat.SOURCE` — tells Databricks to treat the content as a source-format notebook
- `language=Language.PYTHON` — required when format is SOURCE
- `overwrite=True` — allows re-running if the notebook already exists
- The path has **no `.py` extension** — Databricks adds that internally for source-format notebooks in Repos

---

## Naming Convention

| Mapping Name | Notebook Path |
|---|---|
| `M_LOAD_FACT_ORDERS` | `notebooks/nb_m_load_fact_orders` |
| `M_Orders_v2 (PROD)` | `notebooks/nb_m_orders_v2_prod` |
| `W_CLAIMS_DIM` | `notebooks/nb_w_claims_dim` |

Rules:
- Prefix: `nb_`
- Lowercase
- Spaces and special characters → underscores
- No `.py` extension in the path

---

## REST API Fallback

If the SDK is unavailable, use the REST API directly:

```python
import base64
import requests

host = spark.conf.get("spark.databricks.workspaceUrl")
token = dbutils.notebook.entry_point.getDbutils().notebook().getContext().apiToken().get()

response = requests.post(
    f"https://{host}/api/2.0/workspace/import",
    headers={"Authorization": f"Bearer {token}"},
    json={
        "path": target_path,           # no .py extension
        "format": "SOURCE",
        "language": "PYTHON",
        "content": base64.b64encode(NOTEBOOK_SOURCE.encode()).decode(),
        "overwrite": True
    }
)
response.raise_for_status()
print(f"Notebook created: {target_path}")
```

---

## Mapplet Notebooks

When a Mapplet is large or complex enough to warrant its own notebook (see `docs/transformation_mappings.md`), create it at:

```
{repo_root}/notebooks/mapplets/mlt_<name>
```

Reference it from the calling notebook with:

```python
# %run ./mapplets/mlt_address_cleanse
```

The `%run` path is relative to the calling notebook's location — since both live in `notebooks/`, the relative path is `./mapplets/mlt_<name>`.

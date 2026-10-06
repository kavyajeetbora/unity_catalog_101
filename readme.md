# Databricks Unity Catalog

<img src="https://www.databricks.com/sites/default/files/2026-06/2026-unity-catalog-og.png" height=300/>

Hands-on notes and exercises for **Unity Catalog on Azure Databricks**: setting up the environment, connecting your own storage, organizing data, and governing access.

---

## Unity Catalog in brief

Unity Catalog is Databricks' **centralized governance layer for data and AI assets**. One metastore per region is shared by all workspaces attached to it, so access rules, lineage and auditing are defined once and apply everywhere.

**Core ideas**

- **Object model:** metastore → catalog → schema → table / view / volume / function / model, addressed as `catalog.schema.object`.
- **Storage access:** access connector → storage credential → external location. Databricks reaches your cloud storage through these.
- **Managed vs external:** Unity Catalog either owns the files of a table (managed) or only governs files at a path you choose (external).
- **Governance:** privileges are granted to **groups** and inherit downward, with row filters and column masks for fine-grained control.
- **Observability:** lineage is captured automatically at table and column level, and audit and billing data are exposed through system tables.

---

## Table of contents

| # | Topic | What it covers |
|---|---|---|
| 01 | [Azure setup for Databricks](docs/01_azure_setup_databricks.md) | Build the environment from scratch: resource group, ADLS Gen2, workspace, access connector, credentials, external locations, first catalog and tables, plus cost control and troubleshooting |
| 02 | [External vs managed tables](docs/02_external_vs_managed_tables.md) | Where table data lives, what `DROP` and `UNDROP` do, when to use each type, and production storage practices |
| 03 | [Volumes](docs/03_volumes.md) | Governed storage for non-tabular files, managed vs external volumes, and the `/Volumes/...` path |
| 04 | [Governance and access control](docs/04_governance-access-control.md) | Privileges, inheritance, ownership, groups, admin roles, row filters and column masks, dev/prod design, and debugging "permission denied" |
| 05 | [Data lineage](docs/05_data_lineage.md) | Table and column lineage, reading the graph, lineage and audit system tables, and what lineage can't capture |

**Suggested reading order:** 01 → 02 → 03 → 04 → 05.

---

## Prerequisites

- An Azure subscription (free credits or pay-as-you-go).
- Permission to create resources and assign roles in a resource group.
- Basic SQL.

---

## Repository structure

```
databricks-unity-catalog/
├── data/        sample data
├── docs/        notes 01 to 05 (see the table above)
└── README.md
```

---

## A note on cost

Databricks bills for compute **time**, not query volume. Use short sessions, prefer a small SQL warehouse for SQL-only work, and set a budget alert before you start. Details are in [01 Azure setup](docs/01_azure_setup_databricks.md).
# Unity Catalog: Storage, Managed vs External Tables (Summary)

Notes from the hands-on labs on Azure Databricks. Covers how Databricks reaches your storage, where table data lives, managed vs external tables, and the production way of doing it.

---

## 1. The big picture

A table is two things:

1. **Metadata**: name, columns, permissions. Stored by Unity Catalog in the **metastore**.
2. **Data files**: Delta/Parquet files in cloud storage (ADLS Gen2).

Databricks itself does not store your data. Unity Catalog governs it, and the files live in storage.

**Object hierarchy:** metastore → catalog → schema → table / view / volume / function / model.
Storage-related objects (storage credential, external location, connection, share…) sit directly under the metastore, as siblings of the catalog.

**Workspace vs metastore:** a workspace is a working environment (UI, notebooks, jobs, compute). The metastore is the shared governance layer. One metastore per region, shared by every workspace attached to it.

---

## 2. How Databricks gets access to your storage

```
Storage account (ADLS Gen2)
        ↑  role: Storage Blob Data Contributor
Access Connector (Azure managed identity)
        ↑  referenced by its Resource ID
Storage credential  (Unity Catalog object)
        ↑  used by
External location   (credential + a storage path)
```

| Object | What it is | Analogy |
|---|---|---|
| Access connector | Azure resource holding a managed identity | The ID badge |
| Role assignment | Gives that identity rights on the storage account | The badge is accepted at the gate |
| Storage credential | UC object that wraps the connector's Resource ID | The badge registered in Databricks |
| External location | A credential paired with one storage path | A door the badge opens |

Key points:

- The role must be **Storage Blob Data Contributor** (data access), not Storage Account Contributor (account management).
- An external location is a **path**, not necessarily a whole container. A folder works.
- One credential can back many external locations, as long as the paths are within what it can reach.
- Credentials and external locations belong to the **metastore**, so any attached workspace can use them.
- Two layers of control: the credential controls what Databricks can reach in the cloud, and grants on the external location control which Databricks users can use that path.
- The file-events warning on a new external location is optional (it speeds up Auto Loader). Choose **Force create**.

---

## 3. Where does a managed table's data go?

Lookup order (closest level wins):

1. **Table**: a `LOCATION` on the table makes it an *external* table.
2. **Schema**: `MANAGED LOCATION` on the schema.
3. **Catalog**: `MANAGED LOCATION` on the catalog.
4. **Metastore root storage**: the fallback.

What we observed in the labs:

| Setup | Table location |
|---|---|
| No location on catalog or schema | Under the metastore root (`.../metastore/__unitystorage/catalogs/<id>/tables/<id>`) |
| Location on the catalog | Under the catalog's folder (`.../catalog/__unitystorage/catalogs/<id>/tables/<id>`) |
| Location on the schema | Under the schema's folder |
| `LOCATION` on the table | Exactly that path (external table) |

- A blank **Storage Root** in `DESCRIBE CATALOG EXTENDED` means the catalog has no storage of its own, so it uses the metastore root.
- Catalog and schema locations only affect **managed** tables and volumes. External tables ignore them.
- "External catalog" or "external schema" are loose terms. They are normal catalogs and schemas that have their own managed location.
- Managed vs external is chosen **per table** (and per volume).

---

## 4. Managed vs external tables

| | Managed | External |
|---|---|---|
| Who chooses the path | Unity Catalog | You (`LOCATION`) |
| Who owns the file lifecycle | Unity Catalog | You |
| On `DROP TABLE` | Metadata and data removed | Only metadata removed; files stay |
| Recovery | `UNDROP TABLE` for 7 days | Re-register with `CREATE TABLE ... LOCATION` |
| Formats | Delta (Iceberg also supported) | Delta, Parquet, CSV, JSON and others |
| Optimization | Predictive optimization can run automatically | You maintain it |
| Direct file access | Not intended | Possible, but bypasses Unity Catalog permissions |

**Drop behaviour in detail:** after `DROP` on a managed table, UNDROP works for 7 days. After that the data is marked for deletion and removed during regular maintenance, typically within 7 to 30 days. The 7 days is the recovery window, not an exact deletion date.

### The simple test for choosing

> Does something **outside Databricks** (Synapse, ADF, a DuckDB API) read or write these files directly?
> **Yes → external. No → managed.**

### Applied to the Coca-Cola project

| Layer | Choice | Why |
|---|---|---|
| Bronze (daily files written by Synapse) | External | Another system writes the files to a fixed path |
| Silver (cleaned, Databricks only) | Managed | No outside access needed |
| Gold (feeds the DuckDB API) | External if the API reads by path, otherwise managed | Depends on who reads the files |

Mixing both inside one catalog is normal. Keep a **single writer** per table: do not let Synapse and Databricks both write the same Delta table.

---

## 5. Production practices

1. **Managed is the default**, in both dev and prod. Use external only for a specific reason (other systems need the files, data must outlive the table, existing files).
2. **Give each catalog its own storage** (ideally its own storage account, at least its own container) using `MANAGED LOCATION`. This gives separate security, cost tracking and delete-risk.
3. **Separate dev and prod by path or storage account**, not just by catalog name. Example: prod paths live under a `koine-prod` container.
4. **Keep a separate shared landing area** for files that people or other tools upload (Storage Explorer, ADF, Synapse). Give it its own external location. Read it as external tables, or as an **external volume** if they are plain files.
5. **Never touch the catalog's managed folder by hand.** Folders are named with IDs, and manual uploads, edits or deletes can corrupt tables.
6. **External table paths must not overlap** the catalog's managed storage or another table's path.
7. **Unity Catalog permissions only apply to access through Databricks.** Anyone using the storage directly is governed by Azure RBAC and ACLs. Give storage-level access only to the access connector and a few admins for managed folders, and wider access only for the landing area.
8. **Single writer per table**, and keep the number of external locations small and meaningful (per layer or per domain, not per table).

Suggested layout:

```
storage account
├── koine-prod/          managed storage for the prod catalog (Unity Catalog only, hands off)
│   └── catalog/
├── landing/             shared uploads from other tools (external location + external volume)
└── bronze/              external bronze data written by Synapse (external location)
```

---

## 6. Things we ran into (and the fix)

| Situation | Explanation / fix |
|---|---|
| File events permissions warning when creating an external location | Optional feature. Choose **Force create**. |
| Account console login: "user does not exist in tenant" | The personal Microsoft account does not exist in your Azure directory. Not needed for the labs; skip it. |
| `Standard` pricing tier | Retired for new workspaces, and Unity Catalog needs Premium anyway. Use **Premium**. |
| Workspace type: Serverless vs Hybrid | **Hybrid** keeps the workspace storage in your own subscription, which matches classic setups and interviews. |
| New workspace already had a metastore and catalogs | New workspaces are attached to the existing regional metastore. Do not delete or recreate it. |
| Catalog "Storage Root" blank | No storage of its own, so it falls back to the metastore root. |
| Catalog not visible in the UI | Refresh, clear filters, and verify with `SHOW CATALOGS` in SQL. |
| Table path overlaps another path | Use a separate folder for each external table, outside the managed storage. |
| Azure Data Explorer vs Storage Explorer | Browsing files is **Azure Storage Explorer** (or the portal). Azure Data Explorer is a different service. |
| Serverless cost settings | Notebooks have no cheaper tier. Standard mode only applies to jobs and pipelines. Labs are tiny, so cost is minimal. |

---

## 7. SQL cheat sheet

```sql
-- Inspect
SELECT current_metastore();
SHOW CATALOGS;
SHOW SCHEMAS IN <catalog>;
SHOW STORAGE CREDENTIALS;
SHOW EXTERNAL LOCATIONS;
DESCRIBE EXTERNAL LOCATION <name>;
DESCRIBE CATALOG EXTENDED <catalog>;
DESCRIBE EXTENDED <catalog>.<schema>.<table>;   -- see Type and Location

-- Catalog with its own storage
CREATE CATALOG IF NOT EXISTS koine_prod
MANAGED LOCATION 'abfss://<container>@<account>.dfs.core.windows.net/catalog';

CREATE SCHEMA IF NOT EXISTS koine_prod.bronze;

-- Managed table
CREATE TABLE koine_prod.bronze.fact_daily_sales
(sale_date DATE, outlet_code STRING, sku_id STRING, volume_uc DOUBLE);

-- External table
CREATE TABLE koine_prod.bronze.fact_daily_sales_ext
(sale_date DATE, outlet_code STRING, sku_id STRING, volume_uc DOUBLE)
LOCATION 'abfss://<container>@<account>.dfs.core.windows.net/<folder>';

-- Drop and recover
DROP TABLE <catalog>.<schema>.<table>;
UNDROP TABLE <catalog>.<schema>.<table>;        -- managed tables, within 7 days
```

---

## 8. Interview questions

**Managed vs external tables: what happens on DROP?**
Managed tables keep their data in Unity Catalog's managed storage, so dropping one removes the data, though UNDROP works for 7 days. External tables point at a path I own, so a drop removes only the metadata. I default to managed for governance and automatic optimization, and use external when other systems need direct access to the files or the data must outlive the table.

**What are a storage credential and an external location, and how do they relate?**
A storage credential wraps an Azure managed identity (through the access connector) and defines what Databricks can reach in the cloud. An external location pairs a credential with a specific storage path, and grants on it control which Databricks users can use that path.

**Where is a managed table's data stored?**
Closest level wins: the table's own path (making it external), then the schema's managed location, then the catalog's, then the metastore root storage.

**Why give a catalog its own storage?**
Isolation of security, cost and ownership between environments or business units, and independence from the metastore's default storage.

**Does Unity Catalog protect files read directly from storage?**
No. Unity Catalog permissions apply only to access through Databricks. Direct storage access is governed by Azure RBAC and ACLs, so storage-level access must be restricted separately.

---

## 9. Next topics

- Volumes (Q8): managed vs external, when to use a volume instead of a table.
- Access control: GRANT/REVOKE, inheritance, `USE CATALOG` / `USE SCHEMA`, debugging "permission denied".
- Row filters and column masks, lineage, audit and system tables, Delta Sharing.
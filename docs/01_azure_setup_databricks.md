# Setting Up Azure Databricks from Scratch (with Unity Catalog and Your Own Storage)

A complete, step-by-step setup: Azure resources, Databricks workspace, access to your own ADLS Gen2 storage, and your first catalog, schema and tables.

Placeholders used below: `<account>` is your storage account name. Replace the example names with your own if you prefer.

---

## 0. Before you start

**What you need**
- An Azure account with an active subscription (credits or pay-as-you-go).
- Owner or Contributor rights on a resource group (needed to create resources and assign roles).

**Decisions made in this guide**

| Decision | Choice | Why |
|---|---|---|
| Workspace pricing tier | **Premium** | Unity Catalog needs Premium or above. The Standard tier is retired for new workspaces. |
| Workspace type | **Hybrid** | Workspace storage sits in your own subscription (the classic setup). |
| Compute | **Serverless** (notebooks) | No cluster to manage and no idle cost. Notebooks have no cheaper tier. |
| Region | One region for everything | The workspace must be in the same region as the Unity Catalog metastore it attaches to. In our setup this was Central India. |

**Cost note:** serverless compute and a small LRS storage account cost very little at lab scale. Set a budget alert (Part G) anyway.

---

## Part A: Azure resources

### Step 1. Create a resource group
1. Azure portal → search **Resource groups** → **Create**.
2. Choose your subscription, name it (for example `rg-uc-learning`), and pick your region.
3. **Review + create** → **Create**.

### Step 2. Create the storage account (the data lake)
1. Search **Storage accounts** → **Create**.
2. **Basics:**
   - Resource group: the one from Step 1.
   - Storage account name: globally unique, lowercase letters and digits only (`<account>`).
   - Region: same as the workspace will use.
   - Performance: **Standard**.
   - Redundancy: **LRS** (cheapest).
3. **Advanced** tab: tick **Enable hierarchical namespace**. This is what turns Blob storage into ADLS Gen2, and it can't be easily changed later.
4. **Review + create** → **Create**, then wait for the deployment.

### Step 3. Create containers
1. Open the storage account → **Containers** → **+ Container**.
2. Create the containers you need. A simple learning layout:

| Container | Purpose |
|---|---|
| `managed-data` | Managed storage for a catalog (Unity Catalog only, don't touch by hand) |
| `external-data` | Files for external tables |

For a production-style layout, use one container per purpose, for example `koine-prod` (catalog managed storage), `landing` (shared uploads) and `bronze` (external bronze data).

### Step 4. Create the Databricks workspace
1. Search **Azure Databricks** → **Create**.
2. **Basics:**
   - Subscription and resource group: as above.
   - Workspace name: for example `dbx-uc-learning`. This is only a label for the Azure resource. It doesn't appear in the workspace URL and can't be renamed later.
   - Region: same region as above.
   - Pricing tier: **Premium** (not Trial, if you want to save the 14-day trial for later).
   - Workspace type: **Hybrid**.
   - Managed resource group name: `managed-dbx-uc-learning` (a `managed-` prefix makes it easy to recognise).
3. Leave the remaining tabs on their defaults → **Review + create** → **Create**.
4. When deployment finishes, open the resource and click **Launch workspace**.

**About the managed resource group:** Databricks creates and manages it. It typically holds a virtual network, a network security group and the workspace storage account. Never edit or delete anything inside it.

### Step 5. Create the Access Connector
The access connector is the Azure identity that Databricks uses to reach your storage.

1. Search **Access Connector for Azure Databricks** → **Create**.
2. Same subscription, resource group and region. Name it `ac-uc-learning`. Keep the default (system-assigned) identity.
3. **Review + create** → **Create**.
4. Open the connector → **Overview** → **JSON View** and copy the **Resource ID**. It looks like `/subscriptions/.../accessConnectors/ac-uc-learning`. You need it in Step 7.

> Your workspace may already contain an automatically created connector (often `unity-catalog-access-connector` in the managed resource group). That one belongs to the workspace's default storage, so leave it alone and create your own.

### Step 6. Give the connector access to your storage
1. Open the **storage account** → **Access control (IAM)** → **Add** → **Add role assignment**.
2. Role: **Storage Blob Data Contributor** → **Next**. This is the data-access role, not Storage Account Contributor.
3. **Assign access to:** Managed identity → **Select members**.
4. Managed identity type: **Access Connector for Azure Databricks** → select `ac-uc-learning` → **Select**.
5. **Review + assign**. It can take a few minutes to take effect.
6. Verify: **Access control (IAM) → Role assignments** should list the connector with Storage Blob Data Contributor.

---

## Part B: Connect the storage to Unity Catalog (in the workspace)

New Azure workspaces already have Unity Catalog enabled and a regional metastore attached. You don't need to create a metastore or use the account console. Workspace admin rights are enough for everything below.

### Step 7. Create the storage credential
1. In the workspace, click **Catalog** (left sidebar) → the **gear icon** → **Credentials** → **Create credential**.
2. Keep **Storage Credential** selected, with credential type **Azure Managed Identity**.
3. Fill in:
   - Credential name: `cred-uc-learning`
   - Access connector ID: the Resource ID from Step 5
   - User assigned managed identity ID: leave blank
4. **Create**.

### Step 8. Create external locations
An external location is a storage path that Unity Catalog is allowed to use, through a credential. Create one for each path you will use.

1. **Catalog → External Data → External Locations → Create location.**
2. Fill in:
   - Name: for example `loc-managed`
   - URL: `abfss://managed-data@<account>.dfs.core.windows.net/`
   - Storage credential: `cred-uc-learning`
3. **Create**, then click **Test connection**.
4. If you see a **file events permissions** warning, ignore it. It is an optional feature, and the credential can already read and write. Choose **Force create**.
5. Repeat for `external-data` (name it `loc-external`).

### Step 9. Verify the setup
Open a notebook (**Workspace → Create → Notebook**, language **SQL**, **Connect → Serverless**) and run each line in its own cell:

```sql
SELECT current_metastore();
SHOW CATALOGS;
SHOW STORAGE CREDENTIALS;
SHOW EXTERNAL LOCATIONS;
```

You should see your credential and both external locations listed.

---

## Part C: Create your first objects

### Step 10. Create a catalog with its own storage
```sql
CREATE CATALOG IF NOT EXISTS learn_uc
MANAGED LOCATION 'abfss://managed-data@<account>.dfs.core.windows.net/catalog';

DESCRIBE CATALOG EXTENDED learn_uc;
```
**Storage Root** should show your path. (A blank Storage Root would mean the catalog uses the metastore's root storage.)

### Step 11. Create a schema
```sql
CREATE SCHEMA IF NOT EXISTS learn_uc.sales;
```

### Step 12. Create a managed table
```sql
CREATE TABLE learn_uc.sales.orders_managed (order_id INT, outlet STRING, volume_uc INT);
INSERT INTO learn_uc.sales.orders_managed VALUES (1,'Outlet A',10),(2,'Outlet B',20);
```

### Step 13. Create an external table
The only difference is the `LOCATION` line. It must be inside an external location and must not overlap the catalog's managed storage.
```sql
CREATE TABLE learn_uc.sales.orders_external (order_id INT, outlet STRING, volume_uc INT)
LOCATION 'abfss://external-data@<account>.dfs.core.windows.net/orders_external';
INSERT INTO learn_uc.sales.orders_external VALUES (1,'Outlet A',10),(2,'Outlet B',20);
```

### Step 14. Compare them
```sql
DESCRIBE EXTENDED learn_uc.sales.orders_managed;
DESCRIBE EXTENDED learn_uc.sales.orders_external;
```
- **Type:** `MANAGED` vs `EXTERNAL`.
- **Location:** the managed table's path is chosen by Unity Catalog (`.../catalog/__unitystorage/.../tables/<id>`). The external table's path is exactly the one you wrote.
- Browse both folders in the portal (**Containers**) and look for the `_delta_log` folder and the Parquet files.

### Step 15. Drop and recover (the key experiment)
```sql
DROP TABLE learn_uc.sales.orders_managed;
DROP TABLE learn_uc.sales.orders_external;
```
Check the storage: the external table's files are still there.

```sql
UNDROP TABLE learn_uc.sales.orders_managed;     -- works for 7 days
CREATE TABLE learn_uc.sales.orders_external
LOCATION 'abfss://external-data@<account>.dfs.core.windows.net/orders_external';
```
Both tables return their 2 rows again.

---

## Part D: Production-style layout (example for the Coke project)

```
storage account
├── koine-prod/          managed storage for the koine_prod catalog (Unity Catalog only)
├── landing/             shared uploads from other tools
└── bronze/              external bronze data written by Synapse
```

```sql
CREATE CATALOG IF NOT EXISTS koine_prod
MANAGED LOCATION 'abfss://koine-prod@<account>.dfs.core.windows.net/catalog';

CREATE SCHEMA IF NOT EXISTS koine_prod.bronze;

-- External, because Synapse writes these files
CREATE TABLE koine_prod.bronze.fact_daily_sales
(sale_date DATE, outlet_code STRING, sku_id STRING, volume_uc DOUBLE)
LOCATION 'abfss://koine-prod@<account>.dfs.core.windows.net/bronze/fact_daily_sales';
```

Rules of thumb:
- Managed is the default. Use external only when another system reads or writes the files, or the data must outlive the table.
- Prod gets its own paths (ideally its own storage account).
- Never upload files into a catalog's managed folder. Use a separate landing area.
- External table paths must not overlap each other or the managed storage.
- One writer per table.

---

## Part E: Cost control

1. **Budget alert:** Azure portal → **Cost Management + Billing** → **Budgets** → create a small budget (a few dollars) with an email alert.
2. **Use serverless only.** Avoid classic all-purpose clusters, which bill for idle VMs.
3. **Don't leave long queries or streams running.** Cancel anything that runs for more than a couple of minutes.
4. **When finished with the labs:** delete the resource group (this removes the workspace, storage account and connector), or keep it and rely on the fact that an idle workspace with no compute costs almost nothing apart from small storage charges.

---

## Part F: Troubleshooting

| Problem | Likely cause and fix |
|---|---|
| External location test fails with 403 | The role assignment hasn't propagated yet (wait a few minutes), or the role is missing on the storage account. Check **IAM → Role assignments**. |
| File events warning | Optional. Choose **Force create**. |
| `CREATE CATALOG` fails with a permission error | You need `CREATE CATALOG` on the metastore, and `CREATE MANAGED STORAGE` on the external location covering the path. Locations you create yourself make you the owner. |
| `CREATE TABLE ... LOCATION` fails | The path isn't inside an external location, overlaps another table or managed storage, or you lack privileges on the location. |
| Catalog not visible in the UI | Refresh, clear filters, and check with `SHOW CATALOGS`. |
| Account console login: "user does not exist in tenant" | The personal Microsoft account isn't in your Azure directory. The account console isn't needed for this setup, so skip it. |
| Workspace creation lists no Standard tier | It is retired. Use **Premium**. |
| Can't attach to a metastore | Workspace and metastore must be in the same region. |

---

## Checklist

- [ ] Resource group created
- [ ] Storage account created with hierarchical namespace enabled
- [ ] Containers created
- [ ] Databricks workspace (Premium, Hybrid) created and launched
- [ ] Access connector created, Resource ID copied
- [ ] Storage Blob Data Contributor assigned to the connector on the storage account
- [ ] Storage credential created
- [ ] External locations created and tested
- [ ] Catalog with managed location created
- [ ] Managed and external tables created and compared
- [ ] Budget alert set
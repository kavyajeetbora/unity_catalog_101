# Unity Catalog Lineage: Refresher

How Unity Catalog tracks where data comes from and where it goes, how to read the lineage graph, what it can and can't tell you, and how it connects to audit and system tables.

<img src="https://www.databricks.com/sites/default/files/inline-images/data-lineage-img.jpeg" height = 400/>

---

## 1. What lineage is

**Data lineage** is the map of how data flows between objects: which tables a table was built from (**upstream**), and which tables, notebooks, jobs and dashboards use it (**downstream**).

Unity Catalog captures this **automatically**, at two levels:

- **Table level:** table A feeds table B.
- **Column level:** column `total` in B is computed from columns `x` and `y` in A.

You don't instrument anything or configure a tool. It is recorded as queries run.

---

## 2. Why it matters

| Need | How lineage helps |
|---|---|
| **Root-cause analysis** | A number in a report looks wrong. Trace upstream through each layer to the source table and column |
| **Impact analysis** | Before changing or dropping a column, see everything downstream that depends on it |
| **Compliance and audit** | Show where sensitive data flows and which assets consume it |
| **Understanding** | New team members see how a dataset is produced without reading all the code |
| **Ownership** | Find which job, notebook or pipeline writes a table |

---

## 2a. How it is captured

- Lineage is recorded for queries run on **Unity Catalog-enabled compute** (SQL warehouses, serverless compute, and clusters attached to a Unity Catalog metastore).
- Statements that read one Unity Catalog object and write another create lineage: `CREATE TABLE ... AS SELECT`, `INSERT ... SELECT`, `MERGE`, views and DataFrame writes.
- It also links **notebooks, jobs, pipelines and dashboards** to the tables they read or write.
- It is **permission-aware**: you only see lineage for objects you have access to. Hidden objects don't appear in your graph.
- It is **not instant**. Lineage can take a few minutes to appear after the queries run.

---

## 3. Viewing lineage in the UI

1. Open **Catalog** and navigate to a table.
2. Open the **Lineage** tab.
3. Click **See lineage graph**.
4. Expand upstream and downstream nodes to walk the flow.
5. Click a **column** to see column-level lineage, meaning which source columns feed it.

**Use it two ways:**
- **Upstream** (what feeds this?) for debugging a wrong number.
- **Downstream** (what uses this?) for impact analysis before a change.

<img src="https://docs.databricks.com/aws/en/assets/images/uc-lineage-details-598f25ffcce29065c3f5e9117f01e62d.png" height=400/>

---

## 4. Lineage in SQL: system tables

The same information is queryable from the system catalog:

| Table | Content |
|---|---|
| `system.access.table_lineage` | Table-to-table relationships |
| `system.access.column_lineage` | Column-to-column relationships |

```sql
SELECT source_table_full_name, target_table_full_name, event_time
FROM system.access.table_lineage
WHERE target_table_full_name = 'my_catalog.gold.summary'
ORDER BY event_time DESC;
```

Use these when you want to **script** impact checks or build reports.

**Access to system tables:**
- An **account admin** must enable each system schema (through the SystemSchemas API, CLI or SDK, not through a SQL statement).
- No user has access by default. An admin has to grant `USE SCHEMA` and `SELECT` on the schema.
- Billing tables hold account-wide data, so limit access to admins.
- If the `access` schema doesn't appear in `SHOW SCHEMAS IN system`, it isn't enabled or visible to you.

---

## 5. Worked example (generic)

Build a small flow and look at its graph:

```sql
CREATE SCHEMA IF NOT EXISTS my_catalog.silver;
CREATE SCHEMA IF NOT EXISTS my_catalog.gold;

-- Silver: cleaned data from a raw table
CREATE OR REPLACE TABLE my_catalog.silver.orders_clean AS
SELECT order_id, region, quantity, price, quantity * price AS amount
FROM my_catalog.bronze.orders_raw
WHERE quantity IS NOT NULL;

-- Gold: aggregate from silver
CREATE OR REPLACE TABLE my_catalog.gold.sales_by_region AS
SELECT region, SUM(amount) AS total_amount, COUNT(*) AS orders
FROM my_catalog.silver.orders_clean
GROUP BY region;
```

Open the lineage graph for `gold.sales_by_region`:
- **Upstream:** `silver.orders_clean`, and behind it `bronze.orders_raw`.
- **Column level:** click `total_amount` and trace it to `amount` in silver, and then to `quantity` and `price` in bronze.
- **Downstream from bronze:** silver, then gold. This is the impact-analysis view.

**Scenario drill:** change a value in the raw table, rebuild silver and gold, then trace the changed gold number back through the graph.

---

## 6. What lineage can't do

- **Work done outside Databricks isn't captured.** Another engine reading files directly, or an external tool writing files to a path, doesn't appear.
- **Lineage shows relationships, not security.** Derived tables do **not** inherit the source table's row filters, column masks or grants. A copy or aggregate of protected data is unprotected unless you protect it too.
- **History is limited.** It isn't kept forever, so don't treat it as a permanent archive.
- **You see only what you can access,** so a graph can look incomplete to a user with limited privileges.

To bring outside systems into the picture, Unity Catalog has an **external metadata** object for registering them, so they appear in the graph as upstream sources or downstream consumers.

---

## 7. Related: audit and other system tables

Lineage is one of several system tables, and the same enabling and access rules apply.

| Table | Answers |
|---|---|
| `system.access.table_lineage`, `column_lineage` | Where did this data come from, and what uses it? |
| `system.access.audit` | Who did what, and when: grants, table creation, access events |
| `system.billing.usage` | What is costing money (DBU usage by product and SKU) |

An audit query has this shape (columns vary by action):
```sql
SELECT event_time, user_identity.email, action_name
FROM system.access.audit
WHERE service_name = 'unityCatalog'
ORDER BY event_time DESC;
```

---

## 8. Common questions, with short answers

**What is lineage in Unity Catalog?**
An automatic record of how data flows between tables and columns, and which notebooks, jobs and dashboards use them. It is captured from queries on Unity Catalog compute, with no setup.

**How do you find the source of a wrong number in a report table?**
Open the table's lineage graph, trace upstream through each layer to the source, and use column-level lineage to see which source columns feed the metric. Then check the transformation at the step where the value diverges.

**How do you know what breaks if you change a column?**
Use the downstream view (or the lineage system tables) to list every table, notebook, job and dashboard that depends on it.

**What are the limitations?**
Only work on Unity Catalog compute is captured, so direct file access and external tools are invisible unless registered as external metadata. Lineage doesn't copy security, and history isn't kept forever.

**How would you audit who accessed a sensitive table?**
Query the audit system table, filtered by the table's name, and look at who accessed it and any grant changes. An account admin has to enable the system schemas first.

---

## 9. Practice labs to repeat

1. Build raw → clean → summary tables with `CREATE TABLE AS SELECT` and open the graph for the summary table.
2. Click a column and trace it back to source columns.
3. Open the raw table's downstream view and list what would be affected by a change.
4. Change a source value, rebuild the layers, and trace the changed number back.
5. Check `SHOW SCHEMAS IN system` to see whether system tables are available to you.
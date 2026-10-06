# Unity Catalog Access Control: Refresher

A practical summary of how permissions and fine-grained security work in Unity Catalog on Azure Databricks: who can see what, how grants flow, how to organize groups and environments, and how to debug access problems.

<img src="https://www.techjaga.com/assets/images/blog/unity-catalog-diagram.png" height=400/>

---

## 1. The mental model

Access control answers one question: **may this principal do this action on this object?**

- **Principal:** a user, a group, or a service principal.
- **Privilege:** the action, such as `SELECT`, `MODIFY` or `USE CATALOG`.
- **Securable:** the object it applies to (catalog, schema, table, view, volume, function, external location, and so on).

Rules that hold everywhere:

1. **Everything is denied by default.** A new user sees nothing until granted something.
2. **There is no explicit DENY.** Access is the sum of grants. To remove access you revoke the grant.
3. **The owner of an object has full control** over it, including granting and revoking on it. Ownership does not appear in `SHOW GRANTS`.
4. **Grants inherit downward** (catalog → schema → table).
5. **Grant to groups, not individuals.**

---

## 2. Common privileges

| Privilege | Applies to | What it allows |
|---|---|---|
| `USE CATALOG` | Catalog | Reference the catalog. Needed for anything inside it |
| `USE SCHEMA` | Schema | Reference the schema. Needed for anything inside it |
| `SELECT` | Table, view, schema, catalog | Read data |
| `MODIFY` | Table, schema, catalog | Insert, update, delete, merge |
| `CREATE TABLE` | Schema, catalog | Create tables in it |
| `CREATE SCHEMA` | Catalog | Create schemas in it |
| `BROWSE` | Catalog | See the **names** of objects without being able to read them |
| `READ VOLUME` / `WRITE VOLUME` | Volume | Read or write files in the volume |
| `CREATE EXTERNAL TABLE`, `READ FILES`, `WRITE FILES` | External location | Use that storage path |
| `MANAGE` | Various | Manage privileges and ownership of the object |

Naming notes:
- The privileges are always **`USE CATALOG`** and **`USE SCHEMA`**. A bare `USE` is rejected, and `USAGE` is the older Hive-metastore term.
- `ALL PRIVILEGES` is a shorthand for everything applicable to an object, so use it carefully.

---

## 3. Inheritance and the three privileges to read a table

To query `catalog.schema.table`, a principal needs **all three**:

1. `USE CATALOG` on the catalog
2. `USE SCHEMA` on the schema
3. `SELECT` on the table, **or** on the schema or catalog (inherited)

What this means in practice:

| Granted at catalog level | Can they query tables? |
|---|---|
| `USE CATALOG` only | No |
| `USE CATALOG` + `SELECT` | No, still missing `USE SCHEMA` |
| `USE CATALOG` + `USE SCHEMA` + `SELECT` | Yes, on **every** table in **every** schema, including future ones |

Inheritance covers **future** objects too. A schema-level `SELECT` applies to tables created later, which is convenient and also a risk (see section 11).

`SHOW GRANTS` on a table shows inherited privileges, and its `ObjectType` column tells you at which level each one was granted.

---

## 4. Ownership

- The **creator** of an object is its owner by default.
- The owner has full control and can grant and revoke privileges on it.
- **Ownership is not inherited.** Tables inside a schema keep their own owners when the schema's owner changes.
- Best practice: let a **group** own catalogs and schemas, so objects don't depend on one person.

```sql
ALTER CATALOG prod_catalog OWNER TO `data_admins`;
ALTER SCHEMA prod_catalog.gold OWNER TO `data_admins`;
```

Check the owner:
```sql
DESCRIBE CATALOG EXTENDED prod_catalog;     -- see the Owner row
DESCRIBE SCHEMA EXTENDED prod_catalog.gold;
```

Cautions: only transfer ownership to a group you belong to (otherwise you lose control and need an admin to fix it), and keep the owner group small, since its members can regrant or revoke anyone's access.

---

## 5. Groups and identities

### Why groups
Granting to groups means joining, leaving or changing role is a **membership change**. The grants themselves stay stable, which avoids stale access and makes audits simple.

### Suggested groups by role
| Group | Typical privileges |
|---|---|
| `data_admins` | Own catalogs and schemas, manage grants |
| `data_engineers` | Build pipelines: `SELECT`, `MODIFY`, `CREATE TABLE` where they work |
| `analysts` | Read curated data only: `USE CATALOG`, `USE SCHEMA`, `SELECT` on the gold schema |

### Account groups vs workspace groups
- **Account-level groups** are the ones Unity Catalog governance should use. They work across workspaces.
- The built-in **`admins`** group is a **workspace** group, so `is_account_group_member('admins')` returns false. Create an account-level admin group for governance.
- Check membership in SQL:

```sql
SELECT current_user() AS me,
       is_account_group_member('analysts') AS account_level,
       is_member('analysts') AS workspace_level;
```

### Compute access is separate
Reading data takes **two** permissions:
1. Permission to **use compute** (for example "Can use" on a SQL warehouse, granted to a group).
2. Privileges on the **data** (the grants above).

Missing either one blocks the user.

### Membership changes take time
Changes can take a few minutes to apply, and sessions read membership at start. If a change doesn't seem to work, open a fresh session or restart the warehouse and wait a few minutes before concluding that anything is wrong.

### Adding an external person
Invite them in Microsoft Entra ID (as a guest), accept the invitation, add them to the workspace, then add them to the right group.

---

## 6. The admin roles

| Role | Where it lives | Can do |
|---|---|---|
| **Account admin** | Account console | Manage the account: metastores, workspace assignment, users and groups, enabling system tables and external sharing |
| **Metastore admin** | The metastore | Manage privileges and ownership across the metastore, create catalogs |
| **Workspace admin** | One workspace | Manage users, compute, jobs and settings in that workspace |

The common trap: **none of these roles gives automatic access to table data.** A workspace admin has no special rights on tables. A metastore admin can manage privileges and take ownership, but that is a deliberate step. Assign admin roles to groups.

Metastore-level privileges such as `CREATE CATALOG` do not flow down to the data inside catalogs.

---

## 7. Visibility: BROWSE and discoverability

- The **`account users`** group holds `BROWSE` on catalogs by default. Everyone can see the **names** of catalogs, schemas and tables, but objects they can't read appear greyed out.
- This is normal and harmless for most teams, and it helps people find data and request access.
- Table names can still reveal things. For a sensitive catalog, switch discoverability off:

```sql
REVOKE BROWSE ON CATALOG prod_catalog FROM `account users`;
```

---

## 8. Fine-grained security

Table grants are all-or-nothing, so two features add finer control.

### Row filters
A function attached to a table. It returns `true` for rows the caller may see.

```sql
CREATE OR REPLACE FUNCTION prod_catalog.gold.region_filter(region_value STRING)
RETURNS BOOLEAN
RETURN
  is_account_group_member('data_admins')
  OR (is_account_group_member('north_team') AND region_value = 'North')
  OR (is_account_group_member('south_team') AND region_value = 'South');

ALTER TABLE prod_catalog.gold.sales
SET ROW FILTER prod_catalog.gold.region_filter ON (region);
```
Anyone matching no condition sees **nothing** (fail closed).

### Column masks
A function attached to a column. It returns the real value or a masked one.

```sql
CREATE OR REPLACE FUNCTION prod_catalog.gold.mask_phone(phone STRING)
RETURNS STRING
RETURN CASE
  WHEN is_account_group_member('data_admins') THEN phone
  ELSE CONCAT(LEFT(phone, 3), '*******') END;

ALTER TABLE prod_catalog.gold.customers
ALTER COLUMN phone SET MASK prod_catalog.gold.mask_phone;
```

### Dynamic views
An ordinary view whose SQL checks the caller's group to filter rows or mask columns. The older technique, still useful for the **approved view** pattern: a view runs with its owner's access, so analysts can be granted `SELECT` on an approved view and **no access to the base table**. Row filters and masks can't be attached to views.

### Attribute-based access control (ABAC)
Tag columns or tables (for example `pii`), then write **one policy** that masks or filters every matching object across a catalog or schema. It scales better than attaching functions table by table.

### Rules for filters and masks
- Group names inside the functions must **match real group names**, or nobody matches.
- **Exempt admins and pipeline identities**, or they see no rows when reading or merging.
- Filters and masks apply to every query made **through Databricks**, however the SQL is written.
- On **dedicated** access mode compute, filtering is handed to serverless compute (extra cost). **Standard** access mode and **SQL warehouses** enforce natively.
- **Derived tables do not inherit** a source table's filters or masks. A copy or aggregate built from a protected table is unprotected unless you protect it too.

---

## 9. Designing access across environments

### Access Control Matrix

<img src="https://www.databricks.com/sites/default/files/2023-08/getting-started-databricks-groups-permissions-02.png" height=800/>

### Layout
- **One catalog per environment** (`dev_catalog`, `qa_catalog`, `prod_catalog`), each with schemas for layers (`bronze`, `silver`, `gold`).
- The same schema and table names exist in every environment, so code only changes the catalog name (a parameter).
- Each catalog gets **its own managed storage**, and production ideally its own storage account.

### Example access matrix
| Group | Dev | Prod |
|---|---|---|
| Engineers | Build freely | Read-only on silver and gold, no writes |
| Pipeline identity (or admin group) | Full | The only group that writes |
| Analysts | None | `USE CATALOG`, plus `USE SCHEMA` and `SELECT` on **gold only** |

Principle: **people write in dev, pipelines write in prod, analysts read only the curated layer.** Code moves between environments, never data.

### Grants in practice
```sql
-- Dev: engineers build (catalog-level is acceptable for low-risk data)
GRANT USE CATALOG ON CATALOG dev_catalog TO `data_engineers`;
GRANT USE SCHEMA, SELECT, MODIFY, CREATE TABLE ON CATALOG dev_catalog TO `data_engineers`;

-- Prod: the pipeline/admin group writes
GRANT USE CATALOG ON CATALOG prod_catalog TO `data_admins`;
GRANT USE SCHEMA, SELECT, MODIFY, CREATE TABLE ON CATALOG prod_catalog TO `data_admins`;

-- Prod: engineers read silver and gold
GRANT USE CATALOG ON CATALOG prod_catalog TO `data_engineers`;
GRANT USE SCHEMA, SELECT ON SCHEMA prod_catalog.silver TO `data_engineers`;
GRANT USE SCHEMA, SELECT ON SCHEMA prod_catalog.gold TO `data_engineers`;

-- Prod: analysts read gold only
GRANT USE CATALOG ON CATALOG prod_catalog TO `analysts`;
GRANT USE SCHEMA, SELECT ON SCHEMA prod_catalog.gold TO `analysts`;
```

### Workspace binding: the second lock
Grants control **who** can reach a catalog. **Workspace-catalog binding** controls **which workspaces** can. Binding the production catalog to the production workspace means a developer with an accidental grant still can't read it from dev.

### Separate catalogs or separate workspaces?
Both. Catalogs isolate the **data**, workspaces isolate **compute and users**, and binding connects the two.

---

## 10. Service principals (concept)

A **service principal** is a non-human identity for automation: scheduled jobs, CI/CD and applications.

- **Why not personal accounts:** pipelines break when a person leaves or a token changes, personal accounts tend to be over-privileged, and audit logs can't separate people from automation.
- **Authentication:** OAuth machine-to-machine. The client ID and secret are exchanged for a **short-lived token**. Keep the secret in a secret store, never in code.
- **Authorization:** the same grants as users, with **least privilege** on specific schemas.
- Jobs can be configured to **run as** the service principal.

---

## 11. Debugging "permission denied"

When someone has `SELECT` but still can't read a table, check in this order:

1. **`USE CATALOG` and `USE SCHEMA`** on the parents. The most common miss.
2. **`SELECT` on the object**, direct or inherited: `SHOW GRANTS ON TABLE ...`.
3. **Group membership:** is the user in the group that holds the grant? Allow for the delay.
4. **Workspace binding:** is the catalog bound to the workspace they're using?
5. **Row filters or column masks** that make data look missing or altered.
6. **Compute:** do they have "Can use" on the warehouse or cluster, and is the compute type supported for Unity Catalog?
7. **Underlying storage:** for external tables and volumes, privileges on the external location (`READ FILES`, `CREATE EXTERNAL TABLE`) also matter.

Error messages usually point at the missing piece. Add privileges one at a time and re-run the query to see which error changes.

---

## 12. Gotchas and security caveats

- **Schema-level `SELECT` covers every table in the schema**, including ones added later. If a schema mixes protected and unprotected tables, grant per table.
- **Catalog-level `SELECT` is broad.** Use it only for low-risk environments.
- **Derived tables don't inherit** filters or masks. Protect the copies or restrict who can read them.
- **Unity Catalog permissions only apply to access through Databricks.** Anyone reading the storage directly is governed by Azure RBAC and ACLs, so keep storage-level access limited to the access connector and a few admins.
- **Lineage and audit don't capture work done outside Databricks**, such as direct file access.
- **`SHOW GRANTS` doesn't list ownership.** Check owners separately.
- **Group names in functions must match exactly.**
- **Membership changes lag.** Test in a fresh session after waiting.
- **Failed multi-privilege grants:** if a long `GRANT` errors, split it into one privilege per statement to find the offending one.
- **System tables** (audit, lineage, billing) must be enabled by an account admin, and no user has access by default. They are the way to audit who accessed what.

---

## 13. SQL cheat sheet

```sql
-- Who am I, and what groups am I in?
SELECT current_user();
SELECT is_account_group_member('analysts'), is_member('analysts');

-- Grant and revoke
GRANT USE CATALOG ON CATALOG my_catalog TO `analysts`;
GRANT USE SCHEMA, SELECT ON SCHEMA my_catalog.gold TO `analysts`;
REVOKE SELECT ON SCHEMA my_catalog.gold FROM `analysts`;

-- Inspect grants
SHOW GRANTS ON CATALOG my_catalog;
SHOW GRANTS ON SCHEMA my_catalog.gold;
SHOW GRANTS ON TABLE my_catalog.gold.sales;
SHOW GRANTS `analysts` ON SCHEMA my_catalog.gold;

-- Ownership
DESCRIBE CATALOG EXTENDED my_catalog;
DESCRIBE SCHEMA EXTENDED my_catalog.gold;
DESCRIBE EXTENDED my_catalog.gold.sales;
ALTER SCHEMA my_catalog.gold OWNER TO `data_admins`;

-- Discoverability
REVOKE BROWSE ON CATALOG my_catalog FROM `account users`;

-- Row filter and column mask
ALTER TABLE my_catalog.gold.sales SET ROW FILTER my_catalog.gold.region_filter ON (region);
ALTER TABLE my_catalog.gold.sales DROP ROW FILTER;
ALTER TABLE my_catalog.gold.customers ALTER COLUMN phone SET MASK my_catalog.gold.mask_phone;
ALTER TABLE my_catalog.gold.customers ALTER COLUMN phone DROP MASK;
```

---

## 14. Common questions, with short answers

**How do GRANT, REVOKE and inheritance work?**
Privileges are granted to a principal on a securable and removed with `REVOKE`. Everything is denied by default and the owner has full control. Grants inherit downward, and to read a table you also need `USE CATALOG` and `USE SCHEMA`. I grant to groups, not individuals.

**A user has SELECT but gets "permission denied". Why?**
The usual cause is a missing `USE CATALOG` or `USE SCHEMA`. Then I check inherited grants, group membership, workspace binding, row filters or masks, compute permissions, and external location privileges.

**What is the difference between the admin roles?**
Account admins manage the account (metastores, workspaces, groups). Metastore admins manage privileges and ownership across the metastore. Workspace admins manage one workspace. None of them automatically grants access to table data.

**How do you restrict rows or hide sensitive columns?**
A row filter function on the table decides which rows each group sees, and a column mask function returns real or masked values by group. Both check the caller's group, exempt admins and pipelines, and fail closed. At scale, tag-based ABAC policies avoid attaching functions table by table.

**Why not just hide the production catalog from developers?**
Hiding isn't securing. Permissions decide access, so developers get no grants on production, only the pipeline identity writes there, and the production catalog is bound to the production workspace.

**Do filters and masks protect copies of the data?**
No. Derived tables don't inherit them, and reading storage files directly bypasses Unity Catalog entirely.

**Why use service principals for automation?**
They keep pipelines independent of individual employees, use short-lived OAuth tokens, get least-privilege grants, and make the audit trail clear.

---

## 15. Practice labs to repeat

1. Create a second user, confirm they see nothing, then grant `USE CATALOG`, `USE SCHEMA` and `SELECT` one at a time and note each error message.
2. Move the same grants to a group, remove and re-add the user, and watch access change without touching any `GRANT`.
3. Revoke a schema-level `SELECT` and re-grant it at catalog level, watching how a table's grants change.
4. Add a row filter and a column mask to a table, and test as admin, as a group member and as a user in no group.
5. Create dev and prod catalogs with their own storage, and build the access matrix.
6. Change a catalog's owner to a group and verify with `DESCRIBE`.
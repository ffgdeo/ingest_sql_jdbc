# ingest_sql_jdbc

Databricks notebooks for ingesting data from SQL Server into Unity Catalog using JDBC.

---

## Notebooks

| Notebook | Purpose |
|---|---|
| `create_secret_scope.ipynb` | Creates a Databricks secret scope and stores SQL Server credentials |
| `sqlserver_jdbc_ingest.ipynb` | Reads from SQL Server via JDBC and writes to a Unity Catalog bronze table |

Run them in the order listed above — secrets must exist before the ingestion notebook can use them.

---

## Step 1 — Set up secrets (`create_secret_scope.ipynb`)

Open the notebook and fill in the variables in **Cell 1**:

```python
scope_name    = "sql-server-scope"       # choose a name for your secret scope
db_user_key   = "sql-server-user"        # key name for the username secret
db_pass_key   = "sql-server-password"    # key name for the password secret

db_user_value = "your_database_user"     # ← replace with actual username
db_pass_value = "your_database_password" # ← replace with actual password
```

Run all cells. The notebook will:
1. Connect to your Databricks workspace automatically
2. Create the secret scope (or skip if it already exists)
3. Store the username and password as secrets

### Wipe credentials after running

Once the notebook has run successfully, **immediately remove the actual credential values** from Cell 1 to avoid exposing them in the notebook file:

1. Go back to **Cell 1** and replace the actual values with placeholders:
    ```python
    db_user_value = "your_database_user"
    db_pass_value = "your_database_password"
    ```
2. Select **Edit → Clear All Outputs** to remove any output that may contain sensitive data
3. Save the notebook (**Ctrl + S** / **Cmd + S**)

> The secrets are now safely stored in the Databricks secret store. The notebook no longer needs to hold the real values.

---

## Step 2 — Ingest data (`sqlserver_jdbc_ingest.ipynb`)

Open the notebook and fill in the variables in **Cell 1**:

```python
# JDBC connection
jdbc_hostname = "your-sql-server.database.windows.net"  # SQL Server hostname
jdbc_port     = 1433                                     # default SQL Server port
jdbc_database = "your_database_name"                     # source database name

# Must match the scope and key names used in create_secret_scope.ipynb
jdbc_user     = dbutils.secrets.get(scope="sql-server-scope", key="sql-server-user")
jdbc_password = dbutils.secrets.get(scope="sql-server-scope", key="sql-server-password")

# Unity Catalog target
uc_catalog = "your_catalog"   # Unity Catalog catalog name
uc_schema  = "bronze"         # target schema

# Query to run against the source database
sql_query = "select * from schema_name.table_name"
```

Run all cells. The notebook will:
1. Parse the target table name from the `FROM` clause of your query
2. Connect to SQL Server via JDBC
3. Read the query results into a Spark DataFrame
4. Write the data to `{uc_catalog}.{uc_schema}.{table_name}` using `CREATE OR REPLACE TABLE` semantics

The load is **idempotent** — rerunning the notebook will fully replace the target table's data and schema.

---

## Notes

- **DBR version:** Requires Databricks Runtime 13.0+ (for `databricks-sdk`) and 7.0+ (for Spark 3.0 `writeTo` API)
- **JDBC parallelism:** For large tables, add `numPartitions`, `partitionColumn`, `lowerBound`, and `upperBound` to the `.jdbc()` call to parallelize reads
- **Table name parsing:** The target table name is derived from the last word in the `FROM` clause. If your query has a complex structure, set `target_table_name` manually in Cell 2

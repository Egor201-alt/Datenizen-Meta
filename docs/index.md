# Datenizen Documentation

Datenizen is a high-performance database management addon for Denizen, designed to provide a robust bridge between script-based logic and external database engines (SQLite, MySQL, MariaDB, PostgreSQL). It leverages HikariCP for efficient connection pooling, ensures thread safety by executing heavy database operations asynchronously, and provides a comprehensive suite of events and tags for seamless data manipulation.

---

## Introduction

### Key Features
*   **Async-First Architecture:** All read/write operations execute asynchronously by default to prevent server-side lag.
*   **Connection Pooling:** Built-in integration with HikariCP for resource optimization.
*   **Event-Driven:** Reactive programming support via specialized database events (e.g., `db executed`, `db error`).
*   **Safe Data Handling:** Automatic support for `PreparedStatement` to mitigate SQL injection risks.
*   **Abstraction Layer:** Simplified commands for CRUD operations (`insert`, `update`, `upsert`, `delete`) and batch processing.
*   **Transaction Management:** Full support for ACID-compliant multi-statement transactions.
*   **Error Intelligence:** The `db error` event exposes SQL state codes and human-readable categories for precise error handling.

### Basic Connection Setup
Database connections are established using the `db_connect` command. Connections are persisted under a unique `id` for subsequent use. The `driver` argument accepts short aliases - no need to type full class names.

| Alias | Database |
|---|---|
| `sqlite` | SQLite |
| `mysql` | MySQL |
| `mariadb` | MariaDB |
| `postgresql` or `postgres` | PostgreSQL |

The `url` argument also supports a short form - no `jdbc:` prefix required.

```yaml
# Connect to SQLite using a short path
- db_connect id:local driver:sqlite url:plugins/Datenizen/local.db

# Connect to MySQL with short url
- db_connect id:main driver:mysql url:localhost:3306/mydb user:root pass:secret

# Connect to PostgreSQL
- db_connect id:pg driver:postgres url:localhost:5432/mydb user:admin pass:secret

# Full JDBC URLs still work too
- db_connect id:legacy driver:sqlite url:jdbc:sqlite:plugins/Datenizen/local.db
```

Fires `on db connected` on success, or `on db error` on failure.

### Reconnecting
If a connection pool becomes stale (e.g. the remote database server restarted), reconnect without repeating credentials:

```yaml
- if !<db_ping[main]>:
  - db_reconnect id:main
```

`db_reconnect` reuses the exact HikariCP config from the original `db_connect` call. Fires `on db connected` on success.

### Basic Data Manipulation

```yaml
# Insert a row without writing SQL
- db_table_insert id:main table:players columns:<list[name|uuid|score]> values:<list[Steve|abc-123|0]> label:new_player

# Update with conditions
- db_table_update id:main table:players set:<list[score=100]> where:<list[uuid=abc-123]> label:update_score

# Insert or update in one command (upsert)
- db_upsert id:main table:players key_column:uuid key_value:<player.uuid> set:<list[name=<player.name>|coins=0]> label:save_player

# React to completion
on db executed label:new_player:
  - narrate "Player inserted. Rows affected: <context.affected_rows>"
```

### Retrieving Data

```yaml
# Single value
- define score <db_value[main].sql[SELECT score FROM players WHERE uuid = ?].args[<list[abc-123]>]>

# First row as a MapTag (cleaner than db_query when you only need one row)
- define player <db_query_first[main].sql[SELECT * FROM players WHERE uuid = ?].args[<player.uuid>]>
- narrate "Coins: <[player].get[coins]>"

# All rows as a ListTag of MapTags
- define players <db_query[main].sql[SELECT * FROM players]>

# Result as JSON string
- define json_data <db_query_as_json[main].sql[SELECT * FROM players]>
```

### Connection Management

```yaml
# Disconnect and release pool resources
- db_disconnect id:main

# Evict idle connections from a specific pool
- db_clean_pool id:main

# Modify pool size dynamically (1–100)
- db_set_pool_size id:main size:20

# Set connection timeout (minimum 250ms)
- db_timeout id:main ms:5000
```

---

## Transaction Management

Transactions allow multiple SQL statements to be grouped into a single atomic operation, ensuring data integrity. If any statement within a transaction fails, the entire sequence can be rolled back.

### Syntax
```yaml
- db_transaction id:<id> action:<start|commit|rollback> tx:<tx_id>
```

### Usage

```yaml
- db_transaction id:main action:start tx:my_tx

- db_execute id:main "sql:UPDATE players SET score = score + 10 WHERE id = 1" tx:my_tx
- db_execute id:main "sql:INSERT INTO logs (action) VALUES ('score_update')" tx:my_tx

- db_transaction id:main action:commit tx:my_tx
```

### Rollback on Failure
Use the `category` switch on `db error` to target specific failure types:

```yaml
on db error id:main category:constraint:
  - db_transaction id:main action:rollback tx:my_tx
  - narrate "Constraint violation, transaction rolled back."

on db error id:main:
  - narrate "Error (<context.sql_state>): <context.error>"
  - narrate "Query: <context.query>"
```

### Transaction Expiration
If a transaction remains open for more than 5 minutes, it is automatically rolled back.

```yaml
on db transaction expired:
  - debug log "Transaction <context.tx> on <context.id> timed out and was rolled back."
```

### Connection Leak Detection
Fires alongside `db transaction expired` when a transaction exceeds the 5-minute threshold.

```yaml
on db connection leaked:
  - narrate "Leak detected: tx <context.tx> was open for <context.duration> seconds."
```

---

## Upsert (Insert or Update)

`db_upsert` is the most common data-persistence pattern in game servers - save a player record regardless of whether it already exists. The correct SQL is built automatically for each database type:

| Database | Strategy |
|---|---|
| SQLite | `INSERT OR REPLACE` |
| MySQL / MariaDB | `INSERT ... ON DUPLICATE KEY UPDATE` |
| PostgreSQL | `INSERT ... ON CONFLICT DO UPDATE SET col=EXCLUDED.col` |

The `key_column` must have a `UNIQUE` or `PRIMARY KEY` constraint.

```yaml
- db_upsert id:main table:players key_column:uuid key_value:<player.uuid> set:<list[name=<player.name>|coins=100|rank=gold]> label:save_player
```

```yaml
on db executed label:save_player:
  - narrate "Player data saved. Rows affected: <context.affected_rows>"
```

---

## Data Export and Import

### Exporting Data

```yaml
- db_export_csv id:<id> sql:<query> path:<path> (args:<list>)
```

```yaml
# Export all players
- db_export_csv id:main sql:"SELECT * FROM players" path:plugins/Datenizen/exports/players.csv

# Export with filter
- db_export_csv id:main sql:"SELECT * FROM players WHERE rank = ?" path:plugins/Datenizen/exports/gold.csv args:<list[gold]>
```

Values containing commas, quotes, or newlines are automatically escaped per RFC 4180.

### Importing Data

```yaml
- db_import_csv id:<id> table:<table> path:<path>
```

```yaml
- db_import_csv id:main table:players path:plugins/Datenizen/imports/backup.csv
```

Supports quoted fields containing commas and escaped quotes.

### Events

```yaml
on db csv exported:
  - narrate "Export saved to <context.path>"

on db csv imported:
  - narrate "Imported <context.rows> rows into <context.table>"
```

---

## Database Schema

### Creating Tables

```yaml
- db_table_create id:main table:players columns:<list[id INTEGER PRIMARY KEY AUTOINCREMENT|name TEXT NOT NULL|uuid TEXT UNIQUE|coins INTEGER DEFAULT 0]>

# Safe to call on startup - uses IF NOT EXISTS by default
# Pass if_not_exists:false to require the table to not exist
- db_table_create id:main table:logs columns:<list[id INTEGER PRIMARY KEY|message TEXT|ts INTEGER]> if_not_exists:false
```

Fires `on db executed` with `label:db_table_create` on success.

### Dropping Tables

```yaml
- db_drop_table id:main table:old_data
```

Table name must be alphanumeric/underscores only.

### Table Inspection Tags

```yaml
# Check if a table exists (two syntaxes)
- if <db_exists_table[main].table[players]>:
- if <db_table_exists[main|players]>:

# List all tables in the database
- define all_tables <db_tables[main]>

# Get column names for a table
- define cols <db_columns[main].table[players]>

# Total row count
- define total <db_count[main].table[players]>
```

---

## Advanced Query Execution

### Synchronous Execution
Executes SQL on the main server thread. Use only for time-critical events like `on shutdown`.

```yaml
on shutdown:
  - db_execute_sync id:main "sql:UPDATE players SET last_online = ? WHERE uuid = ?" args:<list[<util.time_now>|<player.uuid>]>
```

### Batch Processing
Executes one parameterized query many times efficiently using JDBC batching inside a transaction. The entire batch is rolled back on failure.

```yaml
- define batch_data <list[<list[100|uuid1]>|<list[200|uuid2]>|<list[50|uuid3]>]>
- db_execute_batch id:main "sql:UPDATE players SET score = ? WHERE uuid = ?" args:<[batch_data]> label:bulk_update
```

### SQL Script Execution
Reads a `.sql` file, splits by semicolons, and executes all statements inside a transaction with rollback on failure.

```yaml
- db_execute_script id:main path:plugins/Datenizen/sql/init_schema.sql
```

### Asynchronous Sequential Execution
Executes a list of SQL statements sequentially in one async task, inside a transaction with rollback on failure.

```yaml
- define queries <list["DELETE FROM logs WHERE age > 30"|"UPDATE stats SET count = 0"]>
- db_execute_async_list id:main sql:<[queries]>
```

---

## Maintenance

### Database Optimization
Runs `VACUUM` on SQLite or `ANALYZE` on MySQL/PostgreSQL.

```yaml
- db_analyze id:main
```

### Backup (SQLite Only)
Copies the database file asynchronously to a new path.

```yaml
- db_backup id:local path:plugins/Datenizen/backups/local_backup.db
```

---

## Error Handling

The `db error` event fires whenever any SQL exception or validation error occurs. It now includes a SQL state code and a human-readable category to enable precise error handling.

### Contexts

| Context | Description |
|---|---|
| `<context.id>` | The database ID that produced the error |
| `<context.error>` | Full error message from the JDBC driver |
| `<context.query>` | The SQL query that caused the error, or the command name for validation errors |
| `<context.sql_state>` | 5-character SQL state code (e.g. `23000`). Empty for non-SQL exceptions |
| `<context.category>` | Human-readable category derived from the SQL state |

### Error Categories

| Category | SQL State Prefix | Meaning |
|---|---|---|
| `constraint` | `23xxx` | Unique/primary key violation, foreign key error |
| `syntax` | `42xxx` | SQL syntax error or unknown object |
| `connection` | `08xxx` or no state | Network error, pool timeout, driver failure |
| `data` | `22xxx` | Data type mismatch, truncation, out of range |
| `permission` | `28xxx`, `42501` | Access denied, insufficient privileges |
| `timeout` | `HYT00`, `HYT01`, `40001` | Lock wait timeout, query timeout, deadlock |
| `unknown` | anything else | Uncategorized SQL exception |

### Switches

```yaml
# Filter by database id
on db error id:main:

# Filter by category
on db error category:constraint:

# Combine both
on db error id:main category:connection:
```

### Usage Examples

```yaml
# Log everything
on db error:
  - announce to_console "[<context.id>] <context.category> (<context.sql_state>): <context.error>"

# Handle constraint violations (e.g. duplicate player uuid)
on db error id:main category:constraint:
  - narrate "That record already exists."

# Auto-reconnect on connection loss
on db error category:connection:
  - db_reconnect id:<context.id>

# Rollback active transaction on any error
on db error id:main:
  - db_transaction id:main action:rollback tx:active_tx
```

---

## Connection Status Tags

```yaml
# Check if a connection pool is registered and open
- if <db_connected[main]>:
  - narrate "Pool is open."

# Actively test the connection (more reliable - calls isValid(1) on a real connection)
- if !<db_ping[main]>:
  - db_reconnect id:main

# List all currently active database IDs
- narrate "Active connections: <db_list>"

# Get pool statistics
- define stats <db_info[main]>
- narrate "Active: <[stats].get[active_connections]> / Total: <[stats].get[total_connections]>"
```

---

## Full Tag Reference

| Tag | Returns | Description |
|---|---|---|
| `<db_query[id].sql[query].args[list]>` | `ListTag` | All rows as a list of `MapTag` |
| `<db_query_first[id].sql[query].args[list]>` | `MapTag` | First row as a `MapTag`, or null |
| `<db_value[id].sql[query].args[list]>` | `ElementTag` | First column of first row |
| `<db_exists[id].sql[query].args[list]>` | `ElementTag(Boolean)` | True if query returns at least one row |
| `<db_query_as_json[id].sql[query].args[list]>` | `ElementTag` | Result as JSON array string |
| `<db_convert_map[id].sql[query].args[list]>` | `MapTag` | First column → key, second column → value |
| `<db_last_id[tx_id]>` | `ElementTag` | Last inserted row ID (requires transaction ID) |
| `<db_exists_table[id].table[name]>` | `ElementTag(Boolean)` | True if table exists |
| `<db_table_exists[id\|name]>` | `ElementTag(Boolean)` | Same as above, shorter syntax |
| `<db_tables[id]>` | `ListTag` | All table names in the database |
| `<db_columns[id].table[name]>` | `ListTag` | All column names for a table |
| `<db_count[id].table[name]>` | `ElementTag` | Total row count for a table |
| `<db_connected[id]>` | `ElementTag(Boolean)` | True if pool is registered and open |
| `<db_ping[id]>` | `ElementTag(Boolean)` | True if connection is actively alive |
| `<db_list>` | `ListTag` | All active database connection IDs |
| `<db_info[id]>` | `MapTag` | Pool stats: active, idle, total, threads_awaiting |

---

## Full Command Reference

| Command | Description |
|---|---|
| `db_connect` | Connect to a database and register a connection pool |
| `db_reconnect` | Reconnect using saved credentials from the last `db_connect` |
| `db_disconnect` | Disconnect and release all pool resources |
| `db_execute` | Run an async SQL write query (INSERT/UPDATE/DELETE) |
| `db_execute_sync` | Run a synchronous SQL write query on main thread |
| `db_execute_async_list` | Run multiple SQL statements sequentially in one async task |
| `db_execute_batch` | Run one parameterized query many times via JDBC batching |
| `db_execute_script` | Execute a `.sql` file inside a transaction |
| `db_transaction` | Start, commit, or rollback a transaction |
| `db_table_create` | Create a table from a column definition list |
| `db_table_insert` | Insert a row without writing SQL |
| `db_table_update` | Update rows without writing SQL |
| `db_table_delete` | Delete rows without writing SQL |
| `db_upsert` | Insert or update a row in one command |
| `db_drop_table` | Drop a table |
| `db_backup` | Copy an SQLite database file asynchronously |
| `db_import_csv` | Import a CSV file into a table |
| `db_export_csv` | Export a query result to a CSV file |
| `db_analyze` | Run VACUUM (SQLite) or ANALYZE (MySQL/PostgreSQL) |
| `db_clean_pool` | Evict idle connections from a pool |
| `db_set_pool_size` | Change maximum pool size (1–100) |
| `db_timeout` | Change connection timeout (minimum 250ms) |

---

## Full Event Reference

| Event | Contexts | Description |
|---|---|---|
| `on db connected` | `id` | Connection pool successfully initialized. Switch: `id` |
| `on db disconnected` | `id` | Connection pool closed |
| `on db executed` | `id`, `label`, `affected_rows` | Async write operation completed. Switches: `id`, `label` |
| `on db error` | `id`, `error`, `query`, `sql_state`, `category` | SQL or validation error occurred. Switches: `id`, `category` |
| `on db transaction expired` | `tx`, `id` | Transaction auto-rolled back after 5 minutes |
| `on db connection leaked` | `tx`, `duration` | Transaction exceeded the 5-minute threshold |
| `on db csv exported` | `id`, `path` | CSV export completed |
| `on db csv imported` | `id`, `table`, `rows` | CSV import completed |

---

## Retrieving Last Inserted ID

To guarantee thread safety, `db_last_id` **must** be called with a transaction ID - this ensures it reads from the same connection the insert was made on.

```yaml
- db_transaction id:main action:start tx:insert_tx
- db_execute id:main "sql:INSERT INTO logs (action) VALUES ('test')" tx:insert_tx
- define new_id <db_last_id[insert_tx]>
- db_transaction id:main action:commit tx:insert_tx
- narrate "New log entry ID: <[new_id]>"
```

---

## Converting Results to Maps

```yaml
# Map of setting_name -> setting_value
- define settings <db_convert_map[main].sql[SELECT setting_name, setting_value FROM config]>
- narrate "Language: <[settings].get[language]>"
```

---

## Parameterized Queries

All commands that accept SQL support `?` placeholders via the `args` list. This is the recommended approach - never interpolate player input directly into SQL strings.

```yaml
# With args list
- db_execute id:main sql:"INSERT INTO players (name, uuid) VALUES (?, ?)" args:<list[<player.name>|<player.uuid>]>

# The sql argument also accepts quoted form for complex queries
- db_execute id:main "sql:UPDATE players SET coins = coins - ? WHERE uuid = ?" args:<list[<[amount]>|<player.uuid>]>
```

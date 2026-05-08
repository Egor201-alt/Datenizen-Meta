# Datenizen Documentation

Datenizen is a high-performance database management addon for Denizen, designed to provide a robust bridge between script-based logic and external database engines (SQLite, MySQL, PostgreSQL). It leverages HikariCP for efficient connection pooling, ensures thread safety by executing heavy database operations asynchronously, and provides a comprehensive suite of events and tags for seamless data manipulation.

---

## Introduction

### Key Features
*   **Async-First Architecture:** All read/write operations execute asynchronously by default to prevent server-side lag.
*   **Connection Pooling:** Built-in integration with HikariCP for resource optimization.
*   **Event-Driven:** Reactive programming support via specialized database events (e.g., `db executed`, `db error`).
*   **Safe Data Handling:** Automatic support for `PreparedStatement` to mitigate SQL injection risks.
*   **Abstraction Layer:** Simplified commands for CRUD operations (`insert`, `update`, `delete`) and batch processing.
*   **Transaction Management:** Full support for ACID-compliant multi-statement transactions.

### Basic Connection Setup
Database connections are established using the `db_connect` command. Connections are persisted under a unique `id` for subsequent use in queries and tags.

```yaml
# Connect to SQLite
- db_connect id:local driver:org.sqlite.JDBC url:jdbc:sqlite:plugins/Datenizen/local.db

# Connect to MySQL
- db_connect id:main driver:com.mysql.cj.jdbc.Driver url:jdbc:mysql://localhost:3306/db user:root pass:123
```

### Basic Data Manipulation
Data insertion and modification are performed via specialized commands that handle parameterization automatically.

```yaml
# Simple Insert
- db_table_insert id:main table:players columns:name|uuid|score values:<list[Steve|abc-123|0]> label:new_player

# Update with conditions
- db_table_update id:main table:players set:<list[score=100]> where:<list[uuid=abc-123]> label:update_score

# Reacting to operation completion
on db executed label:new_player:
  - narrate "Player successfully inserted. Rows affected: <context.affected_rows>"
```

### Retrieving Data
Datenizen provides specialized tags to fetch data directly into Denizen objects.

```yaml
# Fetching a single value
- define score <db_value[main].sql[SELECT score FROM players WHERE uuid = 'abc-123']>

# Fetching rows as a List of MapTags
- define players <db_query[main].sql[SELECT * FROM players]>

# Converting result sets to JSON strings
- define json_data <db_query_as_json[main].sql[SELECT * FROM players]>
```

### Connection Management

```yaml
# Disconnect and release pool resources
- db_disconnect id:main

# Evict idle connections from a specific pool
- db_clean_pool id:main

# Modify pool size dynamically
- db_set_pool_size id:main size:20
```

---

## Transaction Management

Transactions allow multiple SQL statements to be grouped into a single atomic operation, ensuring data integrity. If any statement within a transaction fails, the entire sequence can be rolled back to its initial state.

### Syntax
```yaml
- db_transaction id:<id> action:<start|commit|rollback> tx:<tx_id>
```

### Usage
Transactions require a `tx_id` (a unique string defined by the script) to associate subsequent `db_execute` commands with the specific transaction connection.

```yaml
# Start a transaction
- db_transaction id:main action:start tx:my_tx

# Execute multiple statements within the transaction
- db_execute id:main sql:"UPDATE players SET score = score + 10 WHERE id = 1" tx:my_tx
- db_execute id:main sql:"INSERT INTO logs (action) VALUES ('score_update')" tx:my_tx

# Commit changes
- db_transaction id:main action:commit tx:my_tx
```

### Rollback on Failure
Using the `db_error` event, you can trigger a rollback if any part of the transaction fails.

```yaml
transaction_handler:
  type: world
  events:
    on db error:
      - if <context.query> contains "tx:my_tx":
        - db_transaction id:main action:rollback tx:my_tx
        - narrate "Transaction failed: <context.error>. Rollback executed."
```

### Transaction Expiration
Datenizen automatically manages transaction lifecycles. If a transaction remains open for more than 5 minutes (300,000ms), it is automatically rolled back to prevent connection leaks.

```yaml
on db transaction expired:
  - debug log "Transaction <context.tx> on database <context.id> timed out and was rolled back."
```

### Connection Leak Detection
If a transaction remains uncommitted for longer than the defined threshold, a `db connection leaked` event is fired.

```yaml
on db connection leaked:
  - narrate "Connection leak detected in transaction <context.tx>. Duration: <context.duration> seconds."
```

---

## Data Export and Import

Datenizen provides native support for CSV file operations, allowing for efficient bulk data migration and logging without writing complex SQL scripts.

### Exporting Data
The `db_export_csv` command runs a `SELECT` query and writes the resulting rows to a CSV file.

```yaml
- db_export_csv id:<id> sql:<query> path:<path> (args:<list>)
```

```yaml
# Export all players to a CSV file
- db_export_csv id:main sql:"SELECT * FROM players" path:plugins/Datenizen/exports/players.csv

# Export with parameterized filtering
- db_export_csv id:main sql:"SELECT * FROM players WHERE rank = ?" path:plugins/Datenizen/exports/gold_players.csv args:<list[gold]>
```

### Importing Data
The `db_import_csv` command parses a CSV file and performs batch insertions into a specified table.

```yaml
- db_import_csv id:<id> table:<table> path:<path>
```

```yaml
# Import CSV data into the 'players' table
- db_import_csv id:main table:players path:plugins/Datenizen/imports/players_backup.csv
```

### Event Handling

```yaml
on db csv exported:
  - narrate "Data exported to <context.path>"

on db csv imported:
  - narrate "Imported <context.rows> rows into <context.table>"
```

---

## Advanced Query Execution

### Synchronous Execution
Executes SQL queries on the main server thread. Use only for time-critical events like `on shutdown`.

```yaml
on shutdown:
  - db_execute_sync id:main sql:"UPDATE players SET last_online = ? WHERE uuid = ?" args:<list[<util.time_now>|<player.uuid>]>
```

### Batch Processing
Utilizes JDBC batching for high-performance inserts or updates.

```yaml
# Update multiple player scores in a single batch
- define batch_data <list[<list[100|uuid1]>|<list[200|uuid2]>|<list[50|uuid3]>]>
- db_execute_batch id:main sql:"UPDATE players SET score = ? WHERE uuid = ?" args:<[batch_data]>
```

### SQL Script Execution
Reads a `.sql` file, splits statements by semicolons, and executes them within a single transaction.

```yaml
# Run an initialization script
- db_execute_script id:main path:plugins/Datenizen/sql/init_schema.sql
```

### Asynchronous Sequential Execution
Processes multiple distinct SQL queries sequentially within a single asynchronous task.

```yaml
# Execute multiple independent queries sequentially
- define queries <list[DELETE FROM logs WHERE age > 30|UPDATE stats SET count = 0]>
- db_execute_async_list id:main sql:<[queries]>
```

---

## Database Schema and Maintenance

### Table Inspection
*   **`<db_exists_table[<id>].table[<name>]>`**: Returns `true` if the table exists.
*   **`<db_columns[<id>].table[<name>]>`**: Returns a `ListTag` of all column names.
*   **`<db_count[<id>].table[<name>]>`**: Returns the current row count of the table.

```yaml
# Verify table existence before processing
- if <db_exists_table[main].table[players]>:
  - define cols <db_columns[main].table[players]>
  - narrate "Table exists with columns: <[cols].formatted>"
```

### Database Optimization
```yaml
- db_analyze id:main
```

### Pool Monitoring
Returns a `MapTag` containing: `active_connections`, `idle_connections`, `total_connections`, `threads_awaiting`.

```yaml
- define stats <db_info[main]>
- narrate "Active connections: <[stats].get[active_connections]>"
```

---

## Error Handling and Events

### Database Error Handling
The `db error` event is triggered whenever an SQL exception occurs.

```yaml
on db error:
  - announce to_console "Database error on <context.id>: <context.error>"
  - if <context.query> contains "INSERT":
    - flag server db_log_failed:true
```

### Connection Events
*   **`on db connected`**: Triggered when a new connection pool is successfully initialized.
*   **`on db disconnected`**: Triggered when a connection pool is closed.

### Operation Completion
Triggered upon the successful completion of asynchronous operations.

```yaml
on db executed label:player_creation:
  - narrate "New player recorded. Rows affected: <context.affected_rows>"
```

---

## Miscellaneous Commands and Tags

### Database Backup (SQLite Only)
```yaml
- db_backup id:local path:plugins/Datenizen/backups/local_backup.db
```

### Table Management
```yaml
- db_drop_table id:main table:old_players_data
```

### Simplified Row Deletion
```yaml
- db_table_delete id:main table:players where:<list[rank=banned]> label:purge_banned
```

### Advanced Retrieval Tags

#### Checking Row Existence
```yaml
- if <db_exists[main].sql[SELECT 1 FROM players WHERE uuid = ?].args[<list[<player.uuid>]>]>:
  - narrate "Player exists in the database."
```

#### Converting Results to Maps
```yaml
- define settings <db_convert_map[main].sql[SELECT setting_name, setting_value FROM config]>
- narrate "Server language is: <[settings].get[language]>"
```

#### Retrieving Last Inserted ID
To guarantee thread safety, this tag **must** be provided with a transaction ID (`<tx_id>`).

```yaml
- db_transaction id:main action:start tx:insert_tx
- db_execute id:main "sql:INSERT INTO logs (action) VALUES ('test')" tx:insert_tx
- define new_id <db_last_id[insert_tx]>
- db_transaction id:main action:commit tx:insert_tx
- narrate "The new log entry ID is <[new_id]>"
```
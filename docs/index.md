# Datenizen Documentation

Datenizen is a high-performance database addon for [Denizen](https://github.com/DenizenScript/Denizen), bridging script logic with SQLite, MySQL, MariaDB, and PostgreSQL via HikariCP connection pooling. All heavy I/O runs asynchronously; results return through events and tags.

---

## Introduction

### Key Features

- **Async-first** — all DB I/O runs off the main thread; results fire events back on it
- **HikariCP pooling** — one managed pool per named connection ID
- **Event-driven** — every operation fires a success or error event for reactive scripting
- **PreparedStatements everywhere** — no SQL injection possible through Datenizen's own commands
- **Full CRUD abstraction** — insert, update, upsert, delete, batch upsert without writing SQL
- **Transactions + savepoints** — full ACID-compliant multi-statement operations
- **Named queries** — register SQL once, run by name anywhere in your scripts
- **Schema migrations** — version-controlled SQL file migration runner
- **Query caching** — TTL-based in-memory result cache with manual invalidation
- **Schema alterations** — add/drop columns, rename tables, create/drop indexes without writing SQL
- **Conditional execution** — run SQL only when a check query matches
- **PostgreSQL LISTEN/NOTIFY** — real-time push notifications between database and scripts
- **Smart error events** — SQL state codes, human-readable categories, per-db and per-category switches

### Supported Databases

| Alias | Database |
|---|---|
| `sqlite` | SQLite |
| `mysql` | MySQL |
| `mariadb` | MariaDB |
| `postgresql` or `postgres` | PostgreSQL |

---

## Getting Started

### Connecting

```yaml
# SQLite (short path, no jdbc: prefix needed)
- db_connect id:local driver:sqlite url:plugins/Datenizen/local.db

# MySQL
- db_connect id:main driver:mysql url:localhost:3306/mydb user:root pass:secret

# PostgreSQL
- db_connect id:pg driver:postgres url:localhost:5432/mydb user:admin pass:secret

# Full JDBC URL also works
- db_connect id:legacy driver:sqlite url:jdbc:sqlite:plugins/Datenizen/local.db
```

Fires `on db connected` on success, `on db error` on failure.

### Connecting from a File

Store credentials in a YAML file so they are not hardcoded in scripts:

```yaml title="plugins/Datenizen/db.yml"
driver: mysql
url: localhost:3306/mydb
user: root
password: secret
```

```yaml
- db_connect_file id:main path:plugins/Datenizen/db.yml
```

Fires `on db connected` on success, `on db error` on failure.

### Checking Status and Reconnecting

```yaml
# Fast in-memory check (pool is registered and open)
- if <db_connected[main]>:
  - narrate "Pool is open."

# Active connection test — calls isValid(1) on a real connection
- if !<db_ping[main]>:
  - db_reconnect id:main

# All currently active IDs
- narrate "Active connections: <db_list>"
```

`db_reconnect` reuses credentials from the original `db_connect` call. Fires `on db connected` on success.

---

## Writing Data

### Insert

```yaml
- db_table_insert id:main table:players columns:<list[name|uuid|coins]> values:<list[Steve|abc-123|0]> label:new_player

on db executed label:new_player:
  - narrate "Inserted <context.affected_rows> row(s)"
```

### Update

```yaml
- db_table_update id:main table:players set:<list[coins=100|rank=gold]> where:<list[uuid=abc-123]> label:update_player
```

### Delete

```yaml
- db_table_delete id:main table:players where:<list[uuid=abc-123]> label:remove_player
```

### Upsert (Insert or Update)

The most common game-server pattern — save a record whether it exists or not. The correct SQL is generated per database type automatically.

| Database | Strategy |
|---|---|
| SQLite | `INSERT OR REPLACE` |
| MySQL / MariaDB | `INSERT ... ON DUPLICATE KEY UPDATE` |
| PostgreSQL | `INSERT ... ON CONFLICT DO UPDATE SET col=EXCLUDED.col` |

The `key_column` must have a `UNIQUE` or `PRIMARY KEY` constraint.

```yaml
- db_upsert id:main table:players key_column:uuid key_value:<player.uuid> set:<list[name=<player.name>|coins=100]> label:save_player

on db executed label:save_player:
  - narrate "Saved. Rows affected: <context.affected_rows>"
```

### Batch Upsert

Upsert many rows in one batched statement — far more efficient than looping `db_upsert`.
Each entry in `rows` is a `MapTag`.

```yaml
- define rows <list[<map[uuid=abc|name=Steve|coins=100]>|<map[uuid=def|name=Alex|coins=200]>]>
- db_upsert_batch id:main table:players key_column:uuid rows:<[rows]> label:bulk_save

on db executed label:bulk_save:
  - narrate "Batch saved. Total rows affected: <context.affected_rows>"
```

---

## Reading Data

### Tags (synchronous)

Tags run on the calling thread. Use them when you need the result immediately in the same script step.

```yaml
# Single scalar value
- define score <db_value[main].sql[SELECT score FROM players WHERE uuid = ?].args[<list[abc-123]>]>

# First row as a MapTag
- define player <db_query_first[main].sql[SELECT * FROM players WHERE uuid = ?].args[<player.uuid>]>
- narrate "Coins: <[player].get[coins]>"

# All rows as a ListTag of MapTags
- foreach <db_query[main].sql[SELECT * FROM players WHERE active = ?].args[<list[1]>]> as:row:
  - narrate "<[row].get[name]>: <[row].get[coins]> coins"

# Result as a JSON string
- define json <db_query_as_json[main].sql[SELECT * FROM players]>

# Paginated query (page 1, 20 rows per page)
- define page <db_query_page[main].sql[SELECT * FROM players ORDER BY coins DESC].page[1].size[20]>

# Map of key -> value from a two-column query
- define settings <db_convert_map[main].sql[SELECT key, value FROM config]>
- narrate "Language: <[settings].get[language]>"

# Check if any row matches
- if <db_exists[main].sql[SELECT 1 FROM players WHERE uuid = ?].args[<player.uuid>]>:
  - narrate "Player exists"
```

### Async Query

Run a SELECT asynchronously and handle results in an event — correct for queries triggered mid-script that don't need to block.

```yaml
- db_query_async id:main sql:"SELECT * FROM players WHERE rank = ?" args:<list[gold]> label:gold_players

on db queried label:gold_players:
  - foreach <context.rows> as:row:
    - narrate "<[row].get[name]> is gold rank"
```

### Cached Query

Avoid hitting the database repeatedly for the same data. Results are cached for the given TTL (in ticks) and invalidated automatically when the database disconnects.

```yaml
# Cache leaderboard for 5 seconds (100 ticks)
- define top <db_cached_query[main].sql[SELECT * FROM players ORDER BY coins DESC LIMIT 10].ttl[100]>
```

---

## Parameterized Queries

All commands that accept SQL support `?` placeholders bound via the `args` list. Never interpolate player input directly into SQL strings.

```yaml
- db_execute id:main sql:"INSERT INTO players (name, uuid) VALUES (?, ?)" args:<list[<player.name>|<player.uuid>]>

- db_execute id:main "sql:UPDATE players SET coins = coins - ? WHERE uuid = ?" args:<list[<[amount]>|<player.uuid>]>
```

---

## Named Queries

Register a SQL query once and call it by name anywhere in your scripts. Reduces duplication and keeps SQL in one place.

```yaml
# Register on startup
- db_register id:main name:get_player sql:"SELECT * FROM players WHERE uuid = ?"
- db_register id:main name:save_player sql:"INSERT INTO players (uuid, name, coins) VALUES (?, ?, ?) ON DUPLICATE KEY UPDATE name=VALUES(name)"

# Call later — SELECT fires 'db queried', DML fires 'db executed'
- db_run id:main name:get_player args:<list[<player.uuid>]> label:got_player
- db_run id:main name:save_player args:<list[<player.uuid>|<player.name>|0]> label:saved_player

on db queried label:got_player:
  - define p <context.rows.first>
  - narrate "Found: <[p].get[name]> with <[p].get[coins]> coins"
```

`db_run` detects the query type automatically: starts with `SELECT` → fires `db queried`; otherwise → fires `db executed`.

---

## Transaction Management

Transactions group multiple statements into one atomic unit. If any statement fails, the entire group rolls back.

### Basic Usage

```yaml
- db_transaction id:main action:start tx:my_tx

- db_execute id:main "sql:UPDATE players SET coins = coins - ? WHERE uuid = ?" args:<list[50|<player.uuid>]> tx:my_tx
- db_execute id:main "sql:INSERT INTO logs (action, player) VALUES (?, ?)" args:<list[purchase|<player.uuid>]> tx:my_tx

- db_transaction id:main action:commit tx:my_tx
```

```yaml
on db error id:main:
  - db_transaction id:main action:rollback tx:my_tx
  - narrate "Transaction rolled back: <context.error>"
```

### Savepoints

Savepoints let you partially roll back within a transaction without discarding the whole thing.

```yaml
- db_transaction id:main action:start tx:my_tx

- db_execute id:main "sql:UPDATE accounts SET balance = balance - 100 WHERE id = 1" tx:my_tx

- db_transaction tx:my_tx action:savepoint savepoint:before_bonus
- db_execute id:main "sql:UPDATE accounts SET balance = balance + 200 WHERE id = 2" tx:my_tx

# Undo only the bonus step if it fails, keep the debit
- db_transaction tx:my_tx action:rollback_to savepoint:before_bonus

- db_transaction id:main action:commit tx:my_tx
```

Available savepoint actions: `savepoint`, `rollback_to`, `release`.

### Transaction Expiry and Leak Detection

Transactions open for more than 5 minutes are automatically rolled back.

```yaml
on db transaction expired:
  - debug log "Transaction <context.tx> on <context.id> timed out."

on db connection leaked:
  - narrate "Leak detected: tx <context.tx> was open for <context.duration> seconds."
```

### Last Inserted ID

Must be called within a transaction to guarantee the same connection is used.

```yaml
- db_transaction id:main action:start tx:insert_tx
- db_execute id:main "sql:INSERT INTO logs (action) VALUES ('test')" tx:insert_tx
- define new_id <db_last_id[insert_tx]>
- db_transaction id:main action:commit tx:insert_tx
- narrate "New ID: <[new_id]>"
```

---

## Conditional Execution

Run a write query only when a check SELECT returns at least one row. If the check matches nothing, fires `db executed` with `affected_rows=0` (skipped — no error).

```yaml
# Only update if player exists
- db_execute_if id:main check:"SELECT 1 FROM players WHERE uuid=?" check_args:<list[<player.uuid>]> sql:"UPDATE players SET online=1 WHERE uuid=?" args:<list[<player.uuid>]> label:mark_online

on db executed label:mark_online:
  - if <context.affected_rows> > 0:
    - narrate "Marked online"
  - else:
    - narrate "Player not in database, skipped"
```

---

## PostgreSQL LISTEN/NOTIFY

Subscribe to a PostgreSQL notification channel. When another connection executes `NOTIFY channel, 'payload'`, Datenizen fires `on db notify` with the channel name and payload. Notifications are polled every second (20 ticks).

```yaml
# Start listening on server start
on db connected id:pg:
  - db_listen id:pg channel:player_events action:start

# Handle incoming notifications
on db notify channel:player_events:
  - narrate "DB notification from <context.id>: <context.payload>"

# Stop listening
- db_listen id:pg channel:player_events action:stop
```

Requires PostgreSQL. Fires `on db error` if the database type is not `postgresql`.

---

## Schema Management

### Creating Tables

```yaml
# Safe to call on startup — uses IF NOT EXISTS by default
- db_table_create id:main table:players columns:<list[id INTEGER PRIMARY KEY AUTOINCREMENT|name TEXT NOT NULL|uuid TEXT UNIQUE|coins INTEGER DEFAULT 0]>

# Require the table to not exist
- db_table_create id:main table:logs columns:<list[id INTEGER PRIMARY KEY|message TEXT|ts INTEGER]> if_not_exists:false
```

Fires `on db executed` with `label:db_table_create`.

### Dropping Tables

```yaml
- db_drop_table id:main table:old_data
```

### Altering Tables

```yaml
# Add a column (full SQL definition)
- db_add_column id:main table:players column:"rank TEXT DEFAULT 'default'"
- db_add_column id:main table:players column:"last_seen INTEGER"

# Drop a column (SQLite 3.35.0+ required)
- db_drop_column id:main table:players column:old_field

# Rename a table
- db_rename_table id:main from:players to:players_old
```

### Indexes

```yaml
# Create a unique index (speeds up lookups by uuid)
- db_create_index id:main table:players name:idx_players_uuid column:uuid unique:true

# Create a non-unique index
- db_create_index id:main table:logs name:idx_logs_ts column:timestamp

# Drop an index
- db_drop_index id:main name:idx_players_uuid
# MySQL/MariaDB require the table name
- db_drop_index id:mysql_db name:idx_players_uuid table:players
```

### Copying Tables

```yaml
# Create a new table from an existing one (copies schema + data)
- db_table_copy id:main from:players to:players_backup

# Only copy data into an already-existing table
- db_table_copy id:main from:players to:players_backup insert_only:true
```

Fires `on db executed` with `label:db_table_copy`.

### Inspecting Schema

```yaml
# Check if a table exists
- if <db_exists_table[main].table[players]>:

# List all tables
- define all_tables <db_tables[main]>

# Get column names for a table
- define cols <db_columns[main].table[players]>

# Total row count
- define total <db_count[main].table[players]>
```

---

## Migrations

Run SQL migration files from a folder in alphabetical order. Already-applied migrations are tracked in a `__datenizen_migrations` table and skipped on subsequent runs. Safe to call on every server start.

```
plugins/Datenizen/migrations/
  001_create_players.sql
  002_add_rank_column.sql
  003_create_logs.sql
```

```yaml
on db connected id:main:
  - db_migrate id:main path:plugins/Datenizen/migrations

on db migrated id:main:
  - narrate "Applied <context.count> migration(s) from <context.path>"

on db error id:main:
  - debug log "Migration failed: <context.error>"
```

If a migration fails, it is rolled back and no further migrations run that session.

---

## Batch and Bulk Operations

### SQL Script File

Reads a `.sql` file, splits statements correctly (handles quoted strings and comments), and runs them all in a single transaction.

```yaml
- db_execute_script id:main path:plugins/Datenizen/sql/init.sql
```

### Sequential Async List

Executes multiple SQL statements in one async task, inside a transaction.

```yaml
- db_execute_async_list id:main sql:<list["DELETE FROM logs WHERE age > 30"|"UPDATE stats SET resets = resets + 1"]> label:cleanup

on db executed label:cleanup:
  - narrate "Cleanup done"
```

### JDBC Batch

One parameterized query executed many times with different args — the most efficient way to bulk-write.

```yaml
- define rows <list[<list[100|uuid1]>|<list[200|uuid2]>|<list[50|uuid3]>]>
- db_execute_batch id:main "sql:UPDATE players SET score = ? WHERE uuid = ?" args:<[rows]> label:bulk_update
```

---

## Data Import and Export

### JSON Export

```yaml
- db_export_json id:main sql:"SELECT * FROM players ORDER BY coins DESC" path:plugins/Datenizen/exports/players.json

on db json exported:
  - narrate "JSON saved to <context.path>"
```

Null database values are written as JSON `null` (not the string `"null"`).

### CSV Export

```yaml
- db_export_csv id:main sql:"SELECT * FROM players WHERE rank = ?" path:plugins/Datenizen/exports/gold.csv args:<list[gold]>

on db csv exported:
  - narrate "Saved to <context.path>"
```

Values containing commas, quotes, or newlines are automatically escaped per RFC 4180.

### CSV Import

```yaml
- db_import_csv id:main table:players path:plugins/Datenizen/imports/backup.csv

on db csv imported:
  - narrate "Imported <context.rows> rows into <context.table>"
```

The first line is treated as the header and skipped. Missing columns in a row are inserted as `NULL`.

---

## Maintenance

### Backup (SQLite only)

```yaml
- db_backup id:local path:plugins/Datenizen/backups/local_<util.date>.db

on db backed up:
  - narrate "Backup saved to <context.path>"
```

### Optimization

```yaml
# VACUUM on SQLite, ANALYZE on others
- db_analyze id:main

# Evict idle connections from pool
- db_clean_pool id:main

# Change max pool size at runtime (1–100)
- db_set_pool_size id:main size:20

# Change connection timeout (minimum 250ms)
- db_timeout id:main ms:5000

# Manually invalidate the query cache for a connection
- db_clear_cache id:main
```

### Pool Statistics

```yaml
- define stats <db_info[main]>
- narrate "Active: <[stats].get[active_connections]> / Total: <[stats].get[total_connections]>"
```

---

## Error Handling

`on db error` fires for any SQL exception or validation error.

### Contexts

| Context | Description |
|---|---|
| `<context.id>` | Database ID that produced the error |
| `<context.error>` | Full error message from the driver |
| `<context.query>` | SQL query or command name for validation errors |
| `<context.sql_state>` | 5-character SQL state code (e.g. `23000`). Empty for non-SQL exceptions |
| `<context.category>` | Human-readable category derived from the SQL state |

### Error Categories

| Category | SQL State | Meaning |
|---|---|---|
| `constraint` | `23xxx` | Unique/PK violation, foreign key error |
| `syntax` | `42xxx` | SQL syntax error or unknown object |
| `connection` | `08xxx` or no state | Network error, pool timeout, driver failure |
| `data` | `22xxx` | Type mismatch, truncation, out of range |
| `permission` | `28xxx`, `42501` | Access denied, insufficient privileges |
| `timeout` | `HYT00`, `HYT01`, `40001` | Lock wait timeout, deadlock |
| `unknown` | anything else | Uncategorized |

### Switches

```yaml
on db error id:main:                     # specific database
on db error category:constraint:         # specific category
on db error id:main category:connection: # both
```

### Examples

```yaml
# Log everything
on db error:
  - announce to_console "[<context.id>] <context.category> (<context.sql_state>): <context.error>"

# Handle duplicate records
on db error id:main category:constraint:
  - narrate "That record already exists."

# Auto-reconnect on connection loss
on db error category:connection:
  - db_reconnect id:<context.id>
```

---

## Full Command Reference

| Command | Description |
|---|---|
| `db_connect` | Connect to a database and register a connection pool |
| `db_connect_file` | Connect using credentials from a YAML file |
| `db_reconnect` | Reconnect using saved credentials |
| `db_disconnect` | Disconnect and release pool resources |
| `db_register` | Register a named SQL query for later use with `db_run` |
| `db_run` | Execute a named query; fires `db queried` or `db executed` |
| `db_execute` | Run an async SQL write query |
| `db_execute_sync` | Run a synchronous SQL write query on the main thread |
| `db_execute_async_list` | Run multiple SQL statements sequentially in one async task |
| `db_execute_batch` | Run one parameterized query many times via JDBC batching |
| `db_execute_script` | Execute a `.sql` file inside a transaction |
| `db_query_async` | Run a SELECT asynchronously; fires `db queried` with results |
| `db_transaction` | Start, commit, rollback, or manage savepoints in a transaction |
| `db_migrate` | Apply pending SQL migration files from a folder |
| `db_table_create` | Create a table from a column definition list |
| `db_table_insert` | Insert a row without writing SQL |
| `db_table_update` | Update rows without writing SQL |
| `db_table_delete` | Delete rows without writing SQL |
| `db_table_copy` | Copy a table's schema and data to a new table |
| `db_upsert` | Insert or update a single row |
| `db_upsert_batch` | Insert or update many rows in one batched operation |
| `db_drop_table` | Drop a table |
| `db_backup` | Copy an SQLite database file asynchronously |
| `db_import_csv` | Import a CSV file into a table |
| `db_export_csv` | Export a query result to a CSV file |
| `db_add_column` | Add a column to a table |
| `db_drop_column` | Drop a column from a table (SQLite 3.35.0+ required) |
| `db_rename_table` | Rename a table |
| `db_create_index` | Create an index on a table column |
| `db_drop_index` | Drop an index |
| `db_execute_if` | Run a write query only when a check SELECT returns rows |
| `db_export_json` | Export a query result to a JSON file |
| `db_listen` | Subscribe to or unsubscribe from a PostgreSQL NOTIFY channel |
| `db_clear_cache` | Manually invalidate the query cache for a connection |
| `db_analyze` | Run VACUUM (SQLite) or ANALYZE (others) |
| `db_clean_pool` | Evict idle connections from a pool |
| `db_set_pool_size` | Change maximum pool size (1–100) |
| `db_timeout` | Change connection timeout (minimum 250ms) |

---

## Full Event Reference

| Event | Switches | Contexts | Description |
|---|---|---|---|
| `on db connected` | `id` | `id` | Connection pool initialized |
| `on db disconnected` | — | `id` | Connection pool closed |
| `on db executed` | `id`, `label` | `id`, `label`, `affected_rows` | Write operation completed |
| `on db queried` | `id`, `label` | `id`, `label`, `rows` | `db_query_async` or `db_run` SELECT completed |
| `on db error` | `id`, `category` | `id`, `error`, `query`, `sql_state`, `category` | SQL or validation error |
| `on db migrated` | `id` | `id`, `count`, `path` | Migration run completed |
| `on db backed up` | `id` | `id`, `path` | SQLite backup completed |
| `on db csv exported` | — | `id`, `path` | CSV export completed |
| `on db csv imported` | — | `id`, `table`, `rows` | CSV import completed |
| `on db json exported` | — | `id`, `path` | JSON export completed |
| `on db notify` | `channel` | `id`, `channel`, `payload` | PostgreSQL NOTIFY received on a subscribed channel |
| `on db transaction expired` | — | `tx`, `id` | Transaction auto-rolled back after 5 minutes |
| `on db connection leaked` | — | `tx`, `duration` | Transaction exceeded the 5-minute threshold |

---

## Full Tag Reference

| Tag | Returns | Description |
|---|---|---|
| `<db_query[id].sql[q].args[list]>` | `ListTag` | All rows as a list of `MapTag` |
| `<db_query_first[id].sql[q].args[list]>` | `MapTag` | First row as a `MapTag`, or null |
| `<db_query_page[id].sql[q].page[n].size[n].args[list]>` | `ListTag` | Paginated rows (LIMIT/OFFSET) |
| `<db_cached_query[id].sql[q].ttl[ticks].args[list]>` | `ListTag` | Cached query result with TTL in ticks |
| `<db_value[id].sql[q].args[list]>` | `ElementTag` | First column of first row |
| `<db_exists[id].sql[q].args[list]>` | `ElementTag(Boolean)` | True if query returns at least one row |
| `<db_query_as_json[id].sql[q].args[list]>` | `ElementTag` | Result as JSON array string |
| `<db_convert_map[id].sql[q].args[list]>` | `MapTag` | Column 1 → key, column 2 → value |
| `<db_last_id[tx_id]>` | `ElementTag` | Last inserted row ID (requires transaction ID) |
| `<db_exists_table[id].table[name]>` | `ElementTag(Boolean)` | True if table exists |
| `<db_tables[id]>` | `ListTag` | All table names in the database |
| `<db_columns[id].table[name]>` | `ListTag` | All column names for a table |
| `<db_count[id].table[name]>` | `ElementTag` | Total row count for a table |
| `<db_connected[id]>` | `ElementTag(Boolean)` | True if pool is registered and open |
| `<db_ping[id]>` | `ElementTag(Boolean)` | True if connection is actively alive |
| `<db_list>` | `ListTag` | All active database connection IDs |
| `<db_info[id]>` | `MapTag` | Pool stats: `active_connections`, `idle_connections`, `total_connections`, `threads_awaiting` |

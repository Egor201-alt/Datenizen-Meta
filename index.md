# Introduction

Datenizen is a high-performance database management addon for Denizen, designed to provide a robust bridge between script-based logic and external database engines (SQLite, MySQL, PostgreSQL). It leverages HikariCP for efficient connection pooling, ensures thread safety by executing heavy database operations asynchronously, and provides a comprehensive suite of events and tags for seamless data manipulation.

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
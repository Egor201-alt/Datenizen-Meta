# Data Export and Import

Datenizen provides native support for CSV file operations, allowing for efficient bulk data migration and logging without writing complex SQL scripts.

### Exporting Data

The `db_export_csv` command runs a `SELECT` query and writes the resulting rows to a CSV file. Fields containing commas, quotes, or newlines are automatically escaped according to RFC 4180.

#### Syntax

```yaml
- db_export_csv id:<id> sql:<query> path:<path> (args:<list>)
```

#### Usage

```yaml
# Export all players to a CSV file
- db_export_csv id:main sql:SELECT * FROM players" path:plugins/Datenizen/exports/players.csv

# Export with parameterized filtering
- db_export_csv id:main sql:"SELECT * FROM players WHERE rank = ?" path:plugins/Datenizen/exports/gold_players.csv args:<list[gold]>
```

### Importing Data

The `db_import_csv` command parses a CSV file and performs batch insertions into a specified table. The first line of the CSV file is treated as the header row, and the structure is mapped to the table columns.

#### Syntax

```yaml
- db_import_csv id:<id> table:<table> path:<path>
```

#### Usage

```yaml
# Import CSV data into the 'players' table
- db_import_csv id:main table:players path:plugins/Datenizen/imports/players_backup.csv
```

### Event Handling

Operations related to CSV data can be tracked using events. These events are triggered in the main thread once the asynchronous I/O operation completes.

#### CSV Export Event
Triggered when an export operation is finished.
- `<context.id>`: Database ID.
- `<context.path>`: Path to the exported file.

#### CSV Import Event
Triggered when an import operation is finished.
- `<context.id>`: Database ID.
- `<context.table>`: Target table name.
- `<context.rows>`: Total number of rows successfully inserted.

```yaml
on db csv exported:
  - narrate "Data exported to <context.path>"

on db csv imported:
  - narrate "Imported <context.rows> rows into <context.table>"
```
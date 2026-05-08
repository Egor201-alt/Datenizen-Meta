# Transaction Management

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

Datenizen automatically manages transaction lifecycles. If a transaction remains open for more than 5 minutes (300,000ms), it is automatically rolled back to prevent connection leaks. Use the `db transaction expired` event to monitor these incidents.

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
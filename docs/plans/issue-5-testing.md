# Issue #5 - Testing Report

## Tests Performed

| Test | Result | Notes |
|------|--------|-------|
| Transaction starts in ACTIVE state | PASS | Verified tx.state == ACTIVE |
| Log INSERT for rollback | PASS | old_data is None, operation == "INSERT" |
| Log DELETE for rollback | PASS | old_data captured for restoration |
| Log UPDATE for rollback | PASS | old_data captured for restoration |
| Commit clears undo log | PASS | undo_log == [], modified_tables == set() |
| Rollback restores deleted record | PASS | Record restored from old_data |
| Rollback removes inserted record | PASS | Record deleted from DB |
| Rollback reverse order (LIFO) | PASS | Multi-operation rollback correct |
| TransactionLog append and read | PASS | JSON log file created |
| Get uncommitted transactions | PASS | Returns only ACTIVE entries |
| Exclude committed from uncommitted | PASS | COMMITTED entries filtered out |
| Clear log | PASS | Log file removed |
| BEGIN starts transaction | PASS | auto_commit=False, current_transaction set |
| BEGIN when active raises error | PASS | ActiveTransactionError raised |
| COMMIT without BEGIN raises error | PASS | NoActiveTransactionError raised |
| COMMIT finalizes transaction | PASS | Data persisted, tx cleared |
| ROLLBACK without BEGIN raises error | PASS | NoActiveTransactionError raised |
| ROLLBACK undoes INSERT | PASS | Inserted record removed |
| ROLLBACK undoes DELETE | PASS | Deleted record restored |
| Insert auto-commits by default | PASS | No tx needed, data persists |
| Schema change auto-commits | PASS | create_table outside tx persists |
| Crash recovery on startup | PASS | Uncommitted inserts rolled back |
| Full transaction commit | PASS | Multi-insert commit persists |
| Full transaction rollback | PASS | Multi-insert rollback removes all |
| Delete then rollback | PASS | Deleted records restored |
| **Edge Cases** | | |
| Rollback empty transaction | PASS | No-op rollback handled |
| Commit empty transaction | PASS | No-op commit handled |
| Multiple rollback cycles | PASS | 5 consecutive begin/rollback cycles |
| Partial operations then rollback | PASS | Mixed insert/delete rollback correct |
| Update no match | PASS | 0 rows updated, no crash |
| Very long char in transaction | PASS | 1000-char string handled |
| Unicode in transaction | PASS | Japanese characters handled |
| **State Manipulation** | | |
| No commit after rollback | PASS | NoActiveTransactionError |
| No rollback after commit | PASS | NoActiveTransactionError |
| Insert without begin auto-commits | PASS | Data persists immediately |
| Delete without begin auto-commits | PASS | Data removed immediately |
| Update without begin auto-commits | PASS | Data updated immediately |
| **Crash Recovery** | | |
| Recovery with no log file | PASS | Startup works fine |
| Recovery clears log | PASS | Log cleared after recovery |
| Recovery multiple uncommitted | PASS | All uncommitted rolled back |
| **FK Integration** | | |
| FK insert then rollback | PASS | Main record undone (known limitation: referenced_by metadata) |

## Bugs Found & Fixed

1. **test_rollback_undoes_delete assertion bug** — The pre-existing test incorrectly asserted `table_db.get(b"(2,)") is None` after rollback. After rollback, deleted records should be restored, so this was changed to `is not None`.

2. **Windows file locking in fixtures** — `shutil.rmtree(DB_DIR)` failed on Windows because dbm files remained locked between tests. Fixed by adding `gc.collect()` before cleanup and using `onerror` handler.

3. **update_query transformer bug** — The assignment was captured at wrong index due to missing SET token offset. Fixed by reading assignment at `items[3]` and where at `items[4]`.

## Verified Robust Against

- Empty transactions (begin->commit/rollback with no operations)
- Multiple rapid begin/rollback cycles
- Insert/delete/update without explicit transaction (auto-commit)
- Unicode and long string values in transactions
- Crash recovery with no prior log file
- Multiple uncommitted transactions on startup
- Nested BEGIN calls (rejected with error)
- COMMIT/ROLLBACK without active transaction (rejected with error)
- UPDATE with no matching WHERE clause (returns 0 rows, no crash)

## Known Limitations

- **FK referenced_by metadata**: When a transaction inserts a record with a foreign key and then rolls back, the main record is correctly removed. However, the `referenced_by` metadata in the parent (referenced) table is updated immediately and is not reverted on rollback. This is acceptable for Phase 3.1 basic transaction support and would require additional side-effect logging for full correctness.

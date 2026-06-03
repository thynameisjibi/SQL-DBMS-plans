# Issue #14 – Testing Report

## Tests Performed

| Test | Result | Notes |
|------|--------|-------|
| Grammar parsing: CREATE INDEX | PASS | `CREATE INDEX idx ON t (col)` parses correctly |
| Grammar parsing: DROP INDEX | PASS | `DROP INDEX idx ON t` parses correctly |
| Transformer: index dict populated | PASS | `index_name`, `table_name`, `column_name` extracted |
| DBMS: create_index with name | PASS | Metadata updated, index file created |
| DBMS: drop_index by name | PASS | Metadata cleaned, index file updated |
| Insert propagates to index | PASS | `IndexManager.insert()` called after `INSERT` |
| Delete propagates to index | PASS | `IndexManager.delete()` called after `DELETE` |
| Range search on hash index | PASS | O(n) scan returns correct values |
| Persistence across restart | PASS | New `DBMS` instance loads `.idx` file |
| Duplicate index name | PASS | `DuplicateIndexError` raised |
| Column already indexed | PASS | `IndexExistenceError` raised |
| Non-existent column | PASS | `NonExistingColumnDefError` raised |
| Non-existent table | PASS | `NoSuchTable` raised |
| Drop non-existent index | PASS | `NoSuchIndexError` raised |
| Drop table cleans index | PASS | `.idx` file removed |
| EXPLAIN shows IDX key | PASS | `PRI/FOR/IDX` displayed correctly |
| Multiple indexes per table | PASS | All tracked in `index_definitions` |
| Index name scoped per table | PASS | Same name on different tables allowed |
| Edge case: long name (200+ chars) | PASS | Accepted and functional |
| Edge case: rapid create/drop/create | PASS | Full cycle works |
| Edge case: double drop | PASS | `NoSuchIndexError` on second drop |
| Edge case: index on PK column | PASS | Allowed (no conflict) |
| Edge case: case-insensitive names | PASS | Stored lowercase, lookup case-insensitive |
| Edge case: delete all with index | PASS | Index correctly emptied |
| Edge case: unicode index name | XFAIL | Grammar `LETTER` token is ASCII-only |

## Bugs Found & Fixed

- **Grammar IDENTIFIER only matched letters/underscores** — Fixed by allowing digits: `IDENTIFIER : C (C | "_" | N)*`. This enables realistic SQL identifiers like `idx1`, `t1`.
- **IndexManager lacked range_search proxy** — Added `range_search(column_name, low, high)` method to delegate to `HashIndex.range_search()`.

## Verified Robust Against

- Duplicate index names on same table
- Same column indexed with different names
- Non-existent tables, columns, indexes
- Table drop with active indexes
- Rapid create/drop cycles
- Case-insensitive index name handling
- Multiple concurrent indexes on one table
- Persistence across DBMS restarts
- Full table delete with indexed columns

## Test Count Summary

| Suite | Tests | Status |
|-------|-------|--------|
| `test_index_sql.py` | 15 | All PASS |
| `test_index.py` | 22 | All PASS |
| `test_dbms_index.py` | 15 | All PASS |
| `test_adversarial_index.py` | 11 | 10 PASS, 1 XFAIL |
| Total relevant | 63 | 62 PASS, 1 XFAIL |

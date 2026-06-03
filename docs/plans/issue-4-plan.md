# Issue #4: Phase 2.1: Hash Index Implementation

## Summary

Implement a hash-based indexing system for O(1) average-case single-column lookups in the SQL-DBMS. This adds index support to replace full table scans (O(n)) with indexed lookups, significantly improving query performance for equality conditions on indexed columns.

**Note:** This implements a Hash Index (dictionary-based), not a B-Tree. True B-Tree implementation is deferred to Phase 2.3 (Advanced Bonus).

## Root Cause Analysis

**Current State:**
- No indexing exists in the DBMS
- All SELECT queries with WHERE clauses perform full table scans
- Query performance is O(n) where n is the number of records
- As tables grow, query performance degrades linearly

**Why This Matters:**
- Without indexes, every query must examine every record
- Large tables become unusable for interactive queries
- The DBMS cannot compete with even basic database systems

## Proposed Solution

Implement a two-tier indexing architecture:

1. **HashIndex Class**: In-memory hash table (Python dict) mapping column values to record keys
   - O(1) average-case insert, delete, and exact match search
   - O(n) range queries (hash indexes are not optimized for ranges)
   - One index per column

2. **IndexManager Class**: Manages multiple indexes per table with disk persistence
   - Creates/drops indexes on columns
   - Maintains indexes during INSERT/UPDATE/DELETE operations
   - Persists indexes to disk using pickle for fast startup

3. **DBMS Integration**: Modify existing DBMS operations to maintain indexes
   - INSERT: Add entry to all table indexes
   - DELETE: Remove entries from all table indexes
   - UPDATE: Remove old value, add new value to indexes
   - SELECT: Use index for WHERE clause optimization (Phase 2.2)

## Files to Modify

| File | Change |
|------|--------|
| `index.py` | **NEW** - HashIndex and IndexManager classes |
| `dbms.py` | Import IndexManager; modify `insert()` to update indexes; modify `delete()` to update indexes |
| `db_model.py` | Add index metadata to Table class (optional: track indexed columns) |

## New Files

| File | Purpose |
|------|---------|
| `index.py` | Core indexing logic with HashIndex and IndexManager classes |
| `test_index.py` | Unit tests for index operations (optional but recommended) |

## Implementation Steps

### Step 1: Create index.py Module

Create `index.py` with the following structure:

```python
from typing import Dict, Set, Any, Optional
from pathlib import Path
import pickle


class HashIndex:
    """Hash-based index for O(1) average-case lookups."""
    
    def __init__(self, column_name: str, ascending: bool = True):
        self.column_name = column_name
        self.ascending = ascending  # Stored for API compatibility, not used in hash index
        self.index: Dict[Any, Set[bytes]] = {}  # value -> set of record_keys
    
    def insert(self, value: Any, record_key: bytes):
        """Add value->key mapping. O(1) average."""
        if value not in self.index:
            self.index[value] = set()
        self.index[value].add(record_key)
    
    def delete(self, value: Any, record_key: bytes):
        """Remove value->key mapping. O(1) average."""
        if value in self.index:
            self.index[value].discard(record_key)
            if not self.index[value]:  # Clean up empty sets
                del self.index[value]
    
    def search(self, value: Any) -> Set[bytes]:
        """Return matching record_keys. O(1) average."""
        return self.index.get(value, set()).copy()
    
    def range_search(self, low: Any, high: Any) -> Set[bytes]:
        """Return keys in range [low, high]. O(n) - hash indexes are NOT optimized for ranges."""
        result = set()
        for value, keys in self.index.items():
            if low <= value <= high:
                result.update(keys)
        return result
    
    def __len__(self):
        """Return number of indexed entries."""
        return len(self.index)


class IndexManager:
    """Manages indexes for a single table with disk persistence."""
    
    def __init__(self, table_name: str, db_dir: Path = None):
        self.table_name = table_name
        self.db_dir = db_dir or Path("./DB")
        self.indexes: Dict[str, HashIndex] = {}
        self.index_file = self.db_dir / f"{table_name}_indexes.idx"
        self.load_indexes()
    
    def create_index(self, column_name: str) -> bool:
        """Create index on column. Returns False if already exists."""
        if column_name in self.indexes:
            return False
        self.indexes[column_name] = HashIndex(column_name)
        self.save_indexes()
        return True
    
    def drop_index(self, column_name: str) -> bool:
        """Drop index on column. Returns False if doesn't exist."""
        if column_name not in self.indexes:
            return False
        del self.indexes[column_name]
        self.save_indexes()
        return True
    
    def get_index(self, column_name: str) -> Optional[HashIndex]:
        """Get index for column. Returns None if doesn't exist."""
        return self.indexes.get(column_name)
    
    def has_index(self, column_name: str) -> bool:
        """Check if index exists on column."""
        return column_name in self.indexes
    
    def insert(self, column_name: str, value: Any, record_key: bytes):
        """Insert into all indexes (or specific column)."""
        if column_name in self.indexes:
            self.indexes[column_name].insert(value, record_key)
    
    def delete(self, column_name: str, value: Any, record_key: bytes):
        """Delete from all indexes (or specific column)."""
        if column_name in self.indexes:
            self.indexes[column_name].delete(value, record_key)
    
    def save_indexes(self):
        """Persist indexes to disk using pickle."""
        self.db_dir.mkdir(exist_ok=True)
        with open(self.index_file, 'wb') as f:
            pickle.dump(self.indexes, f)
    
    def load_indexes(self):
        """Load indexes from disk if they exist."""
        if self.index_file.exists():
            try:
                with open(self.index_file, 'rb') as f:
                    self.indexes = pickle.load(f)
            except (pickle.UnpicklingError, EOFError, KeyError):
                # Corrupted or incompatible index file - rebuild
                self.indexes = {}
    
    def rebuild_index(self, column_name: str, records: list):
        """Rebuild index from scratch using provided records.
        
        Args:
            column_name: Column to index
            records: List of (value, record_key) tuples
        """
        self.indexes[column_name] = HashIndex(column_name)
        for value, record_key in records:
            self.indexes[column_name].insert(value, record_key)
        self.save_indexes()
```

### Step 2: Modify dbms.py to Support Indexes

Update the DBMS class to maintain indexes during data operations:

**In `__init__`:**
```python
def __init__(self):
    self.db_dir = Path("./DB")
    self.db_dir.mkdir(exist_ok=True)
    self.meta_db = MetaDB()
    self.index_managers: Dict[str, IndexManager] = {}  # table_name -> IndexManager
```

**Add helper methods:**
```python
def _get_index_manager(self, table_name: str) -> IndexManager:
    """Get or create IndexManager for a table."""
    if table_name not in self.index_managers:
        self.index_managers[table_name] = IndexManager(table_name, self.db_dir)
    return self.index_managers[table_name]

def create_index(self, table_name: str, column_name: str):
    """Create an index on a table column."""
    self.meta_db.open_db()
    table_key = self.meta_db.create_key_from_value(table_name)
    table = self.meta_db.get(table_key)
    if not table:
        raise NoSuchTable()
    if column_name not in table.columns:
        raise NonExistingColumnDefError(column_name)
    self.meta_db.close_db()
    
    index_manager = self._get_index_manager(table_name)
    if not index_manager.create_index(column_name):
        raise Exception("Index already exists")
    
    # Rebuild index from existing data
    table_db = DB(table_name)
    table_db.open_db()
    records = []
    cursor = table_db.create_cursor()
    key_value_pair = cursor.first()
    while key_value_pair:
        key, value = key_value_pair
        record = Record.deserialize(value)
        if column_name in record.data:
            records.append((record.data[column_name], key))
        key_value_pair = cursor.next()
    table_db.discard_cursor(cursor)
    table_db.close_db()
    
    index_manager.rebuild_index(column_name, records)
    return f"Index created on {table_name}.{column_name}"

def drop_index(self, table_name: str, column_name: str):
    """Drop an index from a table column."""
    index_manager = self._get_index_manager(table_name)
    if not index_manager.drop_index(column_name):
        raise Exception("Index does not exist")
    return f"Index dropped on {table_name}.{column_name}"
```

**Modify `insert()` method:**
After the existing insert logic (after line 216), add:
```python
# Update indexes
if table_name in self.index_managers:
    index_manager = self.index_managers[table_name]
    for column_name, value in data.items():
        index_manager.insert(column_name, value, record_key)
```

**Modify `delete()` method:**
Before deleting records, get the values for index cleanup. After line 238 (after deserializing the record), add:
```python
# Update indexes before deletion
if table_name in self.index_managers:
    index_manager = self.index_managers[table_name]
    for column_name, value in record.data.items():
        index_manager.delete(column_name, value, key)
```

### Step 3: Add Index Metadata to Table Class (Optional)

In `db_model.py`, add tracking for indexed columns:

```python
class Table(DataObject):
    def __init__(
        self, 
        table_name: str, 
        columns: Dict[str, str], 
        not_null_keys: Set[str], 
        primary_key: Tuple[str], 
        foreign_keys: Dict[str, Tuple[str, str]],
        referenced_by: Set[str]=set(),
        indexed_columns: Set[str]=None  # NEW
    ):
        # ... existing code ...
        self.indexed_columns = indexed_columns or set()
    
    def add_index(self, column_name: str):
        """Mark column as indexed."""
        self.indexed_columns.add(column_name)
    
    def remove_index(self, column_name: str):
        """Mark column as not indexed."""
        self.indexed_columns.discard(column_name)
```

### Step 4: Create Unit Tests

Create `test_index.py` to verify index functionality:

```python
import unittest
from pathlib import Path
import shutil
from index import HashIndex, IndexManager


class TestHashIndex(unittest.TestCase):
    
    def setUp(self):
        self.index = HashIndex("test_column")
    
    def test_insert_and_search(self):
        self.index.insert(42, b"key1")
        result = self.index.search(42)
        self.assertEqual(result, {b"key1"})
    
    def test_multiple_values(self):
        self.index.insert(42, b"key1")
        self.index.insert(42, b"key2")
        result = self.index.search(42)
        self.assertEqual(result, {b"key1", b"key2"})
    
    def test_delete(self):
        self.index.insert(42, b"key1")
        self.index.delete(42, b"key1")
        result = self.index.search(42)
        self.assertEqual(result, set())
    
    def test_range_search(self):
        for i in range(10):
            self.index.insert(i, f"key{i}".encode())
        result = self.index.range_search(3, 6)
        self.assertEqual(len(result), 4)  # 3, 4, 5, 6


class TestIndexManager(unittest.TestCase):
    
    def setUp(self):
        self.test_dir = Path("./test_db")
        self.test_dir.mkdir(exist_ok=True)
        self.manager = IndexManager("test_table", self.test_dir)
    
    def tearDown(self):
        shutil.rmtree(self.test_dir, ignore_errors=True)
    
    def test_create_index(self):
        result = self.manager.create_index("column1")
        self.assertTrue(result)
        self.assertTrue(self.manager.has_index("column1"))
    
    def test_create_duplicate_index(self):
        self.manager.create_index("column1")
        result = self.manager.create_index("column1")
        self.assertFalse(result)
    
    def test_drop_index(self):
        self.manager.create_index("column1")
        result = self.manager.drop_index("column1")
        self.assertTrue(result)
        self.assertFalse(self.manager.has_index("column1"))
    
    def test_persistence(self):
        self.manager.create_index("column1")
        self.manager.insert("column1", 42, b"key1")
        
        # Create new manager (should load from disk)
        manager2 = IndexManager("test_table", self.test_dir)
        self.assertTrue(manager2.has_index("column1"))
        result = manager2.search("column1", 42)
        self.assertEqual(result, {b"key1"})


if __name__ == "__main__":
    unittest.main()
```

## Test Strategy

### Unit Tests
- **HashIndex**: Test insert, search, delete, range_search operations
- **IndexManager**: Test create, drop, persistence, rebuild operations
- **Edge cases**: 
  - NULL values in indexed columns
  - Duplicate values (multiple records with same column value)
  - Empty index operations
  - Index persistence across restarts

### Integration Tests
- INSERT with indexes: Verify index entries are created
- DELETE with indexes: Verify index entries are removed
- UPDATE with indexes: Verify old values removed, new values added
- SELECT with indexed WHERE clause: Verify index is used (Phase 2.2)

### Edge Cases
1. **NULL handling**: Indexes should handle NULL values correctly
2. **Composite operations**: Multiple inserts/deletes in sequence
3. **Index rebuild**: Verify rebuild produces correct index from existing data
4. **Corrupted index file**: Verify graceful recovery (rebuild from scratch)
5. **Concurrent access**: Basic thread safety (if applicable)

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| **Index corruption on crash** | Implement atomic writes (write to temp file, then rename); rebuild indexes from data on startup if corrupted |
| **Memory usage for large indexes** | Hash indexes are in-memory; for very large tables, consider disk-based structures (B-Tree in Phase 2.3) |
| **Performance degradation on writes** | Each INSERT/UPDATE/DELETE now updates all indexes; limit number of indexes per table (document best practices) |
| **Pickle security** | Pickle is used for persistence; document that index files should not be loaded from untrusted sources |
| **Range query performance** | Hash indexes are O(n) for ranges; document this limitation and recommend B-Tree for range-heavy workloads (Phase 2.3) |
| **Index staleness** | If DBMS crashes mid-operation, indexes may be stale; implement recovery/rebuild on startup |

## Diagrams

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                      DBMS Layer                          │
├─────────────────────────────────────────────────────────┤
│  create_index()  │  insert()  │  delete()  │  select()  │
└────────┬─────────┴─────┬──────┴─────┬────────┴────┬─────┘
         │               │            │             │
         ▼               ▼            ▼             ▼
┌─────────────────────────────────────────────────────────┐
│                   IndexManager Layer                     │
│  ┌─────────────────┐  ┌─────────────────┐              │
│  │ IndexManager    │  │ IndexManager    │  ...         │
│  │ (table1)        │  │ (table2)        │              │
│  ├─────────────────┤  ├─────────────────┤              │
│  │ indexes: dict   │  │ indexes: dict   │              │
│  │ save/load()     │  │ save/load()     │              │
│  └────────┬────────┘  └────────┬────────┘              │
└───────────┼────────────────────┼────────────────────────┘
            │                    │
            ▼                    ▼
┌───────────────────┐  ┌───────────────────┐
│  HashIndex        │  │  HashIndex        │
│  (column1)        │  │  (column2)        │
├───────────────────┤  ├───────────────────┤
│ index: dict       │  │ index: dict       │
│ {value: {keys}}   │  │ {value: {keys}}   │
└───────────────────┘  └───────────────────┘
            │                    │
            ▼                    ▼
┌─────────────────────────────────────────────────────────┐
│                  Disk Persistence                        │
│  ┌─────────────────┐  ┌─────────────────┐              │
│  │ table1_indexes  │  │ table2_indexes  │              │
│  │ .idx (pickle)   │  │ .idx (pickle)   │              │
│  └─────────────────┘  └─────────────────┘              │
└─────────────────────────────────────────────────────────┘
```

### Data Flow: INSERT with Index

```
┌──────────┐
│  INSERT  │
│  Statement│
└────┬─────┘
     │
     ▼
┌─────────────────┐
│  DBMS.insert()  │
│  - Validate     │
│  - Create Record│
│  - Store in DB  │
└────┬────────────┘
     │
     ▼
┌─────────────────────────┐
│  IndexManager.insert()  │  ◄── For each indexed column
│  - Get HashIndex        │
│  - index[value].add(key)│
│  - Save to disk         │
└────┬────────────────────┘
     │
     ▼
┌──────────┐
│  Success │
└──────────┘
```

### Index Lookup Flow

```
┌──────────┐
│  SELECT  │
│  WHERE   │
│  col = X │
└────┬─────┘
     │
     ▼
┌─────────────────┐
│  Check if index │
│  exists on col  │
└────┬────────────┘
     │
     ├───────┐
     │ Yes   │ No
     ▼       ▼
┌─────────┐ ┌──────────────┐
│ HashIndex│ │ Full Table   │
│ search(X)│ │ Scan         │
│ O(1)    │ │ O(n)         │
└────┬────┘ └──────┬───────┘
     │             │
     ▼             ▼
┌─────────────────────┐
│  Return record keys │
│  Fetch from DB      │
└─────────────────────┘
```

## Performance Characteristics

| Operation | Without Index | With Hash Index |
|-----------|--------------|-----------------|
| INSERT    | O(1)         | O(1) + k*O(1)*  |
| DELETE    | O(n)         | O(1) + k*O(1)*  |
| SELECT (exact match) | O(n) | O(1) |
| SELECT (range) | O(n) | O(n) (no benefit) |

*k = number of indexes on the table

## Future Enhancements (Out of Scope)

1. **Phase 2.2: Query Planner** - Automatically use indexes in SELECT queries
2. **Phase 2.3: B-Tree Index** - True O(log n) for both exact match and range queries
3. **Composite Indexes** - Indexes on multiple columns
4. **Covering Indexes** - Store additional column data in the index
5. **Index Statistics** - Track cardinality, selectivity for query optimization

## References

- IMPLEMENTATION_PLAN.md Phase 2.1 (revised from B-Tree to Hash Index)
- Phase 2.2: Query Planner (uses indexes for optimization)
- Phase 2.3: B-Tree Index (advanced bonus for true O(log n))

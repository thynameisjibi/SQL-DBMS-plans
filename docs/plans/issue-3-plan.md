# Issue #3: Phase 1.2 - UPDATE Statement Implementation

## Summary

Implement the UPDATE statement with full constraint checking, supporting both single and multiple column assignments. This feature completes the CRUD operations by adding the missing UPDATE functionality to the SQL DBMS.

## Root Cause Analysis

**Current State:**
- The UPDATE grammar exists in `grammar.lark` (line 152) but only supports single assignment
- The `SQLTransformer.update_query()` method (sql_transformer.py lines 249-251) returns a placeholder string `"'UPDATE' requested"` instead of parsing the statement properly
- The `DBMS` class has no `update()` method
- The `run.py` dispatcher has no UPDATE statement handler

**Why This Issue Exists:**
The UPDATE statement was defined in the grammar as a stub during initial setup but was never implemented. The grammar currently only supports:
```lark
update_query : UPDATE table_name SET assignment [where_clause]
assignment : column_name EQUAL value
```

This needs to be extended to support multiple assignments and the full implementation pipeline.

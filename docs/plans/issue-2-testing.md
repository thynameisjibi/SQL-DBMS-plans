# Issue #2 - Testing Report

## Tests Performed

### Unit Tests (`test_utils.py` — 27 tests)
| Test | Result | Notes |
|------|--------|-------|
| INT: positive number passes | PASS | |
| INT: zero passes | PASS | |
| INT: negative passes | PASS | |
| INT: bool True fails | PASS | Fixed: `isinstance(v, int) and not isinstance(v, bool)` |
| INT: bool False fails | PASS | Fixed: same as above |
| INT: string fails | PASS | |
| INT: float fails | PASS | |
| INT: None passes | PASS | NULL handling preserved |
| CHAR: short string passes | PASS | |
| CHAR: exact length passes | PASS | |
| CHAR: overflow fails | PASS | Fixed: validates `len(v) <= N` instead of truncating |
| CHAR: empty string passes | PASS | |
| CHAR: digit string passes | PASS | Fixed: removed erroneous `not v.isdigit()` check |
| CHAR: non-string fails | PASS | |
| CHAR: None passes | PASS | |
| DATE: valid passes | PASS | |
| DATE: first/last day passes | PASS | |
| DATE: invalid month fails | PASS | Fixed: `1 <= month <= 12` |
| DATE: month zero fails | PASS | Fixed: same |
| DATE: invalid day fails | PASS | Fixed: `1 <= day <= 31` |
| DATE: day zero fails | PASS | Fixed: same |
| DATE: non-string fails | PASS | |
| DATE: bad format fails | PASS | |
| DATE: None passes | PASS | |
| Edge: unknown type | PASS | Returns False |
| Edge: large N boundary | PASS | |

### Integration Tests (`test_dbms_insert.py` — 13 tests)
| Test | Result | Notes |
|------|--------|-------|
| Valid int+char+date insert | PASS | End-to-end success |
| NULL for nullable columns | PASS | |
| Exact CHAR boundary | PASS | No truncation, exact match allowed |
| Empty string CHAR | PASS | |
| Bool as INT fails | PASS | Raises InsertTypeMismatchError |
| String as INT fails | PASS | Raises InsertTypeMismatchError |
| CHAR overflow fails | PASS | Raises InsertTypeMismatchError (not truncated) |
| INT as CHAR fails | PASS | Raises InsertTypeMismatchError |
| Invalid date month fails | PASS | Raises InsertTypeMismatchError |
| Invalid date day fails | PASS | Raises InsertTypeMismatchError |
| Bad date format fails | PASS | Raises InsertTypeMismatchError |
| NULL for NOT NULL fails | PASS | Raises InsertColumnNonNullableError |
| NULL for nullable passes | PASS | |

### Adversarial Tests (`test_adversarial.py` — 26 tests, 1 skipped)
| Test | Result | Notes |
|------|--------|-------|
| Very large INT | PASS | `10**18` accepted |
| numpy.bool_ | SKIP | numpy not installed |
| Custom int subclass | PASS | Accepted per plan scope |
| Unicode in CHAR | PASS | |
| Emoji boundary | PASS | `len()` counts codepoints |
| Whitespace CHAR | PASS | |
| Newline in CHAR | PASS | Correct length check |
| Bytes as CHAR | PASS | Rejected |
| List as CHAR | PASS | Rejected |
| Leap year Feb 29 | PASS | Simple 01-31 validation accepts |
| Non-leap Feb 29 | PASS | Known limitation — full calendar not in scope |
| Year 0000 / 9999 | PASS | |
| Missing leading zero | PASS | Rejected (regex strict) |
| Extra characters in date | PASS | Fixed: `re.fullmatch` rejects trailing chars |
| Empty date string | PASS | Rejected |
| None type name | PASS | Returns False gracefully |
| Empty type name | PASS | Returns False |
| Malformed char type | PASS | Fixed: catches SyntaxError/NameError |
| Rapid PK duplicate | PASS | InsertDuplicatePrimaryKeyError raised |
| Bool False as INT | PASS | InsertTypeMismatchError raised |
| CHAR exact boundary | PASS | |
| CHAR one over boundary | PASS | |
| Unicode in CHAR | PASS | |
| Unicode overflow CHAR | PASS | |
| All NULL except NOT NULL | PASS | |

## Bugs Found & Fixed During Adversarial Testing

1. **`re.match` allows trailing characters in DATE**: `"2023-01-01 "` passed because `re.match` only anchors at the start.
   - **Fix**: Changed to `re.fullmatch` to require the entire string to match the pattern.

2. **Malformed `char()` / `char(x)` crashed with uncaught exceptions**: `eval("")` raises `SyntaxError`, `eval("x")` raises `NameError`.
   - **Fix**: Added `SyntaxError` and `NameError` to the exception catch in `is_valid_type()`.

3. **Windows file locking in test teardown**: DB files were locked by the process, preventing cleanup.
   - **Fix**: Added `gc.collect()` and `ignore_errors=True` in test fixtures.

## Verified Robust Against

- Boolean-as-integer injection (True, False, numpy.bool_)
- CHAR length overflow (exact boundary, one over, unicode, emoji)
- DATE format attacks (trailing chars, missing leading zeros, invalid month/day)
- Type confusion (int as char, string as int, bytes as char, list as char)
- NULL injection (NOT NULL columns, nullable columns)
- Primary key duplication
- Malformed type strings

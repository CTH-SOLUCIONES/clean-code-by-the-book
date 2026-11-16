# Oracle: what changes when writing and reviewing

Read this when writing or reviewing SQL or PL/SQL that runs on Oracle: Autonomous Database 23ai,
from Go with `sijms/go-ora` or from Python with `python-oracledb`. It is not an Oracle manual: these
are the points where code that looks clean has a bug the compiler does not see. What was verified
was verified on an Oracle 23ai Free; what may vary on ADB is marked.

## Names that get captured
- Inside a static SQL statement, a parameter or variable with the same name as a column in the
  statement is read as the column. `UPDATE t SET status = 'X' WHERE id = id` touches every row, and
  `SET name = name` leaves the old value and reports the row as updated. This does not apply inside
  `VALUES (...)`, where there are no columns in scope, nor to dynamic SQL, which is text.
- Prefix it (`p_id`) or qualify it with the routine name (`revoke_access.id`). When qualifying an
  existing routine, qualify every use, not just the `WHERE`. Renaming a parameter breaks callers that
  use named notation (`revoke_access(id => 1)` gives PLS-00306).

## Errors in PL/SQL
- `RAISE_APPLICATION_ERROR` accepts codes between -20000 and -20999; outside that range it fails
  with ORA-21000.
- **`RAISE` keeps the code; `RAISE_APPLICATION_ERROR` replaces it.** A `WHEN OTHERS` that ends in
  `RAISE_APPLICATION_ERROR(-20999, ..., TRUE)` over an ORA-00001 hands the service ORA-20999: the
  gateway no longer sees the code it was supposed to translate. That is why `WHEN OTHERS` ends in
  `RAISE`, and `RAISE_APPLICATION_ERROR` is left for domain errors: a validation that fails, or the
  translation of a specific exception (`DUP_VAL_ON_INDEX`, `NO_DATA_FOUND`) in its handler. Its
  message carries identifiers, not personal data: it travels to the service and into `SQLERRM`.
- Inside a handler, the third argument set to `TRUE` keeps the original error on the stack; with the
  default (`FALSE`) it disappears. Outside a handler it adds nothing. That stack travels to the
  service with its full text (see "Messages with personal data").
- When an exception leaves the call, Oracle rolls back what that call did, including a log `INSERT`
  made in the handler. Logging is done from a procedure with `PRAGMA AUTONOMOUS_TRANSACTION`, which
  stores the code and identifiers, not `SQLERRM`.
- With `ALTER SESSION SET PLSQL_WARNINGS = 'ENABLE:ALL'`, a `WHEN OTHERS` that ends neither in
  `RAISE` nor in `RAISE_APPLICATION_ERROR` gives PLW-06009. It is off by default: turn it on when
  compiling in tests. It does not warn about a `WHEN OTHERS` that ends in `RAISE_APPLICATION_ERROR`
  (which masks the code) or about one without `TRUE`: review catches those.
- Three-valued logic: `v IN ('A', 'B')` with `v` null gives `NULL`, and `IF NOT ok THEN` does not
  enter. A validation like that lets the null through. Check `IS NULL` first, or validate with
  `COALESCE(..., FALSE)`.

## Messages with personal data
- In 23ai, several errors carry the value that failed: ORA-00001 arrives with ORA-03301 and the key
  values (`row with column values (NATIONAL_ID:'...') already exists`), ORA-01438 carries the number
  that did not fit, and ORA-01722 arrives with ORA-03302 and the text that could not be converted.
  ORA-12899, ORA-01400, ORA-02290 and ORA-02291 do not carry the value. Verified on Free; on ADB,
  unconfirmed.
- The same text is in `SQLERRM` and `FORMAT_ERROR_STACK` inside a handler, and in the error the
  driver returns: `err.Error()` in go-ora, `str(e)` in python-oracledb. With SQLAlchemy, `str(e)`
  also appends the bind variables to any error, including those that do not carry the value: create
  the engine with `hide_parameters=True`.
- That is why, for any Oracle error, the gateway keeps the code and, if known, the constraint or the
  column, and never takes the driver's error or its text to the log, because of personal-data law
  (e.g., GDPR, or Colombia's Law 1581 of 2012). It does not wrap it (`%w`, `source`) or copy it
  (`%v`, `err.Error()`, `str(e)`). In Python, the custom error is raised `from None`: without
  `from`, the implicit context carries the original into the traceback and `log.exception` writes
  it. The gateway knows the constraint name from the context of the statement, or extracts it from
  the message without recording the rest. What the use case does not distinguish comes out as an
  infrastructure error with that code, not as a 503 with no cause.
- How to read the code. In Go, `var oe *network.OracleError` (package
  `github.com/sijms/go-ora/v2/network`) and `errors.As(err, &oe)`, which works even if the error
  comes wrapped, and `oe.ErrCode == 1` for ORA-00001; `oe.ErrMsg` carries the full text. In
  python-oracledb, `oracledb.IntegrityError` and `e.args[0].code == 1`.

## Writes that look atomic and are not
- A `MERGE` is not an atomic upsert: if two sessions run a `MERGE` with the same new key, the second
  gets ORA-00001. Treat it like any key collision (read or retry), never with "query, then insert".
- `RETURNING ... INTO` does not return rows: it fills output variables. In Go with go-ora,
  `sql.Out{Dest: &id}` or `go_ora.Out{Dest: &id}`. The PostgreSQL idiom,
  `QueryRow(... RETURNING id).Scan(&id)`, fails against Oracle; on PostgreSQL it is still the right
  one. In python-oracledb, `cursor.var(...)`, whose value arrives as a list. If the key comes from a
  sequence, it also works to request its `NEXTVAL` first. Inserting and then looking up the row
  (`MAX(id)`) is a race.
- A function with DML cannot be called from a query (ORA-14551). A procedure that inserts and returns
  the key in an `OUT` parameter is fine.

## Empty string and `NULL`
- In a `VARCHAR2`, `''` is `NULL`. A `NOT NULL` rejects the empty string (ORA-01400), `WHERE col = ''`
  finds nothing (write `col IS NULL`), and what the service stored as `""` comes back as `NULL`: in
  Go, `sql.NullString` or a pointer; in Python, `None`. If the domain distinguishes "empty" from
  "absent", that distinction lives in the domain or in another column, not in a sentinel such as
  `' '`.

## Column length
- With `BYTE` semantics, which is what the test container uses (on ADB, confirm with `CHAR_USED` in
  `USER_TAB_COLUMNS`), `VARCHAR2(5)` admits 5 bytes: "ñañañ" is 5 characters and 8 bytes, and fails
  with ORA-12899. Python's `len()` counts characters; Go's counts bytes. Validate the length in the
  column's unit before writing: translating ORA-12899 is the safety net, not the check. Tests use
  values at the maximum, with accented characters and ñ.

## Numbers with decimals
- python-oracledb returns a `NUMBER` with decimals as `float` unless
  `oracledb.defaults.fetch_decimals = True`, which returns it as `Decimal`. Set it at startup, before
  the first query: a statement that has already run keeps the `float` type in its cache.
- In Go, go-ora scans a `NUMBER` into a `string` without loss; from there it is converted to a
  decimal type, or the amount lives as an integer in the smallest unit. Not into `float64` (G26, see
  [chapter-17-heuristics.md](chapter-17-heuristics.md)).

## What cannot be bound
- DDL does not accept bind variables, and neither do identifiers. In idempotent startup, names and
  DDL come from constants in the code or, if they come from configuration, go through `DBMS_ASSERT`
  (`SIMPLE_SQL_NAME`, `ENQUOTE_NAME`); never from user input.

## Changing what already exists in the database
- When the schema has no migrations, the change is made by hand in the database and startup only
  creates what is missing. If startup recreated what exists, it would silently overwrite the manual
  change while the code does not reflect it.
- Replacing a stateful package with different text (package-level variables, cursors or non-literal
  constants) makes the next call from each session that already used it fail with ORA-04068; the
  retry works, with the state reset to zero. A manual change triggers it too: with a pool, each
  connection fails once. A stateless package, or the same text, does not give the error.

## Testing
- What is in this file is tested against Oracle, not against PostgreSQL or a test double. A 23ai Free
  container is enough for the logic, concurrency (two connections) and column limits; what depends on
  ADB or on the driver against it (`MAX_STRING_SIZE`, length semantics, the detail of ORA-03301) is
  confirmed on a dev ADB.

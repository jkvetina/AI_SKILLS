# adt recompile

Recompile invalid database objects and optionally force recompilation of valid ones to change PL/SQL compilation flags. Handles retry logic automatically — objects that fail on the first pass are retried in reverse dependency order.


## Common Usage

```bash
# Recompile invalid objects on DEV (most common)
adt recompile -target DEV

# Recompile on UAT or PROD
adt recompile -target UAT
adt recompile -target PROD

# Force recompile ALL objects with native compilation and full optimization
adt recompile -target DEV -force -native -level 3

# Force recompile with PL/SQL warnings enabled
adt recompile -target DEV -force -native -level 3 -warnings SEVERE PERF -scope ALL

# Recompile only objects matching a name prefix
adt recompile -target DEV -name XX%

# Recompile only packages
adt recompile -target DEV -type PACKAGE%
```


## Flags

| Flag | Purpose |
|---|---|
| `-target {ENV}` | **Required.** Target environment (DEV, UAT, PROD, etc.) |
| `-schema {NAME}` | Limit to a specific schema/connection |
| `-key {PASSWORD}` | Decryption key for encrypted passwords |
| `-type {PATTERN}` | Filter by object type (LIKE syntax, e.g. `PACKAGE%`) |
| `-name {PATTERN}` | Filter by object name (LIKE syntax, e.g. `XX%`) |
| `-force` | Recompile even valid objects (not just invalid ones) |
| `-native` | Force native PL/SQL compilation (`PLSQL_CODE_TYPE = NATIVE`) |
| `-interpreted` | Force interpreted PL/SQL compilation |
| `-level {1-3}` | PL/SQL optimization level (`PLSQL_OPTIMIZE_LEVEL`) |
| `-scope {VALUES}` | PL/Scope identifiers gathering (IDENTIFIERS, STATEMENTS, ALL) |
| `-warnings {VALUES}` | Enable PL/SQL warnings (SEVERE, PERF, INFO) |
| `-silent` | Suppress screen output |


## How It Works

1. Connects to the target environment and fetches an overview of all objects with their validity status.
2. Builds `ALTER ... COMPILE` statements for each invalid object (or all objects when `-force` is used).
3. For PL/SQL object types (packages, procedures, functions, triggers), applies the requested compilation flags with `REUSE SETTINGS`.
4. Compiles objects sequentially, capturing any that fail ("troublemakers").
5. Reconnects and retries troublemakers in reverse order to resolve dependency chains.
6. Reports objects overview (total vs invalid), lists remaining invalid objects with error counts.
7. Sends a team notification if any invalid objects remain after recompilation.


## Supported Object Types

The overview query covers: PACKAGE, PACKAGE BODY, PROCEDURE, FUNCTION, TRIGGER, VIEW, MATERIALIZED VIEW, SYNONYM, TYPE, TYPE BODY, and MVIEW LOG.

Force mode (`-force`) recompiles all of these. Without `-force`, only objects with `status != 'VALID'` are compiled.


## Compilation Flags (PL/SQL objects only)

These flags only apply to PACKAGE, PACKAGE BODY, PROCEDURE, FUNCTION, and TRIGGER:

- **Code type**: `-native` sets `PLSQL_CODE_TYPE = NATIVE`, `-interpreted` sets `PLSQL_CODE_TYPE = INTERPRETED`. Native compilation improves runtime performance but makes compilation slower.
- **Optimization level**: `-level 1` (minimal), `-level 2` (default), `-level 3` (aggressive inlining). Level 3 requires native compilation to be most effective.
- **PL/Scope**: `-scope IDENTIFIERS` or `-scope STATEMENTS` or `-scope IDENTIFIERS STATEMENTS` (or `-scope ALL` for both). Populates `USER_IDENTIFIERS` and `USER_STATEMENTS` views for cross-reference and impact analysis.
- **Warnings**: `-scope SEVERE`, `-scope SEVERE PERF`, `-scope SEVERE PERF INFO`. Maps to `PLSQL_WARNINGS` session parameter. The query filters out PLW-prefixed warnings from the error summary.

All flags use `REUSE SETTINGS`, so any flag you don't explicitly set keeps its current database-level value.


## When to Use

- **After deployments** — clean up cascading invalidations caused by DDL changes (table/type alterations, synonym recreation, grant changes).
- **As a CI/CD post-deployment step** — run `adt recompile -target {ENV} -silent` after patch deployment.
- **When changing PL/SQL flags** — use `-force` together with `-native`, `-level`, `-warnings`, or `-scope` to apply new settings across all objects.
- **To investigate broken objects** — the output lists remaining invalid objects with their error counts and first ORA-/PLS- error code, which helps triage.


## Error Reporting

After recompilation, invalid objects are listed with columns from `USER_ERRORS`:

- `object_type`, `object_name` — the broken object
- `errors` — count of compilation errors
- `error` — the first ORA- or PLS- error code found

If invalid objects remain, a team notification is sent (via the `notify_team` method) with a formatted table of the broken objects.

# adt export_data

Export table data into CSV files with auto-generated SQL MERGE statements. Designed for seed data, LOV (list of values) tables, configuration tables, and other reference data that doesn't contain sensitive information.


## Common Usage

```bash
# Export a single table
adt export_data -name CONFIG_PARAMETERS

# Export multiple tables
adt export_data -name CONFIG_PARAMETERS LOV_STATUS LOV_TYPES

# Export tables matching a pattern
adt export_data -name CONFIG% LOV_%

# Export from a specific schema
adt export_data -name LOOKUP% -schema HR

# Re-export all previously exported tables (no -name = re-export existing .sql files in data/)
adt export_data
```


## Flags

| Flag | Purpose |
|---|---|
| `-name {PATTERN(S)}` | Table(s) to export (LIKE syntax with `%` wildcard, space-separated) |
| `-schema {NAME}` | Target schema (overrides default from connection) |
| `-env {ENV}` | Source environment (overrides default connection) |
| `-key {PASSWORD}` | Decryption key for encrypted passwords |

If `-name` is omitted, ADT re-exports all tables that already have `.sql` files in the `data/` directory — a convenient way to refresh all previously exported data.


## Output

Each table produces two files:

```
database_{schema}/data/{table_name}.csv   # raw data (semicolon-delimited by default)
database_{schema}/data/{table_name}.sql   # generated MERGE statement
```

The `.csv` file contains the actual data with column headers. The `.sql` file contains a MERGE statement that handles INSERT and UPDATE by default (DELETE is commented out). This makes it easy to review the source data and deploy it to other environments.


## Unsupported Column Types

BLOB, CLOB, XMLTYPE, and JSON columns are **not exported** — they are silently skipped. The query filters them out via `data_type NOT IN ('BLOB', 'CLOB', 'XMLTYPE', 'JSON')`. If the user expects these columns in the output, let them know this is a known limitation.


## Skipped Columns (Audit Columns)

Columns listed in `ignored_columns` in `config.yaml` are excluded from the export. The default list is:

- `CREATED_BY`
- `CREATED_AT`
- `UPDATED_BY`
- `UPDATED_AT`

These are typically audit columns that get populated automatically by triggers or APEX and should not be part of seed data.


## MERGE Statement Details

The generated `.sql` file uses a `MERGE INTO ... USING (SELECT ... FROM DUAL) ...` pattern:

- **Match key**: primary key columns, or first unique key if no PK exists
- **WHEN MATCHED**: updates all non-key columns (enabled by default)
- **WHEN NOT MATCHED**: inserts all columns (enabled by default)
- **DELETE**: commented out by default (`--DELETE FROM ...`), can be enabled in config via `tables_global.merge.delete: True`
- **Large tables**: data is split into batches of 10,000 rows per MERGE statement
- **COMMIT**: each batch ends with a commented `--COMMIT;`

Tables without a primary key or unique key are exported to CSV but **no MERGE statement is generated** (there's no reliable way to match rows).


## NLS Settings

The MERGE `.sql` file contains date values as literal strings. On target environments with different NLS date format settings, these may fail. Make sure the correct NLS settings (especially `NLS_DATE_FORMAT` and `NLS_TIMESTAMP_FORMAT`) are set before running the `.sql` file:

```sql
ALTER SESSION SET NLS_DATE_FORMAT = 'YYYY-MM-DD';
ALTER SESSION SET NLS_TIMESTAMP_FORMAT = 'YYYY-MM-DD HH24:MI:SS';
```


## Row Filtering

Tables can have WHERE filters configured in `config.yaml` to export only a subset of rows:

- `tables_global.where` — applies to all tables
- `tables.{TABLE_NAME}.where` — applies to a specific table (merged with global)

This is useful for exporting only active records or filtering out test data.


## Console Output

When this command runs, it lists all tables being exported. Always show the full console output to the user.

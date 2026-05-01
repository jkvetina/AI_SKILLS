# adt export_db

Export database objects from an Oracle schema into the repository folder structure. Each object becomes a clean `.sql` file organized by type. ADT uses `DBMS_METADATA` under the hood but applies extensive cleanup to remove clutter (quoted names, storage clauses, default collation, etc.) so that Git diffs stay meaningful.


## Common Usage

```bash
# Export objects changed in the last 7 days
adt export_db -recent 7

# Export specific object types
adt export_db -type PACKAGE% VIEW%

# Export specific objects by name
adt export_db -name MY_PACKAGE MY_VIEW

# Combine filters (recently changed packages matching a prefix)
adt export_db -recent 7 -type PACKAGE% -name APP_%

# Export jobs (no -recent flag — jobs have no last_ddl_time)
adt export_db -type JOB

# Delete existing folders before export (clean export)
adt export_db -recent 7 -delete

# Export from a specific schema
adt export_db -schema HR

# Export from a different environment
adt export_db -recent 7 -env UAT -key MySecretKey
```


## Flags

| Flag | Purpose |
|---|---|
| `-recent {days}` | Objects changed in the last N days (uses `last_ddl_time`) |
| `-type {PATTERN(S)}` | Filter by object type (LIKE syntax, space-separated: `-type PACKAGE% VIEW%`) |
| `-name {PATTERN(S)}` | Filter by object name (LIKE syntax, space-separated: `-name APP_% HR_%`) |
| `-schema {NAME}` | Target schema; use `%` for all schemas configured in connections |
| `-env {ENV}` | Source environment (overrides default connection) |
| `-key {PASSWORD}` | Decryption key for encrypted passwords |
| `-delete` | Delete existing object folders before export (clean slate) |


## Important: JOB Objects Have No Timestamp

Oracle scheduler jobs (`USER_SCHEDULER_JOBS`) do not have a `last_ddl_time` column. The `-recent` flag filters on `last_ddl_time`, so it will never find jobs. When the user asks to export jobs, omit the `-recent` flag and use `-type JOB` instead:

```bash
# Correct — export all jobs
adt export_db -type JOB

# WRONG — this will find nothing because jobs lack last_ddl_time
adt export_db -recent 7 -type JOB
```

If the user asks for "recent changes including jobs", run two commands: one with `-recent` for regular objects and a separate one with `-type JOB` for jobs.


## Output Structure

Objects are saved into a configurable folder structure under the schema root. The default layout from `config.yaml`:

```
database_{schema}/
    data/                  # exported table data (.csv + .sql)
    functions/             # standalone functions
    grants/                # grants made by this schema
    indexes/               # indexes
    jobs/                  # scheduler jobs
    job_schedules/         # scheduler schedules
    mviews/                # materialized views
    mview_logs/            # materialized view logs
    packages/              # package specs (.spec.sql) and bodies (.sql)
    procedures/            # standalone procedures
    sequences/             # sequences
    synonyms/              # synonyms
    tables/                # tables
    triggers/              # triggers
    types/                 # type specs (.sql) and bodies (.body.sql)
    views/                 # views
    unit_tests/            # unit test packages

database/
    grants_made/{schema}.sql       # grants this schema made to others
    grants_received/{schema}.sql   # grants received from other schemas
```

Each object is one `.sql` file, lowercase filename matching the object name. The `path_objects` pattern in `config.yaml` controls the top-level path (e.g. `database_{schema}/`, `database/`, etc.).

Users can organize files into subfolders (e.g. `views/ABC/`, `views/DEF/`) — ADT respects existing subfolder structure when re-exporting.


## Supported Object Types

Tables, Views, Indexes, Sequences, Synonyms, Packages (spec + body), Procedures, Functions, Triggers, Types (spec + body), Materialized Views, Materialized View Logs, Jobs, Schedules, Grants, Roles, Directories.


## How It Works

1. Connects to the target schema and builds a dependency graph of all objects.
2. Shows an overview table of matching objects grouped by type and count.
3. If `-delete` is set, clears existing object type folders (except data/) before exporting.
4. Detects objects that exist in the repo but no longer exist in the database (deleted objects) and lists them. If `auto_delete` is configured, removes the orphaned files.
5. Exports grants (made, received, roles, privileges, directories).
6. Exports each matching object via `DBMS_METADATA.GET_DDL`, applies cleanup transformations per object type, and writes the file.
7. Appends table/view/mview comments (`COMMENT ON ...`) to the object file.

When `-recent` or `-type` or `-name` flags are used, verbose mode activates automatically — each exported object is listed individually instead of just showing a progress bar.


## Export Cleanup

ADT cleans up the raw DDL output to minimize noise in Git diffs:

- Removes quoted identifiers and schema prefixes
- Strips storage clauses, tablespace references, and collation defaults
- Replaces tabs with spaces, removes trailing whitespace
- Simplifies identity/sequence column defaults (removes verbose MAXVALUE/MINVALUE/etc.)
- Separates package spec from body (body goes into its own file)
- For jobs: generates a completely custom PL/SQL block using `DBMS_SCHEDULER` calls instead of raw DDL
- Adds `IF NOT EXISTS` to table/index DDL on 23ai databases when configured
- Handles editionable/noneditionable attributes per config


## Schema and Environment Overrides

By default, `export_db` connects to the default schema/environment from `connections.yaml`. Override when needed:

- `-schema HR` — export from a different schema in the same environment
- `-schema %` — export from all schemas configured in `connections.yaml`
- `-env UAT` — connect to a different environment (e.g. pull objects from UAT instead of DEV)
- `-key` — provide decryption key if passwords are encrypted

Multi-schema exports iterate through each schema sequentially, respecting per-schema `prefix` and `ignore` filters from the connection config.


## Connection Filters (prefix / ignore)

In `connections.yaml`, each schema can define:

- `prefix` — comma-separated LIKE patterns for object names to include (e.g. `APP%,HR_%`)
- `ignore` — comma-separated LIKE patterns for object names to exclude

These filters apply automatically on every export, keeping unwanted objects out of the repo.


## Console Output

When this command runs, it produces:

1. **Objects overview** — a table showing each object type and count of matching objects
2. **Deleted objects** — objects in the repo that no longer exist in the database (if any)
3. **Export progress** — either a progress bar (default) or verbose per-object listing
4. **Completion beep** — audible notification when done

Always show the full console output to the user so they can see what was exported, what was deleted, and whether anything needs attention.

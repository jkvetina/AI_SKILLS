# adt patch

Create and deploy patch files from Git commits. This is the most powerful ADT command — it reads your commits, resolves dependencies, orders objects correctly, and generates deployment scripts. ADT does not store anything in the database; all information comes from Git (folders, files, config).


## Workflow

1. Work on a task — change database objects and/or APEX pages/components.
2. Export changes with `adt export_db` / `adt export_apex` / `adt export_data`.
3. Commit exported files to Git with the task/card number in the commit summary.
4. Generate the patch: `adt patch -target UAT -patch TASK_ID -create`.
5. Review the generated patch folder, adjust if needed.
6. Deploy: `adt patch -target UAT -patch TASK_ID -deploy` (or deploy via CI/CD pipeline).
7. Commit the patch folder.

You can repeat any of these steps as many times as needed. Before deploying you can change any of the generated files.


## Common Usage


### Overview & Commit Browsing

```bash
# Show help and available options
adt patch

# Show recent commits (default 20)
adt patch -target UAT -commits 50

# Show only my commits
adt patch -target UAT -commits 50 -my

# Show commits by a specific author
adt patch -target UAT -commits 50 -by JKVETINA

# Show only commits that have files attached
adt patch -target UAT -commits 50 -files

# Show recent patches
adt patch -target UAT -patches 20
adt patch -target UAT -patches 20 -my
```


### Patch Creation

```bash
# Preview which commits match the task (dry run — no -create)
adt patch -target UAT -patch TASK_ID

# Create the patch
adt patch -target UAT -patch TASK_ID -create

# Create patch with a specific sequence number (e.g. 2)
adt patch -target UAT -patch TASK_ID -create 2

# Add or ignore specific commits (space-separated, supports ranges with dash)
adt patch -target UAT -patch TASK_ID -commit 1-20 30-32 35 36
adt patch -target UAT -patch TASK_ID -ignore 2 3 8-12 14-15 31
adt patch -target UAT -patch TASK_ID -commit 1-20 30-32 35 36 -ignore 2 3 8-12 14-15 31

# Search commits by words in the summary (instead of task ID matching)
adt patch -target UAT -patch TASK_ID -search WORDS
adt patch -target UAT -patch TASK_ID -search %

# Use latest committed files (HEAD) instead of the version from the matching commit
adt patch -target UAT -patch TASK_ID -head

# Use local files (including all uncommitted changes)
adt patch -target UAT -patch TASK_ID -local

# Create patch from a specific branch
adt patch -target UAT -patch TASK_ID -branch FEATURE_BRANCH

# Create patch with full APEX application export (instead of changed pages/components)
adt patch -target UAT -patch TASK_ID -create -full
adt patch -target UAT -patch TASK_ID -create -full 100 200

# Fetch Git changes before patching
adt patch -target UAT -patch TASK_ID -create -fetch
```


### Deployment

```bash
# Deploy an existing patch
adt patch -target UAT -patch TASK_ID -deploy

# Create and deploy in one step
adt patch -target UAT -patch TASK_ID -create -deploy

# Deploy to a different environment
adt patch -target LAB3 -patch TASK_ID -deploy

# Continue on DB error instead of rollback
adt patch -target UAT -patch TASK_ID -deploy -continue

# Force redeployment (ignore prechecks)
adt patch -target UAT -patch TASK_ID -deploy -force
```


### Supporting Actions

```bash
# Create an install script (combines all patches)
adt patch -install

# Archive patches with specific reference numbers
adt patch -target UAT -archive 1 2 3

# Rebuild the Git cache file (needed after repo surgery or branch changes)
adt patch -target UAT -rebuild

# Refresh used objects and APEX components
adt patch -target UAT -patch TASK_ID -refresh

# Merge files in a folder into a single file
adt patch -implode FOLDER

# Store file hashes on patch creation (for change detection)
adt patch -target UAT -patch TASK_ID -create -hash

# Lock hash file and use it as source
adt patch -target UAT -patch TASK_ID -create -hash -locked
```


## Flags — Complete Reference

### Main Actions

| Flag | Purpose |
|---|---|
| `-patch {CARD_NUMBER}` | Patch code/name (typically the task/card ID from Jira, etc.) |
| `-ref {NUMBER}` | Patch reference number (the sequence number from the screen) |
| `-create` | Create patch file(s); optionally pass a sequence number |
| `-deploy` | Deploy the patch (can combine with `-create`) |
| `-force` | Force redeployment, ignore prechecks |
| `-continue` | Continue on DB error instead of `EXIT ROLLBACK` |

### Supporting Actions

| Flag | Purpose |
|---|---|
| `-archive {REF(S)}` | Archive patches with specific reference numbers |
| `-install` | Create a combined install file from all patches |
| `-refresh` | Refresh used objects and APEX components in the patch |
| `-rebuild` | Rebuild the Git cache file (needed after repo surgery or branch changes) |

### Environment

| Flag | Purpose |
|---|---|
| `-target {ENV}` | Target deployment environment (UAT, PROD, etc.) |
| `-branch {NAME}` | Override active Git branch |

### Scope & Filtering

| Flag | Purpose |
|---|---|
| `-commits {N}` | Show N recent commits |
| `-my` | Show only your commits |
| `-by {AUTHOR}` | Show commits by a specific author |
| `-files` | Show only commits that have files |
| `-search {WORDS}` | Search commit summaries for provided words |
| `-commit {HASH(ES)}` | Process specific commits (space-separated, supports ranges like `1-20`) |
| `-ignore {HASH(ES)}` | Exclude specific commits (space-separated, supports ranges) |
| `-full {APP_ID(S)}` | Use full APEX export for specific apps (or all if no IDs given) |
| `-local` | Use local files (including uncommitted) instead of Git versions |
| `-head` | Use file version from the HEAD commit |

### Additional

| Flag | Purpose |
|---|---|
| `-hash` | Store file hashes during patch creation (for change detection) |
| `-locked` | Lock hash file, use it as the authoritative source |
| `-fetch` | Fetch Git remote changes before patching |
| `-implode {FOLDER}` | Merge all files in a folder into a single file |


## Automatic Object Ordering

ADT resolves dependencies and orders objects in the patch file following this sequence (customizable via `patch_map` in `config.yaml`):

```
Sequences → Tables → Types → Synonyms → Objects (Views, Procedures, Functions, Packages) → Triggers → Materialized Views → Indexes → Data → Grants → Jobs
```

The `patch_map` in `config.yaml` maps group names to object types:

```yaml
patch_map:
    sequences:
        - SEQUENCE
    tables:
        - TABLE
    types:
        - TYPE
        - TYPE BODY
    synonyms:
        - SYNONYM
    objects:
        - VIEW
        - PROCEDURE
        - FUNCTION
        - PACKAGE
        - PACKAGE BODY
    triggers:
        - TRIGGER
    mviews:
        - MVIEW LOG
        - MATERIALIZED VIEW
    indexes:
        - INDEX
    data:
        - DATA
    grants:
        - GRANT
    jobs:
        - JOB
```

You can reorder groups, add object types to existing groups, or create new groups as needed.


## Patch Templates & Scripts

### Templates (apply to every patch)

Place files in `config/patch_template/` using `{group}_{timing}` or `{timing}_{group}` folder naming:

```
config/patch_template/
    tables_before/       # runs before TABLE objects
    tables_after/        # runs after TABLE objects
    before_tables/       # alternative naming (same effect)
    after_objects/       # runs after Views, Packages, etc.
    mviews_before/       # runs before Materialized Views (e.g. recompile)
    ...
```

All files and subfolders within a template folder are sorted alphabetically. Use numeric prefixes for ordering (e.g. `01_nls_setup.sql`, `02_enable_logging.sql`).

Common template uses: NLS setup at start, recompile before materialized views, increment APEX app version after deployment, set authentication scheme on production.

### Patch Scripts (apply to a specific patch only)

Place files in `config/patch_scripts/{CARD_NUMBER}/` using the same `{group}_{timing}` subfolder convention:

```
config/patch_scripts/TASK-123/
    tables_after/
        add_birthday_column.sql    # ALTER TABLE for this specific patch
    data_before/
        migrate_status_values.sql  # data migration before new data load
```

ADT auto-generates script stubs for ALTER TABLE and DROP operations when it detects table changes or deleted objects. You only need to handle data migrations manually.


## Patch Output

```
patch/{date}.{seq}.{patch_code}/             # patch folder
    {schema}.sql                             # patch file per schema
    {app_id}_{app_alias}.sql                 # patch file per APEX app
    snapshots/                               # copy of all files in the patch
    LOGS_{env}/                              # deployment logs (after deploy)
```

The patch file contains: an overview header showing planned objects, prompts for readability, source commit references for traceability, and the actual DDL/DML statements wrapped with proper error handling.


## Features & Behavior

- **Grants auto-included**: grants for the APEX schema are added automatically when relevant objects change.
- **ALTER generation**: when a table is modified, ADT detects the diff and generates ALTER TABLE stubs in `patch_scripts/`.
- **Deleted objects**: detected and flagged with `[DELETED]` marker; DROP scripts are generated.
- **New objects**: flagged with `[NEW]` marker in the overview.
- **View column verification**: during deployment, ADT checks that view column names match between the repo definition and the database to catch naming discrepancies.
- **Table diff detection**: compares table structure between patch versions to detect column additions, removals, and type changes.
- **Conflict detection**: warns when deploying patches out of order could overwrite changes from a later patch.
- **APEX handling**: shared components are installed first, pages last. Page creation/audit columns can be updated to match the patch code and date. Workspace setup scripts are added automatically.
- **Template variables**: variables in template files (like `{$PATCH_CODE}`, `{$TARGET_ENV}`) are replaced at generation time.
- **Snapshot folder**: every file in the patch is also copied to a `snapshots/` subfolder for complete traceability.
- **Hash-based tracking**: with `-hash`, ADT stores file hashes to detect which files actually changed between patches, enabling incremental deployments.


## Known Limitations

- If you change only the grants on an object (not the object itself), grants will not be included in the patch. Workaround: add the grant file to `patch_scripts/` for the specific patch or to `patch_template/` for all patches.
- If you change only the comment on a table (not the table itself), the comment will not make it to the patch.
- Revoked grants are not covered.
- Cross-schema dependency resolution is not supported — each schema is ordered independently.
- Some object types not listed in `patch_map` are not supported.

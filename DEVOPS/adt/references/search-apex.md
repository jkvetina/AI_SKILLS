# adt search_apex

Search for database objects referenced by APEX applications. Parses the Embedded Code report exports to find which objects (packages, views, tables, etc.) are used on which pages and shared components. Faster and more accurate than searching directly in the APEX builder.

**Prerequisite:** the Embedded Code report must be exported first — run `adt export_apex -app {ID} -only -embedded` before searching.


## Common Usage

```bash
# Search all referenced objects in app 100
adt search_apex -app 100

# Search objects on a specific page
adt search_apex -app 100 -page 1

# Search objects on multiple pages
adt search_apex -app 100 -page 1 10 20

# Search for objects matching a name pattern (LIKE syntax)
adt search_apex -app 100 -name MY_PACKAGE%

# Search for specific object types
adt search_apex -app 100 -type PACKAGE VIEW

# Combine filters
adt search_apex -app 100 -page 1 -type PACKAGE -name APP_%

# Use schema prefix for more precise matching
adt search_apex -app 100 -schema HR

# Copy referenced objects to patch scripts folder for a specific patch
adt search_apex -app 100 -patch TASK-123
```


## Flags

| Flag | Purpose |
|---|---|
| `-app {APP_ID}` | **Required.** APEX application ID to search |
| `-page {PAGE_ID(S)}` | Limit to specific page(s) (space-separated) |
| `-type {TYPE(S)}` | Limit to specific object types (space-separated, e.g. `PACKAGE VIEW TABLE`) |
| `-name {PATTERN(S)}` | Limit to object names matching pattern(s) (LIKE syntax with `%`, space-separated) |
| `-schema {PREFIX}` | Schema prefix for more precise matching (without the dot) |
| `-patch {CARD_NUMBER}` | Copy matched objects to `config/patch_scripts/{CARD}/refs/` for patching |


## How It Works

1. Scans all `embedded_code/**/*.sql` files for the specified app.
2. Searches each line for object references — either by schema prefix (`HR.MY_PACKAGE`) for precise matching, or by regex matching known repo object names.
3. Connects to the database to also find objects referenced via `APEX_APPLICATION_PAGE_REGIONS` and similar dictionary views.
4. Maps each found object to the pages and shared components where it appears.
5. Displays an overview table with columns: object name, object type, page IDs, and component count.

If `-schema` is provided, matching is more precise because it looks for `{SCHEMA}.{OBJECT_NAME}` patterns in the code. Without it, ADT matches against all known object names in the repository, which can produce false positives if object names are common words.


## Output

The command displays a table showing:

- **object_name** — the database object found in APEX code
- **type** — object type (PACKAGE, VIEW, TABLE, etc.); `?` if unknown
- **pages** — list of page IDs where the object is referenced
- **comps** — count of shared component references (non-page references)

Objects that exist in the embedded code but can't be matched to any repository file are shown as unknown (`?` type).


## Patch Integration

When `-patch TASK-123` is provided, the command goes beyond just listing objects:

1. Translates found object names to their repository file paths.
2. Copies referenced files to `config/patch_scripts/{CARD}/refs/` — sorted by dependencies.
3. Also includes any files manually placed in `config/patch_scripts/{CARD}/append/`.
4. Generates an installation script with proper object ordering.
5. Appends object grants to the script.

This is useful for creating patches that include all objects an APEX app depends on, or for manually adding objects that aren't directly referenced in APEX code (via the `append/` folder).


## Tips

- Run `adt export_apex -app {ID} -only -embedded` before searching to ensure the embedded code report is up to date.
- Use `-schema` whenever possible — it significantly improves matching accuracy.
- If an object shows type `?`, it means the object is referenced in APEX code but ADT can't find it in the repository. This could mean the object belongs to a different schema, is a built-in Oracle object, or hasn't been exported yet.
- The `-page` filter also restricts database query results, not just embedded code parsing.

# adt export_apex

Export APEX applications, components, REST services, and workspace files into the repository. Supports multiple export formats, scoping by app/workspace/group, and filtering by recent changes or developer.


## Typical Workflow

Always pass `-only` to override config defaults — this way you explicitly control which formats are exported and avoid surprises from `config.yaml` settings. Always pass `-recent 0` to skip the recent changes listing, unless the user specifically asks to see which components changed recently. Note: `-recent` only controls whether a list of recently changed components is shown before the export — the export itself always exports all components regardless of this flag.

```bash
# Export app 100 in typical formats (full + split + files + REST + readable)
adt export_apex -app 100 -only -full -split -files -files_ws -rest -readable -recent 0

# Export multiple apps
adt export_apex -app 100 200 -only -full -split -files -rest -readable -recent 0

# Export app 100, show which components changed in last 3 days (export still covers everything)
adt export_apex -app 100 -only -full -split -files -rest -readable -recent 3

# Quick split-only export
adt export_apex -app 100 -only -split -recent 0
```


## Discovering Applications

When the user wants to see which apps are available:

```bash
# Reveal all workspaces and applications
adt export_apex -reveal

# Reveal apps in a specific workspace
adt export_apex -reveal -ws MY_WORKSPACE

# Reveal apps in a specific group
adt export_apex -reveal -group MY_GROUP
```

**Common issue:** If apps don't show up, the database schema in `connections.yaml` likely doesn't match the APEX application owner. Use `-schema` to override:

```bash
adt export_apex -reveal -schema APPS
```


## Flags

### Export Formats

| Flag | Purpose |
|---|---|
| `-full` | Full monolithic application export (single SQL file per app, used for full import) |
| `-split` | Split export — individual files for each page, component, and shared component (what gets committed to Git) |
| `-readable` | Human-readable YAML/JSON formatted export for code review and search |
| `-embedded` | Embedded Code report — extracts all PL/SQL and JavaScript from APEX components into one reviewable file |
| `-rest` | REST services and modules |
| `-files` | Application binary files (JS, CSS, images) |
| `-files_ws` | Workspace binary files |
| `-only` | **Use this.** Proceed only with explicitly passed format flags, ignoring config.yaml defaults |

Typical exports include: `-full -split -files -files_ws -rest -readable`. The `-embedded` export is usually skipped to keep exports faster — only include it when the user specifically needs it.

### Scope

| Flag | Purpose |
|---|---|
| `-app {ID(S)}` | Limit to specific application(s) (space-separated: `-app 100 200 300`) |
| `-ws {WORKSPACE}` | Limit to a specific workspace |
| `-group {GROUP}` | Limit to an application group |
| `-recent {days}` | Show a list of components changed in the last N days before the export; use `0` to skip. Does **not** limit what gets exported — the export always covers all components |
| `-by {DEVELOPER}` | Filter recently changed components by developer name |

### Discovery and Versioning

| Flag | Purpose |
|---|---|
| `-reveal` | List available workspaces and applications (no export) |
| `-release {VERSION}` | Export as if using a specific APEX release version (see below) |

### Environment Overrides

| Flag | Purpose |
|---|---|
| `-schema {NAME}` | Override the database schema (useful when apps aren't showing up) |
| `-env {ENV}` | Source environment (overrides default connection) |
| `-key {PASSWORD}` | Decryption key for encrypted passwords |


## The `-only` Flag

Without `-only`, ADT merges your CLI flags with defaults from `config.yaml` (the `apex_export_*` settings). This can lead to unexpected exports. With `-only`, only the formats you explicitly pass on the command line are executed.

Always use `-only` when building commands for the user. This makes the behavior predictable and avoids the need for `-no*` negation flags.


## The `-release` Flag (APEX Upgrade Recovery)

When APEX gets upgraded, every exported file includes the new version string, causing massive diffs that bury the user's actual changes. The `-release` flag tells ADT to export as if it were the previous APEX version, so the user can isolate just their real changes.

```bash
# Export as if still on APEX 23.2
adt export_apex -app 100 -only -split -readable -release 23.2 -recent 0
```

After the user recovers their changes, they can do a clean export without `-release` to bring all files up to the new version.


## Export by Developer

The `-by` flag filters the recently changed components list by developer name. Note to the user that this is **not a reliable** way to isolate changes — APEX tracks the last modifier, not all modifiers, so if someone else touched the same component after the original developer, it won't show up.

```bash
# Show what JKVETINA changed in the last 7 days
adt export_apex -app 100 -only -split -readable -recent 7 -by JKVETINA
```


## Output Structure

```
apex/{workspace}/{app_group}/{app_owner}/{app_id}_{app_alias}/
    f{app_id}.sql                           # full export
    f{app_id}/                              # split export
        pages/
            page_0001.sql                   # split page file
            page_0001.yaml                  # readable version (same folder, renamed to match)
        shared_components/
            ...
    embedded_code/                          # embedded code report
    files/                                  # application binary files

apex/{workspace}/
    rest/                                   # REST services and modules
    workspace_files/                        # workspace-level binary files
```

Readable exports are renamed and placed in the same folder as their corresponding split files (e.g. `pages/page_0001.yaml` sits next to `pages/page_0001.sql`), not in a separate `readable/` directory. This keeps related files together for easy comparison.

The folder path includes workspace, app group, owner, and an `{app_id}_{alias}` directory. This is configurable via `config.yaml` (`path_apex` setting). ADT respects any custom subfolder organization the user creates.


## How It Works

1. Connects to the database and sets the APEX security context for the target workspace.
2. Lists matching applications (by `-app`, `-ws`, `-group` filters) and shows them in a table.
3. For each matched app:
   - Shows recently changed components (if `-recent > 0`).
   - Runs each requested export format sequentially with a progress bar and time estimates.
   - Moves exported files from temp folders to the final repository structure.
   - Stores timing data for progress estimates on next run.

All file exports happen in binary format. Encoded files are discarded by default to keep the repo clean.


## Console Output

When this command runs, it produces:

1. **Applications table** — lists matched apps with ID, alias, name, page count, last update
2. **Recent changes** — components changed in the last N days (if `-recent > 0`)
3. **Export progress** — per-format progress bars with time estimates
4. **Completion** — summary of what was exported

Always show the full console output to the user.


## Troubleshooting: Apps Not Found

If `adt export_apex -reveal` shows no apps or the wrong apps:

1. **Wrong schema** — the connection's `schema` doesn't match the APEX app owner. Try `-schema APPS` (or whatever schema owns the apps).
2. **Wrong workspace** — the connection's `workspace` setting filters results. Try `-reveal` without `-ws` to see all workspaces, or pass `-ws %` to see everything.
3. **App not in scope** — if `-app` IDs are set in `connections.yaml`, only those apps appear. Remove the `app` setting from the connection to see all.

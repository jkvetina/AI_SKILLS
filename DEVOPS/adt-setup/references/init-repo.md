# Initialize ADT Project Repository

This guide sets up a project repository for use with ADT by copying config templates and configuring `.gitignore`.

**Important:** All commands must be run from inside the project repository.

---

## Step 1: Locate the ADT Installation

Before copying files, find where ADT is cloned:

### macOS

```bash
find ~ -maxdepth 3 -name "config.py" -path "*/ADT/*" 2>/dev/null
```

### Windows

```cmd
dir /s /b C:\Users\%USERNAME%\Documents\ADT\config.py 2>nul
dir /s /b C:\Users\%USERNAME%\ADT\config.py 2>nul
```

Use the found path as `ADT_PATH` below. If not found, the user needs to install ADT first (see section 1: Installation in the main skill).

---

## Step 2: Copy Config Templates

```bash
cd /path/to/your/project
mkdir -p config

# Copy the main config file (controls export paths, APEX settings, patch behavior, etc.)
cp $ADT_PATH/config/config.yaml config/config.yaml

# Copy patch template folders (used when creating deployment patches)
cp -r $ADT_PATH/config/patch config/patch
cp -r $ADT_PATH/config/patch_scripts config/patch_scripts
cp -r $ADT_PATH/config/patch_template config/patch_template
```

Replace `$ADT_PATH` with the actual ADT install location (e.g., `~/Documents/ADT`).

**What these files do:**
- `config/config.yaml` — project-level settings: export folder structure, object types, APEX export options, patch naming, Git integration, and more. Review and adjust to your project needs.
- `config/patch/` — folder for live patches
- `config/patch_scripts/` — manual SQL scripts to include in patches
- `config/patch_template/` — templates applied before/after patch sections (init scripts, post-install checks, mview refreshes, job re-creation)

---

## Step 3: Create or Update .gitignore

Create or append to `.gitignore` in the project root. These patterns keep ADT temp files, generated configs, cached data, and minified APEX assets out of version control.

The following lines must be present in `.gitignore`:

```gitignore
/.temp.nosync
config/apex_timers.yaml
config/apex_developers.yaml
config/apex_apps.yaml
config/db_dependencies.yaml
config/commits/*.yaml

apex/workspace/rest/oracle.example.hr.sql
apex/workspace/rest/__enable_schema.sql
apex/workspace/app_groups.yaml
apex/workspace/credentials/*.sql

apex/*/files/*/*.min.*
apex/*/files/*.min.*
```

**Important:** Do not overwrite an existing `.gitignore` — append the missing lines. Check which lines are already present before adding.

If the user also wants to keep connection credentials out of version control, add:

```gitignore
config/connections.yaml
```

---

## Verification

After initialization, the project should have:

```
your-project/
    .gitignore
    config/
        config.yaml
        patch/
        patch_scripts/
        patch_template/
```

The user can now proceed to set up database connections (see section 3: Database Connections).

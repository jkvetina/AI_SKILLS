# Database Connection Setup

This guide covers creating and configuring ADT database connections. All work must be done from inside the project repository, because the connection file name and location are critical to ADT.

**Important:** Do NOT use `adt config -create` to generate connection files. Instead, copy the provided sample and edit it directly. This gives you full control over the YAML structure and avoids confusion.

## Prerequisites

Before creating a connection, ensure:
- ADT is installed and `adt config -version` works
- You have a Git project repository (the folder you'll run ADT from)
- You have database credentials (username, password)
- For OCI cloud: you have a wallet downloaded from OCI console
- For on-premise: you know the hostname and service name (or SID)

---

## Step 1: Initialize the Project Config

Before creating a connection, copy the ADT config templates into your project repo. This gives you all the configuration files you can customize per project.

```bash
cd /path/to/your/project

# Copy the main config file (controls export paths, APEX settings, patch behavior, etc.)
mkdir -p config
cp ~/Documents/ADT/config/config.yaml config/config.yaml

# Copy the connection sample as your connection file
cp ~/Documents/ADT/connections/sample.yaml config/connections.yaml

# Copy patch template folders (used when creating deployment patches)
cp -r ~/Documents/ADT/config/patch config/patch
cp -r ~/Documents/ADT/config/patch_scripts config/patch_scripts
cp -r ~/Documents/ADT/config/patch_template config/patch_template
```

Adjust the ADT path (`~/Documents/ADT`) if it was cloned elsewhere.

**What these files do:**
- `config/config.yaml` — project-level settings: export folder structure, object types, APEX export options, patch naming, Git integration, and more. Review and adjust to your project needs.
- `config/connections.yaml` — database connection details (you'll edit this next)
- `config/patch/` — folder for live patches
- `config/patch_scripts/` — manual SQL scripts to include in patches
- `config/patch_template/` — templates applied before/after patch sections (init scripts, post-install checks, mview refreshes, job re-creation)

Recommendation: add `config/connections.yaml` (or the entire `config/` folder) to `.gitignore` if you don't want credentials in version control. The other config files are safe to commit.

---

## Step 2: Edit the Connection File

Open `config/connections.yaml` and modify it based on your connection type. The three types are shown below.

---

## Connection Types

### 1. Cloud (OCI) Connection

For Oracle Cloud Infrastructure (Autonomous Database). Uses a wallet for authentication.

```yaml
DEV:                                          # your environment name, must match ADT_ENV
    db:
        hostname    :                         # leave empty for OCI cloud
        lang        :                         # optional language setting (e.g., AMERICAN_AMERICA.AL32UTF8)
        port        : 1521
        service     : INSTANCE_NAME_high      # from wallet's tnsnames.ora (_high, _medium, _low)
        sid         :
        thick       : ''                      # 'Y' or path to Instant Client for thick mode
    defaults:
        schema_apex : YOUR_SCHEMA_NAME        # your schema used/exposed in APEX
        schema_db   : YOUR_SCHEMA_NAME        # your DB schema
    schemas:
        YOUR_SCHEMA_NAME:                     # your schema name
            db:
                pwd     : ''                  # your password
                pwd!    :                     # 'Y' if you want password encrypted with ADT_KEY
                user    : YOUR_SCHEMA_NAME
                proxy   :                     # proxy user, e.g., 'USER[SCHEMA]' for proxy authentication
            export:
                ignore      : ''              # ignore objects matching these prefixes, e.g., 'ABC%,DEF%'
                prefix      : ''              # export only objects matching these prefixes
                subfolder   :                 # subfolder name for multi-schema exports
    wallet:                                   # wallet for OCI connections, delete for on-premise
        wallet      : config/Wallet_INSTANCE_NAME
        wallet_pwd  : ''                      # wallet password
        wallet_pwd! :                         # 'Y' if you want password encrypted with ADT_KEY
```

Fill in:
- `service` — from the wallet's `tnsnames.ora` (common suffixes: `_high`, `_medium`, `_low`)
- `YOUR_SCHEMA_NAME` — replace all 4 occurrences with your actual schema name
- `pwd` — your database password
- `wallet` — path to the wallet directory (relative to the project repo or absolute)
- `wallet_pwd` — wallet password from OCI console
- Leave `hostname` empty — it's not used for cloud connections

### 2. Normal (On-Premise) Connection

For on-premise databases using service name. This is the most common on-premise setup.

```yaml
DEV:                                          # your environment name, must match ADT_ENV
    db:
        hostname    : your-db-host.example.com
        lang        :                         # optional language setting
        port        : 1521
        service     : YOUR_SERVICE_NAME
        sid         :
        thick       : ''                      # 'Y' or path to Instant Client for thick mode
    defaults:
        schema_apex : YOUR_SCHEMA_NAME        # your schema used/exposed in APEX
        schema_db   : YOUR_SCHEMA_NAME        # your DB schema
    schemas:
        YOUR_SCHEMA_NAME:                     # your schema name
            db:
                pwd     : ''                  # your password
                pwd!    :                     # 'Y' if you want password encrypted with ADT_KEY
                user    : YOUR_SCHEMA_NAME
                proxy   :                     # proxy user, e.g., 'USER[SCHEMA]' for proxy authentication
            export:
                ignore      : ''              # ignore objects matching these prefixes, e.g., 'ABC%,DEF%'
                prefix      : ''              # export only objects matching these prefixes
                subfolder   :                 # subfolder name for multi-schema exports
```

Fill in:
- `hostname` — your database server hostname or IP
- `service` — the Oracle service name
- `port` — change if not the default 1521
- `YOUR_SCHEMA_NAME` — replace all 4 occurrences
- `pwd` — your database password
- **Delete the entire `wallet:` section** — it's not needed for on-premise

### 3. Legacy (On-Premise with SID) Connection

For older on-premise databases that use SID instead of service name.

```yaml
DEV:                                          # your environment name, must match ADT_ENV
    db:
        hostname    : your-db-host.example.com
        lang        :                         # optional language setting
        port        : 1521
        service     :
        sid         : YOUR_SID
        thick       : ''                      # 'Y' or path to Instant Client for thick mode
    defaults:
        schema_apex : YOUR_SCHEMA_NAME        # your schema used/exposed in APEX
        schema_db   : YOUR_SCHEMA_NAME        # your DB schema
    schemas:
        YOUR_SCHEMA_NAME:                     # your schema name
            db:
                pwd     : ''                  # your password
                pwd!    :                     # 'Y' if you want password encrypted with ADT_KEY
                user    : YOUR_SCHEMA_NAME
                proxy   :                     # proxy user, e.g., 'USER[SCHEMA]' for proxy authentication
            export:
                ignore      : ''              # ignore objects matching these prefixes, e.g., 'ABC%,DEF%'
                prefix      : ''              # export only objects matching these prefixes
                subfolder   :                 # subfolder name for multi-schema exports
```

Fill in:
- `hostname`, `port`, `sid` — your database details
- Leave `service` empty — SID is used instead
- `YOUR_SCHEMA_NAME` — replace all 4 occurrences
- `pwd` — your database password
- **Delete the entire `wallet:` section** — it's not needed

---

## Wallet Setup (OCI Cloud)

For OCI connections, you need to download and configure a wallet:

1. Go to OCI Console → Autonomous Database → DB Connection → Download Instance Wallet
2. Unzip the wallet into the project repo's config directory:
   ```
   config/Wallet_INSTANCE_NAME/
   ```
3. The wallet folder should contain: `tnsnames.ora`, `sqlnet.ora`, `cwallet.sso`, `ewallet.p12`, `keystore.jks`, `truststore.jks`, `ojdbc.properties`
4. Set the `wallet` value in `connections.yaml` to this path

Recommendation: add the entire `config/` folder to `.gitignore` to keep credentials out of version control.

---

## The `connections.yaml` File Structure

Here is the full structure with all optional sections shown:

```yaml
DEV:                                          # environment name, matches ADT_ENV
    db:
        hostname    :                         # empty for OCI cloud connections
        lang        :                         # optional language setting (e.g., AMERICAN_AMERICA.AL32UTF8)
        port        : 1521
        service     : SERVICE_NAME_high       # service name (or leave empty if using SID)
        sid         :                         # use instead of service for legacy connections
        thick       : ''                      # 'Y' or path to Instant Client for thick mode
    defaults:
        schema_apex : SCHEMA_NAME             # default schema for APEX operations
        schema_db   : SCHEMA_NAME             # default schema for DB operations
    schemas:
        SCHEMA_NAME:                          # one entry per schema you work with
            db:
                pwd     : ''                  # plain or encrypted password
                pwd!    :                     # 'Y' if password is encrypted with ADT_KEY
                user    : SCHEMA_NAME
                proxy   :                     # proxy user, e.g., 'USER[SCHEMA]'
            apex:                             # optional — only add if you need to limit APEX exports
                workspace : WORKSPACE_NAME
                app       : 100
            export:                           # optional — for filtering exported objects
                ignore      : ''              # ignore objects matching these prefixes, e.g., 'ABC%,DEF%'
                prefix      : ''              # export only objects matching these prefixes
                subfolder   :                 # subfolder name for multi-schema exports
    wallet:                                   # only for OCI cloud connections, delete for on-premise
        wallet      : config/Wallet_INSTANCE_NAME
        wallet_pwd  : ''                      # wallet password
        wallet_pwd! :                         # 'Y' if encrypted with ADT_KEY
```

Key points:
- The `wallet:` section must be **deleted entirely** for on-premise connections (not just left empty)
- The `apex:` section is optional — only add it if you need to specify a workspace or app ID
- The `export:` section fields can be left empty — they are shown so the user knows what's available
- `proxy` is for proxy authentication (e.g., `'JAN[WKSP_PHX]'`) — leave empty if not used
- Passwords with `!` suffix (e.g., `pwd!`) are encryption flags — when set to `'Y'`, the corresponding password field contains an encrypted value

---

## Connection File Locations

ADT looks for connection files in this order:
1. `{repo}/config/connections.yaml` — per-project connection file (most common, recommended)
2. `{ADT_install}/connections/{project_folder_name}.yaml` — centralized connection file, **filename must match the project repo folder name exactly**

The first match wins. For most users, the per-project file at `config/connections.yaml` is the right choice.

**Important:** If using option 2 (centralized connections), the YAML file name must match the name of your project folder. For example, if your project is at `~/Documents/MY_PROJECT/`, the file must be `~/Documents/ADT/connections/MY_PROJECT.yaml`. A mismatch means ADT won't find the connection.

---

## Multi-Schema Setup

To add more schemas, add entries under the `schemas:` section:

```yaml
DEV:
    db:
        hostname    : db.example.com
        lang        :
        port        : 1521
        service     : ORCL
        sid         :
        thick       : ''
    defaults:
        schema_apex : HR
        schema_db   : HR
    schemas:
        HR:
            db:
                pwd     : 'hr_password'
                pwd!    :
                user    : HR
                proxy   :
            export:
                ignore      : ''
                prefix      : ''
                subfolder   :
        FINANCE:
            db:
                pwd     : 'fin_password'
                pwd!    :
                user    : FINANCE
                proxy   :
            export:
                ignore      : ''
                prefix      : ''
                subfolder   : finance       # keep exports separate from HR
```

Use `subfolder` to keep exports organized when multiple schemas coexist in the same repo. Set the schema you work with most as the default in the `defaults:` section.

---

## Multiple Environments

You can have multiple environments (DEV, UAT, PROD) in the same `connections.yaml`. Just add another top-level block:

```yaml
DEV:
    db:
        hostname    : dev-db.example.com
        ...
UAT:
    db:
        hostname    : uat-db.example.com
        ...
```

Switch between them by setting the `ADT_ENV` environment variable or passing `-env` to each command.

---

## Password Encryption

ADT can encrypt passwords in `connections.yaml` using the `ADT_KEY` environment variable:

1. Set `ADT_KEY` in your shell profile (`~/.zshrc` or system environment variables):
   ```bash
   export ADT_KEY=YOUR_SECRET_KEY
   ```
2. Run any ADT command — if `ADT_KEY` is set and `pwd!` is not `'Y'`, ADT will encrypt the password on first use and set `pwd!: 'Y'`
3. To keep passwords in plain text, leave `ADT_KEY` unset or don't set the `pwd!` flag
4. Keep `ADT_KEY` consistent across team members who share the same encrypted `connections.yaml`

---

## Thick vs Thin Mode

ADT uses the Python `oracledb` driver which supports two modes:

- **Thin mode** (default) — pure Python, no Instant Client needed. Works with OCI cloud connections and most modern on-premise setups.
- **Thick mode** — uses Oracle Instant Client native libraries. Required for on-premise databases with older password compatibility (below 12c) or specific Oracle Net features.

To enable thick mode, set the `thick` value in `connections.yaml`:
- `thick: 'Y'` — auto-resolves Instant Client from `$ORACLE_HOME`
- `thick: '/path/to/instantclient'` — explicit Instant Client path

---

## Verifying a Connection

After setting up the connection file, test it:

```bash
cd /path/to/your/project
adt config
```

This connects to the database using the current environment (`ADT_ENV`) and default schema, then displays connection details. If it fails, check:
- Credentials (username/password)
- Network access to the database host
- Wallet path and password (for OCI)
- Thick mode setting (for on-premise with older password hashes)
- That the `wallet:` section is deleted for on-premise connections

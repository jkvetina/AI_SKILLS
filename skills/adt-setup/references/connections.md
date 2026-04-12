# Database Connection Setup

This guide covers creating and configuring ADT database connections. ADT stores connection details in a YAML file (`config/connections.yaml`) inside the project repository.

## Prerequisites

Before creating a connection, ensure:
- ADT is installed and `adt config -version` works
- You have database credentials (username, password)
- For OCI cloud: you have a wallet downloaded from OCI console
- For on-premise: you know the hostname and service name (or SID)

---

## Connection Types

ADT supports three connection types, detected automatically based on which arguments you provide:

### 1. Cloud (OCI) Connection

For Oracle Cloud Infrastructure (Autonomous Database). Uses a wallet for authentication.

Required arguments: `-user`, `-pwd`, `-service`, `-wallet`, `-wallet_pwd`

```bash
adt config -create \
    -user USER_NAME \
    -pwd USER_PASSWORD \
    -service SERVICE_NAME_high \
    -wallet /path/to/Wallet_INSTANCE_NAME \
    -wallet_pwd WALLET_PASSWORD
```

The `-service` value comes from the wallet's `tnsnames.ora`. Common suffixes are `_high`, `_medium`, `_low` (indicating different resource priorities).

### 2. Normal (On-Premise) Connection

For on-premise databases using service name. This is the most common on-premise setup.

Required arguments: `-user`, `-pwd`, `-hostname`, `-service`

```bash
adt config -create \
    -schema SCHEMA_NAME \
    -user USER_NAME \
    -pwd USER_PASSWORD \
    -hostname HOST_NAME \
    -service SERVICE_NAME
```

Port defaults to 1521. Override with `-port` if different.

### 3. Legacy (On-Premise with SID) Connection

For older on-premise databases that use SID instead of service name.

Required arguments: `-user`, `-pwd`, `-hostname`, `-sid`

```bash
adt config -create \
    -schema SCHEMA_NAME \
    -user USER_NAME \
    -pwd USER_PASSWORD \
    -hostname HOST_NAME \
    -sid SID_NAME
```

---

## Wallet Setup (OCI Cloud)

For OCI connections, you need to download and configure a wallet:

1. Go to OCI Console → Autonomous Database → DB Connection → Download Instance Wallet
2. Unzip the wallet into the project repo's config directory:
   ```
   config/Wallet_INSTANCE_NAME/
   ```
3. The wallet folder should contain: `tnsnames.ora`, `sqlnet.ora`, `cwallet.sso`, `ewallet.p12`, `keystore.jks`, `truststore.jks`, `ojdbc.properties`
4. Pass the wallet path to `-wallet` when creating the connection

Recommendation: add the entire `config/` folder to `.gitignore` to keep credentials out of version control.

---

## The `-create` Command — All Flags

| Flag | Description | Default |
|---|---|---|
| `-user` | Database username | (required) |
| `-pwd` | Database password | (required) |
| `-hostname` | Database host (on-premise only) | |
| `-port` | Database port | 1521 |
| `-service` | Service name | |
| `-sid` | SID (legacy on-premise only, use instead of `-service`) | |
| `-wallet` | Path to wallet directory (OCI only) | |
| `-wallet_pwd` | Wallet password (OCI only) | |
| `-schema` | Schema name (defaults to `-user` value if omitted) | |
| `-env` | Environment name (DEV, UAT, PROD...) | `$ADT_ENV` |
| `-thick` | Enable thick mode: `'Y'` for auto-resolve or path to Instant Client | |
| `-key` | Encryption key or path to key file | `$ADT_KEY` |
| `-prefix` | Export only objects matching these prefixes (comma-separated) | |
| `-ignore` | Ignore objects matching these prefixes (comma-separated) | |
| `-subfolder` | Subfolder name for exports (useful with multiple schemas) | |
| `-workspace` | APEX workspace name | |
| `-app` | APEX application ID(s) to export as default | |
| `-default` | Mark this schema as the default for APEX or DB operations | |
| `-decrypt` | Store passwords in plain text (not encrypted) | false |

---

## The `connections.yaml` File Structure

ADT creates and reads `config/connections.yaml` in the project repo. Here is the full structure:

```yaml
DEV:                              # environment name, matches ADT_ENV
    db:
        hostname    :             # empty for OCI cloud connections
        lang        :             # optional language setting
        port        : 1521
        service     : SERVICE_NAME_high
        sid         :             # use instead of service for legacy connections
        thick       : ''          # 'Y' or path to Instant Client for thick mode
    defaults:
        schema_apex : SCHEMA_NAME # default schema for APEX operations
        schema_db   : SCHEMA_NAME # default schema for DB operations
    schemas:
        SCHEMA_NAME:              # one entry per schema you work with
            db:
                pwd     : ''      # plain or encrypted password
                pwd!    :         # 'Y' if password is encrypted with ADT_KEY
                user    : SCHEMA_NAME
            apex:                 # optional, only if APEX workspace/app are set
                workspace : WORKSPACE_NAME
                app       : 100
            export:               # optional, only if prefix/ignore/subfolder are set
                ignore      : 'ABC%,DEF%'
                prefix      : ''
                subfolder   :
    wallet:                       # only for OCI cloud connections
        wallet      : /path/to/Wallet_INSTANCE_NAME
        wallet_pwd  : ''
        wallet_pwd! :             # 'Y' if encrypted
```

Key points about the YAML structure:
- The `wallet` section is automatically removed for on-premise connections
- The `apex` section is removed if workspace is empty
- The `export` section is removed if all fields (prefix, ignore, subfolder) are empty
- Passwords with `!` suffix (e.g., `pwd!`) are encryption flags — when set to `'Y'`, the corresponding password field contains an encrypted value

---

## Connection File Locations

ADT looks for connection files in this order:
1. `{repo}/config/connections.yaml` — per-project connection file (most common)
2. `{ADT_install}/connections/{project_folder_name}.yaml` — centralized connection file named after your project folder

The first match wins. For most users, the per-project file at `config/connections.yaml` is the right choice.

---

## Multi-Schema Setup

To work with multiple schemas in the same environment, run `-create` once per schema:

```bash
# First schema
adt config -create -schema HR -user HR_USER -pwd HR_PWD -hostname db.example.com -service ORCL -default

# Second schema (same environment, different subfolder)
adt config -create -schema FINANCE -user FIN_USER -pwd FIN_PWD -hostname db.example.com -service ORCL -subfolder finance
```

Each schema gets its own entry under the `schemas` section. Use `-subfolder` to keep exports organized when multiple schemas coexist in the same repo. Use `-default` on the schema you work with most — it becomes the default for `schema_apex` or `schema_db` in the `defaults` section.

---

## Multiple Environments

You can have multiple environments (DEV, UAT, PROD) in the same `connections.yaml`:

```yaml
DEV:
    db:
        hostname: dev-db.example.com
        ...
UAT:
    db:
        hostname: uat-db.example.com
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
2. When running `adt config -create`, passwords are automatically encrypted if `ADT_KEY` is set
3. Alternatively, pass `-key` directly: `adt config -create ... -key YOUR_SECRET_KEY`
4. To store passwords in plain text, add `-decrypt` to the create command
5. Encrypted passwords are marked with `pwd!: 'Y'` (or `wallet_pwd!: 'Y'`) in the YAML file

Keep `ADT_KEY` consistent across team members who share the same encrypted `connections.yaml`.

---

## Thick vs Thin Mode

ADT uses the Python `oracledb` driver which supports two modes:

- **Thin mode** (default) — pure Python, no Instant Client needed. Works with OCI cloud connections and most modern on-premise setups.
- **Thick mode** — uses Oracle Instant Client native libraries. Required for on-premise databases with older password compatibility (below 12c) or specific Oracle Net features.

To enable thick mode, either:
- Pass `-thick Y` when creating the connection (auto-resolves Instant Client from `$ORACLE_HOME`)
- Pass `-thick /path/to/instantclient` for a specific Instant Client path
- Set `thick: 'Y'` or `thick: '/path/to/instantclient'` directly in `connections.yaml`

---

## Verifying a Connection

After creating a connection, test it:

```bash
cd /path/to/your/project
adt config
```

This connects to the database using the current environment (`ADT_ENV`) and default schema, then displays connection details. If it fails, check:
- Credentials (username/password)
- Network access to the database host
- Wallet path and password (for OCI)
- Thick mode setting (for on-premise with older password hashes)

---

## Editing Connections Manually

You can also edit `config/connections.yaml` directly in any text editor. This is useful for:
- Adjusting export filters (`ignore`, `prefix`) without re-running `-create`
- Adding or removing schemas
- Changing default schemas
- Fine-tuning settings that don't have a `-create` flag

After manual edits, verify with `adt config` to ensure the YAML is valid and the connection works.

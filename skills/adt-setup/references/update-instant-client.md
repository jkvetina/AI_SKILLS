# Updating Oracle Instant Client

Oracle Instant Client provides the native Oracle client libraries needed for **thick mode** connections. Thick mode is required for on-premise databases with older password compatibility modes (below 12c) or when specific Oracle Net features are needed (like certain encryption/authentication methods).

If you only connect to OCI cloud databases via wallets (thin mode), you may not need Instant Client at all and can skip this guide.

## Check Current Version

Via ADT:

```bash
adt config -version
```

Look for the "Instant Client" line. ADT reads the version from `$ORACLE_HOME/BASIC_README`.

If that line is blank, either Instant Client is not installed or `ORACLE_HOME` is not set correctly.

## Update

Instant Client is a manual download and replace — there is no package manager update.

### Step 1: Download the New Version

**macOS (Intel/x86_64):**
https://www.oracle.com/database/technologies/instant-client/macos-intel-x86-downloads.html

**macOS (Apple Silicon/ARM64):**
https://www.oracle.com/database/technologies/instant-client/macos-arm64-downloads.html

**Windows (64-bit):**
https://www.oracle.com/database/technologies/instant-client/winx64-64-downloads.html

Download the **Basic** package. For most ADT use cases, the Basic package is sufficient.

### Step 2: Extract and Replace

#### macOS

```bash
# Back up the old version
mv ~/instantclient_19_16 ~/instantclient_19_16_old

# Extract the new version (adjust filename)
unzip instantclient-basic-macos.x64-19.XX.0.0.0dbru.zip -d ~/

# If upgrading to a different major version (e.g., 19 -> 21), update ORACLE_HOME
# in ~/.zshrc to point to the new directory name
```

#### Windows

1. Rename the old folder (e.g., `instantclient_21_10` to `instantclient_21_10_old`)
2. Extract the new zip to `C:\Users\USER_NAME\`
3. If the folder name changed (e.g., `instantclient_21_10` to `instantclient_21_15`), update environment variables:
   - `ORACLE_HOME`
   - `CLIENT_HOME`
   - `TNS_ADMIN`
   - PATH entries

### Step 3: Update Environment Variables (if path changed)

If the new version has a different folder name (e.g., upgrading from 19.16 to 21.x), update `ORACLE_HOME` and related variables.

#### macOS — edit `~/.zshrc`:

```bash
export ORACLE_HOME=~/instantclient_21_XX   # adjust to actual folder name
export DYLD_LIBRARY_PATH=$ORACLE_HOME
export LD_LIBRARY_PATH=$ORACLE_HOME
export OCI_LIB_DIR=$ORACLE_HOME
export OCI_INC_DIR=$ORACLE_HOME
```

Then reload:

```bash
source ~/.zshrc
```

#### Windows — update system environment variables:

Update `ORACLE_HOME`, `CLIENT_HOME`, `TNS_ADMIN`, and PATH to reflect the new folder name.

### Step 4: Copy Wallet and Network Config (if applicable)

If you had `tnsnames.ora`, `sqlnet.ora`, or wallet files in the old Instant Client directory, copy them to the new one:

```bash
# macOS example
cp ~/instantclient_19_16_old/tnsnames.ora ~/instantclient_21_XX/
cp ~/instantclient_19_16_old/sqlnet.ora ~/instantclient_21_XX/
```

Or if your wallet/TNS config lives in `config/` inside the project repo (which is the ADT convention), no action is needed.

### Step 5: Update DBVERSION (if major version changed)

If you changed the major Instant Client version, update the `DBVERSION` environment variable:

#### macOS — in `~/.zshrc`:

```bash
export DBVERSION=21   # was 19
```

#### Windows:

```cmd
set DBVERSION=21
```

### Step 6: Verify

```bash
adt config -version
```

Confirm the "Instant Client" line shows the new version. Then test an actual connection:

```bash
adt config
```

## Version Compatibility Notes

- **Instant Client 19.x** — supports connecting to Oracle 11.2 through 23ai. Good for legacy on-premise databases.
- **Instant Client 21.x** — supports Oracle 12.1 through 23ai. Does not connect to 11g databases. Better Windows compatibility.
- **Instant Client 23.x** — supports Oracle 19c through 23ai. Latest features but drops support for older database versions.

Choose the version that covers your oldest target database. If you only target Oracle 19c or newer, use the latest Instant Client for best feature support.

## Clean Up

Once everything works, remove the old backup:

```bash
# macOS
rm -rf ~/instantclient_19_16_old

# Windows — delete the old folder manually
```

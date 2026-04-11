# Updating SQLcl

SQLcl (SQL Developer Command Line) is Oracle's Java-based SQL tool. ADT uses it for APEX application exports (split, readable, embedded code reports). SQLcl requires Java JDK 17+.

SQLcl is installed as a subfolder inside the Oracle Instant Client directory (`$ORACLE_HOME/sqlcl/`). The Instant Client path varies per user — it is set in the `ORACLE_HOME` environment variable (e.g., `~/instantclient_19_16` on Mac, `C:\Users\USER_NAME\instantclient_21_10` on Windows). All instructions below use `$ORACLE_HOME` to refer to whatever that path is.

## Check Current Version

```bash
sql -V
```

Or via ADT:

```bash
adt config -version
```

Look for the "SQLcl" line in the output. Note the current version number (e.g., 24.1) — you will use it for the backup folder name.

## Update

SQLcl does not have a package manager — it is a manual download and replace.

### Step 1: Back Up the Current SQLcl

Rename the existing `sqlcl` folder inside `$ORACLE_HOME` to include the current version number:

#### macOS

```bash
# Example: current version is 24.1
mv $ORACLE_HOME/sqlcl $ORACLE_HOME/sqlcl24-1
```

#### Windows

```cmd
# Example: current version is 24.1
ren %ORACLE_HOME%\sqlcl sqlcl24-1
```

### Step 2: Download the Latest SQLcl

Download from: https://www.oracle.com/database/sqldeveloper/technologies/sqlcl/download/

### Step 3: Extract Into the Instant Client Directory

#### macOS

```bash
unzip sqlcl-*.zip -d $ORACLE_HOME/
# This creates $ORACLE_HOME/sqlcl/bin/sql
```

#### Windows

Extract the zip so that the `sqlcl` folder ends up at `%ORACLE_HOME%\sqlcl\`.

### Step 4: Verify

Confirm the new version and verify all ADT components:

```bash
adt config -version
```

Check that the "SQLcl" line shows the new version.

### Step 5: Clean Up (optional)

Once confirmed working, remove the backup:

#### macOS

```bash
rm -rf $ORACLE_HOME/sqlcl24-1
```

#### Windows

Delete the `%ORACLE_HOME%\sqlcl24-1` folder manually.

## Java Dependency

SQLcl requires Java 17 or higher. If you also need to update Java:

- **macOS**: `brew upgrade openjdk@17` or download from https://www.oracle.com/java/technologies/downloads/
- **Windows**: Download from https://www.oracle.com/java/technologies/downloads/ and update `JAVA_HOME` if the path changed

Check Java version:

```bash
java --version
```

## Notes

- SQLcl versions are tied to Oracle database releases (e.g., 24.1, 24.3). Newer versions generally add support for newer APEX and database features.
- ADT uses SQLcl primarily for APEX exports — if you only use ADT for database object exports or patching, SQLcl version is less critical.
- If SQLcl shows "is not recognized" or blank in `adt config -version`, make sure `$ORACLE_HOME/sqlcl/bin` is in your PATH.
- The PATH entry should point to `$ORACLE_HOME/sqlcl/bin` (Mac) or `%ORACLE_HOME%\sqlcl\bin` (Windows). Since the folder name stays `sqlcl`, the PATH does not need updating after the swap.

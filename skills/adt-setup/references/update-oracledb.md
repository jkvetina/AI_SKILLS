# Updating Python oracledb Module

The `oracledb` module is ADT's Python driver for connecting to Oracle databases. It supports both thin mode (no Instant Client needed, works with OCI cloud) and thick mode (requires Instant Client, needed for on-premise or older password modes).

## Check Current Version

```bash
python3 -c "import oracledb; print(oracledb.__version__)"
```

Or via ADT:

```bash
adt config -version
```

Look for the "Oracle DB module" line in the output.

## Update

### macOS

```bash
pip3 install oracledb --upgrade
```

### Windows

```cmd
pip install oracledb --upgrade
```

## Update All ADT Python Dependencies at Once

To update oracledb along with all other Python modules ADT uses:

### macOS

```bash
pip3 install -r ~/Documents/ADT/requirements.txt --upgrade
```

### Windows

```cmd
pip install -r C:\Users\USER_NAME\Documents\ADT\requirements.txt --upgrade
```

This upgrades everything listed in `requirements.txt`: oracledb, cryptography, GitPython, PyYAML, requests, rcssmin, rjsmin, chime, and sshtunnel.

## Verify

```bash
python3 -c "import oracledb; print(oracledb.__version__)"
```

Then test a real connection:

```bash
adt config -version
```

## Compatibility Notes

- The oracledb module is the successor to cx_Oracle. ADT uses the `oracledb` package (not cx_Oracle).
- Thin mode (default) requires no additional native libraries and works with OCI cloud wallets.
- Thick mode is activated when the `-thick` flag is passed or when connecting to databases that require it. Thick mode needs a compatible Instant Client installed.
- Major version jumps (e.g., 2.x to 3.x) may introduce breaking changes — check the [oracledb release notes](https://python-oracledb.readthedocs.io/en/latest/release_notes.html) if updating across major versions.

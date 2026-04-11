# ADT Installation on macOS

Step-by-step guide for installing ADT on macOS (Intel or Apple Silicon).

---

## Step 1: Install Git

If Git is not already installed:

```bash
# Check if Git is available
git --version
```

If not installed, either:
- Install Xcode Command Line Tools: `xcode-select --install`
- Or download [GitHub Desktop](https://desktop.github.com) (includes Git)

---

## Step 2: Clone the ADT Repository

```bash
cd ~/Documents
git clone https://github.com/jkvetina/ADT.git
```

This places ADT at `~/Documents/ADT/`. The user can choose a different location — just adjust all paths below accordingly.

---

## Step 3: Install Python 3.11

ADT requires Python 3.11 or higher (3.12, 3.13, 3.14 all work fine).

Download from: https://www.python.org/downloads/

Choose the **macOS 64-bit universal2 installer** for the latest stable release.

After installation, verify:

```bash
python3 --version
```

---

## Step 4: Install Java JDK 17+

SQLcl requires Java. Check if it is already installed:

```bash
java -version
```

If not present, download from https://www.oracle.com/java/technologies/downloads/ or install via Homebrew:

```bash
brew install openjdk@17
```

---

## Step 5: Install SQLcl

Download SQLcl from: https://www.oracle.com/database/sqldeveloper/technologies/sqlcl/download/

Extract into the Instant Client directory (SQLcl lives as a subfolder inside `$ORACLE_HOME`):

```bash
unzip sqlcl-*.zip -d ~/instantclient_19_16/
# This creates ~/instantclient_19_16/sqlcl/bin/sql
```

Verify:

```bash
~/instantclient_19_16/sqlcl/bin/sql -version
```

Note: If you installed Instant Client at a different path, adjust accordingly. The `$ORACLE_HOME` variable (set in Step 8) will point to this location.

---

## Step 6: Install Oracle Instant Client (optional)

Only needed for **thick connections** (on-premise databases, older password compatibility modes). Skip this step if connecting to OCI cloud databases only.

Download Instant Client **19.16** (Basic package) from:
https://www.oracle.com/database/technologies/instant-client/macos-intel-x86-downloads.html

For Apple Silicon, use the ARM64 version if available, or the x86_64 version under Rosetta.

Extract to home directory:

```bash
# Creates ~/instantclient_19_16/
unzip instantclient-basic-macos.x64-19.16.0.0.0dbru.zip -d ~/
```

---

## Step 7: Install Python Dependencies

```bash
pip3 install --upgrade pip
pip3 install -r ~/Documents/ADT/requirements.txt --upgrade
```

---

## Step 8: Configure Shell Environment

Edit `~/.zshrc` (or `~/.bash_profile` if using bash). Add the following block — adjust paths to match actual install locations:

```bash
# Oracle Instant Client (skip if not installed)
export ORACLE_HOME=~/instantclient_19_16
export DYLD_LIBRARY_PATH=$ORACLE_HOME
export LD_LIBRARY_PATH=$ORACLE_HOME
export OCI_LIB_DIR=$ORACLE_HOME
export OCI_INC_DIR=$ORACLE_HOME

# Add Instant Client and SQLcl to PATH
export PATH=$PATH:$ORACLE_HOME:$ORACLE_HOME/sqlcl/bin

# Database version (used by ADT for compatibility)
export DBVERSION=19

# Point 'python' to python3 if needed
alias python=python3

# ADT encryption key and default environment
export ADT_KEY=YOUR_SECRET_KEY    # for encrypting/decrypting connection passwords
export ADT_ENV=DEV                # default environment name

# ADT function — keeps your command line clean
function adt {
    script=$1
    shift
    clear; python ~/Documents/ADT/$script.py "$@"
}
```

After saving, reload the profile:

```bash
source ~/.zshrc
```

---

## Step 9: Verify Installation

Navigate to your project repository (or any folder) and run:

```bash
adt config -version
```

This should display the versions of Python, SQLcl, and other components ADT uses. If it works, the installation is complete.

If the `adt` function does not work, the user can invoke ADT directly:

```bash
python ~/Documents/ADT/config.py -version
```

---

## Notes

- The `ADT_KEY` environment variable is used to encrypt/decrypt database passwords in `config/connections.yaml`. Choose a strong key and keep it consistent across team members who share the same encrypted config.
- The `ADT_ENV` variable sets the default environment (DEV, UAT, PROD, etc.) so it does not need to be passed with every command.
- Always `cd` into the project repository before running ADT commands — ADT reads config files and writes exports relative to the current directory.

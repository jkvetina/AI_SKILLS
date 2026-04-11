# ADT Installation on Windows

Step-by-step guide for installing ADT on Windows 10/11.

---

## Step 1: Install Git

Download and install [GitHub Desktop](https://desktop.github.com) — this is the easiest option and includes Git.

Alternatively, install [Git for Windows](https://git-scm.com/download/win) standalone.

After installation, verify in Command Prompt or PowerShell:

```cmd
git --version
```

**Important:** GitHub Desktop installs Git in a nested path like:
`C:\Users\USER_NAME\AppData\Local\GitHubDesktop\app-X.X.X\resources\app\git\mingw64\bin\`

This path needs to be added to your system PATH (see Step 8).

---

## Step 2: Clone the ADT Repository

```cmd
cd C:\Users\USER_NAME\Documents
git clone https://github.com/jkvetina/ADT.git
```

This places ADT at `C:\Users\USER_NAME\Documents\ADT\`. Adjust paths below if using a different location.

---

## Step 3: Install Python 3.11

ADT requires Python 3.11 or higher (3.12, 3.13, 3.14 all work fine).

Download from: https://www.python.org/downloads/

Choose the **Windows installer (64-bit)** for the latest stable release.

During installation:
- Check **"Add python.exe to PATH"**
- Choose **"Customize installation"** and ensure pip is included

Verify:

```cmd
python --version
pip --version
```

---

## Step 4: Install Java JDK 17+

SQLcl requires Java. Check if already installed:

```cmd
java -version
```

If not present, download from: https://www.oracle.com/java/technologies/downloads/

Install and set `JAVA_HOME`:

```cmd
set JAVA_HOME=C:\Program Files\Java\jdk-17
```

---

## Step 5: Install SQLcl

Download SQLcl from: https://www.oracle.com/database/sqldeveloper/technologies/sqlcl/download/

Extract into the Instant Client directory (SQLcl lives as a subfolder inside `%ORACLE_HOME%`):

Extract the zip so that the `sqlcl` folder ends up at `C:\Users\USER_NAME\instantclient_21_10\sqlcl\`

Verify:

```cmd
C:\Users\USER_NAME\instantclient_21_10\sqlcl\bin\sql.exe -version
```

Note: If your Instant Client is at a different path, adjust accordingly.

---

## Step 6: Install Oracle Instant Client (optional)

Only needed for **thick connections** (on-premise databases, older password compatibility modes). Skip this step if connecting to OCI cloud databases only.

Download Instant Client **21.x** (Basic package) from:
https://www.oracle.com/database/technologies/instant-client/winx64-64-downloads.html

Note: On Windows, use Instant Client 21 (not 19) for better compatibility.

Extract to: `C:\Users\USER_NAME\instantclient_21_10\`

---

## Step 7: Install Python Dependencies

```cmd
pip install --upgrade pip
pip install -r C:\Users\USER_NAME\Documents\ADT\requirements.txt --upgrade
```

---

## Step 8: Configure Environment Variables

Open **System Properties > Environment Variables** (search "environment variables" in Start menu), or set them via Command Prompt.

### Option A: System Environment Variables (GUI)

Add/edit the following **System variables**:

| Variable | Value |
|---|---|
| `ORACLE_HOME` | `C:\Users\USER_NAME\instantclient_21_10` |
| `CLIENT_HOME` | `C:\Users\USER_NAME\instantclient_21_10` |
| `TNS_ADMIN` | `C:\Users\USER_NAME\instantclient_21_10` |
| `CLASSPATH` | `%ORACLE_HOME%` |
| `DYLD_LIBRARY_PATH` | `%ORACLE_HOME%` |
| `LD_LIBRARY_PATH` | `%ORACLE_HOME%` |
| `OCI_LIB_DIR` | `%ORACLE_HOME%` |
| `OCI_INC_DIR` | `%ORACLE_HOME%\sdk\include` |
| `JAVA_HOME` | `C:\Program Files\Java\jdk-17` |
| `ADT_KEY` | `YOUR_SECRET_KEY` |
| `ADT_ENV` | `DEV` |
| `GIT_PYTHON_REFRESH` | `quiet` |
| `GIT_PYTHON_GIT_EXECUTABLE` | `C:\Users\USER_NAME\AppData\Local\GitHubDesktop\app-X.X.X\resources\app\git\mingw64\bin\git.exe` |

Add to the **PATH** variable:
- `%ORACLE_HOME%`
- `%ORACLE_HOME%\sqlcl\bin`
- The Git executable directory (from GitHub Desktop or Git for Windows)

### Option B: Batch script (setenv.bat)

Create a file `setenv.bat` and run it before using ADT, or add these to your system variables:

```bat
@echo off
set LANG=en_US.UTF-8
set ORACLE_HOME=C:\Users\USER_NAME\instantclient_21_10
set CLIENT_HOME=%ORACLE_HOME%
set TNS_ADMIN=%ORACLE_HOME%
set CLASSPATH=%ORACLE_HOME%
set DYLD_LIBRARY_PATH=%ORACLE_HOME%
set LD_LIBRARY_PATH=%ORACLE_HOME%
set OCI_LIB_DIR=%ORACLE_HOME%
set OCI_INC_DIR=%ORACLE_HOME%\sdk\include
set JAVA_HOME=C:\Program Files\Java\jdk-17

set PATH=%PATH%;%ORACLE_HOME%;%ORACLE_HOME%\sqlcl\bin
set PATH=%PATH%;C:\Users\USER_NAME\AppData\Local\GitHubDesktop\app-X.X.X\resources\app\git\mingw64\bin
set GIT_PYTHON_REFRESH=quiet
set GIT_PYTHON_GIT_EXECUTABLE=C:\Users\USER_NAME\AppData\Local\GitHubDesktop\app-X.X.X\resources\app\git\mingw64\bin\git.exe

set ADT_KEY=YOUR_SECRET_KEY
set ADT_ENV=DEV
```

---

## Step 9: Create the ADT Command Alias

On Windows there is no shell function like on Mac. Instead, create a batch file `adt.bat` and place it somewhere in your PATH (e.g. `C:\Users\USER_NAME\bin\` — add this to PATH too):

```bat
@echo off
set SCRIPT=%1
shift
python C:\Users\USER_NAME\Documents\ADT\%SCRIPT%.py %1 %2 %3 %4 %5 %6 %7 %8 %9
```

This lets you run `adt config -version` from any Command Prompt.

Alternatively, if using PowerShell, add a function to your `$PROFILE`:

```powershell
function adt {
    $script = $args[0]
    $remaining = $args[1..($args.Length-1)]
    python C:\Users\USER_NAME\Documents\ADT\$script.py @remaining
}
```

---

## Step 10: Verify Installation

Open a **new** Command Prompt (to pick up environment changes) and run:

```cmd
adt config -version
```

This should display version information for Python, SQLcl, and other components. If it works, the installation is complete.

If the `adt` alias does not work, invoke ADT directly:

```cmd
python C:\Users\USER_NAME\Documents\ADT\config.py -version
```

---

## Notes

- Replace `USER_NAME` with your actual Windows username throughout.
- Replace `app-X.X.X` with the actual GitHub Desktop version directory.
- The `ADT_KEY` variable encrypts/decrypts database passwords in `config/connections.yaml`. Keep it consistent across team members sharing the same config.
- The `ADT_ENV` variable sets the default environment (DEV, UAT, PROD, etc.).
- Always `cd` into the project repository before running ADT commands — ADT reads config and writes exports relative to the current directory.
- If you encounter `zsh: no such word in event` or similar errors with passwords containing special characters like `!`, wrap passwords in single quotes.

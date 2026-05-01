# adt live_upload

Monitor a local folder for file changes and automatically upload modified files to APEX application or workspace static files. When you save a JS or CSS file in your editor, it gets minified and uploaded to APEX before you even switch to the browser.

This is a development-time tool — it runs in a continuous loop until you press Ctrl+C.


## Common Usage

```bash
# Start monitoring with defaults (app and folder from connection config)
adt live_upload

# Monitor a specific app
adt live_upload -app 100

# Monitor a custom folder
adt live_upload -app 100 -folder ./my_static_files/

# Upload to workspace files instead of app files
adt live_upload -workspace

# Show all files in the monitored folder at startup
adt live_upload -show

# Set a custom polling interval (seconds)
adt live_upload -interval 3

# Use a different schema or environment
adt live_upload -schema APPS -env DEV
```


## Flags

### Main Actions

| Flag | Purpose |
|---|---|
| `-app {APP_ID}` | Override the APEX application ID (defaults to connection config) |
| `-folder {PATH}` | Override the static files folder to monitor (defaults to app's files directory) |
| `-workspace` | Upload to workspace-level files instead of application files |
| `-interval {SECONDS}` | Polling interval between file checks (default: 1 second) |
| `-show` | List all files in the monitored folder at startup |

### Environment

| Flag | Purpose |
|---|---|
| `-schema {NAME}` | Override database schema |
| `-env {ENV}` | Source environment for connection overrides |
| `-key {PASSWORD}` | Decryption key for encrypted passwords |


## How It Works

1. Connects to the database and sets the APEX security context for the target application.
2. Determines the monitored directory — either the app's static files folder (e.g. `apex/.../files/`) or workspace files folder, based on `-workspace` flag.
3. Enters a continuous polling loop:
   - Scans all files recursively in the monitored directory.
   - Compares each file's modification timestamp against the last known timestamp.
   - For new or changed files: uploads to APEX, then minifies if applicable.
   - Sleeps for the configured interval before checking again.
4. Stops on Ctrl+C (KeyboardInterrupt).


## Automatic Minification

When a `.css` or `.js` file is uploaded, ADT automatically creates a minified version (`.min.css` / `.min.js`) alongside the original. Both the original and minified versions are uploaded to APEX. Files that already have `.min.` in the name are skipped for minification.

This means you can reference `my_script.min.js` in your APEX app and always get the latest minified version without any build step.


## Monitored Directory

By default, ADT monitors the application's static files directory within the repo structure:

```
apex/{workspace}/{group}/{owner}/{app_id}_{alias}/files/
```

With `-workspace`, it monitors the workspace-level files directory:

```
apex/{workspace}/workspace_files/
```

Override with `-folder` to monitor any arbitrary directory.


## Tips

- Keep the interval at 1 second (default) for the most responsive experience during active development.
- Use `-show` on first run to verify ADT is monitoring the right directory and files.
- If you get an error about the monitored directory not existing, export the app files first: `adt export_apex -app {ID} -only -files`.
- For workspace files (shared across apps), use `-workspace` instead of `-app`.
- The upload uses the relative file path within the monitored folder as the file name in APEX, so your folder structure is preserved.

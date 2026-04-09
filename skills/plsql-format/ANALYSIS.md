# CORE23 — PL/SQL Package Analysis

**Project:** CORE23 (Oracle APEX utility framework)
**Author:** Jan Kvetina — MIT licence
**Repository:** https://github.com/jkvetina/CORE23
**Analysed:** 5 packages (10 files: specs + bodies)
**Date of analysis:** 2026-04-05

---

## Package Inventory

| Package | Spec (lines est.) | Body (lines est.) | Purpose |
|---|---|---|---|
| `core` | ~1,200 | ~4,647 | Main utility library — APEX session, logging, items, jobs, HTTP, email |
| `core_custom` | ~130 | ~76 | Configuration constants & env-specific overrides |
| `core_lock` | ~60 | ~352 | Optimistic locking & object hash versioning |
| `core_gen` | ~40 | ~306 | Code generator (TAPI / Table API) |
| `core_reports` | ~116 | ~667 | Automated daily email reports for APEX app monitoring |

---

## Package-by-Package Summary

### `core` — The Main Library

This is the heart of the framework. It uses `AUTHID CURRENT_USER RESETTABLE`, meaning all SQL runs under the caller's privilege context and the package state can be reset programmatically (an Oracle 23c feature).

**Functional areas:**

- **Session management** — `create_session`, `attach_session`, `exit_session`, `create_security_context`. Lets background jobs and scheduled tasks operate within a fully initialised APEX session context.
- **Context & APEX app utilities** — `get_app_id`, `get_page_id`, `get_app_name`, `get_app_home_url`, `get_workspace`, `get_env`, `get_user_id`, `get_tenant_id`. These wrap `SYS_CONTEXT('APEX$SESSION', ...)` calls with safe fallback chains.
- **Page item management** — `get_item`, `set_item`, `set_page_items` (overloaded for query string or `SYS_REFCURSOR`), `clear_items`, `apply_items`. Handles the `$` wildcard shorthand for dynamic page item names.
- **Logging framework** — `log_error`, `log_warning`, `log_debug`, `log_start`, `log_end`, all routing through an internal `log__` function that writes to `core_logs`. Each variant accepts up to 20 named key/value argument pairs.
- **Error handling** — `raise_error` raises `-20990` (app_exception), `handle_apex_error` integrates with the APEX error handling API, and the package defines `app_exception`, `assert_exception`, and `bad_depth` pragma exceptions.
- **Job management** — `create_job`, `stop_job`, `drop_job`, `run_job`. Wraps DBMS_SCHEDULER with APEX session bootstrapping so jobs can call APEX APIs.
- **Date/time utilities** — overloaded `get_date`, `get_date_time`, `get_duration`, `get_timer`, `get_local_date`, `get_utc_date`, `get_time_bucket`. Includes smart multi-format parsing via `DEFAULT NULL ON CONVERSION ERROR`.
- **Constant lookup** — `get_constant` / `get_constant_num` with `RESULT_CACHE`. Reads constants from package specs dynamically via `USER_SOURCE` — useful for configuration that needs to be reachable from SQL without a session.
- **HTTP & email** — wrappers for UTL_HTTP / UTL_SMTP / APEX_MAIL.
- **DB maintenance** — `refresh_mviews`, `recalc_table_stats`, `shrink_table`.
- **Security** — `is_developer`, `is_developer_y`, `is_authorized` (delegates to APEX auth schemes).
- **APEX grid helpers** — `get_grid_action`, `get_grid_data`, `set_grid_data`.
- **Utilities** — `get_id` (SYS_GUID-based), `get_token` (numeric OTP), `get_slug`, `get_yn`, `encode_payload` / `decode_payload`, `get_icon` (Font Awesome HTML).

---

### `core_custom` — Configuration Layer

Also `AUTHID CURRENT_USER RESETTABLE`. This is the single file you edit when deploying the framework to a new environment.

Contains all environment-level constants: SMTP settings, app IDs, format masks, flag characters, developer email list, and the APEX app list to monitor. It also implements four functions (`get_env`, `get_user_id`, `get_tenant_id`, `get_sender`) that applications can override simply by editing this one package.

`get_env` extracts the environment name from the Oracle DB name using a regex (`DB_NAME` → strip prefix → result is DEV/UAT/PROD etc.). `get_user_id` uses a carefully ordered `COALESCE` chain: custom global item → APEX session user → proxy user → session user → `USER`.

---

### `core_lock` — Object Locking & Versioning

Not `AUTHID CURRENT_USER` (runs as definer). Uses `PRAGMA AUTONOMOUS_TRANSACTION` in every DML procedure so locks are committed independently of the caller's transaction.

The lock lifecycle: when a database object is compiled via a DDL trigger, `create_lock` fires. It captures the object source via `ora_sql_txt()`, computes a SHA-256 hash (`DBMS_CRYPTO.HASH_SH256`), and either creates a new row in `core_locks` or extends the existing one. Locks expire after **20 minutes** (`g_lock_length = 20/1440`); a new snapshot record is taken every **10 minutes** (`g_lock_rebook`).

Three error scenarios are enforced: another user holds an unexpired lock (→ `LOCK_TIME_ERROR`), the object's hash has changed since the lock was last seen (→ `LOCK_HASH_ERROR`), and missing user context (→ `USER_ERROR`). `unlock` sets `expire_at = NULL` on the relevant rows.

---

### `core_gen` — Code Generator (TAPI)

Generates Table API (TAPI) boilerplate: `create_tapi` writes a procedure that handles INSERT/UPDATE/DELETE for a given table, targeting either APEX interactive grids or forms. Internal helpers (`get_width`, `column_exists`, `table_where`) are restricted with `ACCESSIBLE BY (core_gen)` — good encapsulation.

---

### `core_reports` — Daily Monitoring Reports

Scans the APEX workspace for application metadata, then formats and emails HTML reports based on the `core_daily_*` database views (compile errors, invalid objects, disabled objects, missing VPD policies, broken APEX components, failed authentication attempts, etc.). Integrates with DBMS_SCHEDULER via `core.create_job`. Can send two report types: a developer-focused daily digest (`send_daily`) and an application overview (`send_apps`). HTML is assembled via `get_html_header` / `get_html_footer` wrappers around CLOB content built from views.

---

## Dependency Graph

```
core_custom  ←── core (constants read at package-init time)
     ↑
     core (body calls core.get_item, core.get_constant — circular dep at body level, allowed in Oracle)

core  ←── core_lock (uses core.raise_error, core.app_exception)
core  ←── core_gen  (uses core utilities)
core  ←── core_reports (uses core.create_job, core.log_*, etc.)
```

The circular dependency between `core` and `core_custom` at the **body** level is acceptable in Oracle (spec-to-spec cycles would cause compilation failures; body-to-body or body-to-spec is fine), but it is worth being aware of during compilation ordering — the spec files must be compiled before the bodies.

---

## Strengths

**Architecture & Design**
- Clean separation of concerns: configuration (`core_custom`) is fully isolated from logic (`core`).
- `AUTHID CURRENT_USER` on the main packages is the right default for a shared framework library — avoids privilege escalation.
- `RESETTABLE` (Oracle 23c) allows clean package state reset without a session bounce.
- `RESULT_CACHE` on `get_constant` / `get_constant_num` avoids repeated `USER_SOURCE` lookups.
- `PRAGMA AUTONOMOUS_TRANSACTION` in `core_lock` is correctly scoped to only the locking procedures, not the whole package.
- SHA-256 via `DBMS_CRYPTO` for object integrity is a solid choice.
- `ACCESSIBLE BY` in `core_gen` properly hides internal helpers.

**Error Handling**
- The `WHEN core.app_exception THEN RAISE; WHEN OTHERS THEN core.raise_error()` pattern is applied consistently across all packages, ensuring unhandled exceptions always get logged.
- Named exception constants (`app_exception_code = -20990`, `assert_exception_code = -20992`) are centralised in `core_custom` and re-exposed via `core` — avoids magic numbers scattered in application code.
- `DEFAULT NULL ON CONVERSION ERROR` (Oracle 12.2+) used in `get_date` for graceful multi-format date parsing.

**Usability**
- Overloaded `get_date` / `get_duration` / `log_*` signatures make the API ergonomic.
- The `$` wildcard in item names (`$FIELD` → `P42_FIELD`) is a nice APEX-specific convenience that reduces repetitive page-number references.
- `set_page_items` accepting both a query string and a `SYS_REFCURSOR` is practical for different call sites.

---

## Observations & Concerns

### 1. Verbosity of the 20-pair argument pattern (Medium)
Every logging function (log_error, log_warning, log_debug, log_start, log_end, raise_error) declares 20 `in_name##` / `in_value##` pairs. This is a workaround for PL/SQL having no variadic arguments and it works fine, but it makes the spec file extremely long (it's the main reason `core.spec.sql` is ~1,200 lines). Oracle 23c introduces `PARAMETERS ... DEFAULT ...` improvements and `CLOB`-based JSON arguments could be a lighter alternative for future refactoring. Not a bug, but worth flagging for maintainability.

### 2. SMTP password visible in package spec (Security / Low severity in current state)
`global_smtp_password CONSTANT VARCHAR2(128) := ''` is declared in the public spec of `core_custom`. Since the value is currently empty this is not an active problem, but if a password were placed there it would be readable by any user with `SELECT` on `USER_SOURCE` / `ALL_SOURCE`. Consider moving SMTP credentials to a database vault, an encrypted application setting, or at minimum to the **package body** (which is not exposed in `ALL_SOURCE` by default).

### 3. `core_lock.get_object` relies on DDL trigger context (Design awareness)
`get_object` uses `ora_sql_txt()` to capture the SQL text of the currently executing DDL statement. This function only returns meaningful data when called from within a DDL trigger. If `create_lock` is called outside that context, `get_object` returns NULL and `create_lock` silently returns early (via `IF rec.object_payload IS NULL THEN RETURN; END IF;`). This is intentional behaviour for that use case, but it could surprise a caller who invokes `create_lock` manually expecting it to store something.

### 4. `unlock` UNION query may produce duplicate lock_ids (Low)
The `unlock` procedure uses a `UNION ALL` of two queries over `core_locks`: the first returns active (non-expired) rows matching the filters, the second returns `MAX(lock_id)` grouped by object. In edge cases where an active lock is also the maximum lock for that object, the same `lock_id` could appear in both result sets, causing two `UPDATE` statements on the same row. The second update is idempotent so it won't corrupt data, but the `DBMS_OUTPUT.PUT_LINE('... UNLOCKED ...')` message would print twice, which could be confusing during manual operations.

### 5. `get_item_name` always returns the name regardless of existence (Minor)
The function checks `APEX_CUSTOM_AUTH.APPLICATION_PAGE_ITEM_EXISTS` but returns `v_item_name` unconditionally in both branches. The existence check result is currently not used to alter the return value. This is not harmful (APEX's `GET_SESSION_STATE` will just return NULL for non-existent items), but the check has no effect on the outcome — it reads like dead code or an incomplete refactoring.

### 6. Hardcoded developer email in `core_custom` spec (Minor)
`jan.kvetina@gmail.com` appears directly in the `g_developers` collection in the package spec. For a personal/open-source project this is completely fine, but in an enterprise fork this would need to be managed through a table or an app setting rather than recompiling the package to change the recipient list.

### 7. `get_token` uses `DBMS_RANDOM.VALUE` (Note)
`get_token` generates a numeric token using `DBMS_RANDOM.VALUE`. `DBMS_RANDOM` is a pseudo-random generator — for OTP / security tokens used in authentication flows, `DBMS_CRYPTO.RANDOMBYTES` would produce cryptographically stronger output. If `get_token` is used only for non-security purposes (e.g., short display codes) this is fine.

### 8. Missing `NOCOPY` on large `IN OUT` parameters in `core_reports`
`close_cursor(io_cursor IN OUT PLS_INTEGER)` passes a scalar so this is not relevant here. However, `set_page_items(io_cursor IN OUT SYS_REFCURSOR, ...)` passes a cursor by reference without `NOCOPY`. For `SYS_REFCURSOR` Oracle typically passes by reference anyway, but explicit `NOCOPY` makes the intent clear and avoids copy overhead if the behaviour changes.

---

## Quick-Reference: Key Public APIs

```sql
-- Session
core.create_session(in_user_id, in_app_id);
core.attach_session(in_session_id, in_app_id);

-- Items
core.get_item('P1_NAME')
core.set_item('P1_NAME', 'value');
core.set_page_items('SELECT col1, col2 FROM my_table WHERE id = :id');

-- Logging
core.log_start('param1', :p1, 'param2', :p2);
core.log_debug('MESSAGE', 'key', value);
core.log_end();
core.raise_error('MY_ERROR_CODE', 'field', :bad_value);

-- Locking (from DDL trigger)
core_lock.create_lock(in_object_owner, in_object_type, in_object_name);
core_lock.unlock(in_locked_by => core_lock.get_user());

-- Constants
core.get_constant('MY_CONST');          -- from default package
core.get_constant('MY_CONST', 'MY_PKG', 'OWNER');

-- Jobs
core.create_job('JOB_NAME', 'BEGIN my_proc; END;', in_app_id => :app_id);
```

---

## Summary

CORE23 is a well-structured, mature APEX framework library. The separation of `core_custom` from `core` is a sound design decision that makes the framework straightforward to adopt without modifying the main package. The locking mechanism in `core_lock` is particularly thoughtful — time-based expiry combined with SHA-256 hash comparison gives real protection against concurrent object edits. The main maintenance burden is the verbose 20-pair argument pattern in logging APIs, which inflates the spec file significantly.

The concerns raised above are mostly low-severity design observations rather than bugs. The codebase shows consistent patterns and good exception discipline throughout.

# PL/SQL Formatting: plsql-formatter vs oracle-db-skills Comparison

**Date:** 2026-04-09
**Purpose:** Reference for building a dedicated PL/SQL code quality skill focused on naming conventions, patterns, and anti-patterns.

---

## What Each Skill Is

**plsql-formatter** — A formatting and visual style guide, laser-focused on how PL/SQL code should *look*: indentation, alignment, case, whitespace, comment style, section separators. Extracted from a production Oracle PL/SQL codebase. Very opinionated and specific.

**oracle-db-skills** (krisrice/oracle-db-skills) — A broad reference library of 126+ topic guides. Formatting-adjacent content lives mainly in `plsql-code-quality.md` and `pl-sql-best-practices.md`, approaching formatting from a Trivadis/industry standard angle.

---

## Key Differences

### 1. Variable Naming Prefixes — Direct Conflict

| Concept | plsql-formatter | oracle-db-skills (Trivadis) |
|---|---|---|
| Local variables | `v_` | `l_` |
| Parameters | `in_`, `out_`, `io_` | `p_` (for all directions) |
| Package globals | `g_` | `g_` (same) |
| Local constants | `c_` | `c_` (same) |
| Global constants | (not specified) | `gc_` |
| Records | `rec` (no prefix) | `r_` |
| Exceptions | (not specified) | `e_` |
| Types | (not specified) | `t_` |
| Cursors | (not specified) | `cur_` or `c_` |

The `in_`/`out_`/`io_` parameter convention is more descriptive about data flow direction than the generic `p_`.

### 2. Indentation — Conflict

plsql-formatter mandates **4 spaces**. oracle-db-skills GUIDELINES.md proposes **2-space indent** (SQLFluff defaults, ML-training rationale). plsql-code-quality says "2 or 4 spaces; be consistent."

### 3. Comma Style

plsql-formatter uses **trailing commas** throughout. oracle-db-skills GUIDELINES.md proposes **leading commas** for better git diffs. plsql-formatter is silent on leading vs trailing as a formal rule.

### 4. Package Naming Convention — Conflict

oracle-db-skills uses `_pkg` suffix (`order_mgmt_pkg`). plsql-formatter doesn't use any suffix — packages are named `core`, `core_lock`, etc.

### 5. Object Naming Conventions — oracle-db-skills is more comprehensive

oracle-db-skills covers naming for tables, triggers (`_trg`), sequences (`_seq`), indexes (`idx_`), types (`t_`), exceptions (`e_`). plsql-formatter focuses exclusively on variables/parameters and doesn't prescribe object naming.

### 6. Comment Style — plsql-formatter is stricter

plsql-formatter explicitly bans `/* */` block comments — only `--` allowed. oracle-db-skills doesn't address this; its examples use `--` but never formally prohibits block comments.

### 7. Vertical Alignment — plsql-formatter is far more detailed

Extensive rules for aligning parameters, `:=` assignments, `=>` named parameters, column aliases in SELECT, `=` in WHERE clauses. oracle-db-skills mentions alignment briefly (G-2130) but not at the same column-level detail.

### 8. Section Separators — Unique to plsql-formatter

The bare `--` as a visual separator, the `-- / -- comment / --` subprogram header pattern, and the "max 5 statements per block" rule are entirely absent from oracle-db-skills.

### 9. Code Quality / Anti-patterns — oracle-db-skills is far richer

Covers: `WHEN OTHERS THEN NULL`, `SELECT *`, magic numbers, implicit conversions, autonomous transaction abuse, hardcoded schemas, cyclomatic complexity metrics, max procedure length guidelines, static analysis tooling (PL/SQL Cop, SonarQube). plsql-formatter doesn't cover anti-patterns or code quality — it's purely visual formatting.

### 10. Performance Patterns — Only in oracle-db-skills

BULK COLLECT/FORALL, NOCOPY, context switches, cursor management, pipelined functions.

---

## Recommendations for New Code Quality Skill

**Don't merge with plsql-formatter — they serve different purposes.** plsql-formatter is a formatting rulebook; the new skill should cover naming conventions, patterns, and anti-patterns.

### Things to pull from oracle-db-skills into the new skill:
- Object naming conventions (tables, triggers, sequences, indexes, types, exceptions, cursors)
- Anti-patterns reference (WHEN OTHERS THEN NULL, SELECT *, magic numbers, implicit conversions, autonomous transaction abuse, hardcoded schemas)
- Code review checklist (correctness, performance, security, maintainability, testing)
- Cyclomatic complexity guidelines
- Max procedure length / nesting depth guidelines
- Static analysis tooling references (PL/SQL Cop, SonarQube, SQL Developer Code Analysis)

### Things to adapt from plsql-formatter conventions (keep consistency):
- Use `v_` for locals (not `l_`), `in_`/`out_`/`io_` for params (not `p_`), `g_` for globals
- Add missing prefixes: decide on types, exceptions, cursors, global constants

### Things unique to the new skill (not in either source):
- Pattern catalog: TAPI, autonomous logging, pipelined functions, object types
- When to use AUTHID CURRENT_USER vs DEFINER
- ACCESSIBLE BY patterns (12.2+)
- Safe dynamic SQL patterns (bind variables, DBMS_ASSERT)
- Package cohesion and coupling guidelines
- Connection pooling + package state pitfalls

### Source files in oracle-db-skills to reference:
- `skills/plsql/plsql-code-quality.md` — naming, anti-patterns, review checklist, static analysis
- `skills/plsql/plsql-package-design.md` — package architecture, spec vs body, overloading, forward declarations
- `skills/sql-dev/pl-sql-best-practices.md` — BULK COLLECT/FORALL, exception handling, cursor management, NOCOPY
- `skills/plsql/plsql-error-handling.md` — exception hierarchy, FORMAT_ERROR_BACKTRACE
- `skills/plsql/plsql-security.md` — AUTHID, injection vectors, DBMS_ASSERT
- `skills/plsql/plsql-patterns.md` — TAPI, autonomous logging, pipelined functions

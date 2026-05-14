---
name: exasol-migrate
description: Migrate a schema from an external RDBMS into Exasol using the studio backend's IMPORT FROM JDBC machinery. Creates an Exasol-side CONNECTION object, runs per-table IMPORTs via the existing migration scripts in EXA_DB_MIGRATION, and tracks results in JOB_LOG / JOB_DETAILS (Snowflake script only; other vendors don't auto-log). 17 vendor migration scripts shipped in studio/backend/fixtures/migration-scripts/ (azure_sql, bigquery, db2, exasol, mariadb, mysql, netezza, oracle, postgres, redshift, s3, sap_hana, snowflake, sqlserver, teradata, vectorwise, vertica) plus 3 utility scripts (query_wrapper, delta_import_on_primary_keys, restore_stage_to_target). **Per-vendor signatures are not normalized** — separate ongoing studio-team project. Treat `snowflake_to_exasol` as the canonical example; check `references/flow.md` "Per-vendor signatures vary" + the live script source before invoking another vendor.

preconditions:
  - jdbc_driver_present:
      doc: "JDBC driver jar for the source vendor is uploaded to the runtime-appropriate driver location (Docker: /exa/jdbc/<DRIVER_NAME_UPPER>/; native local: ~/.exanano/jdbc/<DRIVER_NAME_UPPER>/; production: BucketFS path). The studio backend's services/drivers.py knows the canonical jar name for each preset; services/bucketfs.py:_preferred_source_dir() picks the placement per runtime mode."
      check: |
        # Bash — checks the Docker-container path (most common for /oneshot-tour):
        # docker exec <container> ls /exa/jdbc/<DRIVER_NAME_UPPER>/ | grep -c '\.jar$'
        # Returns >=1 when satisfied.
      satisfied_by: exasol-nano-local   # places the driver, then restart not needed (JVM scans on each adapter call)
  - migration_scripts_installed:
      doc: "EXA_DB_MIGRATION schema contains the vendor-specific migration UDF (e.g., snowflake_to_exasol, postgres_to_exasol). Installed from studio/backend/fixtures/migration-scripts/ by scripts_library.py."
      check: |
        # SQL:
        # SELECT 1 FROM SYS.EXA_ALL_SCRIPTS
        # WHERE SCRIPT_SCHEMA = 'EXA_DB_MIGRATION'
        #   AND SCRIPT_NAME = '<vendor>_TO_EXASOL';
      satisfied_by: null   # one-time install; studio backend boot-installs all migration scripts. Manual: run scripts_library.install_fixture("migration-scripts").
  - source_credentials_stored:
      doc: "Source credentials are stored in the studio's local SQLite (CONNECTION_PROFILES table, encrypted via STUDIO_SESSION_SECRET) OR supplied directly to the skill call as parameters. The skill never writes plaintext passwords to disk."
      check: |
        # Python (via studio backend) OR direct parameter pass-through.
        # No SQL check — credentials live client-side until a CREATE CONNECTION is fired.
      satisfied_by: null   # user supplies via studio UI, env var, or direct parameter

provides:
  - schema_imported:
      doc: "Target Exasol schema exists and contains the migrated tables. Per-table results land in EXA_DB_MIGRATION.JOB_LOG + JOB_DETAILS (when EXECUTE mode + logging enabled)."
      verify: |
        # SQL — table count check:
        # SELECT COUNT(*) FROM SYS.EXA_ALL_TABLES WHERE TABLE_SCHEMA = '<TARGET_SCHEMA>';
        # SQL — run audit (when logging enabled):
        # SELECT RUN_ID, SCRIPT_NAME, STATUS, START_TIME, END_TIME
        # FROM EXA_DB_MIGRATION.JOB_LOG ORDER BY RUN_ID DESC LIMIT 5;

parameters:
  required:
    - source_type:
        doc: "Source-vendor key matching a row in studio's DRIVER_PRESETS. Valid values from services/drivers.py: snowflake, postgresql, oracle, sqlserver, redshift, teradata, mysql, mariadb, bigquery, azure_sql, db2, sap_hana, netezza, vectorwise, vertica, exasol, s3. Determines which migration script + driver jar."
    - source_schema:
        doc: "Source schema name (vendor-specific format: Snowflake DB.SCHEMA, Postgres schema, etc.). Passed to the vendor migration script."
    - target_schema:
        doc: "Exasol schema name. Created by the migration script if missing."
  optional:
    - target_table:
        default: null
        doc: "Optional single-table migration. If null, the migration script enumerates all tables in source_schema."
    - migration_schema:
        default: EXA_DB_MIGRATION
        doc: "Schema housing the migration UDFs. Default matches studio backend convention. Rename in lockstep with scripts_library.py FIXTURE_DIRS."
    - connection_name:
        default: null
        doc: "Name for the Exasol CONNECTION object the skill creates. Default: <SOURCE_TYPE_UPPER>_MIGRATE_<TIMESTAMP>. The skill DROPs the connection after migration completes for credential hygiene."
    - jdbc_url:
        default: null
        doc: "Explicit JDBC URL. Required when source_type is not in DRIVER_PRESETS OR when corporate proxy / PrivateLink overrides the preset's default URL template."
    - source_user / source_password:
        doc: "Source credentials. Skill prefers reading from studio's CONNECTION_PROFILES SQLite store; direct params override. Plaintext password is passed to the CREATE CONNECTION statement which Exasol stores encrypted. Never logged."
    - execute_mode:
        default: EXECUTE
        doc: "DEBUG (print generated SQL only, do not run) | EXECUTE (run + log to JOB_LOG/JOB_DETAILS) | EXECUTE_NOLOG (run without log tables). **Only `snowflake_to_exasol` and `restore_stage_to_target` expose this parameter** — the other 17 vendor scripts have no EXECUTION_MODE and run unconditionally on EXECUTE SCRIPT. Live-verified 2026-05-14. For non-Snowflake vendors, use `POST /api/import/jdbc/preview` for a pure dry-run instead — it returns the CREATE CONNECTION + IMPORT SQL without executing."
    - sample_only:
        default: false
        doc: "If true, append `WHERE ROWNUM <= 10000` (or vendor equivalent) to each STATEMENT. Used for fast structural verification."

estimated_impact:
  v01_full_run: "2–10 min depending on source size. No Exasol-side destructive ops without explicit DROP. Migration script logs to JOB_LOG/JOB_DETAILS for audit."
  per_table_overhead: "~1 sec CONNECT setup + IMPORT throughput (typically 50–200 MB/sec wire, vendor-dependent)."
---

# Exasol Migrate Skill

Trigger when the user asks to **migrate**, **import**, **load**, **copy**, **bring in**, **pull**, or **federate** a schema/database/tables from an external RDBMS into Exasol. Also when an orchestrator skill declares a `schema_imported` precondition with a `source_type` parameter.

## What this skill is

A thin orchestration layer over the studio backend's existing migration machinery. The studio already implements:

- **Driver presets** (`studio/backend/services/drivers.py`) — vendor → driver jar name + Java class + URL prefix mapping for 7 JDBC vendors plus extended migration scripts for 9 more
- **Migration UDFs** (`studio/backend/fixtures/migration-scripts/<vendor>_to_exasol.sql`) installed into `EXA_DB_MIGRATION` schema — one Lua/SQL script per source system that generates the IMPORT statements
- **Driver placement** (`studio/backend/services/bucketfs.py:_preferred_source_dir()`) — runtime-aware jar placement: Docker container, native local, configured filesystem, or BucketFS
- **Connection management** (`studio/backend/services/connection_profiles.py`) — local SQLite (`studio/backend/data/sqlcube-studio.sqlite3`) with Fernet-encrypted password storage
- **Import builder** (`studio/backend/routers/imports.py:_build_import_sql()`) — canonical `IMPORT INTO ... FROM JDBC AT <connection_name> STATEMENT <source_sql>` shape
- **Job logging** — migration scripts auto-create `JOB_LOG` + `JOB_DETAILS` tables in the migration schema; per-table run audit

This skill teaches an agent how to invoke that machinery cleanly, OR (for headless / non-studio contexts) reproduce it via plain SQL.

What this skill is NOT:

- Not a separate migration engine. The studio backend's migration scripts are canonical; this skill calls them.
- Not a transformation tool. Imports source rows 1:1. Type mapping is per the source script's logic.
- Not a CDC tool. Each invocation is a point-in-time snapshot.

## Routing algorithm

| Phase | Reference | When |
|---|---|---|
| End-to-end flow | `references/flow.md` | Always — the IMPORT FROM JDBC pipeline using studio machinery |
| Driver presets table | `references/driver-presets.md` | When picking the source_type or troubleshooting jar placement |
| Snowflake specifics | `references/snowflake.md` | source_type=snowflake (largest vendor reference) |
| CSV / S3 fallback | `references/csv-fallback.md` | When JDBC blocked; goes through s3_to_exasol script |

## Supported source systems

Per `studio/backend/routers/imports.py` and `services/drivers.py`:

**With DRIVER_PRESETS (recommended path):**
- `snowflake` (snowflake-jdbc-3.22.0.jar)
- `postgresql` (postgresql-42.7.5.jar)
- `oracle` (ojdbc10.jar)
- `sqlserver` (mssql-jdbc-12.8.1.jre11.jar — JRE 11 default)
- `redshift` (redshift-jdbc42-2.1.0.32.jar)
- `teradata` (terajdbc4_20.jar)
- `mysql` / `mariadb` (MySQL Connector/J — see preset)

**Migration script only (BYO driver):**
- `bigquery`, `azure_sql`, `db2`, `sap_hana`, `netezza`, `vectorwise`, `vertica`, `exasol`-to-`exasol`, `s3` (CSV/Parquet)

A migration script exists for every listed system in `studio/backend/fixtures/migration-scripts/`.

## Step 0: Preconditions

Three checks before any DDL:

1. **JDBC driver placed.** Path depends on runtime mode (Docker / native / production). For `exanano-sqlcube` (the most common case), `/exa/jdbc/<DRIVER_NAME_UPPER>/`. Skill defers to `exasol-nano-local/jdbc-drivers.md` for placement mechanics.

2. **Migration script installed.** Lives in `EXA_DB_MIGRATION` schema. Studio backend auto-installs from `fixtures/migration-scripts/` on first connection. Manual install via `scripts_library.install_fixture("migration-scripts")` or by `EXECUTE SCRIPT EXA_DB_MIGRATION.<script>` after copying SQL files.

3. **Source credentials available.** Three accepted sources:
   - **Studio CONNECTION_PROFILES** (recommended in studio context) — read at runtime via `services/connection_profiles.py`
   - **Direct skill parameters** (`source_user`, `source_password`) — for headless / CI invocations
   - **Pre-existing Exasol CONNECTION** — user already ran `CREATE CONNECTION` and passes the connection name via `connection_name` parameter; skill skips its own connection creation

## Pipeline

```
1. Verify preconditions
2. Resolve driver preset → driver_name + jar_name + url_prefix
3. Build JDBC URL                          (preset template + source params, or jdbc_url override)
4. CREATE CONNECTION <name> TO <url> USER <u> IDENTIFIED BY <p>
5. Test the connection                    (IMPORT-with-SELECT-1 round-trip)
6. Invoke the vendor migration script:
   EXECUTE SCRIPT EXA_DB_MIGRATION.<vendor>_TO_EXASOL (
       <source_schema>, <target_schema>, <connection_name>,
       <execute_mode>, ...
   )
7. Watch JOB_LOG / JOB_DETAILS for run progress
8. Verify schema_imported provides
9. DROP CONNECTION <name>                  (credential hygiene)
10. Hand back to caller
```

The migration script handles per-table enumeration, type mapping, IMPORT generation, and result logging internally. The skill is mostly a wrapper that picks the right script and supplies the right parameters.

## Conventions

- **Connection objects are ephemeral.** Skill creates one CONNECTION per migration run, named `<SOURCE_TYPE_UPPER>_MIGRATE_<TIMESTAMP>`, DROPs it on completion. Don't reuse long-lived connections — Exasol stores them encrypted but the credential refresh story is awkward.
- **EXECUTE mode is default — but only on Snowflake / restore_stage_to_target.** Those two scripts expose `EXECUTION_MODE` as a parameter. DEBUG returns the per-table SQL as a result set instead of running it. **The other 17 vendor scripts have no EXECUTION_MODE** — they run unconditionally on `EXECUTE SCRIPT`. For dry-running a non-Snowflake migration: drive `POST /api/import/jdbc/preview` instead (returns the masked CREATE CONNECTION + IMPORT SQL with no execution).
- **DEBUG is not a pure dry-run even on Snowflake.** Live-verified 2026-05-14: `EXECUTION_MODE='DEBUG'` still queries the source catalog to enumerate databases / schemas / tables. Without a working source CONNECTION the script errors with `Error on getting db list from Snowflake:` before reaching the result-set generation. To dry-run an offline migration: use the `/jdbc/preview` endpoint (pure, no source roundtrip) instead.
- EXECUTE_NOLOG skips JOB_LOG/JOB_DETAILS tables (saves a few KB; loses audit). Snowflake-only.
- **Connection name is identifier-quoted.** `_ident()` in `routers/imports.py` shows the studio's quoting convention. Names with mixed case must round-trip through `"name"` quoting consistently.
- **Source statement is fully under user control via the migration script.** Studio's migration scripts have well-defined parameters; this skill respects them rather than emitting bespoke SQL.

## When to invoke

- `/oneshot-tour` orchestrator declares `schema_imported` precondition with `source_type` parameter.
- Developer asks "load TPCH from Snowflake into the local Nano."
- CI smoke harness needs a fixture schema staged from an external source.
- Repeat sync: re-run with same parameters to re-import (idempotent if migration script supports it; check vendor's script docs).

## Related skills

- **exasol-nano-local** — satisfies `jdbc_driver_present` by placing the driver jar in the runtime-appropriate path.
- **exasol-optimize** — natural next step. Migrated schemas don't have declared PKs/FKs by default; optimize discovers them.
- **exasol-semantic-layer** — runs after optimize when the caller wants a cube on the migrated data.
- **oneshot-tour** — orchestrator that chains nano-local → migrate → optimize → semantic-layer → dashboard.

## Limitations

- **Snapshot, not CDC.** Each run is a point-in-time copy. For incremental updates, the migration scripts include `delta_import_on_primary_keys.sql` — but it requires source-side PKs and a delta column.
- **No DDL translation beyond column types.** Views, procedures, triggers, vendor-specific SQL not migrated.
- **Studio-context credential storage is local-only.** `CONNECTION_PROFILES` SQLite lives in `studio/backend/data/`. Not synced across users. Production / CI flows must pass credentials per-call.
- **Driver jar availability is the user's responsibility.** `exasol-nano-local` doesn't auto-download; the studio backend's `routers/drivers.py` provides upload endpoints that the agent can call when running against the studio.
- **JDBC URL templates vary by vendor.** When the preset's default doesn't match (proxy, PrivateLink, custom auth), pass `jdbc_url` explicitly.

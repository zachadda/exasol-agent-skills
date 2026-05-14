# Migration flow (vendor-agnostic)

The shared pipeline the skill follows against any source system in `studio/backend/fixtures/migration-scripts/`. Vendor-specific knobs are in `references/<vendor>.md`; read this first.

## Two ways to drive the flow

When the studio FastAPI backend is reachable, prefer its HTTP endpoints — they wrap the SQL with credential storage, driver placement, and progress polling already wired in:

| Endpoint | Purpose |
|---|---|
| `GET  /api/import/driver-catalog` | List `DRIVER_PRESETS` (source_type → jar name / Java class / URL prefix) |
| `POST /api/import/jdbc/save-connection` | Persist credentials into the studio's local SQLite (Fernet-encrypted) under a profile name |
| `POST /api/import/jdbc/test-config` / `/test-connection` | Validate a profile or a freshly-saved connection without committing anything |
| `POST /api/import/jdbc/preview` | Generate the IMPORT SQL for review without running it. **Pure dry-run** — does NOT hit the source. Returns `{connection_sql_masked, import_sql, connection_name, jdbc_url}`. Use this when you want a no-side-effects preview on any vendor (including non-Snowflake where the script itself has no DEBUG mode). |
| `POST /api/import/jdbc/run-import` | Execute the generated IMPORT batch and stream progress |
| `GET  /api/import/jdbc/databases` / `/schemas` | List remote databases / schemas through an existing CONNECTION (uses `IMPORT FROM JDBC STATEMENT '<vendor catalog query>'`) |
| `POST /api/import/snowflake/preview-migration` / `/run-sql` | Snowflake-specific flow that calls `EXA_DB_MIGRATION.SNOWFLAKE_TO_EXASOL` with EXECUTION_MODE=DEBUG or EXECUTE |
| `GET  /api/import/migration-scripts` / `/installed` | Inspect available migration scripts + which are installed |
| `POST /api/import/migration-scripts/install` | Install or re-install a migration script from `fixtures/migration-scripts/`. Body field is `filenames` (array), not `filename` (singular) — e.g. `{"filenames":["snowflake_to_exasol.sql"]}`. Returns `{success, message, installed:[...], errors:[...]}`. |
| `POST /api/import/cancel-running` | Interrupt a long-running import |
| `POST /api/import/artifacts/upload` | Upload BucketFS artifacts (driver jars, CSV/Parquet sources) used by the import |

Fall back to direct SQL (below) when the FastAPI backend isn't reachable. The SQL is the same machinery the endpoints wrap — see the canonical-source list and the `CREATE CONNECTION` pattern that follows.

Canonical source code:
- IMPORT statement builder: `studio/backend/routers/imports.py:_build_import_sql()`
- CONNECTION test: `_build_connection_test_sql()` (same file)
- Driver presets: `studio/backend/routers/imports.py:DRIVER_PRESETS`
- Driver placement: `studio/backend/services/bucketfs.py:_preferred_source_dir()`
- Credential storage: `studio/backend/services/connection_profiles.py`
- Migration scripts: `studio/backend/fixtures/migration-scripts/<vendor>_to_exasol.sql`

## The CREATE CONNECTION pattern

The studio backend's flow is **not** "inline credentials in every IMPORT statement." It's:

1. **Create a named Exasol CONNECTION** with the JDBC URL + credentials
2. **Migration script reads from that CONNECTION** by name when generating per-table IMPORTs
3. **DROP the CONNECTION** when done (credential hygiene)

```sql
-- 1. Create
CREATE OR REPLACE CONNECTION "SNOWFLAKE_MIGRATE_20260513"
  TO 'jdbc:snowflake://acct.snowflakecomputing.com/?warehouse=COMPUTE_WH'
  USER 'svc_acct'
  IDENTIFIED BY '<password>';

-- 2. Invoke migration script (creates and executes IMPORTs internally)
EXECUTE SCRIPT EXA_DB_MIGRATION.SNOWFLAKE_TO_EXASOL(
    'SNOWFLAKE_MIGRATE_20260513',     -- CONNECTION_NAME
    FALSE,                            -- DB2SCHEMA (keep db.schema.table flat → schema.table)
    'ACME_DEMO',                      -- DB_FILTER (Snowflake DB)
    'ADVENTUREWORKS',                 -- SCHEMA_FILTER
    'ADVENTUREWORKS',                 -- TARGET_SCHEMA (empty = use source name)
    '%',                              -- TABLE_FILTER (all tables)
    TRUE,                             -- IDENTIFIER_CASE_INSENSITIVE (uppercase)
    'EXECUTE',                        -- EXECUTION_MODE
    'AUTO',                           -- PARALLEL_CONNECTIONS (75% of cluster VCPUs)
    'EXA_DB_MIGRATION'                -- LOGGING_SCHEMA
);

-- 3. Drop the connection
DROP CONNECTION "SNOWFLAKE_MIGRATE_20260513";
```

Why a named CONNECTION over inline credentials:

- **Cleaner audit trail.** `SYS.EXA_DBA_CONNECTIONS` lists active connections.
- **Credentials encrypted at rest.** Exasol stores connection passwords encrypted with the cluster master key.
- **Reuse across multiple IMPORTs.** Migration scripts issue many IMPORTs (one per source table); each references the same connection by name.
- **Match studio convention.** `routers/imports.py:_build_import_sql()` exclusively uses `AT <connection_name>`.

## Pipeline steps

### 1. Resolve driver preset

Look up `source_type` in `studio/backend/routers/imports.py:DRIVER_PRESETS`. Each row:

```python
DriverPreset(
    source_type="snowflake",
    driver_name="SNOWFLAKE",                                    # Used in IMPORT FROM JDBC DRIVER='<driver_name>' (legacy) or just for jar lookup
    driver_main="net.snowflake.client.jdbc.SnowflakeDriver",
    prefix="jdbc:snowflake:",                                   # URL stem
    versions=[
        DriverVersion(version="3.22.0", jar_name="snowflake-jdbc-3.22.0.jar", is_recommended=True),
    ],
)
```

If `source_type` is not in DRIVER_PRESETS but IS in `fixtures/migration-scripts/` (bigquery, azure_sql, db2, sap_hana, etc.), the user must supply their own driver jar + `jdbc_url`.

### 2. Verify driver jar present

For Docker-container mode (`EXA_NANO_CONTAINER_NAME` set):

```bash
docker exec exanano-sqlcube ls /exa/jdbc/<DRIVER_NAME_UPPER>/ | grep '\.jar$'
```

Other modes per `services/bucketfs.py:_preferred_source_dir()`:

| Mode | Path |
|---|---|
| Docker container | `/exa/jdbc/<DRIVER_NAME_UPPER>/` |
| Native local | `~/.exanano/jdbc/<DRIVER_NAME_UPPER>/` |
| Local filesystem | `<configured-root>/<source_type>/` |
| BucketFS | `<exa_bucketfs_url>/<import_path>/<source_type>/` |

Driver casing: ExaNano (container) mode uppercases. Don't mix.

### 3. Build JDBC URL

Composition:
- If `jdbc_url` parameter supplied → use verbatim
- Else → `preset.prefix + vendor-specific suffix`

Snowflake example:
```
jdbc:snowflake://${SNOWFLAKE_ACCOUNT}.snowflakecomputing.com/
  ?warehouse=${SNOWFLAKE_WAREHOUSE}
  &db=${SOURCE_DB}
  &schema=${SOURCE_SCHEMA}
```

(Single line at exec time; readability formatted here.)

### 4. CREATE CONNECTION

```sql
CREATE OR REPLACE CONNECTION "<CONNECTION_NAME>"
  TO '<jdbc_url>'
  USER '<source_user>'
  IDENTIFIED BY '<source_password>';
```

Connection name convention: `<SOURCE_TYPE_UPPER>_MIGRATE_<UNIX_TIMESTAMP>`. Identifier-quoted so case-sensitive names round-trip.

Credentials source priority:
1. Direct skill parameters (`source_user`, `source_password`)
2. Studio's local SQLite (`connection_profiles.py:get_profile(name)`) — Fernet-encrypted, key derived from `STUDIO_SESSION_SECRET`
3. Halt with credential-missing error — never prompt on stdout

### 5. Test the connection

Per `routers/imports.py:_build_connection_test_sql()`:

```sql
SELECT * FROM (
  IMPORT FROM JDBC AT "<CONNECTION_NAME>"
  STATEMENT 'SELECT CURRENT_DATABASE(), CURRENT_SCHEMA(), CURRENT_WAREHOUSE()'
) t;
```

Returns one row when connection is alive. Vendor-specific column references — adapt the STATEMENT per source dialect:

- Snowflake: `CURRENT_DATABASE(), CURRENT_SCHEMA(), CURRENT_WAREHOUSE()`
- Postgres: `SELECT current_database(), current_schema()`
- Oracle: `SELECT USER FROM DUAL`
- SQL Server: `SELECT DB_NAME(), SCHEMA_NAME()`

Failure → halt before invoking the migration script.

### 6. Invoke the vendor migration script

```sql
EXECUTE SCRIPT EXA_DB_MIGRATION.<VENDOR>_TO_EXASOL(
    '<connection_name>',
    <vendor-specific params per references/<vendor>.md>,
    '<execution_mode>',
    <parallel_connections>,
    '<logging_schema>'
);
```

The script:
- Connects via the named CONNECTION
- Enumerates source tables matching `SCHEMA_FILTER` + `TABLE_FILTER`
- Generates per-table IMPORT statements using `_build_import_sql()` shape (`IMPORT INTO "schema"."table" FROM JDBC AT "conn" STATEMENT 'SELECT ...'`)
- Runs them serially OR in parallel (PARALLEL_CONNECTIONS param)
- Writes to JOB_LOG / JOB_DETAILS when LOGGING_SCHEMA set

### 7. Monitor JOB_LOG / JOB_DETAILS

When `EXECUTION_MODE='EXECUTE'` AND `LOGGING_SCHEMA` is non-null, the script auto-creates and writes to:

```sql
EXA_DB_MIGRATION.JOB_LOG (
    RUN_ID       INT IDENTITY PRIMARY KEY,
    SCRIPT_NAME  VARCHAR(100),
    STATUS       VARCHAR(100),       -- RUNNING | OK | FAILED
    START_TIME   TIMESTAMP,
    END_TIME     TIMESTAMP
);

EXA_DB_MIGRATION.JOB_DETAILS (
    DETAIL_ID    INT IDENTITY,
    RUN_ID       INT REFERENCES JOB_LOG(RUN_ID),
    LOG_TIME     TIMESTAMP,
    LOG_LEVEL    VARCHAR(10),        -- INFO | WARN | ERROR
    LOG_MESSAGE  VARCHAR(2000000),
    ROWCOUNT     DECIMAL(18)
);
```

Skill polls:

```sql
SELECT STATUS FROM EXA_DB_MIGRATION.JOB_LOG WHERE RUN_ID = <run_id>;
-- RUNNING → poll again
-- OK → migration successful
-- FAILED → fetch JOB_DETAILS where LOG_LEVEL='ERROR'
```

The migration script also pre-installs `EXA_DB_MIGRATION.QUERY_WRAPPER` (from `query_wrapper.sql` fixture) — that's the helper class the migration scripts use for logging. If JOB_LOG/JOB_DETAILS creation fails because QUERY_WRAPPER isn't installed, run `scripts_library.install_fixture("migration-scripts")` or directly `EXECUTE SCRIPT` against `query_wrapper.sql`.

Set `LOGGING_SCHEMA=NULL` to disable logging — slightly faster, no audit. Use only for throwaway runs.

### 8. Verify `schema_imported` provides

```sql
SELECT COUNT(*) FROM SYS.EXA_ALL_TABLES WHERE TABLE_SCHEMA = '<TARGET_SCHEMA>';
```

Expected: matches the table count from JOB_DETAILS or a DEBUG-mode dry run.

### 9. DROP the connection

```sql
DROP CONNECTION "<CONNECTION_NAME>";
```

Credential hygiene. Skill SHOULD do this even on failure paths.

### 10. Return summary

```
Migrated <N>/<M> tables from <source_type>.<schema> to <TARGET_SCHEMA>.
Run id: <run_id>. JOB_LOG status: OK.
Errors: <list from JOB_DETAILS where LOG_LEVEL='ERROR'>.
```

## Type mapping

The migration scripts handle type mapping per-vendor. Conservative defaults across most scripts:
- Numeric without scale → `DECIMAL(36,18)`
- Date/time → preserved when Exasol has a matching type, else converted to UTC `TIMESTAMP`
- VARCHAR no max → `VARCHAR(2000000)` (Exasol max)
- Semi-structured (JSON / VARIANT / ARRAY) → `VARCHAR(2000000)` text
- Binary → skipped with manifest warning (no Exasol BLOB type)

Per-vendor specifics in `references/<vendor>.md` and within the script's Lua body.

## Parallelization

`PARALLEL_CONNECTIONS` controls IMPORT concurrency:
- Integer N → N parallel IMPORTs across tables
- `'AUTO'` → 75% of cluster VCPUs (queried from `EXA_STATISTICS.EXA_SYSTEM_EVENTS` at script start)
- `'INFO'` → print VCPU stats, don't migrate
- NULL / unset → serial (1 connection)

For a 1-node `exanano-sqlcube` Docker (typical dev): AUTO gives 6-12 parallel depending on host. For large source tables, parallelism helps; for many-small-table schemas, the per-table fixed cost dominates and AUTO ≈ serial.

## DEBUG mode for inspection (Snowflake only)

`EXECUTION_MODE='DEBUG'` returns the generated SQL as a result set instead of executing. **The studio's UI exposes this for review-before-execute workflows on Snowflake — only Snowflake.** The other 17 vendor scripts have no EXECUTION_MODE parameter (see "Per-vendor signatures vary" below) and run unconditionally on `EXECUTE SCRIPT`. To dry-run a non-Snowflake migration, drive `POST /api/import/jdbc/preview` instead — it returns the CREATE CONNECTION + IMPORT SQL without executing and with no source-side roundtrip.

```sql
EXECUTE SCRIPT EXA_DB_MIGRATION.SNOWFLAKE_TO_EXASOL(
    'SNOWFLAKE_MIGRATE_20260513',
    FALSE, 'ACME_DEMO', 'ADVENTUREWORKS', 'ADVENTUREWORKS', '%',
    TRUE,
    'DEBUG',                          -- ← inspect mode (Snowflake only)
    NULL,
    NULL
);
```

Returns N rows where each row is `(SQL_TEXT, SUCCESS, ERROR_MESSAGE)` for a single CREATE SCHEMA / CREATE TABLE / IMPORT INTO statement. Read, review, run manually OR re-invoke with `EXECUTION_MODE='EXECUTE'`.

**DEBUG is not a pure dry-run even on Snowflake.** Live-verified 2026-05-14 against `exanano-sqlcube`: the script still queries the source catalog to enumerate databases / schemas / tables before generating the result set. With a fake `CONNECTION` the script aborts with `Error on getting db list from Snowflake:` before reaching the DEBUG output. For a real no-side-effects dry-run on any vendor, use `POST /api/import/jdbc/preview`.

## Per-vendor signatures vary

The migration-scripts library is **not normalized** — separate ongoing studio-team project. Each vendor script has its own argument list. Snowflake is the canonical, fullest-featured example (10 args including EXECUTION_MODE + LOGGING_SCHEMA). Others differ substantially.

Live-checked 2026-05-14:

| Script | Has `EXECUTION_MODE` | Has `LOGGING_SCHEMA` (auto-creates JOB_LOG / JOB_DETAILS) |
|---|---|---|
| `snowflake_to_exasol.sql` | ✓ | ✓ |
| `restore_stage_to_target.sql` | ✓ | — (utility, not a vendor script) |
| All 17 other vendor scripts (azure_sql, bigquery, db2, exasol, mariadb, mysql, netezza, oracle, postgres, redshift, s3, sap_hana, sqlserver, teradata, vectorwise, vertica) | ✗ | ✗ |

Example signature divergence — `exasol_to_exasol.sql` takes 8 args (`CONNECTION_NAME, CONNECTION_SETTING, IDENTIFIER_CASE_INSENSITIVE, SCHEMA_FILTER, TABLE_FILTER, GENERATE_VIEWS, VIEW_FILTER, PK_SETTING`) — no execution-mode, no logging-schema, no target-schema, no db-filter, no parallel-connections. Completely different shape.

Agent rule of thumb when adding a new vendor:

1. List installed scripts: `GET /api/import/migration-scripts/installed`.
2. If vendor's script isn't installed: `POST /api/import/migration-scripts/install {"filenames":["<vendor>_to_exasol.sql"]}` — note `filenames` is a list.
3. Read the script source from `studio/backend/fixtures/migration-scripts/<vendor>_to_exasol.sql`. Grep for `^create or replace script ... \(` to extract the canonical argument list — the comments above each arg explain the contract.
4. For Snowflake only, use the typed `/snowflake/preview-migration` endpoint. For everything else, compose the EXECUTE SCRIPT call by hand based on the script signature, or use `/jdbc/preview` for the per-table IMPORT shape.
5. JOB_LOG / JOB_DETAILS auto-creation is Snowflake-only. Other vendors' results don't land in `EXA_DB_MIGRATION.JOB_LOG` — verify by row count, by table count in target schema, or by inspecting the script's own pquery-result handling.

## Error handling

Per-table failures don't abort the whole migration. The script:
- Catches `pquery` errors at IMPORT level
- Logs to JOB_DETAILS with LOG_LEVEL='ERROR'
- Continues to next table
- Marks JOB_LOG.STATUS='FAILED' at end if any error logged

Connection-level failures (auth, network, TLS) ARE fatal and halt before any table is touched.

For partial migrations: re-run with TABLE_FILTER narrowed to the failed subset. Migration scripts respect the filter.

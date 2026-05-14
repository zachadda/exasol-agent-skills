# Migration flow (vendor-agnostic)

The shared pipeline the skill follows against any source system in `studio/backend/fixtures/migration-scripts/`. Vendor-specific knobs are in `references/<vendor>.md`; read this first.

## Two ways to drive the flow

When the studio FastAPI backend is reachable, prefer its HTTP endpoints — they wrap the SQL with credential storage, driver placement, and progress polling already wired in:

| Endpoint | Purpose |
|---|---|
| `GET  /api/imports/driver-catalog` | List `DRIVER_PRESETS` (source_type → jar name / Java class / URL prefix) |
| `POST /api/imports/jdbc/save-connection` | Persist credentials into the studio's local SQLite (Fernet-encrypted) under a profile name |
| `POST /api/imports/jdbc/test-config` / `/test-connection` | Validate a profile or a freshly-saved connection without committing anything |
| `POST /api/imports/jdbc/preview` | Generate the IMPORT SQL for review without running it |
| `POST /api/imports/jdbc/run-import` | Execute the generated IMPORT batch and stream progress |
| `GET  /api/imports/jdbc/databases` / `/schemas` | List remote databases / schemas through an existing CONNECTION (uses `IMPORT FROM JDBC STATEMENT '<vendor catalog query>'`) |
| `POST /api/imports/snowflake/preview-migration` / `/run-sql` | Snowflake-specific flow that calls `EXA_DB_MIGRATION.SNOWFLAKE_TO_EXASOL` with EXECUTION_MODE=DEBUG or EXECUTE |
| `GET  /api/imports/migration-scripts` / `/installed` | Inspect available migration scripts + which are installed |
| `POST /api/imports/migration-scripts/install` | Install or re-install a migration script from `fixtures/migration-scripts/` |
| `POST /api/imports/cancel-running` | Interrupt a long-running import |
| `POST /api/imports/artifacts/upload` | Upload BucketFS artifacts (driver jars, CSV/Parquet sources) used by the import |

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

## DEBUG mode for inspection

`EXECUTION_MODE='DEBUG'` returns the generated SQL as a result set instead of executing. The studio's UI exposes this for review-before-execute workflows.

```sql
EXECUTE SCRIPT EXA_DB_MIGRATION.SNOWFLAKE_TO_EXASOL(
    'SNOWFLAKE_MIGRATE_20260513',
    FALSE, 'ACME_DEMO', 'ADVENTUREWORKS', 'ADVENTUREWORKS', '%',
    TRUE,
    'DEBUG',                          -- ← inspect mode
    NULL,
    NULL
);
```

Returns N rows where each row is `(SQL_TEXT, SUCCESS, ERROR_MESSAGE)` for a single CREATE SCHEMA / CREATE TABLE / IMPORT INTO statement. Read, review, run manually OR re-invoke with `EXECUTION_MODE='EXECUTE'`.

## Error handling

Per-table failures don't abort the whole migration. The script:
- Catches `pquery` errors at IMPORT level
- Logs to JOB_DETAILS with LOG_LEVEL='ERROR'
- Continues to next table
- Marks JOB_LOG.STATUS='FAILED' at end if any error logged

Connection-level failures (auth, network, TLS) ARE fatal and halt before any table is touched.

For partial migrations: re-run with TABLE_FILTER narrowed to the failed subset. Migration scripts respect the filter.

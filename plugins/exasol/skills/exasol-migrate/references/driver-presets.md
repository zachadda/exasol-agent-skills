# Driver presets

Quick lookup table for vendor → driver jar + Java class + URL prefix. Mirrors `studio/backend/routers/imports.py:DRIVER_PRESETS` and the migration scripts inventory at `studio/backend/fixtures/migration-scripts/`.

The studio backend uses this list to:
- Validate the `source_type` parameter on import requests
- Place driver jars in the runtime-appropriate location (`services/bucketfs.py`)
- Generate the per-vendor `settings.cfg` text used by Exasol's adapter
- Test driver availability before attempting CREATE CONNECTION

## With DRIVER_PRESETS (full integration)

7 vendors have full preset rows — driver jar pre-staged, URL template ready, migration script in place:

| `source_type` | `driver_name` | Driver class | URL prefix | Recommended jar |
|---|---|---|---|---|
| `snowflake` | `SNOWFLAKE` | `net.snowflake.client.jdbc.SnowflakeDriver` | `jdbc:snowflake:` | `snowflake-jdbc-3.22.0.jar` |
| `postgresql` | `POSTGRESQL` | `org.postgresql.Driver` | `jdbc:postgresql:` | `postgresql-42.7.5.jar` |
| `oracle` | `ORACLE` | `oracle.jdbc.driver.OracleDriver` | `jdbc:oracle:thin:` | `ojdbc10.jar` |
| `sqlserver` | `SQLSERVER` | `com.microsoft.sqlserver.jdbc.SQLServerDriver` | `jdbc:sqlserver:` | `mssql-jdbc-12.8.1.jre11.jar` |
| `redshift` | `REDSHIFT` | `com.amazon.redshift.jdbc42.Driver` | `jdbc:redshift:` | `redshift-jdbc42-2.1.0.32.jar` |
| `teradata` | `TERADATA` | `com.teradata.jdbc.TeraDriver` | `jdbc:teradata:` | `terajdbc4_20.jar` |
| `sqlserver_jtds` | `JTDS` | `net.sourceforge.jtds.jdbc.Driver` | `jdbc:jtds:sqlserver:` | `jtds-1.3.1.jar` |

(MySQL / MariaDB also have presets — check `routers/imports.py:DRIVER_PRESETS` for the exact rows in the current build.)

For these, the agent path is:
```
source_type → preset → driver jar in /exa/jdbc/<DRIVER_NAME_UPPER>/
            → migration script EXA_DB_MIGRATION.<VENDOR>_TO_EXASOL exists
```

Both are pre-wired on `exanano-sqlcube` after `scripts_library.install_fixture("migration-scripts")`.

## Migration script only (no driver preset)

These vendors have a migration script in `fixtures/migration-scripts/` but **no DRIVER_PRESETS row**, meaning the user supplies the driver jar:

| `source_type` | Migration script | Driver source |
|---|---|---|
| `mysql` | `mysql_to_exasol.sql` | MySQL Connector/J (download from MySQL site) |
| `mariadb` | `mariadb_to_exasol.sql` | MariaDB Connector/J |
| `bigquery` | `bigquery_to_exasol.sql` | Simba JDBC for BigQuery (Google download page) |
| `azure_sql` | `azure_sql_to_exasol.sql` | mssql-jdbc (same as sqlserver preset works) |
| `db2` | `db2_to_exasol.sql` | IBM Data Server Driver for JDBC and SQLJ (IBM) |
| `sap_hana` | `sap_hana_to_exasol.sql` | ngdbc (SAP) |
| `netezza` | `netezza_to_exasol.sql` | nzjdbc (IBM Netezza) |
| `vectorwise` | `vectorwise_to_exasol.sql` | Actian Vectorwise JDBC |
| `vertica` | `vertica_to_exasol.sql` | vertica-jdbc (Micro Focus / OpenText) |
| `exasol` | `exasol_to_exasol.sql` | exasol-jdbc (bundled) |
| `s3` | `s3_to_exasol.sql` | No driver — uses Exasol's built-in S3 client via CONNECTION |

For these, the agent path is:
```
source_type → no preset → user uploads driver jar
            → migration script EXA_DB_MIGRATION.<VENDOR>_TO_EXASOL exists
            → user supplies jdbc_url manually
```

## Bonus: bundled utilities

These aren't migrations but support them:

| Script | Purpose |
|---|---|
| `query_wrapper.sql` | Helper class used by all migration scripts for logging to JOB_LOG / JOB_DETAILS. Pre-install required. |
| `delta_import_on_primary_keys.sql` | Incremental sync — re-import only rows whose PK matches a watermark. Requires source-side PKs and a delta column. |
| `restore_stage_to_target.sql` | After STAGE schema is loaded, atomically swap into TARGET. Uses SQLCUBE_META.SCHEMAS lineage hints. |

## How the agent picks a source_type

1. **User specifies explicitly** — `source_type=snowflake` parameter passed to the skill.
2. **Orchestrator infers from context** — `/oneshot-tour` reads user's "load from X" question and maps to a known source_type.
3. **Fallback: enumerate** — when ambiguous, the skill SHOULD `SELECT SCRIPT_NAME FROM SYS.EXA_ALL_SCRIPTS WHERE SCRIPT_SCHEMA='EXA_DB_MIGRATION'` and show the user what's available.

## Driver jar version policy

Studio backend pins to the `is_recommended=True` version of each preset (see `DriverPreset.versions` list). Older versions are kept available for compatibility but new deploys default to recommended.

When upgrading:
1. Drop new jar in `/exa/jdbc/<DRIVER_NAME_UPPER>/`
2. Remove old jar (Exasol scans all jars in the directory; multiple versions = ambiguous classpath)
3. Restart container OR adapter (`metadata_registry.lua` is rerun on next query)
4. Re-run a test connection to confirm

The studio's `routers/drivers.py` exposes upload endpoints for this; agents not running against the studio can `docker cp` directly.

## Adding a new vendor

If a source system isn't in the list:

1. **Author a migration script.** Use `snowflake_to_exasol.sql` as a template — it's the most complete.
2. **Add to `fixtures/migration-scripts/`** and re-run `scripts_library.install_fixture("migration-scripts")`.
3. **Optional: add a DRIVER_PRESETS row** in `routers/imports.py` if the driver jar is freely redistributable.
4. **Test against a real source** + smoke-load a small fixture.

This is out of scope for this skill — it consumes existing presets/scripts, doesn't author new ones. File a request against the factory-foundation repo if you need a new vendor.

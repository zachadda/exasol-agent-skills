# Snowflake source

Vendor-specific knobs for migrating a Snowflake schema into Exasol via `EXA_DB_MIGRATION.SNOWFLAKE_TO_EXASOL`. Read `flow.md` first for the shared pipeline.

Canonical source: `studio/backend/fixtures/migration-scripts/snowflake_to_exasol.sql`.

## Driver preset

From `routers/imports.py:DRIVER_PRESETS`:

| Field | Value |
|---|---|
| `source_type` | `snowflake` |
| `driver_name` | `SNOWFLAKE` |
| `driver_main` | `net.snowflake.client.jdbc.SnowflakeDriver` |
| `prefix` | `jdbc:snowflake:` |
| Recommended jar | `snowflake-jdbc-3.22.0.jar` |
| Older supported | `snowflake-jdbc-3.19.1.jar` |

Driver placement (Docker container mode): `/exa/jdbc/SNOWFLAKE/snowflake-jdbc-3.22.0.jar`.

## JDBC URL template

```
jdbc:snowflake://<ACCOUNT>.snowflakecomputing.com/?warehouse=<WAREHOUSE>&db=<DB>&schema=<SCHEMA>&role=<ROLE>
```

Single line at exec time. `<ACCOUNT>` is the Snowflake account locator (e.g., `xy12345.us-east-1`) without the `.snowflakecomputing.com` suffix.

PrivateLink accounts: substitute the privatelink hostname:
```
jdbc:snowflake://<ACCOUNT>.privatelink.snowflakecomputing.com/...
```

Skill cannot infer the privatelink suffix — user must supply `jdbc_url` parameter.

## Credentials

Required for `CREATE CONNECTION`:

| Field | Where it lives |
|---|---|
| Snowflake user | `source_user` skill parameter, OR studio CONNECTION_PROFILES |
| Snowflake password | `source_password` skill parameter (plaintext), OR studio profile (Fernet-encrypted SQLite). Plain text round-trips to `CREATE CONNECTION ... IDENTIFIED BY '<pw>'` which Exasol stores cluster-encrypted. |
| Warehouse | URL parameter; Snowflake compute warehouse to run source SELECTs against |
| Role | URL parameter; defaults to user's default if omitted |

### Key-pair auth (recommended for service accounts)

Append to JDBC URL:
```
&authenticator=SNOWFLAKE_JWT
&private_key_file=/exa/jdbc-keys/snowflake_key.p8
```

**Critical**: `private_key_file` is a path **inside the Exasol container**, not on the host. Copy the PEM first:

```bash
docker exec exanano-sqlcube mkdir -p /exa/jdbc-keys
docker cp ./rsa_key.p8 exanano-sqlcube:/exa/jdbc-keys/snowflake_key.p8
docker exec exanano-sqlcube chmod 600 /exa/jdbc-keys/snowflake_key.p8
```

Then in CREATE CONNECTION:
```sql
CREATE OR REPLACE CONNECTION "SNOWFLAKE_MIGRATE_..."
  TO 'jdbc:snowflake://...&authenticator=SNOWFLAKE_JWT&private_key_file=/exa/jdbc-keys/snowflake_key.p8'
  USER 'svc_acct'
  IDENTIFIED BY '';     -- password ignored for key-pair, but the clause is required
```

### OAuth / SSO — not supported in v0.1

Snowflake OAuth and external-IDP federations need a token broker the studio backend doesn't currently expose. Workaround: pre-fetch a session token outside the agent, pass via `&token=<...>&authenticator=OAUTH` query parameters.

## SNOWFLAKE_TO_EXASOL parameter contract

Per `studio/backend/fixtures/migration-scripts/snowflake_to_exasol.sql`. Ten positional parameters:

| # | Parameter | Type | Purpose | Skill default |
|---|---|---|---|---|
| 1 | `CONNECTION_NAME` | VARCHAR | Pre-created Exasol CONNECTION | `SNOWFLAKE_MIGRATE_<UNIX_TS>` (skill-generated) |
| 2 | `DB2SCHEMA` | BOOLEAN | If TRUE: flatten `db.schema.table` → Exasol `db_schema.table`. If FALSE: drop the db prefix, use `schema.table` | `FALSE` |
| 3 | `DB_FILTER` | VARCHAR | Snowflake DB filter. `'%'` for all, `'master'` for exact, `'ma%'` for LIKE, `'first_db, second_db'` for csv list | Required from caller |
| 4 | `SCHEMA_FILTER` | VARCHAR | Same patterns as DB_FILTER | Required from caller |
| 5 | `TARGET_SCHEMA` | VARCHAR | Exasol target schema name. Empty string `''` → use source schema name | `target_schema` parameter (defaults to source_schema) |
| 6 | `TABLE_FILTER` | VARCHAR | Same patterns as DB_FILTER | `'%'` (all tables) |
| 7 | `IDENTIFIER_CASE_INSENSITIVE` | BOOLEAN | If TRUE: uppercase all identifiers on the Exasol side. If FALSE: preserve source casing | `TRUE` (Exasol's default behavior matches) |
| 8 | `EXECUTION_MODE` | VARCHAR | `'DEBUG'` (return SQL, don't run) or `'EXECUTE'` | `'EXECUTE'` |
| 9 | `PARALLEL_CONNECTIONS` | VARCHAR / INT / NULL | Integer N, `'AUTO'` (75% of cluster VCPUs), `'INFO'` (stats only), or NULL (serial) | `'AUTO'` |
| 10 | `LOGGING_SCHEMA` | VARCHAR / NULL | Schema for JOB_LOG / JOB_DETAILS audit. NULL to disable | `'EXA_DB_MIGRATION'` |

Example invocation:

```sql
EXECUTE SCRIPT EXA_DB_MIGRATION.SNOWFLAKE_TO_EXASOL(
    'SNOWFLAKE_MIGRATE_1715638800',
    FALSE,
    'ACME_DEMO',
    'ADVENTUREWORKS',
    'ADVENTUREWORKS',
    '%',
    TRUE,
    'EXECUTE',
    'AUTO',
    'EXA_DB_MIGRATION'
);
```

## Filter pattern syntax

The DB / SCHEMA / TABLE filters share a syntax derived from the migration script:

| Pattern | Meaning |
|---|---|
| `'%'` | Match all |
| `'ADVENTUREWORKS'` | Exact match |
| `'AW%'` | LIKE pattern — starts with AW |
| `'%SALES'` | Ends with SALES |
| `'AW%, SF%'` | CSV list of LIKE patterns (any matches) |

Anything containing `%` is translated to a `LIKE '%'` clause server-side; otherwise the script builds an `IN ('a','b','c')` clause. See lines 117-120 of the script for the parser.

## Type mapping

The Snowflake script handles type mapping internally. Key conversions:

| Snowflake type | Exasol type | Notes |
|---|---|---|
| `NUMBER(p,s)` with explicit p,s | `DECIMAL(p,s)` if p≤36 | Exasol DECIMAL max precision is 36 |
| `NUMBER(p,s)` p>36 | `DECIMAL(36,s)` | Truncated; warning in JOB_DETAILS |
| `NUMBER` no params | `DECIMAL(36,18)` | Snowflake default is 38,0 |
| `FLOAT` / `DOUBLE` / `REAL` | `DOUBLE PRECISION` | |
| `VARCHAR(n)` n≤2000000 | `VARCHAR(n)` | |
| `VARCHAR(n)` n>2000000 | `VARCHAR(2000000)` | Exasol cap |
| `STRING` / `TEXT` | `VARCHAR(2000000)` | Unbounded → max |
| `BOOLEAN` | `BOOLEAN` | |
| `DATE` | `DATE` | |
| `TIME` | `VARCHAR(20)` | No Exasol TIME-only type |
| `TIMESTAMP_NTZ` | `TIMESTAMP` | |
| `TIMESTAMP_LTZ` / `TIMESTAMP_TZ` | `TIMESTAMP` | Converted to UTC; logged |
| `VARIANT` / `OBJECT` / `ARRAY` | `VARCHAR(2000000)` | JSON text |
| `BINARY` / `VARBINARY` | — | Skipped with warning (no Exasol BLOB) |
| `GEOGRAPHY` / `GEOMETRY` | `VARCHAR(2000000)` | WKT representation |
| `VECTOR(n)` | `VARCHAR(2000000)` | Serialized as text array |

## STATEMENT pushdown

The script generates `STATEMENT 'SELECT * FROM <DB>.<SCHEMA>.<TABLE>'` per table. To restrict columns or rows, pass a custom STATEMENT — but the SNOWFLAKE_TO_EXASOL script doesn't expose this directly. Workarounds:

- **DEBUG mode + manual edit**: `EXECUTION_MODE='DEBUG'`, capture the generated SQL, modify, run manually.
- **Source-side views**: Create a Snowflake view that projects/filters, then point the filter at the view name.

## Cost notice

`PARALLEL_CONNECTIONS=AUTO` fires multiple concurrent IMPORTs against the Snowflake warehouse. The warehouse runs for the duration of the migration; compute cost lands on `<WAREHOUSE>`.

For TPCH SF1 (~12M rows, 8 tables) on an X-Small: ~5 min, ~$0.30. For TPCH SF100: ~45 min on Medium, several dollars.

The skill SHOULD surface this estimate during plan-presentation:

```
Source size: ~6 GB compressed
Estimated Snowflake compute cost: ~$0.50 (X-Small warehouse, 5 min)
PARALLEL_CONNECTIONS=AUTO → ~8 concurrent IMPORTs
Proceed?
```

## Known failure modes

| Symptom | Cause | Fix |
|---|---|---|
| `390100: Incorrect username or password` | Bad creds in CREATE CONNECTION | Re-check source_user / source_password |
| `390114: Authentication token has expired` | OAuth / JWT staleness | Re-fetch token, re-create connection |
| `391701: Network policy blocked` | Snowflake account network policy excludes the cluster's egress IP | Add IP to allowlist OR use PrivateLink jdbc_url |
| `JDBC driver internal error: Cannot acquire warehouse` | Suspended warehouse, AUTO_RESUME not granted | `ALTER WAREHOUSE <WH> RESUME` from Snowflake side first |
| `Object does not exist or operation cannot be performed` | Role lacks USAGE on schema | `GRANT USAGE ON SCHEMA ... TO ROLE` |
| Migration starts then aborts at first IMPORT | `EXA_DB_MIGRATION.QUERY_WRAPPER` missing | `EXECUTE SCRIPT` against `studio/backend/fixtures/migration-scripts/query_wrapper.sql` OR `scripts_library.install_fixture("migration-scripts")` |
| JOB_LOG.STATUS=FAILED but some tables imported | Per-table error — check JOB_DETAILS WHERE LOG_LEVEL='ERROR' | Inspect, fix root cause (often type-out-of-range or NULL on NOT NULL), re-run with TABLE_FILTER narrowed |

## ACME_DEMO.adventureworks fixture path (`/oneshot-tour` demo)

For the `/oneshot-tour` demo, the canonical input is:

```
source_type     = snowflake
source_db       = ACME_DEMO            (→ DB_FILTER)
source_schema   = ADVENTUREWORKS       (→ SCHEMA_FILTER)
target_schema   = ADVENTUREWORKS
table_filter    = '%'
```

User pre-sets credentials (skill parameters, or studio CONNECTION_PROFILES). Skill auto-generates connection name, runs SNOWFLAKE_TO_EXASOL with EXECUTION_MODE=EXECUTE, polls JOB_LOG, drops connection on completion.

Default warehouse: `DEMO_WH` on X-Small. Expected duration: 2–4 min.

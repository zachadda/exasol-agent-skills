# Snowflake source

Vendor-specific knobs for migrating a Snowflake schema into Exasol. Read `flow.md` first for the shared pipeline.

## Required env vars

| Var | Required | Notes |
|---|---|---|
| `SNOWFLAKE_USER` | yes | Login name |
| `SNOWFLAKE_ACCOUNT` | yes | Account locator, e.g. `xy12345.us-east-1`. Strip `.snowflakecomputing.com`. |
| `SNOWFLAKE_PASSWORD` | if auth_mode=password | Plaintext or env-injected; never log |
| `SNOWFLAKE_PRIVATE_KEY_PATH` | if auth_mode=key_pair | Path to PEM file (inside the container if running JDBC there — see below) |
| `SNOWFLAKE_PRIVATE_KEY_PASSPHRASE` | if encrypted key | |
| `SNOWFLAKE_WAREHOUSE` | yes | Compute warehouse to use for source SELECTs. Cost lands on this. |
| `SNOWFLAKE_ROLE` | recommended | Defaults to user's default role if unset; explicit is safer for reproducibility |

Pre-flight check before any IMPORT:

```bash
[[ -n "$SNOWFLAKE_USER" && -n "$SNOWFLAKE_ACCOUNT" && -n "$SNOWFLAKE_WAREHOUSE" ]] && \
  [[ -n "$SNOWFLAKE_PASSWORD" || -n "$SNOWFLAKE_PRIVATE_KEY_PATH" ]] && echo ok
```

## JDBC URL template

```
jdbc:snowflake://${SNOWFLAKE_ACCOUNT}.snowflakecomputing.com/
  ?warehouse=${SNOWFLAKE_WAREHOUSE}
  &db=${SOURCE_DATABASE}
  &schema=${SOURCE_SCHEMA}
  &role=${SNOWFLAKE_ROLE}
```

Joined to one line at execution time. The `db` and `schema` parameters set the session context inside Snowflake — STATEMENT SQL can still fully-qualify if needed.

For PrivateLink / VPC-only Snowflake accounts:

```
jdbc:snowflake://${SNOWFLAKE_ACCOUNT}.privatelink.snowflakecomputing.com/...
```

User must supply `jdbc_url_override` parameter in that case — agent can't infer the privatelink suffix.

## Auth modes

### password (default)

```sql
USER 'username'
IDENTIFIED BY 'password'
```

Use for personal dev accounts and demos. Snowflake supports MFA; if enforced on the account, password alone fails — switch to key-pair.

### key_pair

Snowflake's standard service-account pattern. Two-step:

1. Generate key pair (one-time):
   ```bash
   openssl genrsa 2048 | openssl pkcs8 -topk8 -inform PEM -out rsa_key.p8 -nocrypt
   openssl rsa -in rsa_key.p8 -pubout -out rsa_key.pub
   ```
2. Register public key in Snowflake:
   ```sql
   ALTER USER svc_acct SET RSA_PUBLIC_KEY='<contents of rsa_key.pub without header/footer>';
   ```

JDBC connection appends:

```
&authenticator=SNOWFLAKE_JWT
&private_key_file=${SNOWFLAKE_PRIVATE_KEY_PATH}
```

Critical: `private_key_file` is a path **inside the Exasol container**, not on the host. The JDBC driver runs in Exasol's JVM, not in the agent's environment. Copy the PEM:

```bash
docker exec exanano-sqlcube mkdir -p /exa/jdbc-keys
docker cp ./rsa_key.p8 exanano-sqlcube:/exa/jdbc-keys/snowflake_key.p8
docker exec exanano-sqlcube chmod 600 /exa/jdbc-keys/snowflake_key.p8
```

Then set `SNOWFLAKE_PRIVATE_KEY_PATH=/exa/jdbc-keys/snowflake_key.p8` in the agent env.

### oauth / iam — not supported in v0.1

Snowflake OAuth and external-IDP federations require a token broker that the Exasol JDBC layer doesn't expose hooks for. Workaround: pre-fetch a session token outside the agent and pass via `&token=` query param + `&authenticator=OAUTH`. Document as user-handled, don't automate in v0.1.

## Table enumeration

Snowflake INFORMATION_SCHEMA is per-database. STATEMENT SQL:

```sql
SELECT TABLE_NAME, ROW_COUNT
FROM ${SOURCE_DATABASE}.INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = '${SOURCE_SCHEMA}'
  AND TABLE_TYPE = 'BASE TABLE'
ORDER BY TABLE_NAME
```

`ROW_COUNT` here is Snowflake's cached statistic — usually accurate but can lag for actively-written tables. Acceptable for migration planning. Don't use as source-of-truth for reconciliation; per-table `SELECT COUNT(*)` is authoritative.

## Column-level type mapping

| Snowflake type | Exasol DDL | Notes |
|---|---|---|
| `NUMBER(p,s)` with explicit p,s | `DECIMAL(p,s)` if p≤36 | Exasol DECIMAL max precision is 36 |
| `NUMBER(p,s)` p>36 | `DECIMAL(36,s)` | Lossy; log warning to manifest |
| `NUMBER` no params | `DECIMAL(36,18)` | Snowflake default is 38,0 — we downsize to fit |
| `FLOAT` / `DOUBLE` / `REAL` | `DOUBLE PRECISION` | |
| `VARCHAR(n)` n≤2000000 | `VARCHAR(n)` | |
| `VARCHAR(n)` n>2000000 | `VARCHAR(2000000)` | Exasol cap; log warning if data actually exceeds |
| `STRING` / `TEXT` | `VARCHAR(2000000)` | Snowflake STRING is unbounded → use Exasol max |
| `CHAR(n)` | `CHAR(n)` | Up to 2000 in Exasol |
| `BOOLEAN` | `BOOLEAN` | |
| `DATE` | `DATE` | |
| `TIME` | `VARCHAR(20)` | Exasol has no TIME-only type; encode as text or compose with DATE |
| `TIMESTAMP_NTZ` | `TIMESTAMP` | Direct |
| `TIMESTAMP_LTZ` / `TIMESTAMP_TZ` | `TIMESTAMP` | Converted to UTC; log to manifest |
| `VARIANT` / `OBJECT` / `ARRAY` | `VARCHAR(2000000)` | Semi-structured stays as JSON text |
| `BINARY` / `VARBINARY` | — | Skip with manifest warning (no Exasol BLOB) |
| `GEOGRAPHY` / `GEOMETRY` | `VARCHAR(2000000)` | WKT representation |
| `VECTOR(n)` | `VARCHAR(2000000)` | Serialize as text array; lossy for ANN ops downstream |

Column-list query for one table:

```sql
SELECT COLUMN_NAME, DATA_TYPE, NUMERIC_PRECISION, NUMERIC_SCALE,
       CHARACTER_MAXIMUM_LENGTH, IS_NULLABLE
FROM ${SOURCE_DATABASE}.INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = '${SOURCE_SCHEMA}'
  AND TABLE_NAME = '${TABLE}'
ORDER BY ORDINAL_POSITION
```

## STATEMENT pushdown

Snowflake handles ANSI SQL plus extensions. Safe to push:

- `SELECT col1, col2 FROM ...` — projection
- `WHERE` predicates
- `LIMIT n` — used for `sample_only=true`
- `ORDER BY` — but expensive in Snowflake, skip unless caller asks
- `QUALIFY` — Snowflake-specific, fine since source-side

Avoid:

- `WITH RECURSIVE` — query planner difference, IMPORT round-trip times out
- Snowflake stored procedures / UDFs in the SELECT — they execute in Snowflake but error surfaces are opaque through JDBC

## Driver

Maven Central:

```
net.snowflake:snowflake-jdbc:3.14.5
```

(Pin a known-good version; latest may break against older Exasol JVM.)

Placement: `/exa/jdbc/SNOWFLAKE/snowflake-jdbc-3.14.5.jar` inside the container. See `exasol-nano-local/jdbc-drivers.md`.

## Cost notice

The compute warehouse named in `SNOWFLAKE_WAREHOUSE` runs during the migration. Time = warehouse cost. For TPCH SF1 (~12M rows, 8 tables) on an X-Small: ~5 min, ~$0.30. For TPCH SF100: ~45 min on Medium, several dollars. The skill SHOULD surface this estimate during step 6 (plan presentation) when source size is known:

```
Source size: ~6 GB compressed
Estimated Snowflake compute cost: ~$0.50 (X-Small warehouse, 5 min)
Proceed?
```

Don't hide cost. Users with a corporate Snowflake bill care.

## Known failure modes

| Symptom | Cause | Fix |
|---|---|---|
| `390100: Incorrect username or password` | Bad creds | Re-verify env vars |
| `390114: Authentication token has expired` | OAuth/JWT staleness | Re-fetch token (see auth_mode notes) |
| `391701: Network policy ... blocked` | Snowflake account network policy excludes the container's egress IP | Add IP to allowlist, or use jdbc_url_override with PrivateLink |
| `JDBC driver internal error: Cannot acquire warehouse` | Suspended warehouse and account lacks AUTO_RESUME privilege | `ALTER WAREHOUSE <WH> RESUME` from Snowflake side first |
| `Object does not exist or operation cannot be performed` | Role lacks USAGE on schema | `GRANT USAGE ON SCHEMA ... TO ROLE` |
| Timeout on large table | Default IMPORT timeout (10 min) hit | Split via include_tables, or set Exasol `STATEMENT_TIMEOUT_MS` higher |

## ACME_DEMO.adventureworks sample path (oneshot-tour fixture)

For the `/oneshot-tour` demo, the canonical input is:

```
source_vendor = snowflake
source_database = ACME_DEMO
source_schema = ADVENTUREWORKS
target_schema = ADVENTUREWORKS
```

User pre-sets the env vars referenced above (or has them in `.env`). Skill auto-runs through the pipeline, manifest lands in current working directory.

Default warehouse for that demo: `DEMO_WH` on X-Small. Expected duration: 2–4 min.

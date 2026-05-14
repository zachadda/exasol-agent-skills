# Migration flow (vendor-agnostic)

Shared mechanics for IMPORT FROM JDBC migrations. Vendor-specific bits — connection string, auth, type quirks — live in `references/<vendor>.md`. Read this first, then layer the vendor file.

## The IMPORT FROM JDBC statement

Canonical shape Exasol uses for every vendor:

```sql
IMPORT INTO <TARGET_SCHEMA>.<TABLE> (<COL_LIST>)
FROM JDBC
  DRIVER  = '<VENDOR_UPPER>'
  AT      '<JDBC_URL>'
  USER    '<USER>'
  IDENTIFIED BY '<PASSWORD>'
  STATEMENT 'SELECT * FROM <SOURCE_SCHEMA>.<TABLE>';
```

Three things to notice:

1. **Target schema must exist and target table must exist with matching columns.** Exasol does NOT infer schema from JDBC metadata at IMPORT time. The CREATE TABLE step is separate and precedes IMPORT.
2. **STATEMENT is run on the source side.** Filtering / projection / sampling are pushed to source via the SQL string. Cheap when source supports the operators.
3. **DRIVER is the directory name under `/exa/jdbc/`.** Uppercase. See `exasol-nano-local` `jdbc-drivers.md`.

## Pipeline steps

### 1. Resolve vendor reference

Load `references/<source_vendor>.md`. Extract:

- JDBC URL template (`jdbc:snowflake://<account>.snowflakecomputing.com/?...`)
- Required env vars (`SNOWFLAKE_USER`, etc.)
- Auth modes supported (password, key-pair, OAuth)
- Type mapping table (source type → Exasol DDL)
- INFORMATION_SCHEMA dialect for table enumeration

If `source_vendor` has no reference file → halt:

> "exasol-migrate v0.1 only supports: snowflake. For others, fall back to CSV (see csv-fallback.md) or wait for vendor-specific reference."

### 2. Build the JDBC URL

```
url = jdbc_url_override OR <vendor_template>.substitute(env)
```

For Snowflake:

```
jdbc:snowflake://${SNOWFLAKE_ACCOUNT}.snowflakecomputing.com/
  ?warehouse=${SNOWFLAKE_WAREHOUSE}
  &db=${SOURCE_DATABASE}
  &schema=${SOURCE_SCHEMA_NAME}
```

(Single-line; readability formatted above.)

If `jdbc_url_override` is set, use it verbatim. Don't merge — full override.

### 3. Test connection

Round-trip a SELECT 1 before committing to a long migration:

```sql
IMPORT INTO (X DECIMAL(18))
FROM JDBC
  DRIVER  = '<VENDOR>'
  AT      '<URL>'
  USER    '<USER>'
  IDENTIFIED BY '<PASSWORD>'
  STATEMENT 'SELECT 1';
```

Returns 1 row → connection works. Errors land in `troubleshooting.md` of the vendor reference (or `exasol-nano-local/troubleshooting.md` §7).

### 4. Enumerate source tables

Each vendor exposes a different INFORMATION_SCHEMA dialect. Generic shape:

```sql
SELECT TABLE_NAME, ROW_COUNT
FROM <VENDOR_INFORMATION_SCHEMA>.TABLES
WHERE TABLE_SCHEMA = '<SOURCE_SCHEMA>'
  AND TABLE_TYPE = 'BASE TABLE'
```

Execute via IMPORT-with-STATEMENT round-trip:

```sql
SELECT * FROM (
  IMPORT INTO (TABLE_NAME VARCHAR(200), ROW_COUNT DECIMAL(18))
  FROM JDBC
    DRIVER = '<VENDOR>' AT '<URL>' USER '<U>' IDENTIFIED BY '<P>'
    STATEMENT '<enumeration SQL>'
);
```

Cache result in Exasol-side staging table `<TARGET_SCHEMA>.__SOURCE_TABLES__` for the rest of the pipeline.

### 5. Apply include / exclude filters

In-memory filter over the enumerated list:

```
candidates = enumerated_tables
if include_tables: candidates = [t for t in candidates if t in include_tables]
if exclude_tables: candidates = [t for t in candidates if t not in exclude_tables]
```

Both lists are plain comma-separated. No glob support in v0.1.

### 6. Plan + present

Render a table to the user:

```
Source: <vendor>.<source_database>.<source_schema>
Target: <target_schema> (exists / will be created / will be replaced)

Tables to migrate:
- ORDERS           ~ 1,500,000 rows
- LINEITEM         ~ 6,000,000 rows
- CUSTOMER         ~   150,000 rows
- ...
Total: 12 tables, ~12.5M rows (sample_only=false)

Estimated time: 3–8 min (vendor-network dependent)

Proceed? [y/n/inspect]
```

`inspect` mode: show the CREATE TABLE DDL we'll run + the first IMPORT statement before continuing. Used by skeptical users who want to eyeball the type mapping.

### 7. Create target schema

```sql
-- If drop_target=true AND user confirmed:
DROP SCHEMA IF EXISTS <TARGET_SCHEMA> CASCADE;

-- Always:
CREATE SCHEMA IF NOT EXISTS <TARGET_SCHEMA>;
OPEN SCHEMA <TARGET_SCHEMA>;
```

DROP is destructive — never silent. User confirmation captured in step 6.

### 8. Per-table migration loop

For each table:

```
a. Inspect source columns (vendor INFORMATION_SCHEMA.COLUMNS)
b. Map types vendor → Exasol  (vendor reference's mapping table)
c. CREATE TABLE target.T (<mapped columns>)
d. IMPORT INTO target.T (cols) FROM JDBC ... STATEMENT 'SELECT cols FROM source.T'
   (+ LIMIT 10000 if sample_only)
e. SELECT COUNT(*) FROM target.T   → log target_rowcount
f. Compare to source rowcount      → status = ok | warn | mismatch
g. Append to manifest
```

Failure modes:

- **CREATE TABLE fails (e.g., reserved keyword as column name):** quote-wrap the offending name, retry once. If still fails, mark row as `ddl_failed`, skip IMPORT, continue to next table.
- **IMPORT fails (auth / network / TLS):** halt the entire pipeline — these are systemic. Report current state and exit.
- **IMPORT fails (single-table type mismatch / value out of range):** log to manifest as `import_failed`, continue.
- **Row count mismatch with sample_only=false:** flag as `warn`, don't halt. User decides.

### 9. Verify provides

After the loop, run the `schema_imported` verify SQL:

```sql
SELECT COUNT(*) FROM SYS.EXA_ALL_TABLES WHERE TABLE_SCHEMA = '<TARGET_SCHEMA>';
```

Must equal `|candidates|` minus tables marked `ddl_failed` or `import_failed`. If less → something silently disappeared; halt and report.

Also write the manifest file:

```
<TARGET_SCHEMA>__migration_manifest.csv
source_table,source_rowcount,target_rowcount,status,error_message
```

Manifest survives the agent session and is consumed by downstream skills (`exasol-optimize` reads it to know which tables to skip).

### 10. Hand back to caller

Return summary:

```
Migrated <N>/<M> tables (~<R> rows) from <vendor>.<source_schema>
to <target_schema> in <duration>. Manifest at <path>.
Failures: <list> (see manifest for details).
```

## Type mapping principles

Conservative wins over precise. The optimize skill downstream can tighten widths (it's literally what `analyze_constraints` is for). At IMPORT time:

- **Source NUMERIC / NUMBER without scale info:** Exasol `DECIMAL(36,18)`. Don't lose precision.
- **Source VARCHAR with explicit max length:** Exasol `VARCHAR(<n>)` if `n <= 2000000`, else `VARCHAR(2000000)`.
- **Source VARCHAR without max:** Exasol `VARCHAR(2000000)` (max). Optimize will narrow.
- **Source DATE / TIMESTAMP / TIMESTAMP WITH TIME ZONE:** preserve. Exasol has `DATE`, `TIMESTAMP`, no TZ-aware type — TZ-aware sources convert to UTC TIMESTAMP, document in manifest.
- **Source BOOLEAN:** Exasol `BOOLEAN`.
- **Source binary / blob:** Exasol has no BLOB. Skip with a manifest warning unless user explicitly requests CHAR-encoded fallback.
- **Source JSON / VARIANT / STRUCT:** Exasol `VARCHAR(2000000)`. Schema-on-read pattern; optimize won't touch it.

Vendor reference files list specific quirks (Snowflake VARIANT, Postgres ARRAY, MySQL ENUM, etc.).

## Parallelization

Per-table IMPORTs are sequential in v0.1 — single connection, clean failure attribution. If the user has a 1000-table schema and complains, future versions can dispatch in parallel via `EXECUTE SCRIPT` batches. Don't optimize until someone hits the wall.

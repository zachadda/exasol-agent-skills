# CSV / S3 fallback

When JDBC isn't viable, the studio backend's fallback is **not** "IMPORT FROM CSV against local files." It's the `S3_TO_EXASOL` migration script — same shape as the JDBC migration scripts, just sourcing from S3-compatible object storage instead of a database.

Canonical source: `studio/backend/fixtures/migration-scripts/s3_to_exasol.sql`.

When to use this path:

- **Source vendor not in DRIVER_PRESETS** AND no migration script exists (rare — see the full list in driver-presets.md)
- **Network policy blocks the Exasol cluster's egress** to the source DB host
- **Corporate TLS interception** breaks JDBC handshakes
- **Source database is a flat-file export** (vendor handed you CSV/Parquet, no live system)
- **Reproducible CI fixtures** — checked-in S3 objects more stable than live JDBC dependency

## High-level flow

```
1. Source exports to CSV/Parquet/JSON files on S3 (out of skill scope)
2. CREATE CONNECTION to S3 (with AWS credentials)
3. EXECUTE SCRIPT EXA_DB_MIGRATION.S3_TO_EXASOL(...)
4. Script enumerates files, generates IMPORT FROM CSV statements
5. Logs to JOB_LOG / JOB_DETAILS (same audit shape as JDBC migrations)
6. DROP CONNECTION
```

The studio backend's `S3_TO_EXASOL` script supports:

- CSV (gzip allowed)
- Parquet (via parquet driver — typically pre-installed)
- JSON Lines (limited; CSV preferred)

## CREATE CONNECTION for S3

Different shape than JDBC:

```sql
CREATE OR REPLACE CONNECTION "S3_MIGRATE_<TS>"
  TO 'https://<bucket>.s3.<region>.amazonaws.com'
  USER '<AWS_ACCESS_KEY_ID>'
  IDENTIFIED BY '<AWS_SECRET_ACCESS_KEY>';
```

For private buckets requiring session tokens (STS-issued temp creds), append the token via additional params — see the script's docs at the top of `s3_to_exasol.sql`.

For S3-compatible alternatives (MinIO, Backblaze B2, R2): substitute the endpoint URL. The connection format is the same as long as the service accepts AWS S3 API.

## S3_TO_EXASOL parameter contract

Read the head of `studio/backend/fixtures/migration-scripts/s3_to_exasol.sql` for the canonical parameter list. Typical shape:

```sql
EXECUTE SCRIPT EXA_DB_MIGRATION.S3_TO_EXASOL(
    'S3_MIGRATE_<TS>',                -- CONNECTION_NAME
    '<bucket>/<prefix>/',             -- SOURCE_PATH
    'TARGET_SCHEMA',                  -- TARGET_SCHEMA
    'csv',                            -- FILE_FORMAT: csv | parquet | json
    {csv format options},             -- delimiter, quote, header skip, etc.
    'EXECUTE',                        -- EXECUTION_MODE
    'EXA_DB_MIGRATION'                -- LOGGING_SCHEMA
);
```

(The parameter count and exact names may differ — consult the live script. The skill should read the script header at runtime to extract the contract rather than hardcoding.)

## Format options

| Format | When |
|---|---|
| CSV (RFC 4180) | Default. Most exports support it. |
| TSV | Set delimiter to `\t` in CSV format options |
| Parquet | Native Exasol support if parquet driver loaded; columnar and compressed → faster |
| Gzip CSV | Auto-detected by `.csv.gz` extension |
| JSON Lines | Limited; convert to CSV first when possible |

Parquet is preferred for large datasets — columnar, type-preserving, smaller wire transfer.

## Direct IMPORT FROM CSV (without S3)

If files are on BucketFS or a local filesystem (the Exasol cluster's perspective, not the agent host), the studio's flow still uses `IMPORT FROM CSV` directly — not via a migration script:

### BucketFS-staged CSV

```bash
# Upload from agent host
exapump bucketfs put -p sqlcube-nano ./customers.csv /default/migrate/customers.csv
```

```sql
-- Target schema must exist; target table must exist with matching columns
CREATE TABLE ADVENTUREWORKS.CUSTOMERS (
    CUSTOMER_ID DECIMAL(18,0),
    FIRST_NAME VARCHAR(50),
    ...
);

IMPORT INTO ADVENTUREWORKS.CUSTOMERS
FROM CSV
  AT 'http://localhost:8888/default/migrate/customers.csv'
  USER 'r' IDENTIFIED BY 'read'
  SKIP = 1
  COLUMN SEPARATOR = ','
  COLUMN DELIMITER = '"';
```

BucketFS URL is the canonical reference inside the Exasol JVM. Auth (`r` / `read`) is the default read-only bucket creds on `exanano-sqlcube`.

### `FROM LOCAL CSV FILE` — container-local files

For tiny CSVs (< 100 MB) on the container filesystem:

```bash
docker cp ./customers.csv exanano-sqlcube:/tmp/customers.csv
```

```sql
IMPORT INTO ADVENTUREWORKS.CUSTOMERS
FROM LOCAL CSV FILE '/tmp/customers.csv'
SKIP = 1 COLUMN SEPARATOR = ',';
```

`FROM LOCAL` paths are from the container's POV, not the agent host.

## Column inference (when no source DDL available)

Unlike JDBC where source metadata gives us types, CSV is types-on-read. Options:

### Option A: User-supplied DDL (preferred)

Run `CREATE TABLE ... ` first with explicit types, then IMPORT into the resulting table. Most reliable.

### Option B: Sample-and-infer

Read first N rows (default 1000), regex-detect types, generate conservative DDL.

| Pattern in samples | Inferred type |
|---|---|
| All match `^-?\d+$`, max digits ≤ 18 | `DECIMAL(18,0)` |
| All match `^-?\d+\.\d+$` | `DECIMAL(36,18)` |
| All match `^\d{4}-\d{2}-\d{2}$` | `DATE` |
| All match `^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}` | `TIMESTAMP` |
| Mix of `true` / `false` / `1` / `0` only | `BOOLEAN` (warn — fragile) |
| Anything else | `VARCHAR(2000000)` |

Always `VARCHAR(2000000)` for unknowns. `exasol-optimize` will narrow downstream.

## IMPORT FROM CSV reference

Full clause syntax:

```sql
IMPORT INTO <target_schema>.<table> (<col_list>)
FROM CSV
  AT '<url_or_path>'
  USER '<u>' IDENTIFIED BY '<p>'        -- omit for FILE
  FILE '<filename_in_bucket>'           -- when AT points to bucket
  SKIP = 1                              -- header row
  COLUMN SEPARATOR = ','
  COLUMN DELIMITER = '"'                -- quote character
  ROW SEPARATOR = 'LF'                  -- or 'CRLF', 'CR'
  NULL = ''                             -- treat empty string as NULL
  ENCODING = 'UTF-8'
  ERRORS INTO <error_table>             -- recover malformed rows
  REJECT LIMIT UNLIMITED;               -- or a number
```

`ERRORS INTO` is the underused feature: malformed rows land in a side table; the IMPORT still succeeds for the good rows. Use this for any non-trivial CSV — there's always a row with an unescaped comma.

## When NOT to use CSV/S3 path

- **Vendor is in DRIVER_PRESETS** — use the JDBC migration script directly. Faster, type-aware, no intermediate disk.
- **Source has BLOB columns** — Base64-in-CSV is gross; skip those columns at source instead.
- **Transactional snapshot consistency needed across N tables** — source-side CSV export is staggered, snapshot drifts. JDBC migration is more honest.

## Known failures

| Symptom | Cause | Fix |
|---|---|---|
| `Row n: column count mismatch` | Embedded newline in quoted field, parser thinks new row | Set `ROW SEPARATOR = 'CRLF'`; or use `ERRORS INTO` |
| `Conversion error: cannot convert 'N/A' to DECIMAL` | NULL sentinel isn't empty string | Set `NULL = 'N/A'`; or pre-process |
| `File not found` from BucketFS URL | Wrong path or wrong bucket | `exapump bucketfs ls` to enumerate |
| Slow import (~MB/s instead of 100MB/s) | Single-file, no parallelism | Split CSV into N files, IMPORT in a loop |
| S3 auth fails with valid keys | Bucket policy denies the IAM identity | Check bucket policy + ACL; some buckets require explicit `s3:GetObject` grant |

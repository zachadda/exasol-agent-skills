# CSV fallback

When IMPORT FROM JDBC is blocked, the universal fallback is: export source to CSV/Parquet, stage on local disk or BucketFS, then `IMPORT FROM CSV` into Exasol.

When to use this path:

- **Source vendor not yet supported by exasol-migrate** (v0.1 = Snowflake only).
- **Network policy blocks Exasol container's egress** to the source host.
- **Corporate TLS interception** breaks JDBC handshakes that don't accept arbitrary CAs.
- **Source database is a flat-file export** (a vendor handed you `.csv` / `.parquet`, no live system).
- **Reproducible CI fixtures** — checked-in CSV is more stable than a live JDBC dependency.

Slower and lossier than JDBC but works anywhere.

## Pipeline shape

```
1. Export source → CSV (vendor-side; out of skill scope)
2. Stage CSV in BucketFS or local mount
3. Inspect headers + sample types
4. CREATE TABLE target with mapped columns
5. IMPORT INTO target FROM CSV AT '<path>'
6. Verify row counts + manifest entry
```

Steps 4–6 are in-skill; 1–2 are user-provided inputs.

## Staging options

### BucketFS (preferred for large files)

Exasol's built-in object store. Visible from inside the Exasol JVM. Already mounted in `exanano-sqlcube` as `/buckets/bfsdefault/default/`.

```bash
# Upload from host:
exapump bucketfs put -p sqlcube-nano \
  ./customers.csv \
  /default/migrate/customers.csv
```

Then in SQL:

```sql
IMPORT INTO ADVENTUREWORKS.CUSTOMERS
FROM CSV
  AT 'http://localhost:8888/default/migrate/customers.csv'
  USER 'r' IDENTIFIED BY 'read'
  SKIP = 1
  COLUMN SEPARATOR = ','
  COLUMN DELIMITER = '"';
```

BucketFS URL is the canonical reference inside the Exasol JVM. Auth (`r` / `read`) is the default read-only bucket creds in `exanano-sqlcube`.

### Local FILE mount

For tiny CSVs (< 100 MB) and ad-hoc dev:

```bash
docker cp ./customers.csv exanano-sqlcube:/tmp/customers.csv
```

```sql
IMPORT INTO target
FROM LOCAL CSV FILE '/tmp/customers.csv'
SKIP = 1 COLUMN SEPARATOR = ',';
```

`FROM LOCAL` is from the container's POV, not the agent host.

## Format options

| Format | Exasol clause | Notes |
|---|---|---|
| CSV (RFC 4180) | `FROM CSV` | Header on row 1 → `SKIP = 1` |
| TSV | `FROM CSV COLUMN SEPARATOR = '\t'` | |
| Parquet | `FROM FBV` (binary) **NOT supported directly** | Pre-convert to CSV; or use ETL tool |
| Gzip CSV | `FROM CSV AT ... .csv.gz` | Exasol decompresses inline |
| JSON Lines | — | No native support; convert to CSV first |

Parquet is the gap. The factory-foundation roadmap has a "exasol-parquet" skill planned but not v0.1. For now, `duckdb -c "COPY tbl TO 'out.csv' (FORMAT CSV);"` against the parquet file is the cheap converter.

## Column inference (the hard part)

Unlike JDBC where source metadata gives us types, CSV is types-on-read. Two options:

### Option A: User-supplied DDL

Best when you have it. User provides CREATE TABLE; skill runs it as-is, then IMPORT into the resulting table.

### Option B: Sample-and-infer

Read first N rows (default 1000), regex-detect numeric vs date vs text, generate conservative DDL. Inference rules:

| Pattern in samples | Inferred type |
|---|---|
| All match `^-?\d+$`, max digits ≤ 18 | `DECIMAL(18,0)` |
| All match `^-?\d+\.\d+$` | `DECIMAL(36,18)` |
| All match `^\d{4}-\d{2}-\d{2}$` | `DATE` |
| All match `^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}` | `TIMESTAMP` |
| Mix of `true` / `false` / `1` / `0` only | `BOOLEAN` (warn — fragile inference) |
| Anything else | `VARCHAR(2000000)` |

Always `VARCHAR(2000000)` for unknowns. `exasol-optimize` will narrow downstream. Don't gamble on tight types during import.

## IMPORT FROM CSV reference

Full clause:

```sql
IMPORT INTO <target_schema>.<table> (<col_list>)
FROM CSV
  AT '<url_or_path>'
  USER '<u>' IDENTIFIED BY '<p>'        -- omit for FILE
  FILE '<filename_in_bucket>'            -- when AT points to bucket
  SKIP = 1                               -- header row
  COLUMN SEPARATOR = ','
  COLUMN DELIMITER = '"'                 -- quote character
  ROW SEPARATOR = 'LF'                   -- or 'CRLF', 'CR'
  NULL = ''                              -- treat empty string as NULL
  ENCODING = 'UTF-8'
  ERRORS INTO <error_table>              -- recover malformed rows
  REJECT LIMIT UNLIMITED;                -- or a number
```

`ERRORS INTO` is the underused feature: malformed rows land in a side table you can inspect later, the IMPORT still succeeds for the good rows. Use this for any non-trivial CSV — there's always a row with an unescaped comma.

## Sequencing for the migrate pipeline

When `source_vendor` is unsupported OR `auth_mode=csv`:

1. Halt the JDBC path. Tell user: "JDBC not available — falling back to CSV."
2. Prompt for staging location (BucketFS URL, local container path, or host path).
3. For each CSV file (one per table assumed):
   - Inspect headers
   - Sample-infer types (or use user DDL)
   - CREATE TABLE
   - IMPORT FROM CSV with ERRORS INTO
   - Reconcile row counts vs file `wc -l - 1`
4. Manifest entry includes `staging_path`, `rejected_count`, `inferred_types` flag.

## Known failures

| Symptom | Cause | Fix |
|---|---|---|
| `Row n: column count mismatch` | Embedded newline in quoted field, parser thinks new row | Set `ROW SEPARATOR = 'CRLF'` if file is CRLF; else use `ERRORS INTO` and inspect |
| `Conversion error: cannot convert 'N/A' to DECIMAL` | NULL sentinel isn't empty string | Set `NULL = 'N/A'`; or pre-process CSV |
| `File not found` from BucketFS URL | Wrong path or wrong bucket | `exapump bucketfs ls` to enumerate |
| Slow import (~MB/s instead of 100MB/s) | Single-file, no parallelism | Split CSV into N files, IMPORT in a loop |

## When NOT to fall back to CSV

- **You're doing a 10TB migration.** CSV intermediate doubles your disk; do JDBC even if you have to set up Snowflake PrivateLink first.
- **The schema has BLOB columns.** Base64-in-CSV is gross. Better to skip those columns at the source.
- **You need transactional snapshot consistency across N tables.** Source-side CSV export is staggered; the snapshot drifts. JDBC `BEGIN TRANSACTION READ ONLY` (vendor-dependent) is more honest.

---
name: exasol-migrate
description: Migrate a schema from an external RDBMS into Exasol via IMPORT FROM JDBC (preferred) or IMPORT FROM CSV (fallback). Handles connection setup, type mapping, table-by-table import, and verification. v0.1 ships Snowflake; Postgres / MySQL / Redshift / BigQuery follow the same shape via per-vendor reference files.

preconditions:
  - nano_jdbc_dir:
      doc: "JDBC driver for the source vendor is present inside the Exasol container at /exa/jdbc/<VENDOR_UPPER>/."
      check: |
        # Bash:
        # docker exec <container> ls /exa/jdbc/<VENDOR_UPPER>/ | grep -c '\.jar$'
        # Returns >=1 when satisfied.
      satisfied_by: exasol-nano-local   # places the driver, then restarts container
  - source_credentials_available:
      doc: "Credentials for the source database are available in the agent's environment or .env file (vendor-specific variable names — see references/<vendor>.md)."
      check: |
        # Bash, vendor-specific. Snowflake example:
        # [[ -n "$SNOWFLAKE_USER" && -n "$SNOWFLAKE_ACCOUNT" ]] && echo ok
      satisfied_by: null   # user must provide; skill prompts if missing

provides:
  - schema_imported:
      doc: "Target Exasol schema exists and contains the migrated tables. Row counts match source within agreed tolerance (default exact; sample_only=true skips row-count match)."
      verify: |
        # SQL:
        # SELECT COUNT(*) FROM SYS.EXA_ALL_TABLES WHERE TABLE_SCHEMA = '<TARGET_SCHEMA>';
        # Plus per-table SELECT COUNT(*) reconciliation logged to a manifest.

parameters:
  required:
    - source_vendor:
        doc: "One of: snowflake, postgres, mysql, redshift, bigquery. Routes to references/<vendor>.md. v0.1 implements snowflake only."
    - source_schema:
        doc: "Fully-qualified source schema (e.g., `SNOWFLAKE_SAMPLE_DATA.TPCH_SF1` for Snowflake, `public` for Postgres)."
    - target_schema:
        doc: "Exasol schema name to create / populate. Created if missing."
  optional:
    - source_database:
        doc: "Source database/catalog name. Often part of source_schema in dotted form; provide explicitly for vendors with separate database+schema (Snowflake, SQL Server)."
    - jdbc_url_override:
        default: null
        doc: "Skip the vendor reference's default URL template and use this literal JDBC URL. Use when corporate proxy / non-standard port / PrivateLink is involved."
    - auth_mode:
        default: password
        doc: "password | key_pair | oauth | iam. Vendor-supported set differs — see vendor reference."
    - include_tables:
        default: null
        doc: "Comma-separated allowlist. Empty / null = all tables in source schema."
    - exclude_tables:
        default: null
        doc: "Comma-separated denylist. Applied after include_tables."
    - sample_only:
        default: false
        doc: "If true, LIMIT each table to 10000 rows. Used for fast demo paths and structural verification."
    - drop_target:
        default: false
        doc: "If true, DROP SCHEMA <target_schema> CASCADE before recreating. Destructive — agent must confirm with user before setting true."

estimated_impact:
  v01_snowflake_demo: "2–10 min depending on schema size and sample_only flag. Network egress to source. No production writes — Exasol-side schema is created/replaced per drop_target."
  per_table_overhead: "~1 sec setup + IMPORT throughput (typically 50–200 MB/sec wire, vendor-dependent)."
---

# Exasol Migrate Skill

Trigger when the user asks to **migrate**, **import**, **load**, **copy**, **bring in**, **pull**, or **federate** a schema/database/tables from an external RDBMS into Exasol. Also trigger when an orchestrator skill (e.g., `/oneshot-tour`) declares a `schema_imported` precondition with a `source_vendor` parameter.

## What this skill is

A general-purpose migration adapter. Single SKILL.md, shared IMPORT FROM JDBC flow, one reference file per vendor for the bits that diverge (connection string, auth, type quirks). All vendors satisfy the same `schema_imported` provides contract.

Architecture lives in `references/flow.md`. Vendor-specific knobs in `references/<vendor>.md`.

## v0.1 status

| Vendor | Status | Reference |
|---|---|---|
| Snowflake | ✅ Implemented | `references/snowflake.md` |
| Postgres | ⏳ Planned | (stub) |
| MySQL | ⏳ Planned | (stub) |
| Redshift | ⏳ Planned | (stub) |
| BigQuery | ⏳ Planned | (Simba JDBC complexity — may slip to v0.3) |
| CSV fallback | ✅ Implemented | `references/csv-fallback.md` |

Routing: if `source_vendor` matches an implemented vendor → use that reference. If not → halt and tell the user. Don't fabricate connection strings for unimplemented vendors.

## Routing algorithm

| Phase | Reference | When |
|---|---|---|
| Plan + type mapping | `references/flow.md` | Always — shared logic |
| Connection + auth | `references/<source_vendor>.md` | Required, vendor-specific |
| CSV intermediate | `references/csv-fallback.md` | When JDBC blocked (network / TLS / vendor not yet supported) |

## Step 0: Preconditions

Before any other action, verify:

1. **JDBC driver present.** Use the orchestrator pattern — `exasol-nano-local` skill satisfies `nano_jdbc_dir` by placing the jar. If `docker exec <container> ls /exa/jdbc/<VENDOR_UPPER>/` returns nothing, route to `exasol-nano-local` first.

2. **Source credentials available.** Vendor-specific. For Snowflake: `SNOWFLAKE_USER`, `SNOWFLAKE_ACCOUNT`, plus password or key-pair. If missing → prompt user (don't fabricate).

3. **Target schema decision.** If `drop_target=true`, confirm with user — irreversible against the existing target schema.

## Pipeline

```
1. Resolve vendor reference  (references/<source_vendor>.md)
2. Build JDBC URL             (vendor default or jdbc_url_override)
3. Test connection            (IMPORT ... STATEMENT 'SELECT 1' round-trip)
4. Enumerate source tables    (vendor-specific INFORMATION_SCHEMA query)
5. Apply include / exclude    (in-memory filter)
6. Plan + present to user     (table list + estimated rowcount if cheap to compute)
7. CREATE SCHEMA target       (or DROP+CREATE if drop_target=true)
8. For each table:
   a. IMPORT INTO target.T  FROM JDBC ... STATEMENT 'SELECT * FROM source.T'
   b. Capture row count, log to manifest
9. Verify (provides contract)
10. Hand back to caller
```

Each step has explicit error handling in `flow.md`. Per-table failures don't abort the whole migration — collect, report at end, let user decide.

## Conventions

- **IMPORT FROM JDBC over CSV when possible.** Single round-trip, in-Exasol type inference, no intermediate disk. CSV fallback is for blocked-network / unsupported-vendor / corporate-policy cases.
- **One IMPORT statement per table.** Cleaner error attribution, parallelizable. Bulk multi-table imports are an Exasol feature but obscure failure surfaces.
- **Type mapping is vendor-specific but conservative.** Default to widest Exasol type for source numeric ambiguity (e.g., Snowflake NUMBER → Exasol DECIMAL(36,18)). User can override per-column via post-import ALTER if precision matters.
- **Schema-level operations are explicit.** Never DROP SCHEMA without `drop_target=true` AND user confirmation.
- **Manifest is the source of truth.** Every migration emits `<target_schema>__migration_manifest.csv` with table name, source rowcount, target rowcount, status, error message. Used by `provides` verification.

## When to invoke

- `/oneshot-tour` orchestrator declares `schema_imported` precondition with `source_vendor=snowflake`.
- Developer asks "load TPCH from Snowflake into the local Nano."
- CI smoke harness needs a fixture schema staged from an external source.
- Migration audit: re-run with `sample_only=true` to validate connectivity without full re-ingest.

## Related skills

- **exasol-nano-local** — satisfies `nano_jdbc_dir` precondition by placing the driver jar.
- **exasol-optimize** — natural next step after schema import: discover PKs, FKs, narrow types, build join graph.
- **exasol-semantic-layer** — runs after optimize when caller wants a cube on top of the migrated schema.
- **oneshot-tour** — orchestrator that chains nano-local → migrate → optimize → semantic-layer → dashboard.

## Limitations

- **Snapshot, not CDC.** Each invocation is a point-in-time copy. No incremental / streaming / log-based replication.
- **Row-count tolerance is exact by default.** Schemas with active writes during migration will report mismatches. Caller must coordinate write quiesce or accept tolerance.
- **No DDL translation beyond column types.** Triggers, stored procedures, views with vendor-specific SQL are NOT migrated — only base tables. Views can be replicated separately, but the body has to be translated to Exasol SQL (manual or via a future translation skill).
- **Credentials must be in environment.** Skill never reads or prompts for plaintext passwords on stdout. User sets env vars; skill references them.

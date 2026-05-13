# Installing the EXA_OPTIMIZE UDFs

The four optimize UDFs are pure Lua scripts. They live in schema `EXA_OPTIMIZE`. No external dependencies, no SLC build, no BucketFS upload — just `CREATE OR REPLACE SCRIPT` on whatever runtime you're pointed at.

## Source

Until an extracted Labs repo exists, the canonical source is `exasol-factory-foundation` (the workspace repo where these UDFs were authored):

```
exasol-factory-foundation/studio/backend/fixtures/optimization-scripts/
  analyze_constraints.sql
  infer_join_paths.sql
  build_unknown_member_insert.sql
  dry_run_plan.sql
  validate_join_plan.sql
  post_load_set_primary_keys.sql
  post_load_convert_datatypes.sql
  post_load_convert_varchar.sql
  generate_dim_date.sql
  analyze_dim_date.sql
```

The first four files are the public surface this skill describes. `validate_join_plan.sql` is the internal dependency of `dry_run_plan.sql`; install it too. The `post_load_*` and `generate_dim_date` scripts are adjacent helpers — install only if you're using the post-load pipeline.

**`post_load_set_primary_keys.sql` is remote-only.** The UDF (`SET_PRIMARY_AND_FOREIGN_KEYS`) reads PK / FK metadata via `IMPORT FROM ... AT <CONNECTION>` over JDBC and applies the constraints to the target schema. It requires a working JDBC `CONNECTION` to a *source* database (`ORACLE`, `MYSQL`, `SQLSERVER`, `POSTGRES`, or `EXASOL`). Self-loopback against the local Nano hangs because the JDBC driver isn't configured for IMPORT against the same instance. Smoke coverage is therefore documentation-only — call sites in production should ensure the JDBC connection is reachable from the Exasol database (not the studio backend).

When a public `exasol-labs/exasol-optimizer-udfs` repo lands, this section will point there instead.

## Install via the Studio backend `/api/scripts/install`

The studio backend parses each fixture file's `--/ … /` script blocks and submits each `CREATE OR REPLACE SCRIPT` body whole, which is what Exasol needs for Lua scripts with `;` chars in the body. This is the only path tested end-to-end.

```bash
curl -X POST http://localhost:8000/api/scripts/install \
  -H 'Content-Type: application/json' \
  -d '{"fixture_relpath":"optimization-scripts/analyze_constraints.sql"}'
```

Repeat for each script. Each file is idempotent (`create schema if not exists exa_optimize;` + `create or replace script …`).

### Why not exapump

`exapump sql -f <file>` splits the input on `;`. Lua script bodies use `;` extensively, so the SQL splitter shreds the script and silently runs only the preamble (`create schema if not exists exa_optimize`). The deployed catalog ends up missing the UDF or carrying a stale version, and there's no error to alert you. Use the `/api/scripts/install` path, or a pyexasol script that submits each `CREATE SCRIPT` body as a single statement.

## Install via SQL paste

If you don't have local file access — typical when working through a coding agent against a remote cluster — paste each script body via `exapump sql --profile <profile>` directly. Wrap each `CREATE OR REPLACE SCRIPT … / `  in its own invocation (Lua scripts can't be batched with `;` because the script body itself uses `;`).

## Verify installation

```sql
SELECT "SCRIPT_NAME"
FROM SYS.EXA_ALL_SCRIPTS
WHERE "SCRIPT_SCHEMA" = 'EXA_OPTIMIZE'
ORDER BY "SCRIPT_NAME";
```

Expect at minimum the five core scripts:
- `ANALYZE_CONSTRAINTS`
- `BUILD_UNKNOWN_MEMBER_INSERT`
- `DRY_RUN_PLAN`
- `INFER_JOIN_PATHS`
- `VALIDATE_JOIN_PLAN`

If the post-load and date-dim helpers are also installed:
- `CONVERT_VARCHAR` · `CONVERT_DATATYPES` (advisory ALTER proposals)
- `GENERATE_DIM_DATE` · `ANALYZE_DIM_DATE` (date-dim materializer + attribute scorer)
- `SET_PRIMARY_AND_FOREIGN_KEYS` (remote-only post-load PK/FK importer)

## Privilege requirements

The user invoking the UDFs needs:

- `SELECT ANY DICTIONARY` (to read `SYS.EXA_ALL_*` catalog views inside the scripts).
- `SELECT` on the target schema (the schema being analyzed).
- For `DRY_RUN_PLAN`: `ALTER` / `INSERT` / `UPDATE` on the target schema, since the script issues those DDL/DML statements directly via `pquery`.

For installation:
- `CREATE SCRIPT` on the `EXA_OPTIMIZE` schema (or `CREATE ANY SCRIPT`).
- `CREATE SCHEMA` if `EXA_OPTIMIZE` doesn't exist yet — each script's first line creates it.

A typical setup grants the install user `SYSTEM PRIVILEGES` once, then individual analysts use a lower-privilege role to invoke.

## Local Nano vs. AWS cluster

Both targets work. The smoke harness in `exasol-factory-foundation/scripts/optimize_torture_smoke.py` defaults to `--target nano` and supports `--target aws` for manual verification. Identical SQL, identical UDF behavior. The harness uses the same `exapump`-profile resolution pattern as `optimize_e2e_smoke.py`.

For a fresh local Nano, the docs at `exasol-factory-foundation/CURRENT_STATE.md` describe the Docker boot path (`docker start exanano-sqlcube`). The Java workaround for ExaNano UDF execution is applied once at provision time.

## Reinstall after schema changes

If you change the UDF source and want the new version live:

```sql
-- Old version is replaced atomically by CREATE OR REPLACE.
-- No need to DROP first.
-- All sessions immediately see the new script body on next call.
```

The compiled Lua bytecode is regenerated on the next invocation. There's no warm-up cost worth worrying about.

## Uninstall

```sql
DROP SCRIPT "EXA_OPTIMIZE"."ANALYZE_CONSTRAINTS";
DROP SCRIPT "EXA_OPTIMIZE"."INFER_JOIN_PATHS";
DROP SCRIPT "EXA_OPTIMIZE"."BUILD_UNKNOWN_MEMBER_INSERT";
DROP SCRIPT "EXA_OPTIMIZE"."DRY_RUN_PLAN";
DROP SCRIPT "EXA_OPTIMIZE"."VALIDATE_JOIN_PLAN";
DROP SCHEMA "EXA_OPTIMIZE";   -- once all scripts in it are gone
```

## Versioning

There is no formal versioning today. The source-of-truth is whichever commit on `workbench-shell` of `exasol-factory-foundation` you cloned. The smoke harness asserts the contract for the version shipped from that commit. Pin a SHA when you install on a production cluster.

When the public `exasol-labs/exasol-optimizer-udfs` repo lands, semver tags will replace SHA pinning.

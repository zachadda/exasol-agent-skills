# SQLCube infrastructure install

When `sqlcube_installed` precondition fails (one of: `SQLCUBE.ADAPTER` UDF missing, registry schema missing, registry tables missing), the skill can't proceed. This reference covers what's needed and where to find the canonical install scripts.

This skill **does not automate the install** — it requires deploying a Lua adapter UDF to the Exasol cluster, uploading the bundled adapter to BucketFS, and running DDL. Too many environment-specific knobs (BucketFS auth, cluster privileges, schema name convention) to safely automate.

## Canonical source

Lives in `factory-foundation/sqlcube/`:

```
sqlcube/
├── adapter/
│   ├── build_adapter.py        # concatenates Lua modules into sqlcube_adapter.lua
│   ├── sqlcube_adapter.lua     # built artifact (~106 KB)
│   ├── main.lua                # adapter entry point
│   ├── metadata_registry.lua   # reads SQLCUBE_REGISTRY at query time
│   ├── common.lua, resolver.lua, measure_builder.lua, ...
└── meta/
    ├── 01_create_tables.sql    # SQLCUBE_REGISTRY DDL
    ├── 02_awd_fixture.sql      # AdventureWorks demo seed
    ├── 03_create_virtual_schema.sql
    └── 04_tpch_demo_models.sql # TPC-H demo seed
```

Treat `01_create_tables.sql` as ground truth for table shapes — `references/meta-model.md` mirrors it but may drift over time.

## Install steps (one-time per Exasol cluster)

```bash
# 1. Build the adapter Lua bundle
cd /path/to/exasol-factory-foundation/sqlcube/adapter
python3 build_adapter.py
# Output: sqlcube_adapter.lua (single concatenated file)

# 2. Upload to BucketFS (assumes default bucket on exanano-sqlcube)
exapump bucketfs put -p sqlcube-nano \
  ./sqlcube_adapter.lua /default/sqlcube_adapter.lua

# 3. Create the registry tables
exapump sql -p sqlcube-nano -f /path/to/sqlcube/meta/01_create_tables.sql

# 4. Create the adapter script
exapump sql -p sqlcube-nano <<'SQL'
CREATE SCHEMA IF NOT EXISTS SQLCUBE;
CREATE OR REPLACE LUA ADAPTER SCRIPT SQLCUBE.ADAPTER AS
    %run /buckets/bfsdefault/default/sqlcube_adapter.lua;
/
SQL
```

After step 4, the precondition check passes:

```sql
SELECT 1 FROM SYS.EXA_ALL_SCRIPTS
WHERE SCRIPT_SCHEMA='SQLCUBE' AND SCRIPT_NAME='ADAPTER';   -- returns 1

SELECT 1 FROM SYS.EXA_ALL_TABLES
WHERE TABLE_SCHEMA='SQLCUBE_REGISTRY' AND TABLE_NAME='MODELS';  -- returns 1
```

## What `exanano-sqlcube` ships with

The pre-provisioned dev container has all of the above already installed. The skill's precondition check passes out-of-the-box.

If a user wipes the container's volume (`docker volume rm exanano_sqlcube`), the install must be repeated. The `factory-foundation` repo's container build / setup scripts handle this automatically — see `exasol-factory-foundation/docs/ops/exanano-local-runtime.md`.

## Customizing the registry schema name

The default schema name `SQLCUBE_REGISTRY` is configurable via `settings.exa_registry_schema` in the studio backend. To change:

1. **Pick a new name** (e.g., `SEMANTIC_REGISTRY`).
2. **Edit `01_create_tables.sql`** (or the equivalent `build_registry_ddl()` output) to use the new name.
3. **Update `settings.exa_registry_schema`** in the studio backend config so deploys read/write the right schema.
4. **Rebuild the adapter** if the registry schema is referenced inside the Lua (currently it isn't — adapter receives schema name as an adapter parameter, but verify with `metadata_registry.lua`).
5. **Re-deploy**: drop old `SQLCUBE_REGISTRY`, create new schema, re-populate by re-running each cube's deploy.

This rename is currently a manual operation. There's no automated migration tool.

## Why install is not automated

Three reasons:

- **Adapter deployment.** Requires Lua + UDF privileges. Many clusters limit who can deploy adapter scripts.
- **BucketFS auth.** Default `r/read` credentials work on `exanano-sqlcube` but are wrong for production clusters. Auto-detecting the right auth is brittle.
- **Schema name choice.** If someone is intentionally diverging from `SQLCUBE_REGISTRY` (rename in progress), automation would silently overwrite their choice.

Manual is safer for v0.1.

## What the agent should do

When the precondition fails:

1. **Stop and report.** Don't try to install headlessly.
2. **Tell the user the install path:**
   ```
   SQLCube infrastructure missing. Install required.
   See: exasol-factory-foundation/sqlcube/ + docs/ops/exanano-local-runtime.md
   Or: re-create the exanano-sqlcube container if you wiped its volume.
   ```
3. **Offer to walk through it interactively** if asked.
4. **After install confirmed** → re-run skill's preconditions; should pass.

## Adapter compatibility

The adapter version (in `sqlcube_adapter.lua`) and the registry schema version (in `01_create_tables.sql`) must move together. Some changes have happened historically:

- Adding `IS_VISIBLE` to ATTRIBUTES (DDL has `ALTER TABLE ADD COLUMN IF NOT EXISTS` to handle the migration idempotently)
- Adding `OWNER_NAME`, `OWNER_TYPE`, `IS_SYSTEM_MANAGED` to MODELS
- Adding `DISPLAY_TYPE` and `DISPLAY_SCALE` to ATTRIBUTES

The `build_registry_ddl()` function in `sqlcube_builder.py` runs all the ALTER statements at deploy time, so an older registry survives a newer adapter. But the inverse — newer registry on older adapter — silently breaks because the adapter doesn't know to read the new columns.

If you're on `exanano-sqlcube` you don't need to think about this. If you've cloned the registry to a different cluster, redeploy the adapter at the same time.

## Custom RCLS (row/column-level security)

The registry includes RCLS_ROW_POLICIES and RCLS_ATTRIBUTE_POLICIES but **this skill does not populate them** in v0.1. They're available for manual / future-skill use.

If a downstream skill needs RCLS, it should:

1. INSERT into RCLS_ROW_POLICIES / RCLS_ATTRIBUTE_POLICIES with the appropriate MODEL_ID, SUBJECT_TYPE, SUBJECT_NAME, predicate.
2. NOT touch the rest of the registry (this skill manages those rows).
3. Coordinate with the cube refresh — adapter reads RCLS at every query, so changes are immediate (no virtual-schema rebuild needed).

Reference adapter source: `metadata_registry.lua` for which RCLS columns are read.

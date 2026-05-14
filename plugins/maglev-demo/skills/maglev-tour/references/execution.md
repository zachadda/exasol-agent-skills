# Phase 3 — Execution

Run the 8 steps in order. Hand each off to its owning skill. Narrate via TaskCreate + TaskUpdate. Halt on first verify failure.

## Step → endpoint → narration shape

### Step 1 — Target (activate Exasol profile)

```
GET  /api/connections/profiles
POST /api/connections/profiles/<id>/activate
```

Narration:
```
### Step 1 of 8 — Target (Exasol connection)
  tool: list profiles → 3 active (ACME Production / AWS / Cloud)
  tool: activate <id>
  ✓ pyexasol connection to localhost:8568, persona=admin, sys role
```

### Step 2 — Source (Snowflake JDBC test)

```
POST /api/import/jdbc/test-connection
Body: { "connection_name": "SNOWFLAKE_CONNECTION" }
```

Backend issues `IMPORT FROM JDBC AT "SNOWFLAKE_CONNECTION" STATEMENT 'SELECT 1'` — first call ~6s for OCSP cache + session, subsequent fast. Narration mentions this so the user doesn't think we hung.

### Step 3 — Schema (Snowflake browse)

```
GET /api/import/jdbc/databases?connection_name=SNOWFLAKE_CONNECTION&source_type=snowflake
GET /api/import/jdbc/schemas?connection_name=SNOWFLAKE_CONNECTION&database_name=ACME_DEMO&source_type=snowflake
```

Query params per `MAGLEV_CLI_HANDOFF.md` — `database_name` (not `database`), and `source_type=snowflake` is required.

Server-side IMPORT FROM JDBC against Snowflake INFORMATION_SCHEMA. Used to populate the picker; the agent locks `database=ACME_DEMO`, `schema=ADVENTUREWORKS` per parameters.

### Step 4 — Migrate (Snowflake → Exasol)

Four-phase per GUI (do not run the script's EXECUTE mode — it serializes tables and blows past the 15s target):

```
1. POST /api/import/snowflake/preview-migration
   Body: {
     connection_name, db_filter, schema_filter, target_schema,
     execution_mode: "EXECUTE",           // emits plan_sql in DEBUG form regardless
     identifier_case_insensitive: true,
     parallel_connections: "1",           // string, but Lua needs numeric → backend special-cases "1" / "AUTO"
     db2schema: false
   }
   → returns {bootstrap_sql, connection_test_sql, migration_sql, plan_sql}

2. POST /api/import/snowflake/run-sql {sql: bootstrap_sql}
   → creates EXA_DB_MIGRATION schema

3. POST /api/query/execute {sql: plan_sql}
   → DEBUG-mode EXECUTE SCRIPT returns rows: (SQL_TEXT, SUCCESS, ERROR_MESSAGE) per CREATE/IMPORT
   → NOTE: `/api/query/execute` is required here, not `/run-sql` — run-sql doesn't return rows
     from EXECUTE SCRIPT, only success/failure (per MAGLEV_CLI_HANDOFF.md gotcha)

4. Split rows into CREATE statements (CREATE SCHEMA / CREATE TABLE / ALTER / COMMENT) and
   IMPORT statements (IMPORT FROM JDBC).

5. Run CREATEs serially via POST /api/import/snowflake/run-sql {sql}
   → CREATE SCHEMA + CREATE TABLE + comments

6. Fan out IMPORTs in parallel via concurrent POST /api/import/snowflake/run-sql {sql}
   → one HTTP call per table; backend opens fresh Snowflake JDBC session per call.
   → In Python: concurrent.futures.ThreadPoolExecutor or httpx.AsyncClient.
```

Perf target: ~15s for 84k rows on JDBC 3.20 + Arrow. Narration ticks per-table as rows complete. Verify post-migrate:

```sql
SELECT TABLE_NAME, TABLE_ROW_COUNT
FROM SYS.EXA_ALL_TABLES
WHERE TABLE_SCHEMA = 'ACME_ADVENTUREWORKS'
ORDER BY 1;
-- Expect 9 tables, ~84k total rows.
```

First Snowflake call is always slow (~6s for OCSP cache + session warmup). Mention once, don't keep apologizing.

### Step 5 — Optimize (UDFs + two-pass)

```
POST /api/optimize/analyze
Body: { "schema_name": "ACME_ADVENTUREWORKS", "categories": ["constraints"] }
→ findings rows with sql_steps[].sql

POST /api/optimize/run-sql per finding
→ applies the ALTER TABLE (idempotent; catches "constraint already used")

POST /api/optimize/analyze  (second pass)
→ newly-validated FKs emerge once dim PKs exist

POST /api/optimize/run-sql per new finding

POST /api/optimize/analyze categories=["date_dim"]
→ may run ANALYZE_DIM_DATE for calendar enrichment
```

**Two-pass is required** — the UDF's value-match only confirms FKs against dims that already have PKs. First pass adds PKs + ~3 high-confidence FKs (PRODUCT, PROMOTION, SALESTERRITORY in ADW) + unknown-member INSERTs. Second pass surfaces the remaining ~5 FKs (CUSTOMERKEY, ORDERDATEKEY, DUEDATEKEY, SHIPDATEKEY, CURRENCYKEY) once dim PKs exist for value-matching. Without the second pass: DIMSALESTERRITORY can stay un-joined → Sales Territory Country attr won't be in the cube → Step 7 RCLS predicate references a name the adapter can't translate → Step 8 fails with `object "Sales Territory Country" not found`. Documented in `exasol-zemantic-layer/MAGLEV_RCLS_DEEP_DIVE.md` root-cause #1.

**Idempotent ADD CONSTRAINT** — re-runs hit "constraint name already used" errors. Catch the error, `DROP CONSTRAINT <name>` by parsing the name from the error message, retry ADD. The GUI's OptimizeStepContainer does this; CLI/agent path must match.

**Pass-2 apply errors are noisy — classify them.** Live-verified during cold-start rehearsal 2026-05-14: pass 2 returned 32 findings, only 9 cleanly applied. The 23 errors split into two categories:

- **"constraint name already used"** — idempotent collision. Drop + retry, then count as success.
- **"constraint violation - foreign key (FK_<NAME> on table <TABLE>)"** — data quality issue. The UDF is being optimistic; the proposed FK fails value-match because actual rows reference dim keys that don't exist (or value-match thresholds were marginal on pass 1 and crossed on pass 2). **Skip these silently** — the FK that would have landed wasn't justified by the data. The cube introspection will still see the FKs that DID land.

Heuristic: catch errors whose message contains `constraint violation - foreign key`, log at INFO not ERROR, continue. Don't surface to the user as "Optimize failed" — Optimize succeeded for every finding the data actually supports. Demo punchline still lands because DIMSALESTERRITORY's FK is high-confidence and consistently passes value-match.

AI "thinking" stream (8s) renders fake recommendations from `factoryDemoLLM.js` in the GUI. **The agent SHOULD run a real LLM enrichment narration here** — not pre-baked text — since the agent IS the LLM. The UDF findings are real; the narration just summarizes them.

### Step 6 — Cube (full deploy)

Six sub-calls in order. Each is owned by `exasol-semantic-layer` — see its `references/deploy-flow.md` for the full HTTP contract:

```
1. GET  /api/sqlcube/introspect-sql?schema_name=ACME_ADVENTUREWORKS
   → { sql: "<UNION-ALL SELECT>" }

2. POST /api/query/execute   (run the introspection SQL)
   → rows

3. POST /api/sqlcube/introspect-from-rows {schema_name, rows}
   → ParsedDDL

4. POST /api/sqlcube/generate
   Body: {
     parsed_ddl,
     view_strategy: "native",
     llm_enrichment: false,
     grain_options: {
       grain_profile: "atomic_plus_rollups",
       row_grain: "transaction_row",
       date_grain: "month",
       include_entity_rollups: true,
       business_process: ""
     }
   }
   → SqlcubeModel with 4 domains (atomic + by_month + by_product + by_month_product), ~140 attrs

5. **Sanitize attributes** (client-side step — matches GUI `sanitizeGeneratedModelAttributes` in `studio/frontend/src/components/panels/CubeCreatorPanel.jsx:1953`).
   For each `metric_agg` attribute: if `formula` is a bare aggregate keyword (`"SUM"`, `"COUNT"`, etc.), move it into `agg_function` and null the formula. If `formula` is a simple fact-column reference (`"f.SALESAMOUNT"`), move it into `physical_col`.
   For each `metric_calc` attribute with a bare-aggregate `formula`: convert to `metric_agg` if there's a `physical_col`, else **drop the attribute** with a `dropped_bare_metric_calc` warning.
   Returns `{attributes, warnings, infos}`. The agent must apply this BEFORE metric-pack — otherwise validator at deploy-time will reject bare-aggregate formulas.

6. POST /api/sqlcube/apply-metric-pack    (** Maglev Tour-specific **)
   Body: { source_schema, domains, attributes, pack_name: null }
   → auto-detects 'adventureworks' pack when any `domain_id` contains `internet_sales`, adds 4 attrs per domain (`gross_profit`, `gross_margin_pct`, `distinct_customer_count`, `distinct_product_count`) + replaces UNIT_PRICE-family metric_agg with weighted metric_calc. All injected attrs land with `is_visible=true` (overrides validator ranking). For 4 domains: +16 net new attrs.

7. POST /api/deploy/execute
   Body: {
     source_schema,
     domains,
     attributes,
     join_paths,
     owner_name: "admin",
     owner_type: "STUDIO_USER",
     is_system_managed: false
   }
   → success: true, virtual_schema_name: "SQLCUBE_ACME_ADVENTUREWORKS"
   → backend creates SQLCUBE_REGISTRY tables if missing, INSERTs MODELS/DIMENSIONS/MEASURES/JOINS,
     deploys adapter via CREATE OR REPLACE LUA ADAPTER SCRIPT SQLCUBE.ADAPTER, runs
     CREATE VIRTUAL SCHEMA "SQLCUBE_<TARGET>" USING SQLCUBE.ADAPTER.
```

Step 5 above is **demo-specific** — `services/metric_packs.py` auto-detects `internet_sales` domain and injects the pack. For non-ADW schemas this is a no-op.

### Step 7 — Security (RCLS seed)

Hand off to `exasol-rcls` skill in this same plugin.

```
POST /api/browser/security/seed-demo
Body: { "domain_id": "internet_sales" }
→ creates RCLS_CANADA / RCLS_EUROPE / RCLS_EXEC users (idempotent)
→ grants USAGE + SELECT on virtual schema
→ grants SELECT on ALL 8 SQLCUBE_REGISTRY tables to each demo user (CRITICAL — adapter
   loads policies AS the impersonated user; without grants the read returns empty and
   RCLS silently falls through)
→ INSERTs 4 row policies
→ ALTER VIRTUAL SCHEMA REFRESH (adapter cache invalidate)
```

Three-patch RCLS history (`exasol-zemantic-layer/MAGLEV_RCLS_DEEP_DIVE.md`):
- **Patch A** (Step 5 above) — two-pass ANALYZE_CONSTRAINTS so DIMSALESTERRITORY lands in the cube
- **Patch B** (this step) — registry-table grants per demo user (was the silent-failure bug)
- **Patch C** (this step) — `ALTER VIRTUAL SCHEMA <name> REFRESH` after INSERTs so adapter drops cached `adapterNotes`

All three must hold for Step 8 to demonstrate RCLS. Skip Patch A and Step 8 errors with "Sales Territory Country not found". Skip Patch B and all four personas return identical row counts. Skip Patch C and the new policies are written but not effective until next manual REFRESH.

### Step 8 — Query (persona showcase)

Hand off to `exasol-query-personas` skill in this same plugin. See `references/persona-showcase.md` for the canonical render.

## Halt-gate checklist

| Between | Verify | If fails |
|---|---|---|
| 1→2 | studio backend `/api/auth/me` returns 200 | Halt: ask user to start backend on 8002 |
| 2→3 | jdbc/test-connection success=true | Halt: Snowflake creds expired or network blocked |
| 4→5 | target schema has expected table count (9 for ADW) | Halt: migration aborted mid-way; inspect JOB_LOG |
| 5→6 | SYS.EXA_ALL_CONSTRAINTS has FOREIGN KEY rows for DIMSALESTERRITORY | Halt: two-pass optimize didn't pick up the FK; RCLS will break later |
| 6→7 | `SELECT 1 FROM SYS.EXA_VIRTUAL_SCHEMAS WHERE SCHEMA_NAME=:target_cube_name` | Halt: deploy/execute failed |
| 7→8 | `SELECT COUNT(*) FROM SQLCUBE_REGISTRY.RCLS_ROW_POLICIES WHERE MODEL_ID=:model_id` >= 4 | Halt: seed-demo didn't write policies |
| 8 final | persona query against RCLS_CANADA returns 1 row, against SYS returns 6 | Halt: RCLS predicate not firing — most likely registry grants missing or adapter cache stale |

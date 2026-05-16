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

### Step 5 — Optimize (PROPOSE/APPLY + two-pass)

Tier 2 replaced the `analyze` + `run-sql` orchestration with stable-id PROPOSE/APPLY UDF pairs. PROPOSE returns proposal rows with `proposal_id`; APPLY consumes id lists and is idempotent + self-correcting (catalog disagreement triggers internal `DROP + ADD`, no client-side retry needed).

```
# Pass 1 — propose + apply for the 4 categories
POST /api/optimize/propose-fk
Body: { "schema_name": "ACME_ADVENTUREWORKS" }
→ { "proposals": [{ proposal_id, fact_table, fact_col, ref_table, ref_col,
                    total_score, action_kind, rationale_text, ... }, ...] }

POST /api/optimize/apply-fk
Body: { "schema_name": "ACME_ADVENTUREWORKS",
        "proposal_ids": ["<id1>", "<id2>", ...] }
→ { "audit": [{ proposal_id, action_name, result_flag, elapsed_ms,
                sql_executed, error_message }, ...] }

POST /api/optimize/propose-dim-date  + /api/optimize/apply-dim-date
POST /api/optimize/propose-unknown-member + /api/optimize/apply-unknown-member

# Pass 2 — repeat propose-fk + apply-fk
POST /api/optimize/propose-fk
→ second-pass proposals (FKs whose dim PK only existed after pass-1 APPLY)
POST /api/optimize/apply-fk
→ idempotent NOOP for already-applied; ADD for newly-justified

# Optional — join-path inference once FKs land
POST /api/optimize/propose-join-paths + /api/optimize/apply-join-paths
```

**Two-pass is still required** — PROPOSE_FK value-match only confirms FKs against dims that already have PKs. First pass adds PKs + ~3 high-confidence FKs (PRODUCT, PROMOTION, SALESTERRITORY in ADW) + unknown-member INSERTs. Second pass surfaces the remaining ~5 FKs (CUSTOMERKEY, ORDERDATEKEY, DUEDATEKEY, SHIPDATEKEY, CURRENCYKEY) once dim PKs exist for value-matching. Without the second pass: DIMSALESTERRITORY can stay un-joined → Sales Territory Country attr won't be in the cube → Step 7 RCLS predicate references a name the adapter can't translate → Step 8 fails with `object "Sales Territory Country" not found`. Documented in `exasol-maglev/MAGLEV_RCLS_DEEP_DIVE.md` root-cause #1.

**Idempotency is server-side now** — `APPLY_FK_PROPOSALS` emits audit rows with `action_name = NOOP | ADD | DROP_AND_ADD` and `result_flag = OK | SKIPPED | ERROR`. Re-runs return `NOOP/SKIPPED` for already-applied proposals. The legacy client-side "catch constraint-name-collision → DROP + retry" loop is no longer needed for proposal-shape findings; the UDF handles catalog disagreement internally.

**Apply audit rows — classify them.** Each audit row carries the per-proposal outcome:

- `result_flag = OK` → applied successfully (ADD or DROP_AND_ADD)
- `result_flag = NOOP/SKIPPED` → already converged or proposal not applicable
- `result_flag = ERROR` + `error_message` contains `constraint violation - foreign key` → data quality issue. UDF was optimistic; proposed FK fails value-match because actual rows reference dim keys that don't exist. **Skip silently** — the FK that would have landed wasn't justified by the data.
- `result_flag = ERROR` otherwise → surface to user; halt or downgrade per discretion

Heuristic: count OK + NOOP + SKIPPED as success. Filter ERROR rows by `error_message` substring; demote `constraint violation - foreign key` to INFO. Don't surface as "Optimize failed" — Optimize succeeded for every proposal the data actually supports. Demo punchline still lands because DIMSALESTERRITORY's FK is high-confidence.

**Batching helper** — the frontend exposes `applyFindings(schemaName, findings)` in `studio/frontend/src/api/lattice.js` that groups findings by `proposal_kind` and fires the matching APPLY in one call per kind. The agent path can either call the per-kind endpoints directly or reuse the helper's grouping logic. Findings without `proposal_id` / `proposal_kind` (legacy shape) still need the old per-SQL apply via `runOptimizeSql` — but the post-Tier-2 backend doesn't emit those for the categories above.

**Legacy `/api/optimize/analyze` + `/api/optimize/run-sql`** — still mounted for back-compat, but the post-Tier-2 tour does NOT use them. They will be removed once every consumer flips. `/api/lattice/optimize/*` (pre-rename intermediate prefix) returns 410 Gone with redirect to `/api/optimize/*`.

AI "thinking" stream (8s) renders fake recommendations from `factoryDemoLLM.js` in the GUI. **The agent SHOULD run a real LLM enrichment narration here** — not pre-baked text — since the agent IS the LLM. The UDF findings are real; the narration just summarizes them.

### Step 6 — Cube (full deploy)

Six sub-calls in order. Each is owned by `exasol-semantic-layer` — see its `references/deploy-flow.md` for the full HTTP contract:

Tier 2 Phase B.2 renamed the customer-facing surface from `/api/sqlcube/*` to `/api/lattice/*`, and the deployed artifacts from `SQLCUBE.ADAPTER` / `SQLCUBE_<TARGET>` to `LATTICE.ADAPTER` / `LATTICE_<TARGET>`. `SQLCUBE_REGISTRY` substrate tables keep their original names per plan §Non-Goals.

```
1. GET  /api/lattice/introspect-sql?schema_name=ACME_ADVENTUREWORKS
   → { sql: "<UNION-ALL SELECT>" }

2. POST /api/query/execute   (run the introspection SQL)
   → rows

3. POST /api/lattice/introspect-from-rows {schema_name, rows}
   → ParsedDDL

4. POST /api/lattice/generate
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
   → LatticeModel with 4 domains (atomic + by_month + by_product + by_month_product), ~140 attrs

5. **Sanitize attributes** (client-side step — matches GUI `sanitizeGeneratedModelAttributes` in `studio/frontend/src/components/panels/CubeCreatorPanel.jsx:1953`).
   For each `metric_agg` attribute: if `formula` is a bare aggregate keyword (`"SUM"`, `"COUNT"`, etc.), move it into `agg_function` and null the formula. If `formula` is a simple fact-column reference (`"f.SALESAMOUNT"`), move it into `physical_col`.
   For each `metric_calc` attribute with a bare-aggregate `formula`: convert to `metric_agg` if there's a `physical_col`, else **drop the attribute** with a `dropped_bare_metric_calc` warning.
   Returns `{attributes, warnings, infos}`. The agent must apply this BEFORE metric-pack — otherwise validator at deploy-time will reject bare-aggregate formulas.

6. POST /api/lattice/apply-metric-pack    (** Maglev Tour-specific **)
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
   → success: true, virtual_schema_name: "LATTICE_ACME_ADVENTUREWORKS"
   → backend creates SQLCUBE_REGISTRY tables if missing (registry name unchanged per Tier 2 §Non-Goals),
     INSERTs MODELS/DIMENSIONS/MEASURES/JOINS, deploys adapter via CREATE OR REPLACE LUA ADAPTER SCRIPT
     LATTICE.ADAPTER, runs CREATE VIRTUAL SCHEMA "LATTICE_<TARGET>" USING LATTICE.ADAPTER.
```

**Legacy `/api/sqlcube/*` returns 410 Gone** with a detail message pointing at `/api/lattice/<rest>`. If migrating an existing demo-tour fixture, drop any pre-Tier-2 `SQLCUBE.ADAPTER` adapter script + `SQLCUBE_<source>` virtual schemas first — coexistence is not supported per the recorded Tier 2 spec.

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

Three-patch RCLS history (`exasol-maglev/MAGLEV_RCLS_DEEP_DIVE.md`):
- **Patch A** (Step 5 above) — two-pass `PROPOSE_FK` + `APPLY_FK_PROPOSALS` so DIMSALESTERRITORY lands in the cube (pre-Tier-2: two-pass `ANALYZE_CONSTRAINTS`)
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

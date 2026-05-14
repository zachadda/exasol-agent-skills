# Phase 3 — Execution

Run the 8 steps in order. Hand each off to its owning skill. Narrate via TaskCreate + TaskUpdate. Halt on first verify failure.

## Step → endpoint → narration shape

### Step 1 — Target (activate Exasol profile)

```
GET  /api/connection/profiles
POST /api/connection/profiles/<id>/activate
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
GET /api/import/jdbc/databases?connection_name=SNOWFLAKE_CONNECTION
GET /api/import/jdbc/schemas?connection_name=SNOWFLAKE_CONNECTION&database=ACME_DEMO
```

Server-side IMPORT FROM JDBC against Snowflake INFORMATION_SCHEMA. Used to populate the picker; the agent locks `database=ACME_DEMO`, `schema=ADVENTUREWORKS` per parameters.

### Step 4 — Migrate (Snowflake → Exasol)

Two-phase per GUI:

```
1. POST /api/import/snowflake/preview-migration
   Body: {connection_name, db2schema=false, db_filter, schema_filter, target_schema, execution_mode=DEBUG, ...}
   → returns {bootstrap_sql, connection_test_sql, migration_sql (DEBUG), plan_sql}

2. POST /api/query/execute  (the DEBUG plan)
   → returns rows: (SQL_TEXT, SUCCESS, ERROR_MESSAGE) per generated CREATE / IMPORT

3. POST /api/import/snowflake/run-sql  (serial CREATE statements)
   → each CREATE SCHEMA / CREATE TABLE runs in order

4. Promise.all of POST /api/import/snowflake/run-sql  (parallel IMPORTs)
   → one HTTP call per table; backend opens fresh Snowflake JDBC session per call
```

Perf target: ~15s for 84k rows on JDBC 3.20 + Arrow. Narration ticks per-table as rows complete (poll JOB_DETAILS or accept the GUI pattern of optimistic completion).

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

**Two-pass is required** — the UDF's value-match only confirms FKs against dims that already have PKs. First pass adds PKs + unknown-member INSERTs; second pass picks up the FKs those new PKs unlock. Without this, DIMSALESTERRITORY FK won't land and RCLS at Step 7 silently breaks.

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
   Body: { parsed_ddl, view_strategy="native", llm_enrichment=false, grain_options={"grain_profile":"atomic_plus_rollups"} }
   → SqlcubeModel with 4 domains (atomic + by_month + by_product + by_month_product)

5. POST /api/sqlcube/apply-metric-pack    (** Factory Tour-specific **)
   Body: { model, parsed_ddl }
   → SqlcubeModel with ADW pack injected (Gross Profit / Gross Margin Pct / Distinct Customer / Distinct Product), is_visible flipped to TRUE

6. POST /api/deploy/execute (the enriched model)
   → success: true, virtual_schema_name: "SQLCUBE_ACME_ADVENTUREWORKS"
```

Step 5 above is **demo-specific** — `services/metric_packs.py` auto-detects `internet_sales` domain and injects the pack. For non-ADW schemas this is a no-op.

### Step 7 — Security (RCLS seed)

Hand off to `exasol-rcls` skill in this same plugin.

```
POST /api/browser/security/seed-demo
Body: { "domain_id": "internet_sales" }
→ creates RCLS_CANADA / RCLS_EUROPE / RCLS_EXEC users (idempotent)
→ grants USAGE + SELECT on virtual schema + SQLCUBE_REGISTRY tables
→ INSERTs 4 row policies
→ ALTER VIRTUAL SCHEMA REFRESH
```

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

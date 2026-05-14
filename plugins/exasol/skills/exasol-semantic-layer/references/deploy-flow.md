# Deploy flow — use the scripts, don't reinvent

This is the **primary** reference for getting a cube live. The studio backend ships UDFs and HTTP endpoints that already do every step. Agents should call them, not hand-write registry INSERTs.

Hand-crafted SQL is the **fallback** for clusters without studio backend reachable — see `cube-creation.md` for that mode. Default to the studio path.

## Two flows, ranked

| Path | When | Effort |
|---|---|---|
| **A. Studio-backend-driven (preferred)** | studio FastAPI reachable (default for `exanano-sqlcube` + factory-foundation `studio/backend` server) | ~6 HTTP calls |
| B. Direct SQL via `EXA_OPTIMIZE` UDFs only | studio FastAPI not reachable but Exasol is | ~3 UDF calls + 1 SQL batch |
| C. Pure hand-crafted SQL | no studio, no UDFs (raw Exasol) | full batch per `cube-creation.md` — last resort |

Most environments are Path A.

## Detecting which path to take

Don't guess — probe. The studio backend (Path A) is conditional on a running uvicorn process and a valid session cookie, neither of which an agent should assume.

```bash
# 1. Reachability — any auth-required endpoint with no cookie returns 401, not connection-refused.
#    401 means the server IS running and we just need a session.
HTTP=$(curl -s -o /dev/null -w '%{http_code}' -m 3 http://localhost:8001/api/auth/me || echo 000)
case "$HTTP" in
  200) echo "Path A ready (session active)";;
  401) echo "Path A available — need to authenticate";;
  000) echo "Path A unavailable — fall back to Path B (UDFs)";;
  *)   echo "Path A reachable but unexpected status $HTTP — check backend logs";;
esac
```

If Path A is available but you need to authenticate: POST `/api/auth/login` with the persona (default `admin`) and password from `studio/backend/.env` (`STUDIO_PASSWORD_HASH` is the SHA-256 of the plaintext; the plaintext lives wherever the operator configured it). The endpoint sets a `session` HttpOnly cookie that every subsequent `/api/*` call must send (use `curl -c cookies.txt` on login + `-b cookies.txt` on every follow-up).

```bash
# Login (live-verified 2026-05-14)
curl -s -X POST -H 'Content-Type: application/json' \
     -d '{"password":"<plaintext>","persona":"admin"}' \
     -c /tmp/cookies.txt http://127.0.0.1:8001/api/auth/login
# {"ok":true,"persona":"admin"}

# Verify
curl -s -b /tmp/cookies.txt http://127.0.0.1:8001/api/auth/me
# {"authenticated":true,"persona":"admin"}
```

Dev escape hatch: setting `STUDIO_AUTH_BYPASS=true` in the backend environment makes `/api/auth/login` accept any password (still issues a real signed cookie). Use only on local Nano dev boxes — never on shared clusters.

If Path A is unavailable, you can drive Path B from any pyexasol-capable runtime. The studio venv at `studio/backend/.venv/bin/python3` already has pyexasol 2.x installed — no separate `pip install` needed for local dev.

### TLS handshake timeout under Nano memory pressure

Live-verified 2026-05-14: under memory pressure on `exanano-sqlcube`, the backend's pyexasol WebSocket+TLS handshake intermittently times out with `_ssl.c:1063: The handshake operation timed out`. `/deploy/execute` returns:

```json
{
  "success": false,
  "virtual_schema_name": "",
  "message": "Could not connect to Exasol: _ssl.c:1063: The handshake operation timed out...",
  "failed_step": "validate_model",
  "deployed_domain_ids": []
}
```

The error is transient. Three correct responses:

1. **Retry** — the same call usually succeeds within 1-3 attempts. The orchestrator should detect `failed_step == "validate_model"` with `handshake` in the message and retry up to 3× with a small backoff.
2. **Lengthen the timeout** — backend `services/exasol.get_connection(..., timeout_seconds=15)` accepts an explicit timeout. Studio defaults to the pyexasol baseline. For mass-deploy or low-memory environments, push the explicit timeout higher in the calling code.
3. **Switch to native transport** — `exapump` uses Exasol's native binary protocol and is unaffected by the TLS handshake delay. The `/deploy/execute` path currently uses pyexasol (WebSocket+TLS only); for the registry SQL portion you can drop to exapump if backend retry is unacceptable.

This is environmental (Nano under load), not a skill bug — but agents should expect it and retry rather than surface a transient SSL error to the user.

---

## Path A: studio-backend-driven (preferred)

Default studio base URL on a local dev box: `http://localhost:8001`. The factory-foundation `scripts/agent_env_check.sh` confirms reachability.

### Step 1 — install / verify infrastructure

Two prerequisites that the studio install handles:

1. `SQLCUBE.ADAPTER` Lua UDF + `SQLCUBE_REGISTRY` schema → installed via `factory-foundation/sqlcube/` build (see `install.md`).
2. `EXA_OPTIMIZE` UDFs (10 scripts) → installed by the `exasol-optimize` skill, or directly via `studio/backend/services/scripts_library.py:install_optimize_scripts()`.

Quick catalog check (rung 4):

```sql
SELECT SCRIPT_NAME FROM SYS.EXA_ALL_SCRIPTS WHERE SCRIPT_SCHEMA='EXA_OPTIMIZE' ORDER BY SCRIPT_NAME;
-- Expect 10: ANALYZE_CONSTRAINTS, ANALYZE_DIM_DATE, BUILD_UNKNOWN_MEMBER_INSERT,
--           CONVERT_DATATYPES, CONVERT_VARCHAR, DRY_RUN_PLAN, GENERATE_DIM_DATE,
--           INFER_JOIN_PATHS, SET_PRIMARY_AND_FOREIGN_KEYS, VALIDATE_JOIN_PLAN

SELECT 1 FROM SYS.EXA_ALL_SCRIPTS WHERE SCRIPT_SCHEMA='SQLCUBE' AND SCRIPT_NAME='ADAPTER';
SELECT 1 FROM SYS.EXA_ALL_TABLES  WHERE TABLE_SCHEMA='SQLCUBE_REGISTRY' AND TABLE_NAME='MODELS';
```

If anything is missing → route to `exasol-optimize` (UDFs) and `install.md` (adapter + registry).

### Step 2 — declare constraints on the source schema (via `exasol-optimize`)

The cube generator needs PKs + FKs in `SYS.EXA_ALL_CONSTRAINTS` to produce a meaningful join graph. The `exasol-optimize` skill drives this:

```sql
-- Suggest constraints from data. Bare EXECUTE SCRIPT only — run via
-- pyexasol, studio /api/query/execute, or an interactive client. The wrap
-- form `SELECT * FROM (EXECUTE SCRIPT ...)` is a parser error in Exasol.
-- See "EXECUTE SCRIPT gotcha" below.
EXECUTE SCRIPT EXA_OPTIMIZE.ANALYZE_CONSTRAINTS('ADVENTUREWORKS');
```

Returns one row per finding (missing PK, inferred FK, nullable FK, etc.) plus the generated `ALTER TABLE` SQL.

Apply via the studio endpoint (preferred — runs DRY_RUN_PLAN around it):

```
POST /api/optimize/apply-review-proposal
Body: { "schema": "ADVENTUREWORKS", "proposals": [ ...rows from ANALYZE_CONSTRAINTS... ] }
```

…or the raw UDF round-trip:

```sql
EXECUTE SCRIPT EXA_OPTIMIZE.DRY_RUN_PLAN(
  'ADVENTUREWORKS',
  '<concatenated ALTER TABLE statements>',
  FALSE  -- dry_run=FALSE to commit
);
```

Don't ALTER constraints from the agent loop without going through one of these — they apply VALIDATE_JOIN_PLAN preflight + rollback discipline.

### Step 3 — infer the join graph (catalog-only, no studio call)

```sql
EXECUTE SCRIPT EXA_OPTIMIZE.INFER_JOIN_PATHS('ADVENTUREWORKS');
```

Returns one row per inferred fact↔dim join (tier 1: declared FK, tier 2: name-stem match, tier 3: empty-schema fallback). Use the result as the `join_paths` array in the cube model. The UDF does fact/dim classification the same way `services.ddl_parser` does in Python, so the output already lines up with what `/generate` expects.

**Halt-gate: zero rows.** If `INFER_JOIN_PATHS` returns 0 rows, neither declared FKs nor stem-match heuristics found a fact↔dim link. Three scenarios:

1. **No FK constraints declared AND no `*_KEY` / `*_ID` columns on the candidate facts.** The cube would have only fact-table measures, no dim attributes. Cube is essentially useless. Halt and surface to the user: "no joins found — schema is fact-only or needs renaming." Suggest the user either: (a) declare FKs via `ANALYZE_CONSTRAINTS` + `DRY_RUN_PLAN`, or (b) rename dim PKs to a stem the fact's FK columns share, or (c) accept a fact-only cube and proceed manually.
2. **Tables exist but classifier didn't pick a fact.** No `FACT_*` / `*_FACT` / `*_TXN` / `*_EVT` naming AND no obvious aggregate-row-count winner. Same halt; suggest user pass `fact_table` parameter explicitly.
3. **Schema is empty.** `SYS.EXA_ALL_TABLES` returns no rows for the source. Route back to `exasol-migrate` — data hasn't landed yet.

Surfaced live 2026-05-14 against the `INVENTORY` fixture on `exanano-sqlcube`: 7 tables, no FK constraints, no FACT-prefixed names, suffix on dim was `_DIM` (caught by `ANALYZE_CONSTRAINTS`) but no fact's FK column shared a stem with any dim's PK → 0 joins. The skill should not silently proceed to step 5 with an empty `join_paths` list — user expects a join graph, will see an empty cube, will not understand why.

### Step 4 — parse the source DDL

Preferred — server-side end-to-end:

```
POST /api/ddl/introspect
Body: { "schema_name": "ADVENTUREWORKS" }
```

Runs the three SYS.EXA_ALL_* queries inside the backend and returns a fully populated `parsed_ddl` dict consumed by every other endpoint. Live-verified 2026-05-14: 9 tables, 11 join_paths, 1 fact + 8 dims auto-classified.

Alternatives:

- `GET /api/sqlcube/introspect-sql?schema_name=ADVENTUREWORKS` — **returns only the SQL** the Workbench shell should run (one row: `{"sql": "..."}`). Caller is expected to execute that SQL via their own driver/exapump, then POST the result rows to `/api/sqlcube/introspect-from-rows` with `{schema_name, rows}`. Use when you want the SQL visible in a user-facing tab + query history audit rather than a hidden server-side call.
- `POST /api/sqlcube/introspect-from-rows` (body `{schema_name, rows}`) — companion to the above; parses an already-fetched UNION-ALL row list.
- `POST /api/sqlcube/source-metadata/parse` if a vendor BIM file (e.g. Power BI) is the source of truth.

**Live-verified divergences from an earlier revision of this doc**:

- Query param is `schema_name`, not `schema`. The shorter form returns a 422 "Field required" error.
- `/api/sqlcube/introspect-sql` does NOT return `parsed_ddl` — only the SQL. The `/api/ddl/introspect` endpoint is the one-shot server-side variant.

### Step 5 — generate the SqlcubeModel

```
POST /api/sqlcube/generate
Body: {
  "parsed_ddl": { ... },               // from step 4
  "source_metadata": null,             // or BIM dict if available
  "view_strategy": "native",           // or "compatibility"
  "llm_enrichment": true,              // false = deterministic profile-based draft
  "grain_options": null                // or { ... } for alt-grain models
}
```

Returns a `ValidationResponse` with the generated `SqlcubeModel` plus warnings / infos.

What this saves the agent: deterministic + LLM enrichment, source-metadata merging, alt-grain rollups, metric_calc sanitization. All the logic the prior hand-crafted SQL was missing.

### Step 6 — validate (optional but recommended)

```
POST /api/sqlcube/validate
Body: { "model": { ...SqlcubeModel... }, "parsed_ddl": { ... } }
```

Returns `ValidationResponse { valid, errors, warnings, infos, model }`. Catches alias collisions, missing PHYSICAL_COL references, derived-measure cycles, etc. — before the deploy.

### Step 7 — pre-flight conflict check

```
POST /api/deploy/check
Body: { "source_schema": "ADVENTUREWORKS", "domain_ids": ["internet_sales"] }
```

Returns `DeployCheckResponse { has_conflicts, conflicts: [...] }`. If `has_conflicts=true`, the deploy will replace existing models on the same source schema — show the user before proceeding (or skip when intentional re-deploy).

### Step 8 — execute

```
POST /api/deploy/execute
Body: { ...SqlcubeModel... }   // straight from /generate (or /validate)
```

`services/deployer.py:deploy()` runs end-to-end:

1. `validate_model(model)` (same as `/validate`)
2. `build_registry_ddl()` — `CREATE TABLE IF NOT EXISTS` for all 11 registry tables (idempotent)
3. `build_registry_sql(model)` — DELETE + INSERT for DOMAINS / ATTRIBUTES / JOIN_PATHS
4. `_build_runtime_registry_sql(model)` — two-pass DELETE (fact-pair purge + per-domain), then INSERT MODELS / JOINS / DIMENSIONS / MEASURES / DERIVED_MEASURES
5. `_deploy_adapter_script()` — uploads the bundled Lua adapter from `factory-foundation/sqlcube/adapter/sqlcube_adapter.lua`
6. `DROP VIRTUAL SCHEMA IF EXISTS` + `CREATE VIRTUAL SCHEMA SQLCUBE_<LAYER>` via `build_virtual_schema_sql`

Returns `DeployResponse { success, virtual_schema_name, message, failed_step, deployed_domain_ids }`.

### Step 9 — verify (provides contract)

Use the three checks from SKILL.md `provides.cube_live.verify`. Plus a real grouped query per `query-patterns.md` (implicit-group shape).

---

## Path B: UDF-only (no studio HTTP)

When the studio FastAPI isn't reachable but the Exasol cluster is, hit the UDFs directly:

1. `EXECUTE SCRIPT EXA_OPTIMIZE.ANALYZE_CONSTRAINTS(<schema>)` — collect proposals.
2. `EXECUTE SCRIPT EXA_OPTIMIZE.DRY_RUN_PLAN(<schema>, <ALTER batch>, FALSE)` — apply with rollback discipline.
3. `EXECUTE SCRIPT EXA_OPTIMIZE.INFER_JOIN_PATHS(<schema>)` — capture the join graph.
4. Hand-write registry INSERTs per `meta-model.md` + `cube-creation.md`. Use the inferred joins from step 3.
5. `CREATE VIRTUAL SCHEMA SQLCUBE_<LAYER> ... MODEL_IDS='<csv>'`.
6. Verify per `provides.cube_live`.

You lose the LLM enrichment (no `/generate`) and the validator (no `/validate`). Acceptable when the cube is small and you supply overrides directly.

---

## Path C: pure hand-crafted SQL

Last resort. See `cube-creation.md` for the full INSERT order + DROP/CREATE pattern. Used in the 2026-05-14 live-walk audit as a stress test of the docs themselves — not the production agent flow.

---

## UDF call reference

All 10 scripts live in schema `EXA_OPTIMIZE`. Installed by `exasol-optimize` skill from `factory-foundation/studio/backend/fixtures/optimization-scripts/*.sql`.

| UDF | Args | Returns | Use |
|---|---|---|---|
| `ANALYZE_CONSTRAINTS` | `SCHEMA_NAME` | findings table (PK/FK proposals + ALTER SQL) | Step 2: derive constraint proposals |
| `INFER_JOIN_PATHS` | `SCHEMA_NAME` | join paths table (fact, dim, fact_key, dim_key, tier) | Step 3: derive `join_paths` for the SqlcubeModel |
| `SET_PRIMARY_AND_FOREIGN_KEYS` | `connection_type, connection_name, db_type, schema_filter, table_filter, target_schema, constraint_status, case_insensitive` (8 args) | applied-constraints table | Cross-DB migration helper (Oracle/MySQL/Exasol → Exasol) — not on the cube path |
| `ANALYZE_DIM_DATE` | `SCHEMA_NAME, FISCAL_YEAR_START_MONTH` | findings table with ALTER + UPDATE SQL | Optional: upgrade an existing date dim in place |
| `GENERATE_DIM_DATE` | `TARGET_SCHEMA, TARGET_TABLE, START_DATE, END_DATE, FISCAL_START_MONTH, FORCE` | log table | Optional: bootstrap a Kimball-standard date dim |
| `BUILD_UNKNOWN_MEMBER_INSERT` | `SCHEMA_NAME, DIM_TABLE, DIM_PK` | one-row table `SQL_TEXT` | Generate idempotent `-1` sentinel INSERT for unknown-member handling |
| `VALIDATE_JOIN_PLAN` | `SCHEMA_NAME, SQL_TEXT` | findings table (PASS / REPAIR / ERROR / SKIP_STATEMENT) | Preflight a constraint-plan batch |
| `DRY_RUN_PLAN` | `SCHEMA_NAME, PLAN_SQL, DRY_RUN` (bool) | validation rows + execution rows | Validate-then-execute orchestrator with rollback on DML failure |
| `CONVERT_DATATYPES` | (schema-level type narrowing helper — see fixture for signature) | findings + SQL | Optional: type-narrow numerics + dates after migrate |
| `CONVERT_VARCHAR` | same as above for VARCHAR length narrowing | findings + SQL | Optional: bring VARCHAR(2000) UTF8 down to reality |

All scripts emit catalog SQL that's portable across Exasol versions — they read `SYS.EXA_ALL_*` views, not version-specific internals.

**`EXECUTE SCRIPT` gotcha** (already in `optimize` SKILL.md, repeated here, **re-verified live 2026-05-14**):

- `exapump sql --profile foo "EXECUTE SCRIPT ..."` fails with `Result set not available: Expected row count, got result set`.
- `exapump sql --profile foo "SELECT * FROM (EXECUTE SCRIPT ...)"` ALSO fails — `Protocol error: syntax error, unexpected EXECUTE_`. The SQL parser itself rejects the wrap form; there is no client that accepts it. (An earlier revision of this doc claimed otherwise — wrong, verified against `exanano-sqlcube` on `EXA_OPTIMIZE.ANALYZE_CONSTRAINTS('INVENTORY')`.)
- **Working invocations**: pyexasol's `.execute("EXECUTE SCRIPT ...")` returns a fetchable statement; studio backend's `POST /api/query/execute` handles the result set; interactive clients (DBeaver, DataGrip) that understand Lua RETURNS TABLE work. The studio venv at `studio/backend/.venv/bin/python3` already has pyexasol 2.x — easiest local route.

Minimal pyexasol recipe:

```python
import pyexasol
conn = pyexasol.connect(
    dsn="localhost:8564", user="sys", password="exasol",
    encryption=True, websocket_sslopt={"cert_reqs": 0},  # self-signed in Nano
)
stmt = conn.execute("EXECUTE SCRIPT EXA_OPTIMIZE.ANALYZE_CONSTRAINTS('INVENTORY')")
cols = list(stmt.columns().keys())
rows = stmt.fetchall()
```

---

## Studio endpoint reference

Base path: `/api/`. All POST unless noted. Reachable from agent over HTTP when studio backend is running.

| Endpoint | Body | Returns | Use |
|---|---|---|---|
| `POST /sqlcube/analyze` | `AnalyzeSchemaRequest` | `AnalyzeSchemaResponse` | Run optimize discovery on a schema (wraps EXA_OPTIMIZE.ANALYZE_CONSTRAINTS) |
| `POST /sqlcube/run-sql` | `RunSqlRequest { sql, parameters?, ... }` | `RunSqlResponse` | Generic SQL passthrough — supports `EXECUTE SCRIPT` syntax for UDF calls |
| `POST /sqlcube/postprocess-findings` | `AnalyzeSchemaResponse` | `AnalyzeSchemaResponse` | Re-rank / merge ANALYZE_CONSTRAINTS rows |
| `POST /sqlcube/reverse-engineer-from-queries` | query corpus | proposed model | Reverse-engineer a cube from a query log (rare; advanced flow) |
| `POST /sqlcube/dry-run` | plan SQL | `DryRunResponse` | Wraps `EXA_OPTIMIZE.DRY_RUN_PLAN` with VALIDATE + auto-install |
| `POST /sqlcube/apply-review-proposal` | proposals | `ApplyReviewProposalResponse` | Apply the optimize panel's constraint proposals through DRY_RUN_PLAN |
| `POST /sqlcube/generate` | `GenerateRequest { parsed_ddl, source_metadata?, view_strategy, llm_enrichment, grain_options? }` | `ValidationResponse { valid, errors, warnings, infos, model }` | **Step 5** — produce SqlcubeModel |
| `POST /sqlcube/validate` | `ValidateRequest { model, parsed_ddl? }` | `ValidationResponse` | **Step 6** — validate before deploy |
| `POST /sqlcube/source-metadata/parse` | `SourceMetadataParseRequest { filename, text }` | parsed BIM dict | Parse a vendor BIM file (Power BI etc.) |
| `POST /sqlcube/source-metadata/report` | `SourceMetadataReportRequest` | parity report | Compare BIM expectations vs live source |
| `POST /ddl/introspect` | `{ "schema_name": "<S>" }` | `parsed_ddl` dict | **Preferred Step 4** — server-side one-shot. Backend runs the three SYS.EXA_ALL_* queries and returns parsed_ddl. |
| `GET  /sqlcube/introspect-sql?schema_name=<S>` | (query string — note `schema_name`, not `schema`) | `{ "sql": "..." }` | Alt step 4 — returns ONLY the introspection SQL for the agent / Workbench tab to run. Pair with `/introspect-from-rows`. |
| `POST /sqlcube/introspect-from-rows` | `{ schema_name, rows }` | `parsed_ddl` dict | Companion to `/sqlcube/introspect-sql` — parses pre-fetched UNION-ALL rows. |
| `POST /deploy/check` | `DeployCheckRequest { source_schema, domain_ids }` | `DeployCheckResponse { has_conflicts, conflicts }` | **Step 7** — pre-flight conflicts |
| `POST /deploy/execute` | `SqlcubeModel` | `DeployResponse { success, virtual_schema_name, message, failed_step, deployed_domain_ids }` | **Step 8** — run `deployer.deploy(model)` end-to-end |

---

## SqlcubeModel shape (request body for `/deploy/execute`)

From `factory-foundation/studio/backend/models/sqlcube.py`. Minimal shape:

```jsonc
{
  "source_schema": "ADVENTUREWORKS",
  "owner_name": null,
  "owner_type": null,
  "is_system_managed": false,
  "domains": [
    { "domain_id": "internet_sales", "domain_name": "Internet Sales", "description": "..." }
  ],
  "attributes": [
    {
      "attr_id": "dimcustomer_firstname",
      "domain_id": "internet_sales",
      "business_name": "First Name",
      "attr_type": "dimension",                 // 'dimension' | 'metric_agg' | 'metric_derived' | 'metric_calc'
      "physical_table": "ADVENTUREWORKS.DIMCUSTOMER",
      "physical_col": "FIRSTNAME",
      "data_type": "VARCHAR(50) UTF8",
      "display_type": "text",
      "is_visible": true
    },
    {
      "attr_id": "internet_sales_extended_amount",
      "domain_id": "internet_sales",
      "business_name": "Extended Amount",
      "attr_type": "metric_agg",
      "physical_table": "ADVENTUREWORKS.FACTINTERNETSALES",
      "physical_col": "EXTENDEDAMOUNT",
      "agg_function": "SUM",
      "data_type": "DECIMAL(19,4)",
      "display_type": "currency",
      "is_visible": true
    }
  ],
  "join_paths": [
    {
      "join_id": "internet_sales_dimcustomer_customerkey",
      "domain_id": "internet_sales",
      "fact_table": "ADVENTUREWORKS.FACTINTERNETSALES",
      "dim_table":  "ADVENTUREWORKS.DIMCUSTOMER",
      "fact_key_col": "CUSTOMERKEY",
      "dim_key_col":  "CUSTOMERKEY",
      "join_type":    "LEFT"                    // deployer resolves to INNER if FK NOT NULL
    }
  ]
}
```

`/sqlcube/generate` produces a shape like this from `parsed_ddl` — agents rarely build it by hand.

---

## Prompt-package enhancement loop

The studio frontend ships an "offline pack" workflow that the agent loop subsumes — it's the same data path, just collapsed because the agent IS the LLM. Two pack types exist:

| Pack | Frontend builder | Output expected back | Apply path |
|---|---|---|---|
| **Metric enrichment** (`sqlcube.offline_pack.v1`) | `studio/frontend/src/lib/offlinePacks.js:buildMetricEnrichmentPack()` | `sqlcube_metric_proposals.md` with fenced JSON `sqlcube.metric_proposals.v1` | Merge into cube model → POST `/sqlcube/validate` → POST `/deploy/execute` |
| **Optimization review** (`sqlcube.optimization_review.v1`) | `offlinePacks.js:buildOptimizationReviewPack()` | `sqlcube_optimization_review.md` with fenced JSON `sqlcube.optimization_review.v1` | Per row: POST `/optimize/apply-review-proposal` (currently `proposal_type='unknown_member'` only) or pipe through `EXA_OPTIMIZE.DRY_RUN_PLAN` |

In the GUI flow: user clicks "Download pack", hands the ZIP to Claude/GPT externally with the bundled `prompt.md` + `cube_context.json` + `output_schema.json`, then uploads the returned `.md` back. Studio parses client-side via `parseMetricProposalText` / `parseOptimizationReviewText`, validates, applies.

In the agent flow: skip the round trip. The skill `prompts/` directory holds the agent-side equivalents:

| Skill prompt | Replaces offline-pack prompt | When |
|---|---|---|
| `prompts/cube-enrichment.md` | Metric enrichment pack `prompt.md` | After `/generate` returns a deterministic draft and you want LLM-generated measures + descriptions |
| `prompts/measure-business-context.md` | Subset of the above for incremental measure additions | Adding measures to a live cube |
| `prompts/domain-detection.md` | Pre-step for naming the domain | When user didn't supply `domain_hint` |

Agent loop (per cube):

```
1. POST /sqlcube/generate llm_enrichment=false       → deterministic SqlcubeModel draft
2. Run prompts/cube-enrichment.md internally         → enrichment JSON (model_description, dimensions, measures, ...)
3. Merge enrichment into the draft model             → enriched SqlcubeModel
4. POST /sqlcube/validate { model, parsed_ddl }      → catch alias collisions, derived-cycle bugs
5. POST /deploy/check { source_schema, domain_ids }  → confirm conflicts before clobbering
6. POST /deploy/execute SqlcubeModel                 → registry + adapter + virtual schema
```

For the constraint side (optimize, not cube):

```
1. EXECUTE SCRIPT EXA_OPTIMIZE.ANALYZE_CONSTRAINTS('<S>') → row table of findings + ALTER SQL
2. Run optimize skill's review prompt internally OR call directly
3. POST /optimize/apply-review-proposal (unknown_member rows) or
   EXECUTE SCRIPT EXA_OPTIMIZE.DRY_RUN_PLAN('<S>', <plan SQL>, FALSE) for the rest
```

What you DON'T do in the agent loop:

- Build a ZIP. The pack format is a UX affordance for GUI users.
- Wait for upload. The agent has both halves of the conversation.
- Re-implement the validator. Always call `/sqlcube/validate` — it catches things the prompt can't (alias collisions, dependency cycles).

What you DO need to match:

- Output JSON shape must satisfy the same `output_schema.json` the offline pack ships, otherwise merge breaks. The skill prompts in `prompts/` already target this shape.
- Existing-metric repairs go in `existing_metric_reviews`, not duplicated as new `proposals`. Same rule as the offline pack.
- `metric_calc` formulas are complete SQL aggregate expressions over `f.<COL>`, not bare aggregate keywords. See `prompts/cube-enrichment.md` and `studio/frontend/src/lib/offlinePacks.js:metricRulesMarkdown()` for the exact rules — keep them aligned.

---

## Anti-pattern: hand-writing the registry batch

This is what the 2026-05-14 live-walk audit did intentionally to stress-test the docs. **Don't repeat it in production**:

- You re-derive MODEL_ID / DIMENSIONS / MEASURES / JOINS from scratch instead of letting `/generate` produce them with the validator + LLM enrichment + source-metadata merge already applied.
- You skip alias collision detection, derived-measure cycle detection, and metric_calc sanitization.
- You don't get the two-pass DELETE that purges stale `MODEL_ID` slugs from earlier deploys.
- You don't get `_deploy_adapter_script()` — adapter Lua updates won't reach the cluster.
- You set MEASURES.AGG_TYPE manually and risk pre-aggregating `f.SUM(...)` incorrectly.

Use Path A. If Path A isn't available, escalate to install the studio backend rather than work around it.

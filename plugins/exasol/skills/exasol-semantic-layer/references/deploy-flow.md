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
-- Suggest constraints from data
SELECT * FROM (EXECUTE SCRIPT EXA_OPTIMIZE.ANALYZE_CONSTRAINTS('ADVENTUREWORKS'));
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

### Step 4 — parse the source DDL

```
POST /api/sqlcube/introspect-sql?schema=ADVENTUREWORKS
```

Returns the `parsed_ddl` dict consumed by every other endpoint. Schema-driven; reads `SYS.EXA_ALL_TABLES`, `SYS.EXA_ALL_COLUMNS`, `SYS.EXA_ALL_CONSTRAINTS` and assembles the canonical view.

Alternatives:

- `POST /api/sqlcube/introspect-from-rows` if you already have a custom row payload from another connector.
- `POST /api/sqlcube/source-metadata/parse` if a vendor BIM file (e.g. Power BI) is the source of truth.

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

**`EXECUTE SCRIPT` gotcha** (already in `optimize` SKILL.md, repeated here): `exapump sql --profile foo "EXECUTE SCRIPT ..."` fails with `Result set not available: Expected row count, got result set`. Use pyexasol, the studio backend `/api/query/execute`, or an interactive SQL client (DBeaver, DataGrip) that handles Lua-result-set tables. Or wrap: `SELECT * FROM (EXECUTE SCRIPT ...)` — which most clients accept.

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
| `GET  /sqlcube/introspect-sql?schema=<S>` | (query string) | `parsed_ddl` dict | **Step 4** — read source schema via SYS.EXA_ALL_* |
| `POST /sqlcube/introspect-from-rows` | row payload | `parsed_ddl` dict | Alt step 4 — use rows already in hand |
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

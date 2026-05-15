# Phase 1 — Interrogation

For the canonical Maglev Tour demo against `exanano-demo-tour`, defaults in the frontmatter usually suffice. Interrogation is short.

## Required to confirm

1. **Which demo backend?** Default `http://127.0.0.1:8002` (Maglev Tour). User running a different studio? Ask.
2. **Approval mode?** `auto_accept` (continuous, ~3-5 min total) or `step_by_step` (pause + confirm each phase). Boss-watching demos almost always want `auto_accept`; first-time runs benefit from `step_by_step`.
3. **Demo state?** Three scenarios:
   - **Cold** — fresh demo-tour container; everything to build from scratch. ~5 min.
   - **Warm** — ACME_ADVENTUREWORKS migrated + cube live + RCLS seeded (the default after running once). Skill verifies via the provides checks and skips no-op steps. ~30 sec.
   - **Mid** — partial state. Skill walks each phase, halts at first verify failure with a clear "Phase N already in state X; re-run from N+1" message.

Detect via:

```sql
-- Note: CUBE is reserved in Exasol (verified live against SYS.EXA_SQL_KEYWORDS).
-- Bare `AS cube` errors with `syntax error, unexpected CUBE_`. Quote it.
SELECT
  (SELECT COUNT(*) FROM SYS.EXA_SCHEMAS WHERE SCHEMA_NAME = 'ACME_ADVENTUREWORKS')                AS "migrated",
  (SELECT COUNT(*) FROM SYS.EXA_VIRTUAL_SCHEMAS WHERE SCHEMA_NAME = 'SQLCUBE_ACME_ADVENTUREWORKS') AS "cube",
  (SELECT COUNT(*) FROM SQLCUBE_REGISTRY.RCLS_ROW_POLICIES WHERE MODEL_ID = 'internet_sales')     AS "rcls";
```

3-of-3 → warm. 0-of-3 → cold. Anything else → mid.

## Warm-state branch — prompt the user explicitly

When the probe returns warm, do **not** auto-skip phases. The user may have run `/maglev-tour` expecting a fresh end-to-end run for the demo (boss watching the 8 steps unfold), not the 5-second showcase. Always ask:

```
agent:
  Warm-state detected — ACME_ADVENTUREWORKS migrated, SQLCUBE_ACME_ADVENTUREWORKS
  cube live with 4 RCLS row policies seeded.

  Three options:

  (a) showcase  — jump straight to Phase 8 persona showcase against the
                  existing cube. ~5 seconds. Use when the demo state is
                  already what you want.

  (b) cold      — drop ACME_ADVENTUREWORKS schema + virtual schema +
                  registry rows for internet_sales* domains, then run all
                  8 phases live. ~60 seconds. RCLS users (RCLS_CANADA / _EUROPE /
                  _EXEC) stay in place — they're idempotent on re-seed.

  (c) abort     — exit, do nothing.

  Pick a / b / c.
```

Defaults: `(a)` if user-context indicates "just show me the punchline"; `(b)` if "watch it build end-to-end". Don't infer — always ask.

### Cold-rerun via EXA_OPTIMIZE.RESET_TOUR_STATE (option b — preferred)

Single UDF call clears every per-tour artefact and emits an audit row per step. Boss-watchable; the agent narrates the result table verbatim. Available on demo-tour from `demo-streamlined` commit `83cabff` onward.

```sql
EXECUTE SCRIPT EXA_OPTIMIZE.RESET_TOUR_STATE(
    'ACME_ADVENTUREWORKS',   -- SOURCE_SCHEMA
    'internet_sales'         -- DOMAIN_PREFIX (matches LIKE 'internet_sales%')
);
```

Returns `(STEP, TARGET_OBJ, ELAPSED_MS, RESULT_FLAG, NOTE)` per step:
- `DROP_VIRTUAL_SCHEMA` — `SQLCUBE_<SOURCE>` cleared
- `DROP_SCHEMA` — `<SOURCE>` cleared (CASCADE)
- `DELETE_REGISTRY` × N — one row per (table, model_id) pair: RCLS_ATTRIBUTE_POLICIES, RCLS_ROW_POLICIES, DERIVED_MEASURES, JOINS, MEASURES, DIMENSIONS, MODELS (model-id-keyed); JOIN_PATHS, ATTRIBUTES, DOMAINS (domain-id-keyed)
- `SUMMARY` — total models cleared

Live-verified 2026-05-14: pre-state 1/1/4/4/4 (vs+src+models+row+attr); post-state 0/0/0/0/0; 44-row audit trail. UDF runs via `EXECUTE SCRIPT` — invoke via pyexasol or `POST /api/query/execute` (exapump CLI errors on EXECUTE SCRIPT result sets — same gotcha as every other EXA_OPTIMIZE UDF).

What survives (intentional):
- `RCLS_CANADA` / `RCLS_EUROPE` / `RCLS_EXEC` users — idempotent on Phase 7 re-seed
- `SNOWFLAKE_CONNECTION` and other Exasol CONNECTION objects
- `EXA_OPTIMIZE.*` UDFs (including this one)
- `EXA_DB_MIGRATION.*` migration scripts
- `SQLCUBE.ADAPTER` Lua script + `SQLCUBE_REGISTRY` table DDL

### Fallback if UDF not installed

If `EXECUTE SCRIPT EXA_OPTIMIZE.RESET_TOUR_STATE` returns "script not found", the studio backend has either an older fixtures dir or the UDF wasn't installed yet. Two recovery paths:

1. Install: `POST /api/optimize/scripts/install` with body `{"filenames": ["reset_tour_state.sql"]}`. Then re-run the UDF.
2. Direct SQL (verbose; lose the audit-row narration):

   ```sql
   DROP VIRTUAL SCHEMA IF EXISTS "SQLCUBE_ACME_ADVENTUREWORKS" CASCADE;
   DROP SCHEMA IF EXISTS "ACME_ADVENTUREWORKS" CASCADE;
   -- Repeat per domain:
   -- internet_sales, internet_sales_by_month,
   -- internet_sales_by_product, internet_sales_by_month_product
   DELETE FROM SQLCUBE_REGISTRY.RCLS_ATTRIBUTE_POLICIES WHERE MODEL_ID = '<domain>';
   DELETE FROM SQLCUBE_REGISTRY.RCLS_ROW_POLICIES       WHERE MODEL_ID = '<domain>';
   DELETE FROM SQLCUBE_REGISTRY.DERIVED_MEASURES        WHERE MODEL_ID = '<domain>';
   DELETE FROM SQLCUBE_REGISTRY.JOINS                   WHERE MODEL_ID = '<domain>';
   DELETE FROM SQLCUBE_REGISTRY.MEASURES                WHERE MODEL_ID = '<domain>';
   DELETE FROM SQLCUBE_REGISTRY.DIMENSIONS              WHERE MODEL_ID = '<domain>';
   DELETE FROM SQLCUBE_REGISTRY.MODELS                  WHERE MODEL_ID = '<domain>';
   DELETE FROM SQLCUBE_REGISTRY.JOIN_PATHS              WHERE DOMAIN_ID = '<domain>';
   DELETE FROM SQLCUBE_REGISTRY.ATTRIBUTES              WHERE DOMAIN_ID = '<domain>';
   DELETE FROM SQLCUBE_REGISTRY.DOMAINS                 WHERE DOMAIN_ID = '<domain>';
   ```

3. Blanket reset: `POST /api/browser/demo/reset {"owner_name": "demo"}` — drops everything owned by the `demo` persona. No audit narration. Fastest path; weakest evidence.

## Free-form Q&A pattern

```
agent:
  This is the Maglev Tour demo — Snowflake → Exasol → optimized cube → RCLS → persona-switch query.

  Quick orientation:
  1. Studio backend at http://localhost:8002 — running?
  2. Approval mode — auto_accept (~3-5 min) or step_by_step?
  3. Snowflake CONNECTION + migration scripts already seeded on the Nano? (Defaults to "yes" since the demo-tour container ships pre-seeded.)

  Once confirmed I'll show the 8-step plan and start.
```

If user wants a custom source / cube name / persona set, branch to the optional parameters block in `SKILL.md`.

## What NOT to ask

- Don't ask for the Snowflake password — backend reads it from the pre-existing `SNOWFLAKE_CONNECTION` Exasol object.
- Don't ask for RCLS persona passwords — backend resolves from `DEMO_RCLS_SCENARIOS`.
- Don't ask about Java JDBC version / Arrow wrapper — operator-managed at container provision time per exasol-maglev/FACTORY_TOUR_DEMO_PATH.md §Setup.

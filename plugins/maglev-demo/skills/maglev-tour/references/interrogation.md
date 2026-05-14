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
- Don't ask about Java JDBC version / Arrow wrapper — operator-managed at container provision time per exasol-zemantic-layer/FACTORY_TOUR_DEMO_PATH.md §Setup.

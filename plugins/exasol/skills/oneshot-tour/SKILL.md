---
name: oneshot-tour
description: End-to-end agent-driven walkthrough — connect a source, import schema, optimize constraints, build a queryable cube, render a dashboard. One session, six steps, real database objects at the end. Trigger when the user wants to "see the full pipeline," "go end to end," "demo," "tour," "build a cube from scratch," "load and query," or invokes `/oneshot-tour` directly.

preconditions: []

provides:
  - cube_live:
      verify: |
        SELECT 1 FROM SYS.EXA_VIRTUAL_SCHEMAS WHERE SCHEMA_NAME = :target_cube_name

parameters:
  required:
    - source_kind:                 "snowflake | s3 | acme_demo_fixture | other_jdbc"
    - source_connection_name:      "Exasol CONNECTION name pointing at the source"
    - source_schema:               "Schema name on the source side"
    - target_cube_name:            "Name for the new virtual schema on Exasol"
    - approval_mode:               "auto_accept | step_by_step"
  optional:
    - source_table_filter:         {default: "%", doc: "Wildcard filter to limit imported tables"}
    - target_stage_schema:         {default: "STAGE_<source_schema>", doc: "Where landed tables go before cube"}
    - llm_enrichment:              {default: true, doc: "Let LLM enrich names + groupings during optimize + cube create"}
---

# /oneshot-tour — single-session end-to-end pipeline

Walks the user from raw source through to a queryable cube + rendered dashboard, in **one agent session**, with one approval gate (or per-step if requested).

## What this skill is

The canonical demonstration of agent-driven Exasol workflows. It composes six domain skills into a named journey. It does no work itself — every step hands off to the skill that owns that capability. The orchestrator's job is interrogation, plan presentation, sequencing, narration, and error halting.

When this skill loads, the agent must:

1. Interrogate the user to fill the required parameters (free-form Q&A; see `references/interrogation.md`).
2. Resolve the precondition DAG (work backwards from `cube_live`).
3. Present the plan + ask for `approval_mode`.
4. Execute each step in order, surfacing progress via `TaskCreate` / `TaskUpdate` so the user sees the work happen.
5. On success, hand off to MCP query mode + render a dashboard against the live cube.

## When to trigger

Match on any of these (case-insensitive):

- `/oneshot-tour`
- "tour me end to end"
- "load and query"
- "build a cube from this source"
- "load X from snowflake and query it" (or s3 / generic JDBC)
- "show me the full pipeline"
- "demo the agent-driven workflow"

Do not trigger on narrower intents (just optimize / just import / just cube). Those have their own domain skills. Triggering `/oneshot-tour` for a narrow ask wastes the user's time.

## Routing algorithm

The skill runs in five phases. Load the matching reference file at each phase.

| Phase | Reference | What happens |
|---|---|---|
| 1. Interrogation | `references/interrogation.md` | Free-form Q&A until every required parameter is filled; LLM owns this |
| 2. Plan presentation | `references/plan-presentation.md` | Show six-step plan, estimated impact, ask for approval mode |
| 3. Execution | `references/execution.md` | Sequence the six steps, hand off to each domain skill, narrate progress |
| 4. Verification | `references/verification.md` | Confirm `cube_live` (run sample SELECT against virtual schema) |
| 5. Dashboard + handoff | `references/dashboard-handoff.md` | Render initial Vega-Lite dashboard, surface MCP query entry |

## Six-step pipeline

| # | Skill responsible | Provides |
|---|---|---|
| 1 | `exasol-nano-local` | `nano_running` |
| 2 | `exasol-migrate` (vendor variant: snowflake / s3 / generic JDBC) | `source_connected` |
| 3 | `exasol-migrate` (continues) | `schema_imported` |
| 4 | `exasol-optimize` | `schema_optimized` (calls `ANALYZE_CONSTRAINTS` + `DRY_RUN_PLAN`) |
| 5 | `exasol-semantic-layer` | `cube_live` |
| 6 | `exasol-dashboard` | `dashboard_rendered` |

Domain skills declare their own preconditions and providers. This orchestrator just walks the chain.

## Approval modes

- **auto_accept**: agent runs all six steps without pausing. Stops only on error.
- **step_by_step**: agent pauses after each step. User must reply `continue` (or `c`, `y`, `yes`) to proceed.

Default the prompt to suggest `auto_accept` for confident users and `step_by_step` for first-time runs. Do not infer — always ask.

## Stop conditions

Halt and surface a clear message to the user when:

1. Any precondition check fails after the upstream skill ran.
2. Any tool call returns a non-recoverable error.
3. The verification SELECT (`provides: cube_live`) returns zero rows.

On halt, the agent must:
- Report which step failed and the actual error.
- Suggest the next manual action (re-run, switch to step_by_step, inspect logs, etc.).
- Not retry automatically. The user picks the next move.

## Re-running against an already-loaded source

`/oneshot-tour` is partially idempotent. What's safe and what isn't:

| Step | Re-run behavior | Source of truth |
|---|---|---|
| 1. Nano boot | Safe. `docker start <name>` no-ops on a running container. | `exasol-nano-local` |
| 2. Connect to source | Safe. `exasol-migrate` creates a CONNECTION with a per-run name (default `<SOURCE_TYPE_UPPER>_MIGRATE_<TIMESTAMP>`) and DROPs it after the migration. The connection object never outlives the step, so there is nothing to collide with on re-run. | `studio/backend/routers/imports.py` |
| 3. Import schema | **Partly destructive.** Migration scripts in `EXA_DB_MIGRATION.<VENDOR>_TO_EXASOL` typically `DROP TABLE IF EXISTS` per target before re-creating. Existing rows in the target schema are lost. Pass `EXECUTION_MODE='DEBUG'` first to inspect generated SQL if uncertain. | `studio/backend/fixtures/migration-scripts/*.sql` |
| 4. Optimize | Safe. `ALTER TABLE ADD PRIMARY KEY ...` errors with `already exists` are caught and treated as success by `DRY_RUN_PLAN`. | `EXA_OPTIMIZE.DRY_RUN_PLAN` |
| 5. Create cube | Registry-safe, virtual-schema-conflict. `sqlcube_builder.build_registry_sql()` does DELETE-then-INSERT per `DOMAIN_ID` in FK-correct order, so `SQLCUBE_REGISTRY.*` rows are upserted cleanly. **But** `CREATE VIRTUAL SCHEMA` will fail with `object already exists` if the target name is reused. Either `DROP VIRTUAL SCHEMA <name>` first or pick a new `target_cube_name`. | `studio/backend/services/sqlcube_builder.py` + `services/deployer.py` |
| 6. Dashboard | Safe. New ephemeral HTML file per run; previous server can be killed first if port collides (see `references/dashboard-handoff.md` port-fallback note). | `exasol-dashboard` |

The orchestrator should detect step-5 collision pre-flight: if `SYS.EXA_VIRTUAL_SCHEMAS` already contains `:target_cube_name`, prompt the user to either drop it or rename before continuing. Do not silently drop.

A clean re-run against the same source typically only needs steps 3 + 5 redone; the orchestrator's precondition DAG should already detect that nano is running, the source connection isn't persistent, and PKs from a prior optimize remain in place.

## What ships at "done"

When `/oneshot-tour` completes successfully:

- Two new Exasol schemas (`STAGE_*` + `CUBE_*`) are live.
- The virtual schema is queryable via SQL, MCP, JDBC, ODBC.
- One Vega-Lite dashboard is rendered locally (URL surfaced to user).
- The user is in MCP-query mode — they can ask follow-up business questions and get charts / tables back without any further setup.

## Out of scope for v0.1

See `specs/_plans/oneshot-tour/transcript-v0.1.md` (in the factory-foundation repo) for the canonical transcript and the explicit out-of-scope list. Key omissions to be aware of:

- Multi-source joins — one source schema → one cube
- Cube versioning / rollback after publish
- Dashboard persistence beyond ephemeral HTML
- AWS Exasol cluster targets (local Nano only)
- Step skipping / partial runs

## Related Skills

- **exasol-nano-local** — Step 1: Docker Nano boot + port + JDBC dir
- **exasol-migrate** — Steps 2-3: `CREATE CONNECTION` + `EXECUTE SCRIPT EXA_DB_MIGRATION.<vendor>_TO_EXASOL(...)` per `studio/backend/routers/imports.py`. 16 vendors; snowflake is one preset.
- **exasol-optimize** — Step 4: `EXA_OPTIMIZE.ANALYZE_CONSTRAINTS` + `DRY_RUN_PLAN`
- **exasol-semantic-layer** — Step 5: `SQLCUBE_REGISTRY` rows (DOMAINS / MODELS / DIMENSIONS / MEASURES / JOINS / JOIN_PATHS / ATTRIBUTES / DERIVED_MEASURES) + `CREATE VIRTUAL SCHEMA` via SQLCube adapter. **Not** `SQLCUBE_META` — that is a different schema (app state + lineage), unrelated to cube metadata.
- **exasol-dashboard** — Step 6: Vega-Lite spec + local HTML server

These five domain skills are the actual workforce. `/oneshot-tour` is the project manager.

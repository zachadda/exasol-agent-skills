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
| 2 | `exasol-migrate-snowflake` (or s3 / jdbc variant) | `source_connected` |
| 3 | `exasol-migrate-snowflake` (continues) | `schema_imported` |
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
- **exasol-migrate-snowflake** — Steps 2-3: connection + IMPORT INTO
- **exasol-optimize** — Step 4: constraint inference + DRY_RUN_PLAN
- **exasol-semantic-layer** — Step 5: cube create + SQLCUBE_META + virtual schema
- **exasol-dashboard** — Step 6: Vega-Lite spec + local HTML server

These five domain skills are the actual workforce. `/oneshot-tour` is the project manager.

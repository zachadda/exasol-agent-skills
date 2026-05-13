# Phase 2 — Plan presentation

After Phase 1 fills every required parameter, the orchestrator walks the precondition DAG, resolves the six steps, and presents the plan to the user. This phase is the final approval gate before any database side effects.

## Goal

Show the user exactly what's about to happen, in concrete terms, before any tool call is made. Get explicit consent (or step-by-step opt-in).

## What the user sees

```
agent:
  ## Plan

  Resolved 6 steps for `source_schema = ACME_DEMO`, target `CUBE_ACME_ADVENTUREWORKS`.

  | # | Step                              | Skill                       | Estimated impact            |
  |---|-----------------------------------|-----------------------------|-----------------------------|
  | 1 | Boot Nano                         | exasol-nano-local           | 5 sec, no DB changes        |
  | 2 | Connect to Snowflake              | exasol-migrate-snowflake    | New CONNECTION object       |
  | 3 | Import schema → STAGE_ACME_ADVENTUREWORKS | exasol-migrate-snowflake | ~14 tables, ~146k rows, ~2 min |
  | 4 | Optimize STAGE_ACME_ADVENTUREWORKS | exasol-optimize             | ~23 ALTER statements, 30 sec |
  | 5 | Create cube CUBE_ACME_ADVENTUREWORKS | exasol-semantic-layer    | SQLCUBE_META rows + virtual schema |
  | 6 | Verify + render dashboard         | exasol-dashboard            | 1 sample query + local HTML |

  - LLM enrichment: **on** (will refine names/groupings at steps 4 + 5)
  - Approval mode requested: **auto_accept**
  - Stop conditions: any tool error or empty verification result.

  Proceed? (yes / no / step — switch to step-by-step)
```

## How the orchestrator builds the plan

1. Start from the target capability (`cube_live`).
2. Walk preconditions backwards as a DAG:
   ```
   cube_live ← schema_optimized ← schema_imported ← source_connected ← nano_running
   ```
3. For each node, run the `check` SQL. If satisfied, mark `✓ already done` and skip in the rendered plan.
4. Resolve unsatisfied nodes to their `satisfied_by` skill. That skill becomes a planned step.
5. Append the final Phase-4-and-5 steps (verification + dashboard).
6. Render the table with column widths sized to the longest field.

## Already-satisfied nodes

If a precondition is already true (e.g., Nano already running, Snowflake connection already exists), surface it explicitly so the user understands what's being skipped:

```
agent:
  Already in place — skipping:
  - ✓ Nano is running (port 8564 reachable)
  - ✓ Snowflake connection `snowflake_demo_conn` exists

  Proceeding with 4 of 6 steps:
  | # | Step                              | ...
  | 3 | Import schema → STAGE_…           | ...
  | 4 | Optimize                          | ...
  | 5 | Create cube                       | ...
  | 6 | Verify + render dashboard         | ...
```

## Estimated-impact lines

Each domain skill declares an `estimated_impact` string in its frontmatter (TBD field — see roadmap). For v0.1, hardcode these in the plan-builder:

| Step | Estimated impact (template) |
|---|---|
| 1 | "5 sec, no DB changes" |
| 2 | "New CONNECTION object" or "Reusing existing" |
| 3 | "~{N} tables, ~{M} rows, ~{Tmin} min" — N from `SHOW SCHEMAS` count, M from `IMPORT` preview, Tmin estimated 1 min per 100k rows |
| 4 | "~{N} ALTER statements, 30 sec" — N from `ANALYZE_CONSTRAINTS` dry-run count |
| 5 | "SQLCUBE_META rows + virtual schema" |
| 6 | "1 sample query + local HTML" |

If any of the source-side counts can't be determined cheaply (slow Snowflake `INFORMATION_SCHEMA` queries), omit the estimate and write "rows count tbd".

## Acceptance handling

The user has three valid responses after seeing the plan:

| Response | Action |
|---|---|
| `yes`, `y`, `go`, `proceed` | Move to Phase 3 in `approval_mode = auto_accept` |
| `step` | Move to Phase 3 in `approval_mode = step_by_step` (override) |
| `no`, `n`, `cancel`, `abort` | Terminate. No work attempted. Acknowledge and exit |

Anything else: reprompt with the three valid options. Do not start running.

## When to re-present

After execution finishes (whether success or halt), do NOT re-present the plan. Phase 5 handles handoff. Phase 2 runs exactly once per tour.

If the user changes a parameter mid-run (rare), back out to Phase 1 and rebuild from scratch. Plans are immutable once accepted.

## Edge cases

| Situation | Action |
|---|---|
| Precondition DAG can't be resolved (no skill `satisfied_by` for some node) | Halt before Phase 2 ends. Report: "Can't continue — no skill provides `<capability>`. Add a skill or pick a different journey." |
| All preconditions already satisfied | Phase 2 still runs. Plan table shows just steps 4-6 (or whatever's left). User must still approve. |
| Estimated impact returns `0 rows` for the import step | Surface a warning: "Source schema has no tables — nothing to import. Continue anyway?" |
| User flips approval mode mid-prompt | Honor the most recent choice |

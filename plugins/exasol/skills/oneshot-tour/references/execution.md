# Phase 3 — Execution

After Phase 2 approval, the orchestrator runs the resolved plan one step at a time. This file is the agent's playbook for sequencing, narration, and hand-off.

## Goal

Run each planned step in order. Hand off to the domain skill responsible. Verify the step's `provides` capability before moving on. Surface task progress to the user in real time. Stop on first error.

## Sequencing

```
for step in plan.steps:
    1. Announce step start (header + step name + skill responsible)
    2. Load the domain skill's SKILL.md (deferred-load, not eager)
    3. Pass the relevant parameter subset from Phase 1 into the skill
    4. Execute the skill's routing algorithm — it owns its own tool calls
    5. On skill completion:
         a. Run the `provides.verify` SQL declared by the skill
         b. If verify passes → mark step done, advance
         c. If verify fails → halt, surface clear error, exit Phase 3
    6. If approval_mode == step_by_step → prompt user before next step
```

## Task progress surfacing

Every step uses `TaskCreate` + `TaskUpdate` so progress renders in the user's IDE the way Speq tasks do. One task per planned step.

Skeleton:

```python
# At plan acceptance (end of Phase 2)
for step in plan.steps:
    TaskCreate(subject=step.title, description=step.skill_name)

# During execution
TaskUpdate(taskId=N, status="in_progress")
# ... domain skill runs ...
TaskUpdate(taskId=N, status="completed")  # or "deleted" on error
```

The user watches a checklist tick off without scrolling through verbose tool output.

## Step-by-step mode prompts

In `step_by_step` mode, after each step's verify passes, prompt:

```
agent:
  Step 4 / 6 complete: STAGE_ACME_ADVENTUREWORKS optimized.
  ✓ 23 ALTER statements applied. 0 errors.

  Continue to Step 5 (Create cube)? (yes / no / inspect)

user:  yes  →  proceed to next step
user:  no   →  halt cleanly, summarize what landed, exit Phase 3
user:  inspect → drop into a read-only query mode against the current state; resume with "continue"
```

## Auto-accept mode behavior

In `auto_accept` mode, no per-step prompt. Move directly to the next step on verify pass. Halt on verify fail.

## Narration shape

Each step renders as a small markdown section. Keep it dense — assume the user is watching but not reading every line.

```
agent:
  ### Step 4 / 6 — Optimize STAGE_ACME_ADVENTUREWORKS

  Skill: exasol-optimize · LLM enrichment: on

    tool: EXECUTE SCRIPT EXA_OPTIMIZE.ANALYZE_CONSTRAINTS('STAGE_ACME_ADVENTUREWORKS')
      → 23 findings: 13 PK warnings, 8 critical FKs, 1 role-playing date, 1 INSERT_UNKNOWN_MEMBER

    tool: LLM enrich findings — flagged 2 self-FK candidates for manual review

    tool: EXECUTE SCRIPT EXA_OPTIMIZE.DRY_RUN_PLAN('STAGE_ACME_ADVENTUREWORKS', <21 stmts>, FALSE)
      → 21 applied, 0 errors

  ✓ Step 4 complete (28 sec). 21 constraints live.
```

Use a code-block-ish indent for tool lines so they're skimmable.

## Provides-verification pattern

After the domain skill returns, the orchestrator MUST run the skill's declared `provides.verify` SQL. Do not trust skill success messages alone — verify against the catalog.

Example for `exasol-optimize` → `provides: schema_optimized`:

```sql
-- declared in exasol-optimize/SKILL.md frontmatter:
verify: |
  SELECT COUNT(*) FROM SYS.EXA_ALL_CONSTRAINT_COLUMNS
  WHERE CONSTRAINT_SCHEMA = :source_schema
    AND CONSTRAINT_TYPE = 'PRIMARY KEY'

-- orchestrator runs:
-- if returned count > 0 → schema_optimized = true → advance
-- if returned count = 0 → halt, report "skill ran but produced no PK constraints"
```

## Parameter passing between phases

The orchestrator maintains a single `tour_state` dictionary that survives across phases:

```yaml
tour_state:
  source_kind:                snowflake
  source_connection_name:     snowflake_demo_conn
  source_schema:              ACME_DEMO
  source_table_filter:        adventureworks
  target_stage_schema:        STAGE_ACME_ADVENTUREWORKS
  target_cube_name:           CUBE_ACME_ADVENTUREWORKS
  llm_enrichment:             true
  approval_mode:              auto_accept
  steps_completed:            [1, 2, 3, 4]
  step_4_findings_count:      23
  step_4_applied_count:       21
```

Domain skills read the relevant fields and write back their counters/results. The orchestrator uses these to render later narration ("21 constraints applied" appears in Phase 5 dashboard handoff).

## Error / halt protocol

When any step fails, the orchestrator must:

1. Mark the current task as `deleted` (not `completed`) so it visually shows red/struck.
2. Print a halt header: `## ✗ Halted at step N — <step title>`
3. Print the actual error verbatim (do not paraphrase).
4. Print 2-3 suggested next moves:
   ```
   Suggested next steps:
   - Run `docker logs exanano-sqlcube` for container errors
   - Switch to step_by_step mode and re-run from this step
   - Inspect catalog: SELECT * FROM SYS.EXA_DBA_AUDIT_SQL WHERE ...
   ```
5. Do NOT retry automatically.
6. Exit Phase 3. Skip Phases 4+5.

User can restart with a new `/oneshot-tour` invocation.

## Inspect mode (step_by_step only)

If user says `inspect` between steps, drop them into a read-only sub-mode:

- Accept any `SELECT` query they type
- Run it against the current state via `mcp__exasol_db__query`
- Render results
- Stay in inspect mode until user says `continue` (advance to next step) or `cancel` (halt cleanly)

This lets users sanity-check between optimize and cube create, etc., without losing the orchestrator state.

## Edge cases

| Situation | Action |
|---|---|
| Skill takes > 5 minutes | Periodic progress pings every 60s ("still working...") |
| Domain skill itself prompts for user input mid-execution | Pass-through. The user sees the sub-skill's prompt and responds. Orchestrator resumes when sub-skill returns. |
| Multiple steps share the same domain skill (e.g. steps 2+3 both use `exasol-migrate-snowflake`) | Load the skill once. Pass distinct task subsets. |
| Tool call returns successful but data is wrong (e.g. import returns 0 rows) | The `provides.verify` SQL is the gate — if it would fail downstream, it must fail HERE |

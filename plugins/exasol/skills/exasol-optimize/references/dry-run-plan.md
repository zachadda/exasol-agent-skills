# DRY_RUN_PLAN — validate, self-heal, execute, optionally roll back

`DRY_RUN_PLAN(SCHEMA_NAME, PLAN_SQL, DRY_RUN)` is the safe path for applying a multi-statement batch of `ALTER TABLE … ADD CONSTRAINT …` and Unknown Member INSERTs. Same pipeline the Optimize panel uses. Validates first, injects missing REPAIR steps, refuses on ERROR, skips already-applied, runs the rest, rolls back DML on failure (or always, in dry-run mode).

## When to invoke

- Applying a review-pack of optimize proposals end-to-end.
- Re-applying a plan when only some steps succeeded last time (skip-already-applied logic handles the half-done state).
- Smoke-testing a plan against a Nano runtime before pushing it to AWS / production.
- CI-driven schema migration: dry-run on every PR, real run only on merge.

## Call

```sql
-- dry-run (default): rolls back DML, DDL still persists
EXECUTE SCRIPT EXA_OPTIMIZE.DRY_RUN_PLAN(
    'HOSPITALITY',
    'ALTER TABLE "HOSPITALITY"."DIM_X" ADD CONSTRAINT PK_DIM_X PRIMARY KEY ("ID");
     ALTER TABLE "HOSPITALITY"."FACT_Y" ADD CONSTRAINT FK_FACT_Y_X FOREIGN KEY ("X_ID") REFERENCES "HOSPITALITY"."DIM_X" ("ID") DISABLE;',
    TRUE
);

-- commit on success
EXECUTE SCRIPT EXA_OPTIMIZE.DRY_RUN_PLAN('HOSPITALITY', '...sql...', FALSE);
```

Call the script bare. `SELECT * FROM (EXECUTE SCRIPT …)` is invalid Exasol syntax.

## Returned columns

| Column | Meaning |
|---|---|
| `STATEMENT_INDEX` | 1-based position in the **input** plan (REPAIR-injected rows share their target index) |
| `CHECK_NAME` | `VALIDATION` · `EXECUTION` · `SUMMARY` — three result-row genres |
| `SEVERITY` | `PASS` · `REPAIR` · `ERROR` · `INFO` |
| `OBJECT_NAME` | The target table / constraint name, when resolvable |
| `MESSAGE` | Human-readable explanation |
| `SQL_TEXT` | The statement (or generated repair SQL) under consideration |
| `ACTION_NAME` | `ADD_PK` · `ADD_FK` · `MODIFY_COLUMN` · `INSERT_UNKNOWN_MEMBER` · `SKIP_STATEMENT` · `REPAIR` · `EXECUTE` · `OK_DDL_PERSISTED` · etc. |
| `REPAIR_SQL` | When `SEVERITY = REPAIR`, the suggested injected statement |

## Three-phase pipeline

### Phase 1 — Validate

Internally calls `EXA_OPTIMIZE.VALIDATE_JOIN_PLAN` over the input. Surfaces:
- `PASS` rows for statements that look applicable.
- `SKIP_STATEMENT` rows for statements already satisfied by the catalog (e.g. PK already declared on the target table — see `dry_run_skip_statement` torture case).
- `REPAIR` rows when a missing-prerequisite is detected (e.g. an FK statement whose PK isn't yet declared and isn't in the plan).
- `ERROR` rows when the target table doesn't exist, the column doesn't exist, or the type contract can't be met.

### Phase 2 — Inject + revalidate

Any `REPAIR` rows from phase 1 are prepended to the plan and validation runs a second time. The second pass must not produce new `REPAIR` or `ERROR` rows. If it does, the plan is unrecoverable and execution refuses.

Plans rarely produce two-level repairs in practice — most missing-PK-before-FK cases resolve in one inject pass. The double pass exists for safety.

### Phase 3 — Execute (or simulate)

For each non-skipped statement, in order:
1. Run via `pquery`.
2. On failure → `ROLLBACK`, emit `EXECUTION` row with the error message and `SEVERITY = ERROR`. Subsequent statements are not attempted.
3. On success in **commit mode** (`DRY_RUN = FALSE`) → `COMMIT`. The change is permanent. Emit `EXECUTION` row with `SEVERITY = PASS`.
4. On success in **dry-run mode** (`DRY_RUN = TRUE`) → `ROLLBACK` at the end. DML is undone. **But Exasol auto-commits DDL** — an `ALTER TABLE` already ran by the time `pquery` returned, and the ROLLBACK cannot reverse it. The execution row flags `ACTION_NAME = OK_DDL_PERSISTED` so the caller knows.

## The DDL-auto-commit caveat

This is the single most important thing to understand about `DRY_RUN_PLAN` in dry-run mode:

> **Successful DDL persists even in dry-run.** The script can't un-ALTER a table. If your plan adds a PK and then the FK fails, the PK stays.

Practical consequences:
- A dry-run that successfully ALTERs five tables and fails on the sixth statement leaves the first five ALTERs in place. Re-running the same plan as a fresh dry-run will see those five as `SKIP_STATEMENT` and only retry the sixth.
- The `SUMMARY` row will report `ddl_persisted = <N>` for any DDL count that ran successfully. **Always read the SUMMARY row** before declaring a dry-run "clean."
- For a true "no side effects" preview, use `VALIDATE_JOIN_PLAN` directly. It never runs anything, only validates.

## SUMMARY row

The final row in the output has `CHECK_NAME = 'SUMMARY'` and aggregates:
- `STATEMENT_INDEX` = total input statement count.
- `MESSAGE` = English summary (e.g. "5 statements: 4 executed, 1 skipped, 0 errors, 1 repair injected").
- `ACTION_NAME` = `SUMMARY`.
- `SQL_TEXT` = empty.

Smoke harness assertions usually key on this row's `SEVERITY` (`PASS` for clean, `ERROR` if any statement failed).

## Statement-splitting rules

`DRY_RUN_PLAN` splits the input on `;` boundaries while respecting:
- Single-quoted string literals (no split inside, `''` escapes correctly).
- Line comments `-- …` to end of line.
- Block comments `/* … */`.

Empty fragments are dropped. The splitter matches the Python `services.optimizer._split_sql_steps` so the same input parses identically in both contexts.

## Common failure patterns

| Failure | Cause | Fix |
|---|---|---|
| `ERROR` on FK ALTER | Type mismatch between FK and PK | Include the `MODIFY COLUMN` widen step before the FK ALTER (see `fk-widening.md`) |
| `ERROR` on PK ALTER | Data has NULLs or duplicates | Drop NULLs / dedupe first, or pick a different candidate (manual constraint editor) |
| `ERROR` on `INSERT INTO DIM_X` Unknown Member | A row with PK = -1 already exists | The generator uses `WHERE NOT EXISTS`, so this is rare. Check if a custom default already ran |
| `ERROR` on `MODIFY COLUMN … NOT NULL` | Column still has NULL rows | Run the UPDATE step (Step 2 in the Unknown Member block) first |
| `REPAIR` injected, validation still errors | Two-level missing prereqs | Manually add the second-level prereq to the plan and re-run |

## Smoke fixture cases

Coverage matrix in `acme-torture-patterns.md` lists the five DRY_RUN_PLAN scenarios the harness asserts:
- `dry_run_repair_injection` — PK injected before FK in the EXECUTION ordering.
- `dry_run_validation_error` — statement against nonexistent table; severity ERROR; no execution row.
- `dry_run_skip_statement` — already-applied PK on `DIM_PRE_PK`; validation row with `ACTION_NAME = SKIP_STATEMENT`.
- `dry_run_volume_50` — 50-statement batch; SUMMARY count matches input.
- `dry_run_mixed_ddl_dml` — DDL persists, DML rolls back; EXECUTION flags `OK_DDL_PERSISTED` for DDL.

All five pass against the current smoke harness. When extending `DRY_RUN_PLAN`, mirror the structure — synthesize a multi-statement batch, assert SUMMARY shape, and add a new case to `optimize_torture_cases.py` so regressions are caught.

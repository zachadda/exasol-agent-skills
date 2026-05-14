---
name: exasol-optimize
description: Database-first schema optimization for Exasol — Kimball-aware constraint inference, FK widening, join-path discovery, Unknown Member generation, and dry-run batch validation via the EXA_OPTIMIZE Lua UDFs.

# Structured contract (see skill-catalog-architecture for the orchestrator contract)
preconditions:
  - schema_imported:
      doc: "Target schema exists with at least one non-virtual table"
      check: |
        SELECT COUNT(*) AS N FROM SYS.EXA_ALL_TABLES
        WHERE UPPER("TABLE_SCHEMA") = UPPER(:source_schema)
          AND "TABLE_IS_VIRTUAL" = FALSE
      satisfied_by: exasol-migrate
  - udf_installed:
      doc: "EXA_OPTIMIZE.ANALYZE_CONSTRAINTS script is registered"
      check: |
        SELECT COUNT(*) AS N FROM SYS.EXA_ALL_SCRIPTS
        WHERE "SCRIPT_SCHEMA" = 'EXA_OPTIMIZE'
          AND "SCRIPT_NAME" IN ('ANALYZE_CONSTRAINTS', 'INFER_JOIN_PATHS',
                                'BUILD_UNKNOWN_MEMBER_INSERT', 'DRY_RUN_PLAN',
                                'VALIDATE_JOIN_PLAN')
        HAVING COUNT(*) >= 5
      satisfied_by: exasol-optimize-install   # self-install reference documented in udf-install.md

provides:
  - schema_optimized:
      doc: "Source schema carries declared PRIMARY KEY constraints on every PK candidate the heuristic could resolve"
      verify: |
        SELECT COUNT(*) AS N FROM SYS.EXA_ALL_CONSTRAINT_COLUMNS
        WHERE "CONSTRAINT_SCHEMA" = UPPER(:source_schema)
          AND "CONSTRAINT_TYPE" = 'PRIMARY KEY'
  - join_graph_emitted:
      doc: "INFER_JOIN_PATHS produces at least one row (declared_fk or stem_match) for the schema. Useful when downstream cube creation needs join discovery"
      verify: |
        -- Independent verification cannot run via SQL because INFER_JOIN_PATHS
        -- is invoked via EXECUTE SCRIPT and produces a result set, not a side-
        -- effect. Treat the orchestrator's captured-row count from Step N as
        -- the source of truth. See references/cube-profiler.md.

parameters:
  required:
    - source_schema:
        doc: "Schema to analyze (case-insensitive, will be uppercased)"
  optional:
    - apply_mode:
        default: dry_run
        doc: "dry_run = validate only via DRY_RUN_PLAN (rollback DML, DDL persists); commit = same plan but committed"
    - llm_enrichment:
        default: true
        doc: "When true, LLM annotates findings with column-naming hints and flags self-FK / hierarchy candidates for manual review before DRY_RUN_PLAN runs"
    - target_action:
        default: full_analyze
        doc: "full_analyze = ANALYZE_CONSTRAINTS + DRY_RUN_PLAN. constraints_only = ANALYZE_CONSTRAINTS only (no apply). joins_only = INFER_JOIN_PATHS only (no constraint changes)"

# Estimated impact templates for plan-presentation phase in orchestrator skills
estimated_impact:
  full_analyze: "~{N} ALTER statements, 30 sec — N from ANALYZE_CONSTRAINTS dry-run count"
  constraints_only: "Read-only catalog scan, 10 sec"
  joins_only: "Read-only catalog scan, 5 sec"
---

# Exasol Optimize Skill

Trigger when the user mentions **optimize Exasol schema**, **infer constraints**, **detect primary keys**, **detect foreign keys**, **add PK**, **add FK**, **build star schema**, **find fact tables**, **find dimensions**, **classify dim vs fact**, **validate plan**, **dry-run plan**, **add NOT NULL**, **widen FK column**, **build unknown member**, **Kimball unknown member**, **EXA_OPTIMIZE**, **ANALYZE_CONSTRAINTS**, **INFER_JOIN_PATHS**, **BUILD_UNKNOWN_MEMBER_INSERT**, **DRY_RUN_PLAN**, **role-playing date dim**, or **schema profile**.

## What this skill is

`EXA_OPTIMIZE` is four Lua UDFs that read `SYS.EXA_ALL_TABLES` / `EXA_ALL_COLUMNS` / `EXA_ALL_CONSTRAINT_COLUMNS` and emit generated SQL plus findings. Pure catalog-driven — no agent, no model. Runs in-database, so the same logic answers an agent prompt, a CI smoke check, or an operator panel.

| UDF | Purpose | Returns |
|-----|---------|---------|
| `ANALYZE_CONSTRAINTS` | Missing-PK proposals + inferred FKs (stem + inclusion-dep + role-playing date) + Unknown Member INSERTs | rows of `CATEGORY · SEVERITY · TABLE_NAME · COLUMN_NAME · TITLE · DESCRIPTION · SQL_TEXT · KIMBALL_CONTEXT · ACTION_NAME` (SEVERITY: `critical` · `warning` · `info`; ACTION_NAME: `ADD_PK` · `ADD_FK` · `WIDEN_FK` · `INSERT_UNKNOWN_MEMBER` · `NOT_NULL_TIGHTEN` · `NO_CANDIDATE` · `REJECTED_NULLS` · `REJECTED_DUPLICATES` · `REJECTED_CANDIDATE` · `FK_NO_MATCH`) |
| `INFER_JOIN_PATHS` | Catalog-only fact/dim classification + 3-tier join emission (declared / stem / same-name) | rows of `FACT_TABLE · FACT_KEY · DIM_TABLE · DIM_KEY · INFERENCE_SOURCE` (no SEVERITY column) |
| `BUILD_UNKNOWN_MEMBER_INSERT` | Type-aware idempotent INSERT for a single dim's sentinel `-1` row; length-clamped sentinel for narrow VARCHAR PKs; refuses declared composite PKs | one row of `SQL_TEXT · SEVERITY · MESSAGE` (`SEVERITY = ERROR` with empty SQL on composite-PK refusal, `PASS` otherwise) |
| `DRY_RUN_PLAN` | Validates a multi-statement plan, injects missing REPAIR steps, skips already-applied, executes, rolls back DML (DDL persists by Exasol policy) | per-validation + per-execution + SUMMARY row (SEVERITY values: `PASS` · `REPAIR` · `ERROR` · `INFO`) |

Two severity vocabularies coexist: `ANALYZE_CONSTRAINTS` uses `critical/warning/info`; `DRY_RUN_PLAN` and `BUILD_UNKNOWN_MEMBER_INSERT` use `PASS/REPAIR/ERROR/INFO`. They survive separately because their consumers want different signals (urgency vs action outcome). The unifying surface is the new `ACTION_NAME` column on `ANALYZE_CONSTRAINTS`: it gives every emitted row a stable verb (`ADD_PK`, `ADD_FK`, `WIDEN_FK`, `INSERT_UNKNOWN_MEMBER`, `NOT_NULL_TIGHTEN`, `NO_CANDIDATE`, `REJECTED_NULLS`, `REJECTED_DUPLICATES`, `REJECTED_CANDIDATE`, `FK_NO_MATCH`) that downstream consumers can route on regardless of severity vocab.

The UDFs live in schema `EXA_OPTIMIZE`. Install them once (see `references/udf-install.md`).

## Step 0: Verify EXA_OPTIMIZE is installed

```sql
SELECT "SCRIPT_NAME" FROM SYS.EXA_ALL_SCRIPTS
WHERE "SCRIPT_SCHEMA" = 'EXA_OPTIMIZE'
ORDER BY "SCRIPT_NAME";
```

Expect 4+ rows including `ANALYZE_CONSTRAINTS`, `INFER_JOIN_PATHS`, `BUILD_UNKNOWN_MEMBER_INSERT`, `DRY_RUN_PLAN`. If missing → load `references/udf-install.md` and install before continuing.

## Invocation syntax — important

These UDFs are Lua scripts that `RETURN TABLE`. Call them with `EXECUTE SCRIPT <name>(args);` and the result set comes back directly. **Do not** wrap in `SELECT * FROM (EXECUTE SCRIPT …)` — Exasol's SQL parser rejects that form with a syntax error. Likewise `SELECT * FROM EXA_OPTIMIZE.ANALYZE_CONSTRAINTS('SCHEMA')` (table-function call form) does not work for `RETURNS TABLE` scripts. Bare `EXECUTE SCRIPT` is the only working form.

Tool note: `exapump sql "EXECUTE SCRIPT …"` rejects the call with `Result set not available: Expected row count, got result set`. Run these scripts via a driver / endpoint that handles result sets (studio backend's `/api/query/execute`, pyexasol's `execute`, or interactive SQL clients that handle Lua result sets).

## Routing Algorithm

Determine the task; load **only** the references needed. Multiple routes can apply — load all that match.

1. **Diagnose missing PKs / inferred FKs / NULL-FK Unknown Member candidates** (the catch-all entry point):
   - Run: `EXECUTE SCRIPT EXA_OPTIMIZE.ANALYZE_CONSTRAINTS('<SCHEMA>');`
   - Load: `references/constraint-inference.md`
   - If FK proposals include `MODIFY COLUMN`: load `references/fk-widening.md`

2. **Empty schema or pre-PK schema — classify dim vs fact and emit join paths**:
   - Run: `EXECUTE SCRIPT EXA_OPTIMIZE.INFER_JOIN_PATHS('<SCHEMA>');`
   - Load: `references/cube-profiler.md`
   - For fact-classification rules (FACT_ prefix vs heuristic counts): also load `references/smoke-fact-detection.md`

3. **Generate the Kimball Unknown Member INSERT for one dim**:
   - Run: `EXECUTE SCRIPT EXA_OPTIMIZE.BUILD_UNKNOWN_MEMBER_INSERT('<SCHEMA>', '<DIM>', '<PK>');`
   - Load: `references/constraint-inference.md` (Unknown Member section)

4. **Validate or execute a multi-statement plan with self-healing**:
   - Run: `EXECUTE SCRIPT EXA_OPTIMIZE.DRY_RUN_PLAN('<SCHEMA>', '<plan_sql>', TRUE);`
   - Load: `references/dry-run-plan.md`

5. **Hit an adversarial / edge-case schema** (NULLs in PK candidates, composite keys, reserved-word columns, charset mismatches, stem collisions, paired FKs in same batch):
   - Load: `references/acme-torture-patterns.md`

6. **Production schema returned unexpected output** (zero join paths, sparse PK proposals, suffix-naming `*_DIM`):
   - Load: `references/real-schema-findings.md` (per-schema notes from running the UDFs against Adventureworks / Hospitality / Inventory)

7. **Install / reinstall the UDFs**, including the build/deploy story for fresh clusters:
   - Load: `references/udf-install.md`

## Conventions every route enforces

- **Always double-quote identifiers** in any SQL the UDFs emit *and* in any SQL you write to call them. Schema, table, column — `"FACT_ORDERS"."CUSTOMER_KEY"`. Prevents reserved-keyword breaks on schemas like `DIM_WIDE_RESERVED` (which has `"YEAR"`, `"ORDER"`, `"GROUP"`, `"SOURCE"` as column names).
- **PK before FK in any batch.** `ANALYZE_CONSTRAINTS` buffers paired FKs and emits them only after every missing-PK finding so downstream apply order is safe.
- **FK widening is non-lossy only.** Same family (VARCHAR↔VARCHAR or CHAR↔CHAR), same charset (UTF8↔UTF8 only), strictly wider PK. Direction is FK-side only — never widen a PK column (would force every other FK referencing it to widen in lock-step).
- **DDL in Exasol auto-commits.** A failed `DRY_RUN_PLAN` cannot un-do an `ALTER TABLE` that already ran. The SUMMARY row flags `ddl_persisted` so the caller knows. Plan order matters.
- **The smoke harness is the contract.** `scripts/optimize_torture_smoke.py` in `exasol-factory-foundation` asserts severity + action + SQL substring per case. When extending a UDF, add a case there or `acme-torture-patterns.md` first.

## Related Skills

- **exasol-database** — connection setup, SQL execution patterns, reserved-keyword handling, `IMPORT INTO` for loading test fixtures
- **exasol-udfs** — `CREATE SCRIPT` syntax, Lua-UDF specifics, how the `EXA_OPTIMIZE` scripts were authored
- **exasol-bucketfs** — when the UDFs need fixture-loaded reference data (rare for `EXA_OPTIMIZE`; not its main path)

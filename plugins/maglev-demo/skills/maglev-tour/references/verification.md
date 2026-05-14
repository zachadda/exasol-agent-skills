# Phase 4 — Verification

Before rendering the persona showcase: prove all three provides hold.

## Three checks

```sql
-- 1. cube_live
SELECT 1 FROM SYS.EXA_VIRTUAL_SCHEMAS
WHERE SCHEMA_NAME = 'SQLCUBE_ACME_ADVENTUREWORKS'
  AND ADAPTER_SCRIPT_SCHEMA = 'SQLCUBE'
  AND ADAPTER_SCRIPT_NAME = 'ADAPTER';

-- 2a. rcls_row_seeded
SELECT COUNT(*) FROM SQLCUBE_REGISTRY.RCLS_ROW_POLICIES
WHERE MODEL_ID = 'internet_sales';
-- Expect 4 (SYS + 3 RCLS personas).

-- 2b. rcls_attribute_seeded (column masks)
SELECT COUNT(*) FROM SQLCUBE_REGISTRY.RCLS_ATTRIBUTE_POLICIES
WHERE MODEL_ID = 'internet_sales';
-- Expect 2 (GROSS_MARGIN DENY for RCLS_CANADA + RCLS_EUROPE).

-- 3. data round-trip
SELECT COUNT(*)
FROM "SQLCUBE_ACME_ADVENTUREWORKS"."internet_sales";
-- Expect > 0 (live-verified: ~84k rows post-Step-4).
```

Plus a Phase-8 sanity check (do not skip — without this you don't know RCLS is firing):

```python
# Pre-flight persona-switch — confirms the impersonation path works before the showcase render.
def preflight_persona_switch(virtual_schema, model_id):
    sql = f'SELECT COUNT(*) AS N FROM "{virtual_schema}"."{model_id}"'
    sys_n     = post_query_execute(sql, execution_user='SYS').rows[0]['N']
    canada_n  = post_query_execute(sql, execution_user='RCLS_CANADA').rows[0]['N']
    assert canada_n < sys_n, "RCLS predicate did not narrow results — check registry grants + ALTER VIRTUAL SCHEMA REFRESH"
```

## Halt conditions

| Check | Failure mode | Diagnosis |
|---|---|---|
| 1 | virtual schema row missing | deploy/execute didn't land — re-run Phase 6 |
| 1 | row exists but ADAPTER mismatch | wrong adapter script wired — re-deploy fixes |
| 2 | 0 rows | seed-demo never ran — re-run Phase 7 |
| 2 | < 4 rows | seed partial — most likely model_id mismatch; run `SELECT MODEL_ID FROM SQLCUBE_REGISTRY.MODELS` and re-seed with the correct id |
| 3 | 0 rows | migration didn't land — re-run Phase 4 |
| 3 | error "object not found" | virtual schema wrong; quote both identifiers `"SCHEMA"."model_id"` |
| pre-flight | canada_n == sys_n | RCLS not firing — three likely causes: registry grants missing (re-run seed-demo), adapter cache stale (`ALTER VIRTUAL SCHEMA "<vs>" REFRESH`), or pyexasol client cache holding old session |
| pre-flight | canada query errors "Sales Territory Country not found" | DIMSALESTERRITORY isn't in the cube's join graph — Phase 5 two-pass didn't run, the FK never landed, the cube generator skipped that dim. Re-run Phases 5 + 6. |

## Cumulative state check

If anything halts, run the full state dump for diagnosis:

```sql
SELECT 'virtual_schemas' AS T, SCHEMA_NAME AS V FROM SYS.EXA_VIRTUAL_SCHEMAS
UNION ALL
SELECT 'models',        MODEL_ID FROM SQLCUBE_REGISTRY.MODELS
UNION ALL
SELECT 'rcls_policies', SUBJECT_NAME FROM SQLCUBE_REGISTRY.RCLS_ROW_POLICIES
UNION ALL
SELECT 'rcls_users',    USER_NAME FROM SYS.EXA_ALL_USERS WHERE USER_NAME LIKE 'RCLS_%'
UNION ALL
SELECT 'connections',   CONNECTION_NAME FROM SYS.EXA_ALL_CONNECTIONS WHERE CONNECTION_NAME LIKE 'SNOWFLAKE%'
ORDER BY 1, 2;
```

Paste the result into the halt message so the user can diagnose without re-running.

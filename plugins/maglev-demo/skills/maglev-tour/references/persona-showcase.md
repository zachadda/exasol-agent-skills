# Phase 5 — Persona showcase

The demo punchline. Same SQL, four sessions, four results.

Use the `exasol-query-personas` skill — this file just documents the narration shape.

## Canonical SQL

```sql
SELECT "Sales Territory Country",
       SUM("Sales Amount") AS "Total Sales"
FROM "SQLCUBE_ACME_ADVENTUREWORKS"."internet_sales"
ORDER BY 2 DESC
```

(Sorted DESC so the variation shows up in the first visible row. No LIMIT — let the row count tell the RCLS story.)

## Render

```
agent:
  ### Step 8 of 8 — Persona showcase

  Running the same SQL as four different users. Cube didn't change.
  CURRENT_USER did — and the Lua adapter joined a different
  RCLS_ROW_POLICIES.ROW_PREDICATE_SQL into the generated query.

  | Persona      | Rows | Top country     | Visible total |
  |--------------|------|-----------------|---------------|
  | SYS          |    6 | United States   |       $29.36M |
  | RCLS_CANADA  |    1 | Canada          |        $1.98M |
  | RCLS_EUROPE  |    3 | United Kingdom  |        $8.93M |
  | RCLS_EXEC    |    6 | United States   |       $29.36M |

  Verified live against the demo-tour cube on 2026-05-14.

  This is real Exasol session impersonation — the backend opened a
  fresh pyexasol connection authenticating as each user. The audit
  trail shows four distinct sessions:

    SELECT SESSION_ID, USER_NAME, COMMAND_NAME
    FROM SYS.EXA_DBA_AUDIT_SQL
    WHERE START_TIME > NOW() - INTERVAL '5' MINUTE
      AND USER_NAME LIKE 'RCLS_%' OR USER_NAME = 'SYS'
    ORDER BY START_TIME DESC;
```

## What to NOT do during the demo

- Don't render the persona table before all four queries return — partial results look like a bug, not a demo.
- Don't switch personas mid-query — fan-out is parallel-safe but mid-query session changes will surface as inconsistent rowcounts.
- Don't reveal the password (`RclsDemo1!`) on screen. The skill handles auth internally; the boss just sees the persona names.
- Don't claim the rules are written into Snowflake. They're in `SQLCUBE_REGISTRY.RCLS_ROW_POLICIES` on Exasol, applied by the SQLCube adapter at query time. The source schema is unchanged.

## After the showcase

- Offer follow-up: "want to try a custom query as one of these personas?" → continues in `/query` builder via studio frontend, or via plain `POST /api/query/execute {execution_user}` in chat.
- Reset for next demo: `POST /api/browser/demo/reset {owner_name: "demo"}` clears non-system-managed cubes. RCLS users and policies persist for re-runs.

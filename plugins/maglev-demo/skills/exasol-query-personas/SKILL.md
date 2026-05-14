---
name: exasol-query-personas
description: Run a single SQL query against a SQLCube virtual schema as N different Exasol personas in sequence, showing the row-count + value divergence that RCLS produces. Wraps POST /api/query/execute with the studio backend's execution_user impersonation feature (real Exasol session-as-user open per call). The side-by-side result is the canonical demo punchline. Demo-only; not in the public exasol-skills marketplace.

preconditions:
  - rcls_seeded:
      doc: "Target cube has demo personas + ROW policies seeded. Without this every persona sees the same rows."
      check: |
        # SQL:
        # SELECT COUNT(DISTINCT SUBJECT_NAME) FROM SQLCUBE_REGISTRY.RCLS_ROW_POLICIES WHERE MODEL_ID = :model_id;
        # Returns >= 4 when seed-demo ran.
      satisfied_by: exasol-rcls
  - studio_backend_reachable:
      doc: "Studio FastAPI backend reachable. The execution_user impersonation happens in services/query.py opening a NEW pyexasol connection as the persona — not exposed as plain SQL."
      check: |
        # Bash:
        # curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:8002/api/auth/me
      satisfied_by: null

provides:
  - persona_results_rendered:
      doc: "For each persona, ran the SAME SQL and captured (row_count, sample_rows). Differences in row count or visible attribute values demonstrate RCLS in action."

parameters:
  required:
    - virtual_schema:
        doc: "e.g., 'SQLCUBE_ACME_ADVENTUREWORKS'"
    - model_id:
        doc: "Virtual table inside the schema, e.g., 'internet_sales'"
  optional:
    - personas:
        default: ["SYS", "RCLS_CANADA", "RCLS_EUROPE", "RCLS_EXEC"]
        doc: "Order of personas to query as. The four-element default IS the canonical demo sequence."
    - demo_sql:
        default: |
          SELECT "Sales Territory Country", SUM("Sales Amount") AS "Total Sales"
          FROM "<virtual_schema>"."<model_id>"
          ORDER BY 2 DESC
        doc: "Query to run as each persona. Default reveals the RCLS predicate cleanly via country breakdown."
    - studio_base_url:
        default: "http://127.0.0.1:8002"

estimated_impact:
  v01_run: "~3-5 sec total for the four-persona default. Each persona opens a fresh pyexasol session as that user — first session is slow (~500ms TLS), subsequent fast."

internal: true
not_in_marketplace: true
---

# Exasol Query Personas skill

Run the same query under each of N personas, capture the divergent results, render side-by-side. This is the demo punchline: "watch the data shrink as we switch from SYS to Canada salesperson."

## When to trigger

- User asks "switch personas", "show me what each role sees", "demo the RCLS impersonation", "compare SYS vs RCLS_CANADA".
- `/maglev-tour` orchestrator at Phase 8.

Do NOT trigger when there's only one persona to query — use plain `POST /api/query/execute` without `execution_user` instead.

## What this skill is

Wraps the `execution_user` parameter on `POST /api/query/execute`. Per the studio backend `services/query.py:_get_query_connection()`:

- `execution_user=None` (or omitted) → query runs as the studio's admin connection.
- `execution_user='<USERNAME>'` → backend opens a **brand new pyexasol connection** to Exasol authenticating as `<USERNAME>` (password resolved from `services/runtime_registry.DEMO_RCLS_SCENARIOS`). The Lua adapter sees `CURRENT_USER='<USERNAME>'` and applies the matching `RCLS_ROW_POLICIES` predicate as `AND ...` on the generated SQL.

Real session impersonation, not a string-substitution mock. The personas must (a) exist as Exasol users with valid passwords, (b) have `CREATE SESSION` privilege, (c) have `USAGE`/`SELECT` on the virtual schema + the registry tables. `exasol-rcls` handles all three.

## The contract

```
POST /api/query/execute
Body: { "sql": "<SELECT ...>", "execution_user": "RCLS_CANADA" }
Returns: { "rows": [...], "columns": [...], "rowcount": N, "error": null }
```

Run once per persona; collate.

## Canonical demo sequence

The four-persona default produces the visible RCLS punchline:

```python
personas = ["SYS", "RCLS_CANADA", "RCLS_EUROPE", "RCLS_EXEC"]
sql = '''
  SELECT "Sales Territory Country", SUM("Sales Amount") AS "Total Sales"
  FROM "SQLCUBE_ACME_ADVENTUREWORKS"."internet_sales"
  ORDER BY 2 DESC
'''
```

Expected results against the demo-tour fixture (live-verified 2026-05-14):

| Persona | Row count | Top row |
|---|---|---|
| SYS | 6 | United States · $9.39M |
| RCLS_CANADA | 1 | Canada · $1.98M |
| RCLS_EUROPE | 3 | United Kingdom · $3.39M (+ Germany, France) |
| RCLS_EXEC | 6 | United States · $9.39M |

The agent should narrate: "same SQL, four sessions, four results — the cube didn't change, the user's row scope did. RCLS is enforced inside Exasol via session impersonation; the Lua adapter reads `CURRENT_USER` on every query and joins in the matching `RCLS_ROW_POLICIES.ROW_PREDICATE_SQL`."

## Render template

For a chat/terminal demo:

```
agent:
  ### Persona showcase — same SQL, different row scope

  | Persona | Rows | Top country | Total visible |
  |---|---|---|---|
  | SYS         | 6 | United States | $29.36M |
  | RCLS_CANADA | 1 | Canada        |  $1.98M |
  | RCLS_EUROPE | 3 | UK            |  $8.93M |
  | RCLS_EXEC   | 6 | United States | $29.36M |

  The cube definition is identical for every user. Only CURRENT_USER changes.
```

## Mechanics — what actually happens per persona

```
POST /api/query/execute {sql, execution_user='RCLS_CANADA'}
→ backend services/query.py:_get_query_connection('RCLS_CANADA')
  → DEMO_RCLS_SCENARIOS lookup for username
  → pyexasol.connect(dsn=<exa_host>, user='RCLS_CANADA', password='RclsDemo1!', ...)
  → conn.execute(sql)
    → Exasol parses + sends to SQLCUBE.ADAPTER (virtual schema)
    → adapter reads SQLCUBE_REGISTRY.RCLS_ROW_POLICIES WHERE SUBJECT_NAME = CURRENT_USER
    → adapter appends '"Sales Territory Country" = ''Canada''' to WHERE
    → adapter emits the rewritten physical SQL
    → Exasol executes against ACME_ADVENTUREWORKS tables
  → result returned to backend → JSON to agent
```

The impersonation is real — `SYS.EXA_DBA_AUDIT_SQL` shows the session opened as the persona, not as the admin user.

## Limitations

- v0.1 demos only the four canonical personas. Custom personas require pre-existing Exasol users with passwords known to the studio backend.
- The hidden-attribute story (RCLS_CANADA hides `GROSS_MARGIN`) requires ATTRIBUTE_POLICIES which `exasol-rcls` does NOT seed in v0.1. The default demo query intentionally doesn't reference Gross Margin to sidestep this gap.
- One persona per HTTP call — no batched-impersonation endpoint. Skill fans out 4 sequential calls; total time bounded by 4× TLS handshake on cold Nano.
- Password reuse across personas: every demo persona shares `RclsDemo1!`. Acceptable for local demo; do NOT carry this to a customer cluster.

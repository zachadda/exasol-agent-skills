---
name: exasol-rcls
description: Seed and verify row-level cube security (RCLS) on a deployed SQLCube domain. Wraps the studio backend's POST /api/browser/security/seed-demo endpoint plus the underlying services/runtime_registry.py:seed_demo_policies. Creates demo personas (RCLS_CANADA / RCLS_EUROPE / RCLS_EXEC), grants SCHEMA + REGISTRY-table USAGE/SELECT, writes ALLOW row predicates into SQLCUBE_REGISTRY.RCLS_ROW_POLICIES, and issues ALTER VIRTUAL SCHEMA REFRESH so the Lua adapter reloads cached metadata. Demo-only; not in the public exasol-skills marketplace.

preconditions:
  - cube_live:
      doc: "Target cube exists. Skill seeds policies against a specific MODEL_ID — that model must be in SQLCUBE_REGISTRY.MODELS."
      check: |
        # SQL:
        # SELECT 1 FROM SQLCUBE_REGISTRY.MODELS WHERE MODEL_ID = :model_id;
      satisfied_by: exasol-semantic-layer
  - studio_backend_reachable:
      doc: "Studio FastAPI backend reachable. Seed-demo wraps services calls that are not exposed as standalone SQL — must go through the backend."
      check: |
        # Bash:
        # curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:8002/api/auth/me
      satisfied_by: null

provides:
  - rcls_seeded:
      doc: "Four demo personas exist (SYS + RCLS_CANADA + RCLS_EUROPE + RCLS_EXEC), four ALLOW row policies live in SQLCUBE_REGISTRY.RCLS_ROW_POLICIES keyed on the target MODEL_ID, virtual schema refreshed."
      verify: |
        SELECT COUNT(*) FROM SQLCUBE_REGISTRY.RCLS_ROW_POLICIES WHERE MODEL_ID = :model_id;
        -- Expect >= 4 (the four canonical predicates plus any user-added rows).

parameters:
  required:
    - model_id:
        doc: "Target domain / model id. For the canonical Factory Tour demo: 'internet_sales'."
  optional:
    - studio_base_url:
        default: "http://127.0.0.1:8002"

estimated_impact:
  v01_seed: "~5 sec. Creates up to 3 new Exasol users (idempotent — skipped if present), grants USAGE + SELECT on the virtual schema + SQLCUBE_REGISTRY tables, INSERTs 4 policy rows, issues ALTER VIRTUAL SCHEMA REFRESH."

internal: true
not_in_marketplace: true
---

# Exasol RCLS skill

Row-level cube security via the SQLCube adapter's RCLS pushdown. Wraps the studio backend's `seed-demo` endpoint and its underlying `services/runtime_registry.seed_demo_policies` function.

## When to trigger

- User asks to "seed RCLS", "set up row-level security", "create demo personas", "wire up Canada/Europe/Exec users".
- `/factory-tour` orchestrator at Phase 7.

Do NOT trigger for ad-hoc security policy work — use `POST /api/browser/security/row-policies` directly for custom predicates.

## What this skill is

A thin wrapper around the canonical "demo RCLS" flow. Three things happen on `seed-demo`:

1. **Personas materialized.** Four Exasol users — `SYS` already exists; `RCLS_CANADA`, `RCLS_EUROPE`, `RCLS_EXEC` are created idempotently with password `RclsDemo1!` if missing. Each gets `GRANT CREATE SESSION`.
2. **Permissions granted.** Every persona gets `GRANT USAGE ON SCHEMA <virtual_schema>` + `GRANT SELECT ON SCHEMA <virtual_schema>` + `GRANT SELECT` on each `SQLCUBE_REGISTRY.*` table. **The registry grants are essential** — the Lua adapter loads policies as the impersonated user; without `SELECT` on `RCLS_ROW_POLICIES` the policies appear empty and the predicate silently doesn't fire.
3. **Policies written.** Four rows into `SQLCUBE_REGISTRY.RCLS_ROW_POLICIES` keyed on `MODEL_ID`:

   | Subject | Predicate |
   |---|---|
   | `SYS` | `1 = 1` (admin baseline; all rows) |
   | `RCLS_CANADA` | `"Sales Territory Country" = 'Canada'` |
   | `RCLS_EUROPE` | `"Sales Territory Country" IN ('France', 'Germany', 'United Kingdom')` |
   | `RCLS_EXEC` | `1 = 1` (all rows; hides nothing) |

4. **Adapter refresh.** `ALTER VIRTUAL SCHEMA "<virtual_schema>" REFRESH` so the Lua adapter drops its cached metadata and picks up the new policies on next query.

## The flow

```
POST /api/browser/security/seed-demo
Body: { "domain_id": "internet_sales" }
Returns: {
  "success": true,
  "model_id": "internet_sales",
  "current_user": "SYS",
  "apply_required": false,
  "created_row_policies": [ ... 4 rows ... ],
  "created_attribute_policies": [],
  "demo_scenarios": [ ... metadata for UI ... ],
  "message": "..."
}
```

The endpoint is **idempotent** — re-running against an already-seeded domain returns success with the existing policy set; user creation is skipped if present.

## Persona contract

Live in `factory-foundation/studio/backend/services/runtime_registry.py:DEMO_RCLS_SCENARIOS`:

```python
[
  { username: "SYS",          password: None,         label: "Admin baseline",     row_scope: "All rows",                                         predicate: "1 = 1" },
  { username: "RCLS_CANADA",  password: "RclsDemo1!", label: "Canada salesperson", row_scope: "Country = Canada",                                  predicate: '"Sales Territory Country" = \'Canada\'' },
  { username: "RCLS_EUROPE",  password: "RclsDemo1!", label: "Europe manager",     row_scope: "Country IN (France, Germany, UK)",                  predicate: '"Sales Territory Country" IN (\'France\', \'Germany\', \'United Kingdom\')' },
  { username: "RCLS_EXEC",    password: "RclsDemo1!", label: "Executive view",     row_scope: "All rows",                                          predicate: "1 = 1" },
]
```

Plus a `DEMO_RCLS_SUBJECTS = ["SYS", "RCLS_CANADA", "RCLS_EUROPE", "RCLS_EXEC", "*"]` whitelist of subjects the studio's attribute-visibility flow accepts.

## Verifying after seed

```sql
-- 1. Four policies for the model
SELECT POLICY_ID, SUBJECT_NAME, ROW_PREDICATE_SQL
FROM SQLCUBE_REGISTRY.RCLS_ROW_POLICIES
WHERE MODEL_ID = 'internet_sales'
ORDER BY POLICY_ID;

-- 2. Personas exist
SELECT USER_NAME FROM SYS.EXA_ALL_USERS
WHERE USER_NAME IN ('SYS', 'RCLS_CANADA', 'RCLS_EUROPE', 'RCLS_EXEC')
ORDER BY 1;

-- 3. Registry grants in place (sample one)
SELECT GRANTEE, OBJECT_NAME, OBJECT_TYPE
FROM SYS.EXA_DBA_OBJ_PRIVS
WHERE GRANTEE LIKE 'RCLS_%'
  AND OBJECT_SCHEMA = 'SQLCUBE_REGISTRY';
```

Then hand off to `exasol-query-personas` for the actual persona-switch demonstration.

## Adapter mechanics (worth knowing)

The Lua adapter at `sqlcube/adapter/metadata_registry.lua` reads `RCLS_ROW_POLICIES` once per query, filters by `CURRENT_USER`, AND-joins the matching predicate to the generated SQL's WHERE clause. Two failure modes the FACTORY_TOUR_DEMO_PATH doc calls out:

1. **Missing grants on `RCLS_ROW_POLICIES`** → adapter sees an empty result set → no policy applies → query returns all rows → silent security bypass. Always re-grant after table changes.
2. **Predicate references a dim attribute the cube doesn't expose** → adapter can't translate the business name to a physical column → query emits the literal "Sales Territory Country" → Exasol fails "column not found". Ensure DIMSALESTERRITORY is in the cube's join graph before seeding.

The skill's idempotent re-run includes the `ALTER VIRTUAL SCHEMA REFRESH` so the adapter drops cached metadata; without that the new policies wouldn't apply until the next adapter restart.

## When to skip / clean up

- **Skip** when the cube doesn't expose Sales Territory Country (predicates won't translate, demo punchline doesn't work).
- **Clean up** via `POST /api/browser/security/row-policies` DELETE per policy_id, or wipe via `POST /api/browser/demo/reset` (deletes all non-system-managed assets owned by the `demo` persona).

## Limitations

- v0.1 is the canonical four-persona demo only. Custom personas require `POST /api/browser/security/row-policies` direct usage.
- ATTRIBUTE policies (column-level masking) are NOT seeded by this skill — `created_attribute_policies` returns `[]`. Use `POST /api/browser/security/attribute-policies` for that.
- Adapter caches metadata per-virtual-schema. Re-seeding after the user has run a query in the current session may require a fresh connection for the new policies to fire — `ALTER VIRTUAL SCHEMA REFRESH` clears the server-side cache but not pyexasol's client buffer.

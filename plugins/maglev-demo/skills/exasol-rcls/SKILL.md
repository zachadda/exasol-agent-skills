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
        doc: "Target domain / model id. For the canonical Maglev Tour demo: 'internet_sales'."
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
- `/maglev-tour` orchestrator at Phase 7.

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

The Lua adapter at `sqlcube/adapter/metadata_registry.lua` reads `RCLS_ROW_POLICIES` once per query (cached in `adapterNotes`), filters by `CURRENT_USER`, OR-joins the matching ALLOW predicates, rewrites business names to physical columns via `rewrite_semantic_predicate`, and AND-joins the result into the pushdown SQL's WHERE clause.

## The three-layer bug (post-mortem — `exasol-zemantic-layer/MAGLEV_RCLS_DEEP_DIVE.md`)

These three issues stacked on each other in the GUI demo. All three patches must hold for RCLS to actually enforce. The current `seed-demo` endpoint embeds patches B + C; patch A lives in the upstream Tour Optimize step.

### 1. ANALYZE_CONSTRAINTS only emits FK ADDs after value-match validation

Hardened `analyze_constraints.sql` suggests a FK only when (a) candidate FK column matches by name, (b) target dim already has a PK on the matching column (so value-match can run), (c) ≥99% of fact rows match. On a fresh post-migrate schema there are zero PKs — first pass adds PKs + ~3 high-confidence FKs only. **Without a second analyze pass after PKs land**, ~5 FKs stay silent — DIMSALESTERRITORY among them. Cube introspection sees no FK → no join → "Sales Territory Country" attr never lands in `model_meta.columns` → adapter's `rewrite_semantic_predicate` can't translate → ships literal `"Sales Territory Country"` to Exasol → `object not found` error at query time.

**Fix**: Two-pass optimize in the Tour's Step 5. See `maglev-tour/references/execution.md`. Apply is idempotent — on `"constraint already used"` parse the name, DROP CONSTRAINT, retry ADD.

### 2. Adapter cache load runs AS the impersonated user → without grants, silently empty

Adapter caches `model_meta` (including row_policies) in `adapterNotes` on first pushdown per session. The cache-load query `SELECT * FROM SQLCUBE_REGISTRY.RCLS_ROW_POLICIES WHERE MODEL_ID=...` runs as `CURRENT_USER` — `RCLS_CANADA` (or whichever persona). Without `GRANT SELECT` on registry tables to that user, the query returns empty silently, the adapter caches an empty policy list, predicate reduces to `1=1`, and every persona sees every row.

**Most insidious failure mode** — seed reports success (rows ARE in `RCLS_ROW_POLICIES`), but enforcement doesn't happen. Discoverable only by querying as a persona and noticing identical row counts.

**Fix**: `seed_demo_policies` grants `SELECT` on every `SQLCUBE_REGISTRY.*` table (`ATTRIBUTES`, `DIMENSIONS`, `MEASURES`, `DERIVED_MEASURES`, `MODELS`, `JOINS`, `RCLS_ROW_POLICIES`, `RCLS_ATTRIBUTE_POLICIES`) to each persona. In `services/runtime_registry.py:_grant_virtual_schema_access`.

### 3. Adapter caches model_meta — new policies need `ALTER VIRTUAL SCHEMA REFRESH`

Even with FKs and grants right, the adapter's `adapterNotes` cache survives session boundaries. Re-running seed writes new policy rows but the next query still hits the cached empty (or stale) policy set until something invalidates the cache.

**Fix**: `ALTER VIRTUAL SCHEMA "<vs>" REFRESH` at end of `seed_demo_policies`. Drops cache; next pushdown rebuilds from the registry.

## Symptom → cause cheat sheet

| Symptom | Likely cause | Confirm |
|---|---|---|
| `object "Sales Territory Country" not found` in pushdown | DIMSALESTERRITORY missing from cube | `SELECT PHYSICAL_TABLE, COUNT(*) FROM SQLCUBE_REGISTRY.ATTRIBUTES WHERE DOMAIN_ID='internet_sales' AND ATTR_TYPE='dimension' GROUP BY 1` — should list 8 dim tables |
| All four personas return identical row counts | Adapter can't read policies as demo user | Run `SELECT COUNT(*) FROM SQLCUBE_REGISTRY.RCLS_ROW_POLICIES` with `execution_user='RCLS_CANADA'` via `/api/query/execute` — 401 or empty result means Patch B regressed |
| New seed didn't take effect | Stale adapter cache | Manually `ALTER VIRTUAL SCHEMA "SQLCUBE_..." REFRESH` and retest; if that fixes it, Patch C regressed |
| Seed reports `success: true` but enforcement wrong | Order of operations | Cube must deploy BEFORE seed. Never the reverse — the virtual schema must exist for REFRESH and the registry tables must exist for grants. |

## Adjacent traps that look like RCLS bugs

- `EXA_ALL_CONSTRAINTS WHERE CONSTRAINT_TYPE='FOREIGN KEY'` can return 0 rows even when FKs exist — catalog quirk on multi-column constraints (COLUMN_NAME NULL on header row). Inspect names instead.
- `COUNT(*) AS SQLCUBE_AGG_<N>` in pushdown for a bare measure column — different adapter bug, `aggregate_translator.lua` substring-matching column names against "COUNT"/"SUM"/etc. Already fixed by guarding column-type exprs.

## When to skip / clean up

- **Skip** when the cube doesn't expose Sales Territory Country (predicates won't translate, demo punchline doesn't work).
- **Clean up** via `POST /api/browser/security/row-policies` DELETE per policy_id, or wipe via `POST /api/browser/demo/reset` (deletes all non-system-managed assets owned by the `demo` persona).

## Limitations

- v0.1 is the canonical four-persona demo only. Custom personas require `POST /api/browser/security/row-policies` direct usage.
- ATTRIBUTE policies (column-level masking) are NOT seeded by this skill — `created_attribute_policies` returns `[]`. Use `POST /api/browser/security/attribute-policies` for that.
- Adapter caches metadata per-virtual-schema. Re-seeding after the user has run a query in the current session may require a fresh connection for the new policies to fire — `ALTER VIRTUAL SCHEMA REFRESH` clears the server-side cache but not pyexasol's client buffer.

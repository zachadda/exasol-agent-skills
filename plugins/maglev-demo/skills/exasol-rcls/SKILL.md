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
  "created_row_policies": [ ... 4 ALLOW rows, one per persona ... ],
  "created_attribute_policies": [ ... 2 DENY rows: GROSS_MARGIN for RCLS_CANADA + RCLS_EUROPE ... ],
  "demo_scenarios": [ ... metadata for UI ... ],
  "message": "Demo users and policies seeded; virtual schema refreshed."
}
```

Per `services/runtime_registry.py:seed_demo_policies`, the endpoint walks every entry in `DEMO_RCLS_SCENARIOS` and for each:

1. Ensures Exasol user exists with the canonical password (`RclsDemo1!`).
2. Grants USAGE + SELECT on the virtual schema + every SQLCUBE_REGISTRY table.
3. Writes the row predicate via `create_row_policy` (ALLOW).
4. For each name in `scenario["hidden_attributes"]`, writes an `RCLS_ATTRIBUTE_POLICIES` row via `create_attribute_policy` (DENY).

Live-verified shape: 4 row policies (SYS + 3 RCLS users) + 2 attribute policies (`GROSS_MARGIN` hidden from `RCLS_CANADA` and `RCLS_EUROPE`).

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

## Per-user column masking (fixed 2026-05-14 — commit `ca05f1b` on `demo-streamlined`)

Earlier the column-mask path was broken on two layers:

1. `DEMO_RCLS_SCENARIOS["hidden_attributes"] = ["GROSS_MARGIN"]` (slug). Cube exposed `"Gross Margin Pct"` (business name from metric pack). Adapter resolved `RCLS_ATTRIBUTE_POLICIES.VIRTUAL_COL` against business names → no match → policy was a no-op.
2. `apply_static_attribute_security` in `metadata_registry.lua` only enforced policies with `SUBJECT_NAME = '*'` (wildcard). Per-user DENY rows (`RCLS_CANADA`, `RCLS_EUROPE`) were loaded into `model_meta.attribute_policies` but **never applied** — there was no runtime-user mask code path.

Fix bundle (commit `ca05f1b`):

- `services/runtime_registry.py`: scenarios now declare business names: `"hidden_attributes": ["Gross Margin Pct"]`.
- `sqlcube/adapter/metadata_registry.lua`: new `apply_attribute_user_security` walks per-user DENY policies and stamps a `user_mask_predicate` SQL fragment (`UPPER(CURRENT_USER) IN ('RCLS_CANADA','RCLS_EUROPE')`) onto each matching measure / derived / dim. Called from `apply_runtime_security` alongside `apply_row_security`.
- `sqlcube/adapter/measure_builder.lua`: `build_select_expr` + `build_derived_select_expr` now wrap with `CASE WHEN <predicate> THEN NULL ELSE <expr> END` AFTER ROUND/CAST when `user_mask_predicate` is set. Masked output is NULL, not zero or a wrong-typed default.
- Adapter rebuilt via `sqlcube/adapter/build_adapter.py` and redeployed via `CREATE OR REPLACE LUA ADAPTER SCRIPT SQLCUBE.ADAPTER`. `ALTER VIRTUAL SCHEMA "SQLCUBE_<X>" REFRESH` after redeploy.

Live-verified vs `exanano-demo-tour` — double-axis showcase (Sales + Profit + Margin):

| Persona | Rows | Top country | Sales | Profit | Margin |
|---|---|---|---|---|---|
| SYS | 6 | US | $29,358,677 | $12,080,883 | 0.4121 |
| RCLS_CANADA | 1 | Canada | $1,977,845 | **NULL** | **NULL** |
| RCLS_EUROPE | 3 | UK | $8,930,042 | **NULL** | **NULL** |
| RCLS_EXEC | 6 | US | $29,358,677 | $12,080,883 | 0.4121 |

Row mask + column mask both fire per persona — demo punchline doubled. Sales still aggregates (column itself isn't dropped — measure_builder nulls at expression level via `CASE WHEN <user_mask_predicate> THEN NULL ELSE <agg_expr> END`).

`DEMO_RCLS_SCENARIOS` ships with two hidden attributes per restricted persona (`commit b314fee` on `demo-streamlined`):

```python
"hidden_attributes": ["Gross Margin Pct", "Gross Profit"]
```

Extension pattern: append any business-name string to `hidden_attributes` to mask additional measures. Adapter handles measure / derived_measure render-time wrapping automatically. Dim-level masking still requires the open extension noted below.

Open extension: dimension-level user masking. `apply_attribute_user_security` already stamps `user_mask_predicate` onto matching `dim_meta`, but `main.lua` dim SELECT-render doesn't wrap yet. Add the same `CASE WHEN` wrap there for a complete story. Demo's hidden attribute is `"Gross Margin Pct"` (a measure), so this didn't block the demo.

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
- ATTRIBUTE policies (column-level masking) ARE seeded by this skill — `GROSS_MARGIN` is hidden from `RCLS_CANADA` + `RCLS_EUROPE` per `DEMO_RCLS_SCENARIOS["hidden_attributes"]`. For custom column masks beyond the demo defaults, use `POST /api/browser/security/attribute-policies` directly.
- Adapter caches metadata per-virtual-schema. Re-seeding after the user has run a query in the current session may require a fresh connection for the new policies to fire — `ALTER VIRTUAL SCHEMA REFRESH` clears the server-side cache but not pyexasol's client buffer.

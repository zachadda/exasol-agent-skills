# Findings from real customer schemas

Notes from running the four `EXA_OPTIMIZE` UDFs against production-shaped schemas on the local Nano (2026-05-13). Captures behaviors and gaps the synthetic `ACME_TORTURE` fixture doesn't surface.

## Adventureworks — Microsoft sample warehouse

**Shape:** 9 tables. Every fact and dim already carries declared PKs and FKs from the source migration.

**ANALYZE_CONSTRAINTS:** 0 rows. Nothing to propose — schema is already fully constrained. The UDF correctly emits silence rather than chasing redundant PKs.

**INFER_JOIN_PATHS:** 11 rows, all `INFERENCE_SOURCE = 'declared_fk'`. Tier 1 dominates because the declared catalog is rich. Worth noting:

- The role-playing date pattern shows up clean: `FACTINTERNETSALES.DUEDATEKEY → DIMDATE.DATEKEY`, `FACTINTERNETSALES.ORDERDATEKEY → DIMDATE.DATEKEY`, `FACTINTERNETSALES.SHIPDATEKEY → DIMDATE.DATEKEY`. The UDF lists them as three separate tier-1 rows — matches the Kimball expectation that each role gets its own join.
- No tier-2 or tier-3 emissions because every column-pair is already declared. Confirms the priority order (`declared_fk` always wins over the inference tiers).

**Implication for the skill:** when a customer's source migration already lays down PK/FK constraints, ANALYZE has nothing to add. That's a *good* answer — don't propose changes the catalog already has. Coding agents should not flag the empty result as a UDF failure.

## Hospitality (Hilton-shaped) — semi-constrained warehouse

**Shape:** 12 tables. Mix of `FACT_*`, `DIM_*`, and bare `CALENDAR` / `CORPORATE_ACCOUNT`. No declared PKs or FKs.

**ANALYZE_CONSTRAINTS:** 13 `warning` rows — one PK proposal per table. Confirms that the entire schema gets surfaced when nothing is declared. Per-table proposals are reasonable: `DIM_CURRENCY` → PK on `CURRENCY_CODE`, `DIM_DATE` → PK on `LY_CAL_DATE` (the first TIMESTAMP column — calendar-table special case), etc.

**INFER_JOIN_PATHS:** 3 rows, all `stem_match`. Examples:

- `FACT_RESERVATION_SUMMARY.PROPERTY_CD → DIM_PROPERTY.PROPERTY_CODE` (stem `PROPERTY` matches after stripping `_CD` and `DIM_` prefixes)
- `FACT_RESERVATION_SUMMARY.LEAD_TIME_BUCKET_ID → DIM_LEADTIME_BUCKET.BUCKET_ID`
- `FACT_RESERVATION_SUMMARY.CURRENCY_CD → DIM_CURRENCY.CURRENCY_CODE`

Confirms the `_CD` ↔ `_CODE` cross-suffix matching (both strip to the same stem). Confirms the dim-prefix stripping (`DIM_PROPERTY` → `PROPERTY`).

**BUILD_UNKNOWN_MEMBER_INSERT:** Called against `DIM_CURRENCY` with PK column `CURRENCY_CODE`. Emits the quoted `'-1'` sentinel because the PK type is VARCHAR. Worth noting: the dim has a second column `CURRENCY_CODE_ID` (DECIMAL surrogate) so the INSERT actually carries two columns. UDF picks correct sentinels per column type — `-1` for the DECIMAL surrogate, `'-1'` for the VARCHAR natural key.

**Implication for the skill:** real-world dims often pair a natural key with a synthetic surrogate. The UDF handles both correctly when called with the natural-key column name.

## Inventory — suffix-naming partial-recovery

**Shape:** 9 tables — `INVENTORY_ON_HAND`, `INV_CALENDAR_DIM`, `INV_IN_TRANSIT`, `INV_ITEM_DRM`, `INV_SOURCE_SYSTEM_DIM`, `INV_STORE`, `ITEM_STATUS`, `EDB_SALES_LOCATIONS`, `SAP_SITE_TO_POS_LOC_CONV`.

Naming pattern is **suffix-based** (`_DIM`) rather than prefix-based (`DIM_`). The original classifiers only checked prefix; suffix recognition was added 2026-05-13 to both UDFs (`_DIM`, `_DIMENSION`, `_LOOKUP`, `_LKP`, `_REF`, `_LU` → dim; `_FACT`, `_FCT`, `_TXN`, `_TRX`, `_EVENT`, `_EVT` → fact). Longest-first iteration so `_DIMENSION` beats `_DIM`.

**ANALYZE_CONSTRAINTS (post-fix):** 2 rows.

- `INV_CALENDAR_DIM` → `warning` / `ADD_PK` with `PRIMARY KEY ("CALENDAR_DATETIME")`. The suffix triggers `is_dim`, which routes the table into the calendar-table special case in `suggest_pk_candidates` and the first DATE/TIMESTAMP column becomes the natural PK.
- `ITEM_STATUS` → `info` / `NO_CANDIDATE`. No key-suffix column and no obvious PK; the heuristic correctly stops rather than proposing a bad ALTER.

The remaining 7 tables emit nothing. Those that *should* be dims (`INV_SOURCE_SYSTEM_DIM`) classify now but have no key-suffixed columns so the heuristic falls through to "first non-nullable" / "first column" fallbacks and proposes a PK only when the column shape supports it.

**INFER_JOIN_PATHS (post-fix):** still 0 rows. Suffix detection lifts dims into the lookup, but no INVENTORY fact table emits a row because:

- `INVENTORY_ON_HAND` is shape-classified as a fact (`numeric_non_key > 2` per rule 2) but its FK-shaped columns are `SOR_ID`, `BRANCH`, `PART`, etc. Stems (`SOR`, `BRANCH`, `PART`) don't match any registered dim stem (`INV_CALENDAR`, `INV_SOURCE_SYSTEM`).
- Tier-3 same-name fallback doesn't fire because the verbatim column names don't appear on any dim's column list.

**This is the limit of catalog-only inference on schemas with weak naming hygiene.** The UDFs can't invent joins out of incompatible column-name conventions. Real-world remediation paths:

1. **Declare FKs manually.** The inclusion-dependency scan in ANALYZE_CONSTRAINTS will then surface the joins regardless of naming.
2. **Rename dim columns to match fact stems** (e.g. `INV_SOURCE_SYSTEM_DIM.SOURCE_SYSTEM_ID` to match `INVENTORY_ON_HAND.SOR_ID` stem). Better-but-invasive.
3. **Accept the gap** and use the cube generator's explicit-join input mode for INVENTORY-shaped schemas.

**Implication for the skill:** suffix detection helps with the dim side. Empty `INFER_JOIN_PATHS` output on a 9-table schema still typically means a naming-stem mismatch on the fact side, not a UDF defect. Document the limitation rather than over-extending the heuristic; over-eager fact-column-to-dim matching produces nonsense joins which are worse than no joins.

## LOC_MASTER — mixed-case identifiers + view inclusion

**Shape:** 4 tables (`DimBlockGrpLocations`, `DimConsumerHierarchy`, `DimLocationMaster`, `FactStoreHoursTimeZoneDaily`) + 1 view (`vwFactStoreHoursTimeZoneDaily`). Catalog preserves the mixed-case names verbatim — not uppercased. All four tables already carry declared PKs.

**ANALYZE_CONSTRAINTS:** 0 rows. Correct — every table is fully constrained so nothing to propose. Confirms the `existing_pks` skip path fires even when catalog names are not uppercase.

**INFER_JOIN_PATHS:** 2 rows, both `same_name_fallback`:

- `FactStoreHoursTimeZoneDaily.LocId → DimLocationMaster.LocId`
- `vwFactStoreHoursTimeZoneDaily.LocId → DimLocationMaster.LocId`

The fact's stem (`LOCID` → strip `_ID` → `LOC`) doesn't match the dim's stripped stem (`DIMLOCATIONMASTER` → `LOCATIONMASTER`), so tier 2 is empty. Tier 3 then finds the verbatim `LocId` column on exactly one dim and emits. The view shows up because `INFER_JOIN_PATHS` reads `EXA_ALL_VIEWS` too — `vwFact...` shape-classifies as a fact (numeric_non_key > 2 inherited from the underlying table).

**Implications for the skill:**
1. **Mixed-case catalog names work** end-to-end. The `:upper()` in classification and the literal-pass of catalog names in `ident()` cooperate cleanly.
2. **Tier-3 fallback covers stem mismatches** the dim's naming convention introduces. Without it, `LocId → DimLocationMaster.LocId` would land in zero output.
3. **Views as facts** is expected behavior when the view re-exposes a fact's measure shape. If you don't want the view emitting, drop it or hide it from `EXA_ALL_VIEWS` — there's no per-row toggle.

## INVESTMENT_RPT / INVESTMENT_CORE / RETAIL_FULFILLMENT — already-constrained or empty

Three schemas tested as a quick sanity sweep:

- **INVESTMENT_CORE** — 0 tables in catalog. Skipped.
- **INVESTMENT_RPT** — `FACT_INVESTMENT_PRICING` + `FACT_PORTFOLIO_TRANSACTION`. Both empty (0 rows). `FACT_INVESTMENT_PRICING` declares a composite PK (`FACT_INVESTMENT_PRICING_EDW_ID, INVESTMENT_SEQUENCE_EDW_ID`) plus seven NOT NULL columns. ANALYZE_CONSTRAINTS correctly emits 0 rows.
- **RETAIL_FULFILLMENT** — single empty `FACT_ORDER_FULFILLMENT_SUMMARY` carrying a declared composite PK on `(ROW_ID, ORDER_NBR)`. ANALYZE_CONSTRAINTS skips it via `existing_pks` and emits 0 rows. Correct.

These three add no new behavior coverage but confirm the silence-when-nothing-to-do contract holds across diverse production shapes.

## What this run did not surface

Limits of the local sample:

- No NULL-FK Unknown Member generation triggered. Hospitality has nullable FK candidates but ANALYZE_CONSTRAINTS' stem-inference stage requires both sides exist — and Hospitality's fact has stem matches but its dim has no declared PK, so the FK ALTER path runs only at the `info` level (no SQL emitted). To see real Unknown Member SQL, point ANALYZE at a schema where dims have declared PKs and facts have NULL FK columns. ADVENTUREWORKS doesn't have NULL FKs either.
- No 50-column wide reserved-keyword dim outside the torture fixture. Production schemas tested here use short, ASCII-safe column names. Quoting is exercised but reserved-keyword edge cases (`YEAR`, `ORDER`, `GROUP`, etc.) only land in `ACME_TORTURE.DIM_WIDE_RESERVED`.
- No DECIMAL precision/scale FK mismatch in production schemas tested. Adventureworks uses uniform `INT` keys, Hospitality uses uniform VARCHAR codes. The widen-refusal logic in `fk-widening.md` is unit-tested by `ACME_TORTURE.FACT_FK_DECIMAL_MISMATCH` only.
- No view-shadowed dim outside `VW_DIM_SHADOW`. Real schemas use materialized tables for cube layers.

When testing further customer schemas, watch for these missing surfaces — they're the cases the torture fixture covers in isolation but might land differently in production data.

## Recommended next schemas

- **Harvey BIM** (Power BI Tabular migration, 100 measures + 296 relationships). Likely to exercise role-playing dim, composite PK, and large-FK-graph behaviors.
- **ACME_DEMO** (from `exasol-customer-demos`). Designed as the canonical Factory Tour demo schema; should run clean end-to-end.
- **A real Snowflake-imported schema** with mixed naming (some `FACT_*` some bare). Surfaces classifier edge cases.

Each run should append a section here with `severity counts`, `inference source counts`, and any surprises.

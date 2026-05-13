# ANALYZE_CONSTRAINTS — PK / FK / Unknown Member inference

`ANALYZE_CONSTRAINTS(SCHEMA_NAME)` walks an Exasol schema's catalog and proposes the missing PRIMARY KEY, FOREIGN KEY, and NOT NULL constraints needed to make the schema a usable Kimball star. It also emits Kimball Unknown Member INSERT/UPDATE/ALTER blocks for any nullable FK column it finds.

## When to invoke

- A schema was loaded as plain tables (CSV / Parquet / IMPORT INTO) and has zero declared constraints.
- A migration ran but missed PK or FK declarations.
- A cube generator needs declared join paths to pick INNER over LEFT.
- A nightly profiler needs to flag drift in NULL counts on FK columns.

## Call

```sql
EXECUTE SCRIPT EXA_OPTIMIZE.ANALYZE_CONSTRAINTS('ADVENTUREWORKS');
```

The script `RETURNS TABLE`; the rows arrive as a result set. Wrapping in `SELECT * FROM (EXECUTE SCRIPT …)` fails with a syntax error — call the script bare.

## Returned columns

| Column | Meaning |
|---|---|
| `CATEGORY` | Always `constraints` today; reserved for future categories |
| `SEVERITY` | `critical` (validated FK / NULL-FK), `warning` (PK proposal / FK candidate), `info` (no candidate or unvalidated) |
| `TABLE_NAME` | Subject table |
| `COLUMN_NAME` | Subject column (`''` when finding is table-level, e.g. missing PK) |
| `TITLE` | One-line headline (`<TBL> has no primary key`, `<COL> -> <DIM>.<PK> (validated)`, …) |
| `DESCRIPTION` | Long-form rationale; includes discovery source (`name heuristic`, `value-matching`, `composite greedy search`, `role-playing date`) |
| `SQL_TEXT` | Generated DDL. Empty when no valid candidate exists |
| `KIMBALL_CONTEXT` | Optional one-paragraph rationale for non-obvious Kimball patterns |
| `ACTION_NAME` | Cross-UDF routing label: `ADD_PK` (PK proposal with SQL), `ADD_FK` (FK proposal, no widen needed), `WIDEN_FK` (FK proposal with `MODIFY COLUMN` widen step), `INSERT_UNKNOWN_MEMBER` (NULL-FK fix block), `NOT_NULL_TIGHTEN` (FK with no NULLs but column declared NULLABLE), `NO_CANDIDATE` (no PK candidate found), `REJECTED_NULLS` (PK candidate has NULL rows), `REJECTED_DUPLICATES` (PK candidate has duplicates), `REJECTED_CANDIDATE` (other validation failure), `FK_NO_MATCH` (FK candidate had no overlapping rows during validation). Stable values; consumers should treat unknown labels as opaque pass-through |

## Discovery stages (PK)

Run in order; the first stage that returns a validated candidate wins.

1. **Name heuristic.** Look at column-name suffixes: `_KEY`, `_ID`, `_FK`, `_SK`, `_CD`, `_CODE`, `_NUM`, `_NBR`, plus a calendar-table check (date-typed column → PK) and a rate-table check (date + two code columns → composite). Fact tables also pair `*LINENUMBER` with `*NUMBER` from the same stem. Validated against the data (NULL-free + unique) before emission.

2. **Inclusion-dependency / value-matching.** For each key-shaped column with no NULLs and full distinct count, scan every other table for a column whose non-NULL values are a strict subset. Tiered: exact name match → stem match → type-compatible value scan. The first match that passes validation wins. Also emits the paired FK on the matching side, queued so it lands after the PK ALTER (PK-before-FK invariant).

3. **Composite greedy search (populated tables).** Rank key-shaped columns by selectivity, take top 8, try every 2-combination then 3-combination, return the first set that validates. Bounded to C(8,2)+C(8,3) = 84 checks.

3a. **Name-based composite detection (empty / fresh tables).** When the row-stats-driven greedy stage can't run because the dim is empty, a name-pattern pass kicks in: dims whose columns share a prefix-then-single-letter pattern (`KEY_A, KEY_B`) or whose first 2–4 columns are consecutive single letters (`A, B, C`) get returned as a composite candidate. Catches the ACME_TORTURE composite-PK fixtures.

If all stages fail, the finding is `severity=info` with no `SQL_TEXT` and an explanatory `DESCRIPTION`. The caller can fall through to the manual-constraint-editor UI.

## Discovery stages (FK)

Three parallel scans, each guarded against duplicates by `(table, fk_col)`:

1. **Stem inference.** For every `FACT_*` table, strip a key suffix off each column to get a stem; look up that stem in the dim table-name index (`DIM_*` minus its prefix). Validate the join returns rows. Emit `ALTER … ADD CONSTRAINT … FOREIGN KEY (…) REFERENCES … DISABLE` with an optional widen step prepended (see `fk-widening.md`).

2. **Inclusion-dependency on declared PKs.** For every table whose PK is already declared and whose PK column is key-shaped, scan every other (non-empty) table for a key-shaped column with the same name stem or a `FACT_*` involvement. Require a key-suffix tail (`_ID`, `_KEY`, `_CD`, `_CODE`, `_NUM`, `_NBR`, `ID`, `KEY`) on the candidate to filter measure-shaped columns like `ADULT_OCCUPANCY_CNT`. Validate as subset.

3. **Role-playing date FK.** Find the schema's date dim (`sqlcube:dim_date` table comment, then well-known names — `DIMDATE`, `DIM_DATE`, `DATE_DIM`, `D_DATE`, `CALENDAR` — then any `DIM_*`/`D_*` with `DATE` in the name). For every fact, if a column ends in `DATEKEY`, `DATE_KEY`, or `DATE_ID` and isn't already a stem-matched FK, emit the FK to the date dim. Kimball-canonical pattern: many temporal columns join the same physical date dim under different aliases.

Each emitted FK ALTER uses constraint name `FK_<fact>_<fk_col>`. **DISABLE** is set on FKs found by stem inference / role-playing — metadata only, no enforcement overhead at write time. Inclusion-dependency-discovered FKs land enforced.

**Cross-stage dedupe**: the inclusion-dependency scan runs before stem/role-playing matching. When both stages find the same FK, the stem-match / role-playing row emits with empty `SQL_TEXT` (the ALTER already went out as a `warning` from the inclusion-dep stage). The downstream NULL-FK Unknown Member analysis still fires for that column — registration into the validated-FK list happens regardless of whether the SQL emitted, so a `FACT.<FK>` row with `SEVERITY=critical`, `ACTION=ADD_FK`, and empty `SQL_TEXT` is normal and pairs with a separate `INSERT_UNKNOWN_MEMBER` row carrying the four-step block.

## Unknown Member generation

When a validated FK column has at least one NULL, `ANALYZE_CONSTRAINTS` emits a four-step Kimball Unknown Member block as one `SQL_TEXT` value:

```sql
-- Step 1: Insert Unknown Member into <DIM>
INSERT INTO <SCHEMA>.<DIM> (...) VALUES (...);   -- type-aware defaults; -1 for PK

-- Step 2: Backfill NULL <FK_COL> to -1
UPDATE <SCHEMA>.<FACT> SET <FK_COL> = -1 WHERE <FK_COL> IS NULL;

-- Step 3: Set NOT NULL
ALTER TABLE <SCHEMA>.<FACT> MODIFY COLUMN <FK_COL> NOT NULL;

-- Step 3b: Widen <FK_COL> (non-lossy) so the FK ALTER matches the PK type   [optional]
ALTER TABLE <SCHEMA>.<FACT> MODIFY COLUMN <FK_COL> <widened_type>;

-- Step 4: Add FK constraint (DISABLED)
ALTER TABLE <SCHEMA>.<FACT> ADD CONSTRAINT FK_<FACT>_<COL> FOREIGN KEY (<FK_COL>)
REFERENCES <SCHEMA>.<DIM> (<DIM_PK>) DISABLE;
```

Type-aware defaults follow the existing Python `default_value` rules — `'Unknown'` for VARCHAR/CHAR/CLOB, `'1900-01-01'` for DATE, `'1900-01-01 00:00:00'` for TIMESTAMP, `FALSE` for BOOL, `0` for numeric measures, `-1` for the dim PK regardless of type (quoted for char-family).

For a single dim with no NULL FK pressure but still missing its Unknown Member row, call `BUILD_UNKNOWN_MEMBER_INSERT` directly — it's the same generator but standalone and type-aware on the PK literal (quotes when VARCHAR/CHAR/CLOB; bare numeric otherwise). The sentinel literal is **length-clamped**: `VARCHAR(1)` columns get `'?'` (not `'-1'`) so the INSERT fits the declared length. The UDF also **refuses declared composite PKs** — it returns `SEVERITY=ERROR` with an empty `SQL_TEXT` because the single-column sentinel can't generalize to multi-column dedupe.

## NULLABLE-but-empty FK columns

If the FK column has zero NULL rows but is declared NULLABLE, emit a `warning` finding with a single `ALTER TABLE … MODIFY COLUMN <FK> NOT NULL;`. The cube generator only upgrades a join to INNER when the FK column is NOT NULL. Without the upgrade it defaults to LEFT, which blocks the optimizer's free join reordering.

## Severity ladder summary

| Severity | When emitted |
|---|---|
| `critical` | Validated FK to a real dim (rows exist on both sides); NULL-FK Unknown Member block |
| `warning` | PK candidate validated NULL-free + unique; FK inclusion-dependency match; NULLABLE-but-no-NULLs FK |
| `info` | PK candidate failed validation (NULLs or duplicates); FK stem match where the join returned zero rows |

## Common pitfalls

- **Unreserved-keyword columns aren't valid SQL identifiers without quotes.** The UDF always quotes. If you wrap the `SQL_TEXT` and re-quote, you'll get `""YEAR""` — strip your own quoting layer.
- **Composite PKs aren't a goal — they're a fallback.** Prefer adding a surrogate when the natural composite has more than two columns.
- **Empty dims (zero rows) get a `warning` PK proposal but no Unknown Member generation.** Catalog-only context; the validator's NULL and uniqueness checks both pass trivially on a zero-row table, so the PK ALTER is emitted, but there's no FK column anywhere with NULLs to repair yet.
- **DECIMAL / charset FK mismatches are silently skipped** (see `fk-widening.md`). The finding lands as `info` or is omitted entirely. Caller must handle these manually.

## Coverage map

The smoke harness in `exasol-factory-foundation/scripts/optimize_torture_smoke.py` asserts one or more `(severity, action_name, sql_fragment)` tuples per torture-fixture table. See `acme-torture-patterns.md` for the full list with current pass/fail status.

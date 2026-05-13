# Fact-table detection heuristics

Both `INFER_JOIN_PATHS` (via `classify_table`) and `ANALYZE_CONSTRAINTS` (via `is_fact` / `is_dim`) need to decide whether a table is a fact or a dimension. The rules are intentionally shallow — name first, shape second — and behave well on production schemas only when you understand their failure modes.

## Name-based prefixes and suffixes (the fast path)

The two UDFs use slightly different prefix sets — a deliberate split. `INFER_JOIN_PATHS` (catalog-only profiler) accepts the wider list; `ANALYZE_CONSTRAINTS` (data-touching, ALTER-emitting) is conservative.

| Role | `INFER_JOIN_PATHS` prefixes | `ANALYZE_CONSTRAINTS` prefixes |
|---|---|---|
| **Fact** | `FACT_`, `FACT`, `F_`, `FT_`, `FCT_` | `FACT_`, `FACT`, `F_` |
| **Dimension** | `DIM_`, `DIM`, `D_`, `LU_`, `LOOKUP_`, `LKP_`, `REF_` | `DIM_`, `DIM`, `D_`, `LU_`, `LOOKUP_` |

**Both UDFs also recognise suffix variants** (added 2026-05-13 for production schemas that name `<X>_DIM` / `<X>_FACT` instead of prefixing):

| Role | Suffixes (both UDFs) |
|---|---|
| **Fact** | `_FACT`, `_FCT`, `_TXN`, `_TRX`, `_EVENT`, `_EVT` |
| **Dimension** | `_DIMENSION`, `_DIM`, `_LOOKUP`, `_LKP`, `_REF`, `_LU` |

Iteration is longest-first so `_DIMENSION` matches before `_DIM` and `INVENTORY_FACT_EVT` matches `_EVT` not `_FACT_EVT`.

A table whose name starts with any of these is classified immediately. No data scan, no column shape check. This is the recommended path: name your tables consistently and the optimizer reads correctly on day one.

Practical effect of the split: `LKP_CURRENCY` shows up in `INFER_JOIN_PATHS`'s join graph as a dim, but `ANALYZE_CONSTRAINTS` won't propose a stem-based FK against it (it isn't recognized as a dim by `is_dim`). Same for `REF_*` tables and `FT_*`/`FCT_*` facts — visible to the profiler, invisible to the constraint generator. When this bites you, either rename to `DIM_*` / `FACT_*`, or declare the PK manually so the inclusion-dependency FK path picks the relationship up regardless of prefix.

## Shape-based fallback (`classify_table` in INFER_JOIN_PATHS)

Triggered only when neither prefix applies. The UDF counts four numbers per table:

- `pk_count` — declared PK columns.
- `fk_count` — declared FK columns.
- `numeric_non_key` — DECIMAL / INTEGER / FLOAT / DOUBLE / NUMERIC / NUMBER columns excluding PK and FKs.
- `text_non_key` — VARCHAR / CHAR / CLOB columns excluding PK and FKs.

Then applies these rules **in order**, first match wins:

1. `fk_count > 0` and `numeric_non_key == 0` and `text_non_key > 0` → **dimension** (a "junction" dim — attributes plus FKs back to other dims).
2. `numeric_non_key > 2` → **fact** (measures dominate).
3. `pk_count == 1` and `fk_count == 0` and `text_non_key > 0` → **dimension** (single-PK attribute-only table).
4. Otherwise → **unknown** (not emitted in join paths).

## How it's wrong and what to do

### Wide bridge tables classify as facts

A bridge table connecting two dims may carry several DECIMAL weighting columns. `numeric_non_key > 2` triggers and it gets classified as a fact. The downstream cube generator then tries to find dim FKs from this "fact" and emits noise.

**Fix:** Rename to start with `BR_` / `BRIDGE_` (still classifies as unknown today — both `INFER_JOIN_PATHS` and `ANALYZE_CONSTRAINTS` skip unknowns). Better, declare the PK and FKs explicitly so the catalog tells the truth and rule (1) classifies it correctly as a dimension.

### Type-2 SCD dims look like facts

A history-tracking SCD-2 dimension like `DIM_CUSTOMER_HISTORY` carries `EFFECTIVE_DATE`, `END_DATE`, `IS_CURRENT`, and several measure-shaped columns. The shape fallback often calls it a fact.

**Fix:** Prefix with `DIM_` (it usually already is) — the prefix path wins before shape ever runs. If your naming convention buries the `DIM_` deeper in the name, fall back to the explicit PK declaration so the cube generator can read it correctly from the catalog later.

### Junk attribute dims classify as facts

A wide attribute dim with no PK declared and no FKs but lots of mixed-type columns can land in rule (2) if it happens to have more than 2 numeric columns (geo coordinates, counts, percentages). The cube generator then proposes joins from it as if it were a fact.

**Fix:** Prefix with `DIM_`. Or — easier — declare the PK. Rule (3) immediately reclassifies it.

### Tables under `unknown` get silently dropped

`INFER_JOIN_PATHS` only emits rows where the fact side classified as `fact` and the dim side classified as `dimension`. An `unknown` table appears in neither side. The user sees no emission and no error.

**Fix:** Run the underlying query manually to spot unknowns:

Run `EXECUTE SCRIPT EXA_OPTIMIZE.INFER_JOIN_PATHS('MYSCHEMA');` to capture the emitted rows, then diff against the catalog:

```sql
-- in your client / harness, collect FACT_TABLE and DIM_TABLE
-- values from the result of the INFER_JOIN_PATHS call, then:
SELECT "TABLE_NAME"
FROM SYS.EXA_ALL_TABLES
WHERE UPPER("TABLE_SCHEMA") = 'MYSCHEMA'
  AND "TABLE_IS_VIRTUAL" = FALSE
  AND "TABLE_NAME" NOT IN (<list_collected_from_INFER_call>);
```

(Exasol's parser won't let you nest `EXECUTE SCRIPT` inside an `EXCEPT` — separate the two calls.)

This surfaces every table that didn't make it into a join. Decide per row: rename, declare a PK, or accept it isn't a part of the star.

## What `ANALYZE_CONSTRAINTS` does differently

`ANALYZE_CONSTRAINTS` uses simpler functions — `is_fact(name)` and `is_dim(name)` — that only check prefixes. There's no shape fallback. A table without a `FACT_*` or `DIM_*` prefix is treated as neither for FK inference purposes (though it still gets a PK proposal via the generic path).

This is why ANALYZE_CONSTRAINTS and INFER_JOIN_PATHS can disagree about a borderline table. ANALYZE is name-only; INFER includes shape. When they disagree, prefer the conservative answer: the table probably needs a rename or an explicit constraint declaration to be unambiguous.

## Views are first-class for join inference (only)

`INFER_JOIN_PATHS` reads `SYS.EXA_ALL_VIEWS` and `SYS.EXA_ALL_COLUMNS` for views. A view can appear as a `DIM_*` row in the join graph. `ANALYZE_CONSTRAINTS` ignores views entirely (you can't add constraints to a view). When a view re-exposes a dim's key column, the smoke fixture `VW_DIM_SHADOW` asserts the join count does not inflate (the view appears as one additional dim, not as duplicate join rows on the same fact column).

## Calendar / rate tables get special-case PK suggestions

Inside `ANALYZE_CONSTRAINTS` (not `INFER_JOIN_PATHS`), three patterns get bespoke PK logic:

| Pattern | Detection | PK suggestion |
|---|---|---|
| Calendar / date dim | Table name starts with `CALENDAR`, `DIM_DATE`, `DIM_CALENDAR`, `DATE_DIM`, or `TIME_DIM` | First DATE / TIMESTAMP column |
| FX / rate table | Name contains `FX_RATE`, `EXCHANGE_RATE`, or `FOREX_RATE` | Composite (date column, two currency-code columns named `CURRENCY*`, `CCY*`, `*_CD`, or `*_CODE`) |
| `FACT_*` with degenerate-dim composite | Column ending `LINENUMBER` paired with same-stem `NUMBER` | `(stem + NUMBER, stem + LINENUMBER)` composite |

These wins matter on Adventureworks-shaped data: `FACT_INTERNETSALES` keys on `(SALESORDERNUMBER, SALESORDERLINENUMBER)` — the generic heuristic would pick the first two surrogate FKs (`PRODUCTKEY`, `CUSTOMERKEY`) and fail validation.

## Rules of thumb

- Always name FACT_ and DIM_ tables with the prefix. The shape fallback is for surprises — don't rely on it.
- Declare the PK before running any downstream join inference. It eliminates ambiguity for `classify_table` rule (3).
- Use `_KEY` for surrogate keys, `_ID` for natural keys, `_CD` / `_CODE` for short codes. The stem matchers care about the suffix.
- Don't put measures in dims. A measure on a dim breaks the shape classifier.

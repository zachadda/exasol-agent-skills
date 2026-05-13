# FK column widening — non-lossy type alignment before ADD CONSTRAINT

Exasol rejects `ALTER TABLE … ADD CONSTRAINT … FOREIGN KEY` when the FK column type doesn't exactly match the referenced PK column type. Length differences count: `VARCHAR(5)` ↛ `VARCHAR(50)`. Charset differences count: `VARCHAR(20) UTF8` ↛ `VARCHAR(20) ASCII`. `ANALYZE_CONSTRAINTS` solves the common subset by emitting a single `MODIFY COLUMN` step prepended to the FK ALTER. Anything outside that subset is left for the caller to handle.

## What the UDF auto-widens

A widen step is emitted **only** when *all* of these hold:

1. **Same family.** Both sides VARCHAR, or both sides CHAR. VARCHAR ↛ CHAR is not auto-widened (storage semantics differ).
2. **Same charset.** Both UTF8 or both ASCII. Default (`<no qualifier>`) ↛ default is allowed. UTF8 ↛ ASCII is not — silent character truncation risk.
3. **Strict widening.** FK length < PK length. Equal-length is already fine. Wider-than-PK is rejected (lossy).
4. **FK side only.** The MODIFY targets the fact / FK side. Widening the PK column would force every other FK pointing at it to widen in lock-step, which Exasol blocks atomically.
5. **VARCHAR length ≤ 256 on the PK side.** The inclusion-dependency FK discovery treats any VARCHAR/CHAR column wider than 256 as text rather than identifier and skips it. Real-world customer keys (composite codes, product SKUs, license keys) routinely use VARCHAR(50–128); the threshold was bumped from the original 64 to accommodate them. If you need to FK-link a column wider than 256, declare the FK manually.

When the parse fails (numeric types, DECIMAL precision/scale changes, TIMESTAMP precision, BOOL ↛ anything), `fk_widen_target` returns `nil` and the UDF emits **no FK ALTER at all** for that pair. Severity collapses to `info` or the finding is omitted entirely. Caller must:
- Either fix the column types manually before re-running, or
- Use the manual constraint editor to add the FK with an explicit type contract.

## Emission format

The widen + FK ALTER ship as one `SQL_TEXT` value with `-- @@STEP@@` markers so the downstream splitter (`services.optimizer._split_sql_steps`) can recover individual statements:

```sql
-- @@STEP@@ Widen FACT_SALES.CUSTOMER_KEY to VARCHAR(50)
ALTER TABLE "ACME_DEMO"."FACT_SALES" MODIFY COLUMN "CUSTOMER_KEY" VARCHAR(50);
-- @@STEP@@ Add FK FACT_SALES.CUSTOMER_KEY -> DIM_CUSTOMER.CUSTOMER_KEY
ALTER TABLE "ACME_DEMO"."FACT_SALES" ADD CONSTRAINT FK_FACT_SALES_CUSTOMER_KEY
  FOREIGN KEY ("CUSTOMER_KEY") REFERENCES "ACME_DEMO"."DIM_CUSTOMER" ("CUSTOMER_KEY") DISABLE;
```

For Unknown Member generation (step 3b — see `constraint-inference.md`) the widen lands inline after the `NOT NULL` step and before the `ADD CONSTRAINT`. Same rules apply.

## When the UDF refuses

These cases land as **no widen, no FK** and are tracked by the smoke harness so regressions can't accidentally start auto-widening them:

| Case | Why refused |
|---|---|
| `FACT_FK_WIDER.CUST_CODE` is wider than `DIM_CUSTOMER_NARROW.CUST_CODE` | Truncating data — lossy direction |
| `FACT_FK_DECIMAL_MISMATCH` — DECIMAL(10,2) ↛ DECIMAL(18,0) | Different precision/scale — would change values |
| `FACT_FK_CHARSET_MISMATCH` — VARCHAR(20) UTF8 ↛ VARCHAR(20) ASCII | Encoding mismatch — character data could change |

All three are positive in the smoke harness (correctly produce no emission). Adding a fourth case requires extending `fk_widen_target` in `analyze_constraints.sql` and a new torture-fixture table.

## Practical guide for callers

- **Always look at the description field.** Successful auto-widen finds include the phrase `Auto-widening <fact>.<col> from <type> (non-lossy)`. Absence means the FK landed without modification.
- **Apply the steps in order.** Don't reorder the `MODIFY COLUMN` after the `ADD CONSTRAINT`. The FK won't apply if the types still don't match.
- **Don't widen a PK column to match an FK.** If the PK side is narrower, that's a model error — extend the PK type at design time, then re-analyze. The UDF will not propose this.
- **DECIMAL is hand-fix territory.** Most DECIMAL mismatches in real Power BI / Snowflake imports come from the fact carrying `DECIMAL(38,0)` for a surrogate key while the dim PK was declared `DECIMAL(18,0)`. The UDF doesn't propose this widen because precision-changing modifications can fail at runtime when data overflows. Resolve with a manual `ALTER` after auditing the data.
- **CHAR padding semantics.** Widening `CHAR(2)` to `CHAR(5)` pads existing values with three trailing spaces — typically what you want for fixed-width codes, but verify before running on production data.

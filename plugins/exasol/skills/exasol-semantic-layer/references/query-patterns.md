# Querying the cube

Canonical SELECT shapes against a live SQLCube virtual schema. Use these to test a fresh cube and to teach downstream skills (dashboard, MCP-driven analytics) what works.

## Schema reference for examples

Assume cube `ADVENTUREWORKS_CUBE`:

```
DIMENSIONS:    Customer, Product, Date
FACTS:         Sales
RELATIONSHIPS:
  - Sales.CUSTOMER_ID → Customer.CUSTOMER_ID
  - Sales.PRODUCT_ID  → Product.PRODUCT_ID
  - Sales.ORDER_DATE  → Date.DATE_ID  (role: OrderDate)
  - Sales.SHIP_DATE   → Date.DATE_ID  (role: ShipDate)
MEASURES:
  - Sales.Total Revenue   (SUM, currency_usd)
  - Sales.Order Count     (COUNT DISTINCT)
  - Sales.Avg Discount    (AVG, percent)
```

## Pattern 1: Single-table fact query

```sql
SELECT "Order Count", "Total Revenue"
FROM ADVENTUREWORKS_CUBE.Sales;
```

Returns one row. The adapter rewrites:

```sql
SELECT COUNT(DISTINCT ORDER_ID)                                       AS "Order Count",
       SUM(QUANTITY * UNIT_PRICE * (1 - DISCOUNT))                    AS "Total Revenue"
FROM ADVENTUREWORKS.FACT_SALES;
```

Identifier casing: SQLCUBE uses Exasol-default uppercase folding. Quote business names that aren't all-uppercase (`"Total Revenue"`). The adapter handles either; quote for safety in agent-emitted SQL.

## Pattern 2: Group-by-dimension fact query

```sql
SELECT c.Segment, SUM("Total Revenue") AS revenue
FROM ADVENTUREWORKS_CUBE.Sales s
JOIN ADVENTUREWORKS_CUBE.Customer c ON ... -- adapter auto-joins
GROUP BY c.Segment
ORDER BY revenue DESC;
```

Wait — the adapter handles the join. Cleaner shape:

```sql
SELECT Customer.Segment, "Total Revenue"
FROM ADVENTUREWORKS_CUBE.Sales
GROUP BY Customer.Segment
ORDER BY "Total Revenue" DESC;
```

That's the SQLCube idiom. Dimensional attributes are addressed as `<DimName>.<Column>`; the adapter inserts the join using the RELATIONSHIPS metadata. No explicit ON clause required when there's only one path.

The adapter rewrites to:

```sql
SELECT C.SEGMENT,
       SUM(S.QUANTITY * S.UNIT_PRICE * (1 - S.DISCOUNT)) AS "Total Revenue"
FROM ADVENTUREWORKS.FACT_SALES S
JOIN ADVENTUREWORKS.DIM_CUSTOMER C ON S.CUSTOMER_ID = C.CUSTOMER_ID
GROUP BY C.SEGMENT
ORDER BY "Total Revenue" DESC;
```

## Pattern 3: Role-playing date

Two date FKs from Sales to Date. Explicit role in the column reference:

```sql
SELECT OrderDate.Year, "Total Revenue"
FROM ADVENTUREWORKS_CUBE.Sales
GROUP BY OrderDate.Year;
```

vs

```sql
SELECT ShipDate.Quarter, "Total Revenue"
FROM ADVENTUREWORKS_CUBE.Sales
GROUP BY ShipDate.Quarter;
```

The adapter resolves `OrderDate.*` to `Date.*` via the RELATIONSHIPS row where `ROLE_PLAYING_AS = 'OrderDate'`, joins on `ORDER_DATE`. Same for ShipDate joining on `SHIP_DATE`.

Both roles in one query (rare, used for fulfillment-latency analysis):

```sql
SELECT OrderDate.Quarter, ShipDate.Quarter, COUNT(*) AS row_count
FROM ADVENTUREWORKS_CUBE.Sales
GROUP BY 1, 2;
```

Adapter joins DIM_DATE twice with different aliases.

## Pattern 4: Multi-dimensional filter

```sql
SELECT Product.Category, OrderDate.Year, "Total Revenue"
FROM ADVENTUREWORKS_CUBE.Sales
WHERE Customer.Country = 'United States'
  AND OrderDate.Year >= 2024
GROUP BY Product.Category, OrderDate.Year
ORDER BY Product.Category, OrderDate.Year;
```

The adapter pushes WHERE predicates down to the source SELECTs. All three dimensions get joined.

## Pattern 5: Top-N

```sql
SELECT Customer.Name, "Total Revenue"
FROM ADVENTUREWORKS_CUBE.Sales
GROUP BY Customer.Name
ORDER BY "Total Revenue" DESC
LIMIT 10;
```

Standard. Adapter pushes ORDER BY + LIMIT. Result: top 10 customers by revenue. No magic.

## Pattern 6: Compound measure

If `Net Margin Rate` was defined as a measure with expression `(UNIT_PRICE - STANDARD_COST) / UNIT_PRICE` and aggregation `AVG`:

```sql
SELECT Product.Category, "Net Margin Rate"
FROM ADVENTUREWORKS_CUBE.Sales
GROUP BY Product.Category;
```

Adapter rewrites to:

```sql
SELECT P.CATEGORY,
       AVG((S.UNIT_PRICE - P.STANDARD_COST) / NULLIF(S.UNIT_PRICE, 0)) AS "Net Margin Rate"
FROM ADVENTUREWORKS.FACT_SALES S
JOIN ADVENTUREWORKS.DIM_PRODUCT P ON S.PRODUCT_ID = P.PRODUCT_ID
GROUP BY P.CATEGORY;
```

Note: NULLIF defense for divide-by-zero is up to the measure expression author. SQLCube doesn't auto-wrap.

## Pattern 7: Just dimension data (no fact)

```sql
SELECT * FROM ADVENTUREWORKS_CUBE.Customer LIMIT 20;
```

Returns raw dim rows. Adapter pulls from DIM_CUSTOMER directly. Useful for browsing entity values.

```sql
SELECT DISTINCT Country FROM ADVENTUREWORKS_CUBE.Customer ORDER BY Country;
```

Same pattern — purely dimensional, no fact join.

## Pattern 8: Measure with no group-by columns

```sql
SELECT "Total Revenue", "Order Count"
FROM ADVENTUREWORKS_CUBE.Sales
WHERE OrderDate.Year = 2024;
```

Single-row result. WHERE filters the fact (and the adapter joins Date dim for the predicate evaluation). The agent often wants this shape for KPI tiles in dashboards.

## What the adapter does NOT support (v0.1)

| Pattern | Why |
|---|---|
| Subqueries in SELECT against cube columns | Adapter parser doesn't recurse into nested expressions yet |
| Window functions over measures | Measures are aggregates by definition; rank-of-aggregate needs `WITH` CTE |
| CROSS JOIN between two facts | Multi-fact joins are a v0.2 feature |
| UNION across cubes | Same |
| Recursive CTEs touching cube | Adapter doesn't unfold |

Workaround for "rank of aggregate" — use a CTE:

```sql
WITH rev AS (
  SELECT Customer.Name AS name, "Total Revenue" AS revenue
  FROM ADVENTUREWORKS_CUBE.Sales
  GROUP BY Customer.Name
)
SELECT name, revenue, RANK() OVER (ORDER BY revenue DESC) AS rk
FROM rev
ORDER BY rk
LIMIT 10;
```

The cube SELECT is the inner query; the window function runs against the materialized CTE.

## Verifying a fresh cube

Test sequence after `CREATE VIRTUAL SCHEMA`:

```sql
-- 1. List virtual tables
SELECT TABLE_NAME FROM SYS.EXA_VIRTUAL_TABLES WHERE SCHEMA_NAME = 'ADVENTUREWORKS_CUBE';

-- 2. Sample each:
SELECT * FROM ADVENTUREWORKS_CUBE.Customer LIMIT 3;
SELECT * FROM ADVENTUREWORKS_CUBE.Product  LIMIT 3;
SELECT * FROM ADVENTUREWORKS_CUBE.Sales    LIMIT 3;

-- 3. One measure, no grouping:
SELECT "Total Revenue" FROM ADVENTUREWORKS_CUBE.Sales;

-- 4. One measure, one dim:
SELECT Customer.Segment, "Total Revenue"
FROM ADVENTUREWORKS_CUBE.Sales
GROUP BY Customer.Segment;

-- 5. Date roles:
SELECT OrderDate.Year, "Total Revenue"
FROM ADVENTUREWORKS_CUBE.Sales
GROUP BY OrderDate.Year
ORDER BY OrderDate.Year;
```

If all five succeed → cube is healthy. If (4) fails but (3) works → RELATIONSHIPS row is missing or wrong. If (5) fails but (4) works → role-playing setup is broken (likely `ROLE_PLAYING_AS` column unset).

## Performance notes

The adapter doesn't materialize. Every query hits physical tables. Query performance = underlying Exasol performance + adapter rewrite overhead (small, ~1-5 ms).

For dashboards firing many measure queries: prefer ONE wide query returning all measures, GROUP BY all needed dimensions in one shot. The adapter is smart enough to fuse, but issuing N separate queries pays N round-trips.

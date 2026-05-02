# CALCULATION TRANSLATION — TABLEAU FORMULA → LIGHTDASH

A Tableau calculated field (`<calculation class="tableau" formula="…"/>`) becomes one of three things in Lightdash:

1. **A dbt column** (deterministic per-row or window) — preferred. Define dimension/metric on top.
2. **A Lightdash metric** with custom `sql:` — if it's an aggregate that doesn't exist as a base column.
3. **A Lightdash table calculation** (`metricQuery.tableCalculations`) — only for chart-local post-query SQL.

---

## 1. Decision tree

Walk Tableau formula tokens:

- Pure aggregate of one column (`SUM([x])`, `AVG([x])`, …) → **drop**, use base aggregation in `meta.metrics`.
- Arithmetic of aggregates (`SUM([Profit])/SUM([Sales])`) → **Lightdash metric `type: number`** with custom SQL.
- Row-level expression (no aggregator, just per-row math/case) → **dbt column**, then expose as dimension or `meta.metrics` aggregation.
- Window function (`WINDOW_*`, `RUNNING_*`, `RANK`, `LOOKUP`, `INDEX`, `FIRST`, `LAST`) → **dbt window column** OR **Lightdash table calculation**.
- LOD `{FIXED dims : agg}` → **dbt model with explicit GROUP BY** + join, OR `sum_distinct` with `distinct_keys`.
- LOD `{INCLUDE/EXCLUDE}` → **dbt remodeling**, no direct mapping.
- `RAWSQL_*` / `RAWSQLAGG_*` → strip out, the inner SQL was already warehouse-native; lift it into dbt.
- Parameter reference → Lightdash dashboard parameter / filter; usually requires a query-time substitution.

---

## 2. Function-by-function map

Lightdash SQL ≈ warehouse SQL (Snowflake, BigQuery, Postgres dialect — depends on adapter). Below is the standard ANSI form; adjust for your warehouse.

### Aggregates

| Tableau | Lightdash (dbt or metric SQL) |
|---|---|
| `SUM([x])` | `meta.metrics.<name>: { type: sum }` on column `x` |
| `AVG([x])` | `type: average` |
| `MIN([x])` | `type: min` |
| `MAX([x])` | `type: max` |
| `COUNT([x])` | `type: count` |
| `COUNTD([x])` | `type: count_distinct` |
| `MEDIAN([x])` | `type: median` |
| `PERCENTILE([x], 0.9)` | `type: percentile`, `percentile: 90` |
| `STDEV([x])` | `type: number`, `sql: "STDDEV(${TABLE}.x)"` |
| `VAR([x])` | `type: number`, `sql: "VARIANCE(${TABLE}.x)"` |
| `ATTR([x])` | not applicable — drop. Tableau ATTR returns the value if unique else `*`; in Lightdash query the dimension directly. |

### Arithmetic of aggregates (ratio metrics)

```
Tableau:   SUM([Profit]) / SUM([Sales])
Lightdash: meta.metrics:
             profit_margin:
               type: number
               sql: "SUM(${TABLE}.profit) / NULLIF(SUM(${TABLE}.sales), 0)"
               format: percent
```

Always wrap denominators in `NULLIF(..., 0)`.

### IF / CASE

```
Tableau:   IF [Sales] > 1000 THEN "Big" ELSE "Small" END
dbt:       CASE WHEN sales > 1000 THEN 'Big' ELSE 'Small' END  AS sales_bucket
```

Add as a dbt column, expose as dimension.

```
Tableau:   IIF([Region] = "East", 1, 0)
dbt:       CASE WHEN region = 'East' THEN 1 ELSE 0 END
```

### String

| Tableau | dbt SQL (ANSI) |
|---|---|
| `LEFT([s], n)` | `LEFT(s, n)` |
| `RIGHT([s], n)` | `RIGHT(s, n)` |
| `MID([s], start, len)` | `SUBSTRING(s, start, len)` |
| `LEN([s])` | `LENGTH(s)` |
| `UPPER([s])` | `UPPER(s)` |
| `LOWER([s])` | `LOWER(s)` |
| `TRIM([s])` | `TRIM(s)` |
| `CONTAINS([s], "x")` | `s LIKE '%x%'` |
| `STARTSWITH([s], "x")` | `s LIKE 'x%'` |
| `ENDSWITH([s], "x")` | `s LIKE '%x'` |
| `REPLACE([s], "a", "b")` | `REPLACE(s, 'a', 'b')` |
| `REGEXP_MATCH([s], "p")` | `REGEXP_LIKE(s, 'p')` (Snowflake), `REGEXP_CONTAINS` (BQ), `s ~ 'p'` (Postgres) |
| `SPLIT([s], ",", n)` | `SPLIT_PART(s, ',', n)` (Postgres/Snowflake), `SPLIT(s, ',')[OFFSET(n-1)]` (BQ) |

### Date

| Tableau | dbt SQL (ANSI) |
|---|---|
| `YEAR([d])` | `EXTRACT(YEAR FROM d)` |
| `MONTH([d])` | `EXTRACT(MONTH FROM d)` |
| `DAY([d])` | `EXTRACT(DAY FROM d)` |
| `DATEPART('week', [d])` | `EXTRACT(WEEK FROM d)` |
| `DATETRUNC('month', [d])` | `DATE_TRUNC('month', d)` |
| `DATEADD('day', n, [d])` | `d + INTERVAL '1 day' * n` (Postgres), `DATEADD(day, n, d)` (Snowflake/SQL Server), `DATE_ADD(d, INTERVAL n DAY)` (BQ) |
| `DATEDIFF('day', [a], [b])` | `DATE_DIFF(b, a, DAY)` (BQ), `DATEDIFF(day, a, b)` (Snowflake) |
| `TODAY()` | `CURRENT_DATE` |
| `NOW()` | `CURRENT_TIMESTAMP` |
| `DATEPARSE("yyyy-MM-dd", [s])` | `TO_DATE(s, 'YYYY-MM-DD')` |
| `MAKEDATE(y, m, d)` | `DATE(y, m, d)` (BQ) / `MAKE_DATE(y, m, d)` (Postgres) |

For dim-level analysis prefer Lightdash `time_intervals` over per-formula DATETRUNC; the engine generates `DAY`, `WEEK`, `MONTH`, `QUARTER`, `YEAR` columns automatically when you set:

```yaml
meta:
  dimension:
    type: timestamp
    time_intervals: [DAY, WEEK, MONTH, QUARTER, YEAR]
```

### Math

`ABS`, `ROUND`, `CEILING/CEIL`, `FLOOR`, `MOD`, `POWER`, `SQRT`, `LN`, `LOG`, `EXP`, `SIGN` map 1:1 to most warehouses.

### Logical / null

| Tableau | dbt SQL |
|---|---|
| `IFNULL([x], 0)` | `COALESCE(x, 0)` |
| `ISNULL([x])` | `x IS NULL` |
| `ZN([x])` | `COALESCE(x, 0)` |

---

## 3. LOD expressions

### `{FIXED dim : agg(measure)}`

> "Aggregate `measure` to dim level, ignoring all other dims in the view."

Two strategies:

**A. Pre-compute in dbt and join (always safe):**

```sql
-- models/customer_lifetime_sales.sql
SELECT
  customer_id,
  SUM(amount) AS lifetime_sales
FROM {{ ref('orders') }}
GROUP BY 1
```

```yaml
# orders.yml — join the new model
meta:
  joins:
    - join: customer_lifetime_sales
      sql_on: "${orders.customer_id} = ${customer_lifetime_sales.customer_id}"
      relationship: many-to-one
```

Then expose `customer_lifetime_sales.lifetime_sales` as a metric.

**B. `sum_distinct` with `distinct_keys` (only for sum-style fanout safety):**

```yaml
meta:
  metrics:
    customer_lifetime_sales:
      type: sum_distinct
      sql: "${TABLE}.amount"
      distinct_keys: [customer_id, order_id]
      label: Customer Lifetime Sales
      format: usd
```

This handles fan-out from joins but doesn't replicate FIXED in queries that also group by other dimensions — Lightdash will still group by all selected dims. Strategy A is more faithful.

### `{INCLUDE dim : agg}` and `{EXCLUDE dim : agg}`

No native equivalent. Two options:
- Re-model with explicit aggregates per dim level (one model per LOD).
- Drop and discuss with the dashboard owner — these are often used for nuanced averages where the workaround is to expose a different metric.

---

## 4. Table calculations

### `WINDOW_SUM`, `RUNNING_SUM`, `RANK`, `LOOKUP`, `INDEX`

Two paths.

**Path A — dbt window column (preferred, always works):**

```sql
SELECT
  *,
  SUM(amount) OVER (PARTITION BY customer_id ORDER BY order_date) AS running_customer_total,
  RANK() OVER (PARTITION BY region ORDER BY amount DESC) AS region_rank
FROM {{ ref('orders') }}
```

Expose as columns.

**Path B — Lightdash chart-local table calculation:**

```yaml
metricQuery:
  ...
  tableCalculations:
    - name: running_total
      displayName: "Running Total"
      sql: "SUM(${orders.total_revenue}) OVER (ORDER BY ${orders.order_date_month})"
      format:
        type: number
        round: 0
```

Path B keeps the calc at the chart level (closer to Tableau's worksheet-scoped table calc). Path A makes it reusable.

---

## 5. Parameters in formulas

Tableau:

```
SUM(IF YEAR([Order Date]) = [Year Parameter] THEN [Sales] ELSE 0 END)
```

Lightdash:
- Define `[Year Parameter]` as a Lightdash parameter on the dashboard.
- Use a dimension `order_year` (from `time_intervals`) in the chart.
- Apply a dashboard filter `order_year equals [Year Parameter]`.

If the parameter is referenced inside an aggregate condition, fold it into a dbt metric that reads the param at query time via Lightdash parameter syntax (`${ld.parameters.year_param}`).

---

## 6. RAWSQL passthrough

```
RAWSQLAGG_REAL("SUM(%1)", [Sales])
```

The wrapped SQL is already warehouse-native. Move it to a dbt column or metric `sql:` field verbatim.

---

## 7. Worked example: Profit Ratio

Tableau:

```xml
<column name="[Profit Ratio]">
  <calculation class="tableau" formula="SUM([Profit]) / SUM([Sales])"/>
</column>
```

Lightdash on `orders` model:

```yaml
columns:
  - name: profit
    meta:
      dimension: { type: number, label: Profit, format: usd }
      metrics:
        total_profit: { type: sum, label: Total Profit, format: usd }
  - name: sales
    meta:
      dimension: { type: number, label: Sales, format: usd }
      metrics:
        total_sales: { type: sum, label: Total Sales, format: usd }

models:
  - name: orders
    meta:
      metrics:
        profit_ratio:
          type: number
          sql: "SUM(${TABLE}.profit) / NULLIF(SUM(${TABLE}.sales), 0)"
          label: Profit Ratio
          format: percent
          round: 2
```

Always nullsafe-divide. Always set `format: percent` for ratios.

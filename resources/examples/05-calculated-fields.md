# Example 5 — Calculated fields: Profit Margin, Customer Lifetime Value, Sales Bucket

## Tableau calcs

```xml
<column name="[Profit Margin]" role="measure" type="quantitative" datatype="real">
  <calculation class="tableau" formula="SUM([Profit])/SUM([Sales])"/>
</column>

<column name="[Customer Lifetime Sales]" role="measure" type="quantitative" datatype="real">
  <calculation class="tableau" formula="{FIXED [Customer ID] : SUM([Sales])}"/>
</column>

<column name="[Sales Bucket]" role="dimension" type="nominal" datatype="string">
  <calculation class="tableau" formula="IF [Sales] &gt;= 10000 THEN 'High' ELSEIF [Sales] &gt;= 1000 THEN 'Mid' ELSE 'Low' END"/>
</column>
```

## Lightdash translation

### Profit Margin — model-level metric

```yaml
# models/orders.yml
models:
  - name: orders
    meta:
      metrics:
        profit_margin:
          type: number
          sql: "SUM(${TABLE}.profit) / NULLIF(SUM(${TABLE}.sales), 0)"
          label: Profit Margin
          format: percent
          round: 2
```

### Customer Lifetime Sales — separate dbt model + join

```sql
-- models/customer_lifetime_sales.sql
{{ config(materialized='table') }}
SELECT
  customer_id,
  SUM(sales) AS lifetime_sales
FROM {{ ref('orders') }}
GROUP BY 1
```

```yaml
# models/customer_lifetime_sales.yml
version: 2
models:
  - name: customer_lifetime_sales
    meta:
      label: Customer Lifetime Sales
      primary_key: customer_id
    columns:
      - name: customer_id
        meta: { dimension: { type: string, label: Customer ID, hidden: true } }
      - name: lifetime_sales
        meta:
          dimension: { type: number, label: Lifetime Sales, format: usd }
          metrics:
            customer_lifetime_sales:
              type: max
              label: Customer Lifetime Sales
              format: usd
```

```yaml
# models/orders.yml — add the join
meta:
  joins:
    - join: customer_lifetime_sales
      sql_on: "${orders.customer_id} = ${customer_lifetime_sales.customer_id}"
      type: left
      relationship: many-to-one
```

`type: max` (rather than `sum`) is correct here because the joined value is already aggregated per customer; we don't want fan-out re-summation.

### Sales Bucket — dbt column with CASE

Add to `models/orders.sql` (or a downstream model):

```sql
SELECT
  ...,
  CASE
    WHEN sales >= 10000 THEN 'High'
    WHEN sales >= 1000  THEN 'Mid'
    ELSE 'Low'
  END AS sales_bucket
FROM {{ ref('raw_orders') }}
```

```yaml
# models/orders.yml
- name: sales_bucket
  meta:
    dimension:
      type: string
      label: Sales Bucket
      colors:
        High: "#10b981"
        Mid:  "#f59e0b"
        Low:  "#ef4444"
```

## Anti-patterns

- ❌ Don't put `IF/THEN/ELSE` directly into `meta.metrics.sql` if it's per-row logic — that's a dimension, not a metric.
- ❌ Don't use `type: number` with naked `SUM(...)` — wrap in `NULLIF` for ratios.
- ❌ Don't try to express `{FIXED}` as a Lightdash custom metric without a join — fan-out from joins will multiply your sums.

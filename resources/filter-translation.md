# FILTER TRANSLATION — TABLEAU → LIGHTDASH

Tableau filters live on `<filter>` inside a worksheet `<view>` (chart-level) or as global "use as context" filters. Lightdash filters live on either:

- `metricQuery.filters` (chart-level)
- dashboard `filters.dimensions` (dashboard-wide)

Lightdash filters are AND-grouped at the top level. Within a single filter the values list is OR'd.

---

## 1. Filter shape — Lightdash chart filter

```yaml
metricQuery:
  filters:
    dimensions:
      and:
        - id: 7e0e4c0a-...        # any uuid; use a slug or hash if generating
          target:
            fieldId: orders_status
            tableName: orders
          operator: equals
          values: [completed, shipped]
        - id: ...
          target:
            fieldId: orders_amount
            tableName: orders
          operator: greaterThan
          values: [1000]
    metrics:
      and: []
```

The dashboard equivalent is one entry per filter in `filters.dimensions[]` with the same `target`/`operator`/`values` plus a `label` and optional `tileTargets`.

---

## 2. Operator mapping

| Tableau filter | Lightdash operator | values |
|---|---|---|
| `categorical` `included-values="all"` | `equals` | list of allowed values |
| `categorical` exclude (negated set) | `notEquals` | list |
| `categorical` `null` only | `isNull` | `[]` |
| `categorical` `non-null` | `notNull` | `[]` |
| `categorical` `top-n` (`function="end" units="N"`) | n/a — encode via `metricQuery.limit` + `sorts` | |
| `quantitative` `in-range` (min,max) | `inBetween` | `[min, max]` |
| `quantitative` `>` | `greaterThan` | `[n]` |
| `quantitative` `>=` | `greaterThanOrEqual` | `[n]` |
| `quantitative` `<` | `lessThan` | `[n]` |
| `quantitative` `<=` | `lessThanOrEqual` | `[n]` |
| `relative-date` `last-n-days/weeks/months/...` | `inThePast` (+ `settings.unitOfTime`) | `[N]` |
| `relative-date` `next-n-*` | `inTheNext` | `[N]` |
| `relative-date` `year-to-date` / `month-to-date` / `quarter-to-date` | `inTheCurrent` | `[1]` with `settings.unitOfTime: years/months/quarters` |
| `relative-date` `today` / `yesterday` | `inTheCurrent` / fixed `equals` on the date dim | |
| Date range fixed (`<min>2024-01-01</min><max>2024-12-31</max>`) | `inBetween` | `[startDate, endDate]` |
| Formula (condition) filter | rewrite the condition into a boolean dimension in dbt; filter `equals: [true]` | |

---

## 3. Relative-date settings

Lightdash splits the unit out:

```yaml
- target: { fieldId: orders_created_at, tableName: orders }
  operator: inThePast
  settings:
    completed: false      # true = exclude current incomplete period
    unitOfTime: months
  values: [3]             # last 3 months
```

`unitOfTime` ∈ `days`, `weeks`, `months`, `quarters`, `years`, `hours`, `minutes`.

`completed: true` ≈ Tableau "Anchor relative to: end of last complete period".

---

## 4. Top-N filter

Tableau:

```xml
<filter column="[Region]" class="categorical">
  <groupfilter function="end" direction="TOP" expression="[sum:Sales:qk]" units="10"/>
</filter>
```

Lightdash:

```yaml
metricQuery:
  dimensions: [orders_region]
  metrics: [orders_total_sales]
  limit: 10
  sorts:
    - fieldId: orders_total_sales
      descending: true
```

If the chart needs to keep its visual sort by a different dim while limiting to top-10 by measure, pre-aggregate in dbt or use a table calculation.

---

## 5. Context filter

Tableau context filters apply before regular filters and fix the population (e.g. for FIXED LODs and Top-N base set). In Lightdash all filters compose into a single SQL WHERE — there is no "context tier". If a context filter exists in Tableau:

- If it's a simple equals/range on a dimension, just include it as a normal Lightdash filter.
- If a downstream FIXED LOD depends on it, materialize the LOD in dbt with the same WHERE clause baked in, **or** use a dashboard filter so it applies uniformly.

---

## 6. Worked example: relative date + categorical

Tableau:

```xml
<filter column="[Order Date]" class="relative-date"
        period-type-v2="last-n-months" first-period="-3" last-period="0"/>
<filter column="[Region]" class="categorical" included-values="all">
  <groupfilter>
    <groupfilter-item value="East"/>
    <groupfilter-item value="West"/>
  </groupfilter>
</filter>
```

Lightdash chart filters:

```yaml
metricQuery:
  filters:
    dimensions:
      and:
        - id: f1
          target: { fieldId: orders_order_date, tableName: orders }
          operator: inThePast
          settings: { completed: false, unitOfTime: months }
          values: [3]
        - id: f2
          target: { fieldId: orders_region, tableName: orders }
          operator: equals
          values: [East, West]
```

For dashboard-level reuse hoist these into `filters.dimensions[]` of the dashboard YAML and remove from the chart.

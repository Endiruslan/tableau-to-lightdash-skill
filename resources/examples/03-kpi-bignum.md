# Example 3 — KPI: Total Revenue (Big Number with comparison)

## Tableau

A "KPI worksheet" in Tableau is usually a sheet with a single measure on Text mark, no rows/cols pills, large font, often paired with a comparison via a calc field.

```xml
<worksheet name="Total Revenue KPI">
  <table>
    <view><datasources><datasource name="Orders"/></datasources></view>
    <encodings><text column="[federated.x].[sum:Sales:qk]"/></encodings>
    <mark class="Text"/>
    <filter column="[Order Date]" class="relative-date" period-type-v2="month-to-date"/>
  </table>
</worksheet>
```

A second worksheet often holds the comparison metric (`SUM(Sales) for previous month`). In Lightdash we get this with a single `big_number` chart with `showComparison: true`.

## Lightdash chart — `lightdash/charts/total-revenue-kpi.yml`

```yaml
chartConfig:
  config:
    comparisonFormat: percentage
    comparisonLabel: vs. Previous Month
    label: Total Revenue
    selectedField: orders_total_sales
    showBigNumberLabel: true
    showComparison: true
    style: M
  type: big_number
contentType: chart
metricQuery:
  dimensions:
    - orders_order_date_month
  exploreName: orders
  filters: {}
  limit: 2
  metrics:
    - orders_total_sales
  sorts:
    - descending: true
      fieldId: orders_order_date_month
name: Total Revenue
slug: total-revenue-kpi
spaceSlug: sales
tableName: orders
version: 1
```

Notes:
- `limit: 2` and the month dim is required so Lightdash has a "this month" + "last month" row to diff.
- `style: M` formats the number in millions.
- If the Tableau filter was `month-to-date`, you may want to apply that on the dashboard rather than the chart so the comparison anchors to "current month so far".

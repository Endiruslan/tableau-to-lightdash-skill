# Example 4 — Table with conditional formatting: Region × Category Sales

## Tableau

A highlight table in Tableau: discrete dim on rows, discrete dim on cols, measure on Color and Text.

```xml
<worksheet name="Region x Category">
  <table>
    <view><datasources><datasource name="Orders"/></datasources></view>
    <rows>[Region]</rows>
    <cols>[Category]</cols>
    <encodings>
      <color column="[federated.x].[sum:Sales:qk]"/>
      <text  column="[federated.x].[sum:Sales:qk]"/>
    </encodings>
    <mark class="Square"/>
  </table>
</worksheet>
```

## Lightdash chart — pivoted table with gradient

```yaml
chartConfig:
  config:
    columns:
      orders_region:
        frozen: true
        name: Region
        visible: true
      orders_category:
        name: Category
        visible: true
      orders_total_sales:
        name: Sales
        visible: true
    conditionalFormattings:
      - applyTo: cell
        color:
          end: "#10B981"
          start: "#FFFFFF"
        rule:
          max: auto
          min: auto
        target:
          fieldId: orders_total_sales
    hideRowNumbers: true
    showColumnCalculation: true
  type: table
contentType: chart
metricQuery:
  dimensions:
    - orders_region
    - orders_category
  exploreName: orders
  filters: {}
  limit: 500
  metrics:
    - orders_total_sales
  sorts:
    - descending: true
      fieldId: orders_total_sales
name: Region x Category Sales
pivotConfig:
  columns:
    - orders_category
slug: region-x-category-sales
spaceSlug: sales
tableName: orders
version: 1
```

Notes:
- `Square` mark + measure on color = highlight table → Lightdash `table` with `pivotConfig` and a gradient `conditionalFormattings`.
- Row totals come from `showColumnCalculation: true`.
- Region is frozen so it stays visible while scrolling pivoted columns.

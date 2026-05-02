# Example 1 — Bar chart: Sales by Region

## Tableau (.twb excerpt)

```xml
<datasources>
  <datasource name="Orders">
    <connection class="snowflake" server="acme.snowflakecomputing.com" database="ANALYTICS"/>
    <column name="[Region]" role="dimension" type="nominal" datatype="string" caption="Region"/>
    <column name="[Sales]"  role="measure"   type="quantitative" datatype="real"   aggregation="sum" caption="Sales" default-format="$#,##0"/>
  </datasource>
</datasources>

<worksheets>
  <worksheet name="Sales by Region">
    <table>
      <view><datasources><datasource name="Orders"/></datasources></view>
      <rows>[Region]</rows>
      <cols>[federated.x].[sum:Sales:qk]</cols>
      <mark class="Bar"/>
    </table>
  </worksheet>
</worksheets>
```

## Lightdash dbt model — `models/orders.yml`

```yaml
version: 2
models:
  - name: orders
    meta:
      label: Orders
      group_label: Sales
      primary_key: order_id
    columns:
      - name: region
        meta:
          dimension:
            label: Region
            type: string
      - name: sales
        meta:
          dimension:
            label: Sales
            type: number
            format: usd
          metrics:
            total_sales:
              type: sum
              label: Total Sales
              format: usd
```

## Lightdash chart — `lightdash/charts/sales-by-region.yml`

```yaml
chartConfig:
  config:
    eChartsConfig:
      series:
        - encode:
            xRef:
              field: orders_total_sales
            yRef:
              field: orders_region
          type: bar
    layout:
      flipAxes: true
      xField: orders_region
      yField:
        - orders_total_sales
  type: cartesian
contentType: chart
metricQuery:
  dimensions:
    - orders_region
  exploreName: orders
  filters: {}
  limit: 500
  metrics:
    - orders_total_sales
  sorts:
    - descending: true
      fieldId: orders_total_sales
name: Sales by Region
slug: sales-by-region
spaceSlug: sales
tableName: orders
version: 1
```

Notes:
- Tableau "Region on Rows + Sales on Cols + Bar mark" → horizontal bar in Lightdash (`flipAxes: true`).
- Sort descending by sales because Tableau bars typically sort that way visually; explicit in YAML.

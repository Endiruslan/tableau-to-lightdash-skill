# Example 6 — Full dashboard migration: Executive Sales Overview

## Tableau dashboard

```xml
<dashboard name="Executive Sales Overview">
  <size sizing-mode="automatic" minwidth="1280" minheight="900"/>
  <datasources><datasource name="Orders"/></datasources>
  <zones is-pixels="true">
    <zone id="title" x="0" y="0" w="1280" h="50" type-v2="text">
      <formatted-text><run fontsize="18" bold="true">Executive Sales Overview</run></formatted-text>
    </zone>
    <zone id="kpi-rev"   x="0"   y="50" w="320" h="120" type-v2="worksheet" name="Total Revenue KPI"/>
    <zone id="kpi-orders" x="320" y="50" w="320" h="120" type-v2="worksheet" name="Total Orders KPI"/>
    <zone id="kpi-aov"   x="640" y="50" w="320" h="120" type-v2="worksheet" name="AOV KPI"/>
    <zone id="kpi-cust"  x="960" y="50" w="320" h="120" type-v2="worksheet" name="New Customers KPI"/>
    <zone id="trend"     x="0"   y="170" w="800" h="400" type-v2="worksheet" name="Monthly Sales by Segment"/>
    <zone id="region"    x="800" y="170" w="480" h="400" type-v2="worksheet" name="Sales by Region"/>
    <zone id="region-flt" x="0"  y="570" w="640" h="60" type-v2="filter" param="[Region]" name="Region"/>
    <zone id="date-flt"   x="640" y="570" w="640" h="60" type-v2="filter" param="[Order Date]" name="Order Date"/>
    <zone id="footer"     x="0"  y="630" w="1280" h="30" type-v2="text">
      <formatted-text><run italic="true">Updated nightly. Source: Snowflake.</run></formatted-text>
    </zone>
  </zones>
  <actions>
    <action name="Filter Region from Region Chart" type="filter">
      <activation method="on-select"/>
      <source sheet="Sales by Region" datasource="Orders"/>
      <target sheet="Monthly Sales by Segment">
        <target-field column="[Region]"/>
      </target>
    </action>
  </actions>
</dashboard>
```

## Lightdash dashboard — `lightdash/dashboards/executive-sales-overview.yml`

```yaml
config:
  isAddFilterDisabled: false
  isDateZoomDisabled: false
contentType: dashboard
description: Executive sales overview — migrated from Tableau.
filters:
  dimensions:
    - id: f-region
      label: Region
      operator: equals
      required: false
      singleValue: false
      target:
        fieldId: orders_region
        tableName: orders
      tileTargets: {}
      values: []
    - id: f-order-date
      label: Order Date
      operator: inThePast
      required: false
      settings:
        completed: false
        unitOfTime: months
      target:
        fieldId: orders_order_date
        tableName: orders
      tileTargets: {}
      values:
        - 12
name: Executive Sales Overview
slug: executive-sales-overview
spaceSlug: leadership
tiles:
  - h: 1
    properties:
      text: Executive Sales Overview
    type: heading
    w: 36
    x: 0
    y: 0
  - h: 3
    properties:
      chartSlug: total-revenue-kpi
      hideTitle: false
      title: Total Revenue
    type: saved_chart
    w: 9
    x: 0
    y: 1
  - h: 3
    properties:
      chartSlug: total-orders-kpi
      title: Total Orders
    type: saved_chart
    w: 9
    x: 9
    y: 1
  - h: 3
    properties:
      chartSlug: avg-order-value-kpi
      title: AOV
    type: saved_chart
    w: 9
    x: 18
    y: 1
  - h: 3
    properties:
      chartSlug: new-customers-kpi
      title: New Customers
    type: saved_chart
    w: 9
    x: 27
    y: 1
  - h: 8
    properties:
      chartSlug: monthly-sales-by-segment
      title: Monthly Sales by Segment
    type: saved_chart
    w: 22
    x: 0
    y: 4
  - h: 8
    properties:
      chartSlug: sales-by-region
      title: Sales by Region
    type: saved_chart
    w: 14
    x: 22
    y: 4
  - h: 1
    properties:
      content: |
        *Updated nightly. Source: Snowflake.*
    type: markdown
    w: 36
    x: 0
    y: 12
version: 1
```

## Translation notes

- Title text zone → `type: heading`.
- Four KPI worksheets → four `saved_chart` tiles `w: 9` (quarter width).
- The two filter zones are **promoted to dashboard filters** (no tile emitted). The geometry they occupied was reabsorbed.
- Footer text zone → narrow markdown tile with italic.
- The "Filter Region" filter action becomes the dashboard's region filter — Lightdash applies it to all matching tiles automatically; no per-action wiring needed.
- Pixel coordinates were converted via `xGrid = round(x / 1280 * 36)`, snapped to common widths (9, 14, 22, 36).
- Worksheet zone width 800/1280 = 22.5 → snapped to 22; companion 480/1280 = 13.5 → 14. (22+14 = 36 ✓.)

# CHART TYPE MAPPING — TABLEAU MARKS → LIGHTDASH CHARTS

Tableau chooses the visual via the `<mark class="…"/>` plus the shelf composition. Lightdash uses a top-level `chartConfig.type`. Mapping is rarely 1-to-1; this file gives the decision tree and worked examples.

---

## 1. Quick lookup

| Tableau `<mark class>` | Tableau context | Lightdash `chartConfig.type` | Series sub-type |
|---|---|---|---|
| `Bar` | discrete dim on cols, measure on rows | `cartesian` | `bar` (vertical) |
| `Bar` | measure on cols, discrete dim on rows | `cartesian` + `flipAxes: true` | `bar` (horizontal) |
| `Bar` | dim + dim split via color | `cartesian` + pivot | stacked / grouped `bar` |
| `Line` | continuous date on cols, measure on rows | `cartesian` | `line` |
| `Line` | multiple measures stacked | `cartesian` + dual axis | `line` x N |
| `Area` | continuous date + measure(s) | `cartesian` | `line` with `areaStyle: {}` |
| `Circle` / `Square` / `Shape` | two measures (scatter) | `cartesian` | `scatter` |
| `Circle` (small mult.) | many dims as detail | `cartesian` `scatter` | |
| `Pie` | one dim + one measure, ≤ ~7 slices | `pie` (set `isDonut` if hollow) | — |
| `Text` | dims on rows + cols, measures as values | `table` | — |
| `Map` | geographic role + lat/lon | `map` | `scatter` / `area` / `heatmap` |
| `Polygon` | choropleth | `map` | `area` |
| `Density` | dense xy points | `map` `heatmap`, or custom Vega-Lite | — |
| `Gantt` | start/end dates | `cartesian` (bar with `markLine`) **or** custom Vega-Lite | — |
| `Bar` with one measure, no dim | KPI header | `big_number` | — |
| `Pie` + funnel-shaped sort | conversion stages | `funnel` | — |
| `Bar` / arc gauge | single metric vs target | `gauge` | — |
| Treemap visual (use Marks → Square + size) | hierarchical part-of-whole | `treemap` | — |
| `Automatic` | Tableau auto-pick | apply rules below | — |

### Rules for `Automatic`

Tableau's `Automatic` mark resolves at render. Replicate the rule:

1. Cols has continuous date AND rows has a measure → **line**.
2. Cols has a discrete dim AND rows has a measure → **bar**.
3. Both rows and cols are measures → **scatter** (`Circle`).
4. Both rows and cols are discrete dims with measure on Text mark → **table**.
5. Single measure on the marks card with no row/col → **big_number**.

---

## 2. Detection from XML

To pick a Lightdash chart type, parse:

```python
mark      = sheet.find(".//mark").get("class")          # may be Automatic
rows_text = sheet.findtext("table/rows")                # shelf expression
cols_text = sheet.findtext("table/cols")
encodings = {e.tag: e.get("column") for e in sheet.findall("table/encodings/*")}
```

Then run the decision tree above. When `mark == "Automatic"` use the shelf rules.

---

## 3. Mapping examples

### Tableau bar — `Region` on rows, `SUM(Sales)` on cols

```xml
<worksheet name="Sales by Region">
  <table>
    <view><datasources><datasource name="Orders"/></datasources></view>
    <rows>[Region]</rows>
    <cols>[federated.x].[sum:Sales:qk]</cols>
    <mark class="Bar"/>
  </table>
</worksheet>
```

→ Lightdash horizontal bar:

```yaml
chartConfig:
  config:
    eChartsConfig:
      series:
        - encode:
            xRef: { field: orders_total_sales }
            yRef: { field: orders_region }
          type: bar
    layout:
      flipAxes: true
      xField: orders_region
      yField: [orders_total_sales]
  type: cartesian
contentType: chart
metricQuery:
  dimensions: [orders_region]
  exploreName: orders
  filters: {}
  limit: 500
  metrics: [orders_total_sales]
  sorts:
    - { fieldId: orders_total_sales, descending: true }
name: "Sales by Region"
slug: sales-by-region
spaceSlug: sales
tableName: orders
version: 1
```

### Tableau line trend — month on cols, `SUM(Sales)` on rows, `Segment` on Color

```xml
<worksheet name="Trend">
  <table>
    <rows>[federated.x].[sum:Sales:qk]</rows>
    <cols>[federated.x].[mdy:Order Date:ok]</cols>
    <encodings><color column="[Segment]"/></encodings>
    <mark class="Line"/>
  </table>
</worksheet>
```

→ Lightdash pivoted line:

```yaml
chartConfig:
  config:
    eChartsConfig:
      legend: { show: true }
      series:
        - encode:
            xRef: { field: orders_order_date_month }
            yRef: { field: orders_total_sales }
          type: line
          smooth: false
    layout:
      xField: orders_order_date_month
      yField: [orders_total_sales]
  type: cartesian
pivotConfig:
  columns: [orders_segment]
contentType: chart
metricQuery:
  dimensions: [orders_order_date_month, orders_segment]
  exploreName: orders
  filters: {}
  limit: 500
  metrics: [orders_total_sales]
  sorts: [{ fieldId: orders_order_date_month, descending: false }]
name: Sales Trend by Segment
slug: sales-trend-by-segment
spaceSlug: sales
tableName: orders
version: 1
```

### Tableau highlight table → Lightdash table with conditional formatting

`<mark class="Square"/>` + dim on rows + dim on cols + measure on Color → use `chartConfig.type: table` with `pivotConfig.columns` and a `conditionalFormattings` gradient on the metric. See `lightdash-yaml-reference.md` table section.

### Tableau pie

```xml
<mark class="Pie"/>
<rows>[federated.x].[sum:Sales:qk]</rows>          <!-- on Angle -->
<encodings><color column="[Region]"/></encodings>  <!-- slices -->
```

→

```yaml
chartConfig:
  config:
    groupFieldIds: [orders_region]
    metricId: orders_total_sales
    valueLabel: outside
    showValue: true
  type: pie
…
```

### Tableau KPI (single measure, no shelves) → Lightdash `big_number`

When the worksheet has no rows/cols pills but one measure on the Marks card, emit `big_number` with `selectedField`. Limit 1.

---

## 4. Sort / limit / Top-N

Tableau's "Top 10 by SUM(Sales)" filter →

```yaml
metricQuery:
  ...
  limit: 10
  sorts: [{ fieldId: orders_total_sales, descending: true }]
```

If Top-N is paired with another sort dim, keep the limit but sort first by the ranking measure, then re-sort the chart by display dim if needed.

---

## 5. When in doubt — fall back to Vega-Lite

If a Tableau viz can't be mapped to a built-in (e.g. dual-encoded bullet charts, lollipop, ridgeline, gantt), emit a `chartConfig.type: custom` with a hand-written Vega-Lite spec. See the custom-viz section of `lightdash-yaml-reference.md`. Note the field names in the spec must match Lightdash field IDs (`<table>_<column>`).

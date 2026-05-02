# LIGHTDASH YAML/JSON GENERATION CHEATSHEET

A complete reference for generating Lightdash YAML/JSON charts, dashboards, and semantic layer models from another BI tool's exports.

---

## TABLE OF CONTENTS

1. [Chart Types Reference](#chart-types-reference)
2. [Dimensions Reference](#dimensions-reference)
3. [Metrics Reference](#metrics-reference)
4. [Tables & Joins Reference](#tables--joins-reference)
5. [Dashboard Reference](#dashboard-reference)
6. [CLI Reference](#cli-reference)
7. [Workflows & Best Practices](#workflows--best-practices)

---

## CHART TYPES REFERENCE

### Overview: Base Chart Structure

All charts share this common base structure:

```yaml
chartConfig:
  config: {}        # Type-specific configuration
  type: <type>      # "cartesian", "pie", "table", "big_number", etc.
contentType: chart  # Required: "chart", "dashboard", or "sql_chart"
dashboardSlug: null # Optional: scope chart to dashboard (won't appear in space)
metricQuery:
  dimensions: []
  exploreName: explore_name
  filters: {}
  limit: 500
  metrics: []
  sorts: []
name: "Chart Name"
slug: unique-chart-slug
spaceSlug: space-name
tableName: explore_name
version: 1
```

**Key ordering rule:** All YAML keys must be alphabetically sorted at every nesting level. Lightdash CLI writes with `sortKeys: true` and warns on upload if keys are unsorted.

---

### 1. CARTESIAN CHARTS (Bar, Line, Area, Scatter)

**Type:** `cartesian`

**Use cases:**
- **Bar:** Category comparisons, part-to-whole within categories
- **Line:** Trends over time with continuous change
- **Area:** Cumulative values, composition over time
- **Scatter:** Correlations, distributions, outliers

#### Configuration Keys

| Path | Type | Required | Description |
|------|------|----------|-------------|
| `chartConfig.config.eChartsConfig.series` | array | Yes | Array of series (bar, line, area, scatter) |
| `chartConfig.config.eChartsConfig.series[].type` | string | Yes | `"bar"`, `"line"`, `"area"`, `"scatter"` |
| `chartConfig.config.eChartsConfig.series[].encode.xRef.field` | string | Yes | Dimension field for X axis |
| `chartConfig.config.eChartsConfig.series[].encode.yRef.field` | string | Yes | Metric field for Y axis |
| `chartConfig.config.eChartsConfig.series[].encode.yRef.pivotValues` | array | No | Pivot configuration for grouping |
| `chartConfig.config.eChartsConfig.series[].name` | string | No | Series display name in legend |
| `chartConfig.config.eChartsConfig.series[].color` | string (hex) | No | Series color (e.g., `"#FF0000"`) |
| `chartConfig.config.eChartsConfig.series[].yAxisIndex` | number | No | Axis index (0=left, 1=right) for dual axes |
| `chartConfig.config.eChartsConfig.series[].stack` | string | No | Stack group name for stacking |
| `chartConfig.config.eChartsConfig.series[].smooth` | boolean | No | Smooth curves (line/area only) |
| `chartConfig.config.eChartsConfig.series[].areaStyle` | object | No | Presence indicates area chart |
| `chartConfig.config.eChartsConfig.series[].showSymbol` | boolean | No | Show data points (line only) |
| `chartConfig.config.eChartsConfig.series[].markLine` | object | No | Reference line configuration |
| `chartConfig.config.eChartsConfig.xAxis` | array | No | X-axis configuration (name, etc.) |
| `chartConfig.config.eChartsConfig.yAxis` | array | No | Y-axis configuration (multiple for dual axes) |
| `chartConfig.config.eChartsConfig.legend` | object | No | Legend positioning and visibility |
| `chartConfig.config.eChartsConfig.grid` | object | No | Chart area padding |
| `chartConfig.config.layout.xField` | string | Yes | Dimension field ID |
| `chartConfig.config.layout.yField` | array | Yes | Array of metric field IDs |
| `chartConfig.config.layout.flipAxes` | boolean | No | Swap X/Y for horizontal bars (default: `false`) |
| `chartConfig.config.layout.showGridX` | boolean | No | Show vertical gridlines |
| `chartConfig.config.layout.showGridY` | boolean | No | Show horizontal gridlines |
| `chartConfig.config.layout.stack` | boolean or string | No | Stack series (true or stack name) |
| `chartConfig.config.layout.colorByCategory` | boolean | No | Color each bar by its category value (default: `false`) |
| `chartConfig.config.layout.categoryColorOverrides` | object | No | Map of category value → hex color |
| `pivotConfig.columns` | array | No | Dimension field IDs to pivot into columns |

#### Enums

- **Series types:** `"bar"`, `"line"`, `"area"`, `"scatter"`
- **Axis index:** `0`, `1` (for dual Y-axes)

#### Minimal Valid Examples

**Bar Chart:**
```yaml
chartConfig:
  config:
    eChartsConfig:
      series:
        - encode:
            xRef:
              field: orders_category
            yRef:
              field: orders_total_revenue
          type: bar
    layout:
      xField: orders_category
      yField:
        - orders_total_revenue
  type: cartesian
contentType: chart
metricQuery:
  dimensions:
    - orders_category
  exploreName: orders
  filters: {}
  limit: 500
  metrics:
    - orders_total_revenue
  sorts:
    - descending: true
      fieldId: orders_total_revenue
name: "Sales by Category"
slug: sales-by-category
spaceSlug: sales
tableName: orders
version: 1
```

**Line Chart:**
```yaml
chartConfig:
  config:
    eChartsConfig:
      series:
        - encode:
            xRef:
              field: orders_created_month
            yRef:
              field: orders_total_revenue
          showSymbol: true
          smooth: true
          type: line
    layout:
      xField: orders_created_month
      yField:
        - orders_total_revenue
  type: cartesian
contentType: chart
metricQuery:
  dimensions:
    - orders_created_month
  exploreName: orders
  filters: {}
  limit: 500
  metrics:
    - orders_total_revenue
  sorts:
    - descending: false
      fieldId: orders_created_month
name: "Monthly Revenue Trend"
slug: monthly-revenue
spaceSlug: sales
tableName: orders
version: 1
```

**Area Chart with Stacking:**
```yaml
chartConfig:
  config:
    eChartsConfig:
      legend:
        show: true
      series:
        - areaStyle: {}
          encode:
            xRef:
              field: orders_created_month
            yRef:
              field: orders_total_revenue
              pivotValues:
                - field: orders_category
                  value: Electronics
          stack: total
          type: line
    layout:
      xField: orders_created_month
      yField:
        - orders_total_revenue
  type: cartesian
contentType: chart
metricQuery:
  dimensions:
    - orders_created_month
    - orders_category
  exploreName: orders
  filters: {}
  limit: 500
  metrics:
    - orders_total_revenue
  sorts:
    - descending: false
      fieldId: orders_created_month
name: "Revenue by Category Trend"
pivotConfig:
  columns:
    - orders_category
slug: revenue-by-category-trend
spaceSlug: sales
tableName: orders
version: 1
```

**Dual Y-Axis:**
```yaml
chartConfig:
  config:
    eChartsConfig:
      series:
        - encode:
            xRef:
              field: orders_created_month
            yRef:
              field: orders_total_revenue
          name: "Revenue"
          type: bar
          yAxisIndex: 0
        - encode:
            xRef:
              field: orders_created_month
            yRef:
              field: profit_margin
          name: "Margin %"
          smooth: true
          type: line
          yAxisIndex: 1
      yAxis:
        - name: "Revenue ($)"
        - name: "Margin (%)"
    layout:
      xField: orders_created_month
      yField:
        - orders_total_revenue
        - profit_margin
  type: cartesian
contentType: chart
metricQuery:
  dimensions:
    - orders_created_month
  exploreName: orders
  filters: {}
  limit: 500
  metrics:
    - orders_total_revenue
  sorts:
    - descending: false
      fieldId: orders_created_month
  tableCalculations:
    - displayName: Profit Margin %
      name: profit_margin
      sql: "${orders.profit}/${orders.total_revenue} * 100"
name: "Revenue & Margin"
slug: revenue-margin
spaceSlug: sales
tableName: orders
version: 1
```

---

### 2. BIG NUMBER CHARTS (KPIs)

**Type:** `big_number`

**Use cases:** Single metric displays, KPIs, headline numbers, comparisons to previous periods

#### Configuration Keys

| Path | Type | Required | Description |
|------|------|----------|-------------|
| `chartConfig.config.selectedField` | string | Yes | Field ID (metric) to display |
| `chartConfig.config.label` | string | No | Custom label (defaults to field name) |
| `chartConfig.config.showBigNumberLabel` | boolean | No | Show label above number |
| `chartConfig.config.showTableNamesInLabel` | boolean | No | Include table name in label |
| `chartConfig.config.style` | string | No | Compact format: `"K"`, `"M"`, `"B"`, `"T"` |
| `chartConfig.config.showComparison` | boolean | No | Show change vs. previous row |
| `chartConfig.config.comparisonFormat` | string | No | `"raw"` or `"percentage"` |
| `chartConfig.config.comparisonLabel` | string | No | Label for comparison (e.g., "vs. Last Month") |
| `chartConfig.config.flipColors` | boolean | No | Reverse colors (use for costs, errors) |
| `chartConfig.config.conditionalFormattings` | array | No | Color rules based on conditions |

#### Enums

- **Style:** `"K"` (thousands), `"M"` (millions), `"B"` (billions), `"T"` (trillions)
- **Comparison format:** `"raw"`, `"percentage"`
- **Operators:** `equals`, `notEquals`, `lessThan`, `lessThanOrEqual`, `greaterThan`, `greaterThanOrEqual`, `inBetween`, `notInBetween`, `isNull`, `notNull`

#### Minimal Valid Example

```yaml
chartConfig:
  config:
    label: Total Revenue
    selectedField: orders_total_revenue
    showBigNumberLabel: true
    style: M
  type: big_number
contentType: chart
metricQuery:
  dimensions: []
  exploreName: orders
  filters: {}
  limit: 1
  metrics:
    - orders_total_revenue
  sorts: []
name: "Total Revenue"
slug: total-revenue
spaceSlug: sales
tableName: orders
version: 1
```

#### Comparison Example

```yaml
chartConfig:
  config:
    comparisonFormat: percentage
    comparisonLabel: vs. Previous Month
    label: Monthly Revenue
    selectedField: orders_total_revenue
    showBigNumberLabel: true
    showComparison: true
    style: M
  type: big_number
contentType: chart
metricQuery:
  dimensions:
    - orders_created_month
  exploreName: orders
  filters: {}
  limit: 2
  metrics:
    - orders_total_revenue
  sorts:
    - descending: true
      fieldId: orders_created_month
name: "Monthly Revenue"
slug: monthly-revenue
spaceSlug: sales
tableName: orders
version: 1
```

---

### 3. TABLE CHARTS

**Type:** `table`

**Use cases:** Detailed data display, drill-down analysis, raw records, pivoted cross-tabs

#### Configuration Keys

| Path | Type | Required | Description |
|------|------|----------|-------------|
| `chartConfig.config.columns` | object | No | Per-column config (name, frozen, visibility, etc.) |
| `chartConfig.config.columns[fieldId].visible` | boolean | No | Column visibility |
| `chartConfig.config.columns[fieldId].name` | string | No | Custom column header name |
| `chartConfig.config.columns[fieldId].frozen` | boolean | No | Freeze column to left |
| `chartConfig.config.columns[fieldId].displayStyle` | string | No | `"text"` or `"bar"` |
| `chartConfig.config.columns[fieldId].color` | string (hex) | No | Bar color for `displayStyle: "bar"` |
| `chartConfig.config.hideRowNumbers` | boolean | No | Hide row number column |
| `chartConfig.config.showColumnCalculation` | boolean | No | Show totals row at bottom |
| `chartConfig.config.showRowCalculation` | boolean | No | Show totals column on right |
| `chartConfig.config.showTableNames` | boolean | No | Show table names in headers |
| `chartConfig.config.showResultsTotal` | boolean | No | Show result count |
| `chartConfig.config.showSubtotals` | boolean | No | Show subtotal rows for grouped data |
| `chartConfig.config.metricsAsRows` | boolean | No | Transpose metrics into rows |
| `chartConfig.config.conditionalFormattings` | array | No | Conditional formatting rules |
| `pivotConfig.columns` | array | No | Dimension field IDs to pivot into columns |

#### Conditional Formatting (Single Color)

```yaml
conditionalFormattings:
  - applyTo: "cell"  # "cell" or "text"
    color: "#EF4444"
    rules:
      - id: "rule-id"
        operator: "lessThan"
        values: [1000]
    target:
      fieldId: orders_revenue
```

#### Conditional Formatting (Gradient)

```yaml
conditionalFormattings:
  - applyTo: "cell"
    color:
      start: "#FFFFFF"
      end: "#10B981"
    rule:
      min: "auto"
      max: "auto"
    target:
      fieldId: orders_revenue
```

#### Minimal Valid Example

```yaml
chartConfig:
  config:
    columns:
      orders_region:
        frozen: true
        name: Region
        visible: true
      orders_sales_rep:
        name: Sales Rep
        visible: true
      orders_total_revenue:
        name: Revenue
        visible: true
    hideRowNumbers: false
    showColumnCalculation: true
  type: table
contentType: chart
metricQuery:
  dimensions:
    - orders_region
    - orders_sales_rep
  exploreName: orders
  filters: {}
  limit: 500
  metrics:
    - orders_total_revenue
  sorts: []
name: "Sales by Region"
slug: sales-by-region
spaceSlug: sales
tableName: orders
version: 1
```

#### Pivoted Table Example

```yaml
chartConfig:
  config:
    columns:
      orders_region:
        frozen: true
        name: Region
        visible: true
      orders_product_category:
        visible: true
      orders_total_revenue:
        name: Revenue
        visible: true
    showColumnCalculation: true
  type: table
contentType: chart
metricQuery:
  dimensions:
    - orders_region
    - orders_product_category
  exploreName: orders
  filters: {}
  limit: 500
  metrics:
    - orders_total_revenue
  sorts: []
name: "Revenue by Region & Category"
pivotConfig:
  columns:
    - orders_product_category
slug: revenue-pivot
spaceSlug: sales
tableName: orders
version: 1
```

---

### 4. PIE & DONUT CHARTS

**Type:** `pie`

**Use cases:** Part-of-whole relationships, proportions, 3-7 categories only

#### Configuration Keys

| Path | Type | Required | Description |
|------|------|----------|-------------|
| `chartConfig.config.groupFieldIds` | array | Yes | One or more dimensions to slice by |
| `chartConfig.config.metricId` | string | Yes | Metric field ID |
| `chartConfig.config.isDonut` | boolean | No | Donut vs. pie (default: `false`) |
| `chartConfig.config.valueLabel` | string | No | Label position: `"hidden"`, `"inside"`, `"outside"` |
| `chartConfig.config.showValue` | boolean | No | Show numeric values on slices |
| `chartConfig.config.showPercentage` | boolean | No | Show percentages on slices |
| `chartConfig.config.showLegend` | boolean | No | Show legend |
| `chartConfig.config.legendPosition` | string | No | `"horizontal"` or `"vertical"` |
| `chartConfig.config.legendMaxItemLength` | number | No | Max characters before truncation |
| `chartConfig.config.groupLabelOverrides` | object | No | Custom labels for slice values |
| `chartConfig.config.groupColorOverrides` | object | No | Custom hex colors for slice values |
| `chartConfig.config.groupSortOverrides` | array | No | Custom sort order (clockwise from top) |
| `chartConfig.config.groupValueOptionOverrides` | object | No | Per-slice options (e.g., `valueLabel: hidden`) |

#### Minimal Valid Example

```yaml
chartConfig:
  config:
    groupFieldIds:
      - orders_product_category
    metricId: orders_total_revenue
  type: pie
contentType: chart
metricQuery:
  dimensions:
    - orders_product_category
  exploreName: orders
  filters: {}
  limit: 10
  metrics:
    - orders_total_revenue
  sorts:
    - descending: true
      fieldId: orders_total_revenue
name: "Revenue by Category"
slug: revenue-by-category
spaceSlug: sales
tableName: orders
version: 1
```

#### Advanced Example with Custom Colors & Labels

```yaml
chartConfig:
  config:
    groupColorOverrides:
      APAC: "#f59e0b"
      EMEA: "#10b981"
      NA: "#3b82f6"
    groupFieldIds:
      - orders_region
    groupLabelOverrides:
      APAC: Asia Pacific
      EMEA: Europe & Middle East
      NA: North America
    groupSortOverrides:
      - NA
      - EMEA
      - APAC
    isDonut: true
    legendPosition: vertical
    metricId: orders_total_revenue
    showLegend: true
    showPercentage: true
    showValue: true
    valueLabel: inside
  type: pie
contentType: chart
metricQuery:
  dimensions:
    - orders_region
  exploreName: orders
  filters: {}
  limit: 8
  metrics:
    - orders_total_revenue
  sorts:
    - descending: true
      fieldId: orders_total_revenue
name: "Regional Sales"
slug: regional-sales
spaceSlug: sales
tableName: orders
version: 1
```

---

### 5. FUNNEL CHARTS

**Type:** `funnel`

**Use cases:** Conversion funnels, sales pipelines, sequential processes, drop-off analysis

#### Configuration Keys

| Path | Type | Required | Description |
|------|------|----------|-------------|
| `chartConfig.config.dataInput` | string | Yes | `"row"` (default) or `"column"` |
| `chartConfig.config.fieldId` | string | Yes | Metric or dimension field ID |
| `chartConfig.config.labels` | object | No | Label configuration |
| `chartConfig.config.labels.position` | string | No | `"inside"`, `"left"`, `"right"`, `"hidden"` |
| `chartConfig.config.labels.showValue` | boolean | No | Show numeric values |
| `chartConfig.config.labels.showPercentage` | boolean | No | Show percentage of max |
| `chartConfig.config.labelOverrides` | object | No | Custom labels for stages |
| `chartConfig.config.colorOverrides` | object | No | Custom hex colors for stages |
| `chartConfig.config.showLegend` | boolean | No | Show legend |
| `chartConfig.config.legendPosition` | string | No | `"horizontal"` or `"vertical"` |

#### Minimal Valid Example

```yaml
chartConfig:
  config:
    dataInput: row
    fieldId: leads_count
  type: funnel
contentType: chart
metricQuery:
  dimensions:
    - leads_stage
  exploreName: leads
  filters: {}
  limit: 10
  metrics:
    - leads_count
  sorts:
    - descending: false
      fieldId: leads_stage
name: "Sales Funnel"
slug: sales-funnel
spaceSlug: sales
tableName: leads
version: 1
```

#### Advanced Example with Custom Colors & Labels

```yaml
chartConfig:
  config:
    colorOverrides:
      opportunities_stage_closed: "#8b5cf6"
      opportunities_stage_lead: "#3b82f6"
      opportunities_stage_proposal: "#10b981"
    dataInput: row
    fieldId: opportunities_count
    labelOverrides:
      opportunities_stage_closed: Closed Won
      opportunities_stage_lead: New Leads
      opportunities_stage_proposal: Proposal Sent
    labels:
      position: inside
      showPercentage: true
      showValue: true
    legendPosition: vertical
    showLegend: true
  type: funnel
contentType: chart
metricQuery:
  dimensions:
    - opportunities_stage
  exploreName: opportunities
  filters: {}
  limit: 10
  metrics:
    - opportunities_count
  sorts:
    - descending: false
      fieldId: opportunities_stage
name: "Sales Pipeline"
slug: sales-pipeline
spaceSlug: sales
tableName: opportunities
version: 1
```

---

### 6. GAUGE CHARTS

**Type:** `gauge`

**Use cases:** KPI monitoring, progress indicators, status visualization, threshold tracking

#### Configuration Keys

| Path | Type | Required | Description |
|------|------|----------|-------------|
| `chartConfig.config.selectedField` | string | Yes | Metric field ID for gauge value |
| `chartConfig.config.min` | number | Yes | Minimum gauge value (default: 0) |
| `chartConfig.config.max` | number | No | Fixed maximum value |
| `chartConfig.config.maxFieldId` | string | No | Field ID for dynamic maximum (mutually exclusive with `max`) |
| `chartConfig.config.customLabel` | string | No | Custom label display |
| `chartConfig.config.showAxisLabels` | boolean | No | Show min/max on axis |
| `chartConfig.config.showPercentage` | boolean | No | Display as percentage of max |
| `chartConfig.config.customPercentageLabel` | string | No | Custom percentage label |
| `chartConfig.config.sections` | array | No | Array of colored range sections |
| `chartConfig.config.sections[].min` | number | No | Section start value (fixed) |
| `chartConfig.config.sections[].minFieldId` | string | No | Section start (dynamic) |
| `chartConfig.config.sections[].max` | number | No | Section end value (fixed) |
| `chartConfig.config.sections[].maxFieldId` | string | No | Section end (dynamic) |
| `chartConfig.config.sections[].color` | string (hex) | No | Section color |

#### Minimal Valid Example

```yaml
chartConfig:
  config:
    customLabel: CSAT Score
    max: 10
    min: 0
    selectedField: customer_metrics_csat_score
  type: gauge
contentType: chart
metricQuery:
  dimensions: []
  exploreName: customer_metrics
  filters: {}
  limit: 1
  metrics:
    - customer_metrics_csat_score
  sorts: []
name: "Customer Satisfaction"
slug: customer-satisfaction
spaceSlug: metrics
tableName: customer_metrics
version: 1
```

#### Advanced Example with Sections

```yaml
chartConfig:
  config:
    customLabel: CSAT Score
    max: 10
    min: 0
    sections:
      - color: "#DC2626"
        max: 5
        min: 0
      - color: "#FBBF24"
        max: 7
        min: 5
      - color: "#10B981"
        max: 10
        min: 7
    selectedField: customer_metrics_csat_score
    showAxisLabels: true
  type: gauge
contentType: chart
metricQuery:
  dimensions: []
  exploreName: customer_metrics
  filters: {}
  limit: 1
  metrics:
    - customer_metrics_csat_score
  sorts: []
name: "CSAT Score"
slug: csat-score
spaceSlug: metrics
tableName: customer_metrics
version: 1
```

---

### 7. TREEMAP CHARTS

**Type:** `treemap`

**Use cases:** Hierarchical data, part-to-whole, multi-level comparisons, space-efficient display

#### Configuration Keys

| Path | Type | Required | Description |
|------|------|----------|-------------|
| `chartConfig.config.groupFieldIds` | array | Yes | 1-3 dimensions for hierarchy levels |
| `chartConfig.config.sizeMetricId` | string | Yes | Metric determining rectangle size |
| `chartConfig.config.colorMetricId` | string | No | Metric for color intensity |
| `chartConfig.config.visibleMin` | number | No | Minimum size threshold for display |
| `chartConfig.config.leafDepth` | number | No | Depth level to display as leaves |
| `chartConfig.config.startColor` | string (hex) | No | Gradient start color |
| `chartConfig.config.endColor` | string (hex) | No | Gradient end color |
| `chartConfig.config.useDynamicColors` | boolean | No | Enable dynamic color scaling |
| `chartConfig.config.startColorThreshold` | number | No | Value threshold for start color |
| `chartConfig.config.endColorThreshold` | number | No | Value threshold for end color |

#### Minimal Valid Example

```yaml
chartConfig:
  config:
    groupFieldIds:
      - products_category
      - products_subcategory
    sizeMetricId: products_total_revenue
  type: treemap
contentType: chart
metricQuery:
  dimensions:
    - products_category
    - products_subcategory
  exploreName: products
  filters: {}
  limit: 50
  metrics:
    - products_total_revenue
  sorts:
    - descending: true
      fieldId: products_total_revenue
name: "Category Revenue"
slug: category-revenue
spaceSlug: sales
tableName: products
version: 1
```

#### Advanced Example with Color Gradient

```yaml
chartConfig:
  config:
    colorMetricId: products_profit_margin
    endColor: "#22c55e"
    groupFieldIds:
      - products_category
    sizeMetricId: products_total_revenue
    startColor: "#ef4444"
  type: treemap
contentType: chart
metricQuery:
  dimensions:
    - products_category
  exploreName: products
  filters: {}
  limit: 25
  metrics:
    - products_total_revenue
    - products_profit_margin
  sorts:
    - descending: true
      fieldId: products_total_revenue
name: "Revenue & Margin by Category"
slug: revenue-margin-treemap
spaceSlug: sales
tableName: products
version: 1
```

---

### 8. MAP CHARTS

**Type:** `map`

**Use cases:** Geographic data, store locations, regional metrics, density analysis

#### Configuration Keys - Core

| Path | Type | Required | Description |
|------|------|----------|-------------|
| `chartConfig.config.locationType` | string | Yes | `"scatter"`, `"area"`, `"heatmap"` |
| `chartConfig.config.mapType` | string | Yes | `"USA"`, `"world"`, `"europe"`, `"custom"` |

#### Configuration Keys - Scatter Maps

| Path | Type | Description |
|------|------|-------------|
| `chartConfig.config.latitudeFieldId` | string | Latitude field |
| `chartConfig.config.longitudeFieldId` | string | Longitude field |
| `chartConfig.config.sizeFieldId` | string | Metric for bubble size |
| `chartConfig.config.minBubbleSize` | number | Min bubble size (default varies) |
| `chartConfig.config.maxBubbleSize` | number | Max bubble size |

#### Configuration Keys - Area Maps

| Path | Type | Description |
|------|------|-------------|
| `chartConfig.config.locationFieldId` | string | Field matching GeoJSON properties |
| `chartConfig.config.geoJsonPropertyKey` | string | GeoJSON property to match |
| `chartConfig.config.customGeoJsonUrl` | string | URL to custom GeoJSON (for mapType: custom) |
| `chartConfig.config.valueFieldId` | string | Metric for region color |
| `chartConfig.config.noDataColor` | string | Color for regions with no data |

#### Configuration Keys - Heatmap

| Path | Type | Description |
|------|------|-------------|
| `chartConfig.config.latitudeFieldId` | string | Latitude field |
| `chartConfig.config.longitudeFieldId` | string | Longitude field |
| `chartConfig.config.valueFieldId` | string | Metric for intensity |
| `chartConfig.config.heatmapConfig.blur` | number | Blur (0-30) |
| `chartConfig.config.heatmapConfig.opacity` | number | Opacity (0.1-1) |
| `chartConfig.config.heatmapConfig.radius` | number | Point size (1-50) |

#### Configuration Keys - Visual

| Path | Type | Description |
|------|------|-------------|
| `chartConfig.config.colorRange` | array | Gradient colors (2-5 hex codes) |
| `chartConfig.config.colorOverrides` | object | Per-region color overrides (area only) |
| `chartConfig.config.backgroundColor` | string | Map background hex color |
| `chartConfig.config.dataLayerOpacity` | number | Data layer opacity (0-1) |
| `chartConfig.config.tileBackground` | string | Base map: `"none"`, `"openstreetmap"`, `"light"`, `"dark"`, `"satellite"` |
| `chartConfig.config.showLegend` | boolean | Show legend |
| `chartConfig.config.fieldConfig` | object | Tooltip field config (visible, label) |
| `chartConfig.config.defaultZoom` | number | Initial zoom level |
| `chartConfig.config.defaultCenterLat` | number | Initial center latitude |
| `chartConfig.config.defaultCenterLon` | number | Initial center longitude |
| `chartConfig.config.saveMapExtent` | boolean | Preserve zoom/pan on save |

#### Minimal Valid Example - Scatter Map

```yaml
chartConfig:
  config:
    defaultCenterLat: 39.8283
    defaultCenterLon: -98.5795
    defaultZoom: 4
    latitudeFieldId: stores_latitude
    locationType: scatter
    longitudeFieldId: stores_longitude
    mapType: USA
  type: map
contentType: chart
metricQuery:
  dimensions:
    - stores_store_name
  exploreName: stores
  filters: {}
  limit: 500
  metrics: []
  sorts: []
name: "Store Locations"
slug: store-locations
spaceSlug: sales
tableName: stores
version: 1
```

#### Minimal Valid Example - Choropleth (Area Map)

```yaml
chartConfig:
  config:
    colorRange:
      - "#f0f9ff"
      - "#2563eb"
    geoJsonPropertyKey: name
    locationFieldId: orders_state
    locationType: area
    mapType: USA
    valueFieldId: orders_total_sales
  type: map
contentType: chart
metricQuery:
  dimensions:
    - orders_state
  exploreName: orders
  filters: {}
  limit: 50
  metrics:
    - orders_total_sales
  sorts: []
name: "Sales by State"
slug: sales-by-state
spaceSlug: sales
tableName: orders
version: 1
```

---

### 9. CUSTOM VISUALIZATIONS (Vega-Lite)

**Type:** `custom`

**Use cases:** Advanced/unsupported chart types, full visual control, complex transformations

#### Configuration Keys

| Path | Type | Required | Description |
|------|------|----------|-------------|
| `chartConfig.config.spec` | object | Yes | Full Vega-Lite specification |
| `chartConfig.config.spec.$schema` | string | Yes | Vega-Lite schema URL |
| `chartConfig.config.spec.mark` | string or object | Yes | Mark type: `"bar"`, `"line"`, `"point"`, `"rect"`, etc. |
| `chartConfig.config.spec.encoding` | object | Yes | Visual encodings (x, y, color, size, etc.) |
| `chartConfig.config.spec.transform` | array | No | Data transformations |
| `chartConfig.config.spec.layer` | array | No | Layered charts |

#### Key Notes

- **Data is provided automatically** by Lightdash (do NOT specify `data` property)
- **Field names** match `metricQuery` fields (e.g., `orders_status`, `orders_count`)
- **Vega-Lite reference:** [https://vega.github.io/vega-lite/](https://vega.github.io/vega-lite/)

#### Minimal Valid Example

```yaml
chartConfig:
  config:
    spec:
      $schema: https://vega.github.io/schema/vega-lite/v5.json
      encoding:
        x:
          field: orders_status
          type: nominal
        y:
          field: orders_count
          type: quantitative
      mark: bar
  type: custom
contentType: chart
metricQuery:
  dimensions:
    - orders_status
  exploreName: orders
  filters: {}
  limit: 50
  metrics:
    - orders_count
  sorts: []
name: "Order Count by Status"
slug: order-count-by-status
spaceSlug: analytics
tableName: orders
version: 1
```

---

## DIMENSIONS REFERENCE

Dimensions are attributes used to group, filter, and segment data.

### Basic Structure

```yaml
columns:
  - name: customer_id
    description: "Unique customer identifier"
    meta:
      dimension:
        type: string
        label: "Customer ID"
        description: "The unique identifier for each customer"
        hidden: false
```

### Dimension Types

| Type | Description | Common SQL Types |
|------|-------------|------------------|
| `string` | Text values | VARCHAR, TEXT, CHAR |
| `number` | Numeric values | INT, FLOAT, DECIMAL |
| `boolean` | True/false values | BOOLEAN, INT (0/1) |
| `date` | Date only (no time) | DATE |
| `timestamp` | Date and time | TIMESTAMP, DATETIME |

### Configuration Keys

| Path | Type | Description |
|------|------|-------------|
| `meta.dimension.type` | string | Required: `string`, `number`, `boolean`, `date`, `timestamp` |
| `meta.dimension.label` | string | Human-readable display name |
| `meta.dimension.description` | string | Detailed documentation |
| `meta.dimension.hidden` | boolean | Hide from UI (default: `false`) |
| `meta.dimension.sql` | string | Custom SQL expression (overrides column reference) |
| `meta.dimension.round` | number | Decimal places (numbers only) |
| `meta.dimension.format` | string | Format preset: `usd`, `gbp`, `eur`, `percent`, `km`, `mi`, `id` |
| `meta.dimension.compact` | string | Compact display: `thousands`, `millions`, `billions`, `trillions`, `kilobytes`, `megabytes`, `gigabytes`, `terabytes`, `petabytes`, `kibibytes`, `mebibytes`, `gibibytes`, `tebibytes`, `pebibytes` |
| `meta.dimension.time_intervals` | array or string | Time intervals: `RAW`, `YEAR`, `QUARTER`, `MONTH`, `WEEK`, `DAY`, `HOUR`, `MINUTE`, `SECOND`, `MILLISECOND`, `YEAR_NUM`, `QUARTER_NUM`, `MONTH_NUM`, `WEEK_NUM`, `DAY_OF_YEAR_NUM`, `DAY_OF_MONTH_NUM`, `DAY_OF_WEEK_INDEX`, `HOUR_OF_DAY_NUM`, `MINUTE_OF_HOUR_NUM`, `QUARTER_NAME`, `MONTH_NAME`, `DAY_OF_WEEK_NAME`. Set to `"OFF"` or `false` to disable. |
| `meta.dimension.groups` | array | Organizational groups (hierarchical: `["Group", "Subgroup"]`) |
| `meta.dimension.colors` | object | Value → hex color mapping |
| `meta.dimension.urls` | array | Array of clickable links |
| `meta.dimension.urls[].label` | string | Link label |
| `meta.dimension.urls[].url` | string | URL with `${value}` and `${row.field}` variables |
| `meta.dimension.required_attributes` | object | Access control (role/department restrictions) |
| `meta.dimension.ai_hint` | string or array | Guide for AI query generation |
| `meta.dimension.tags` | array | Categorization tags |
| `meta.additional_dimensions` | object | Derived dimensions from base column |

### Time Intervals

For `timestamp` and `date` types, generate auto-grouping options:

```yaml
meta:
  dimension:
    type: timestamp
    time_intervals:
      - RAW
      - YEAR
      - QUARTER
      - MONTH
      - WEEK
      - DAY
      - HOUR
      - MINUTE
```

Disable time intervals:

```yaml
meta:
  dimension:
    type: timestamp
    time_intervals: "OFF"
```

### Complete Example

```yaml
columns:
  - name: created_at
    description: Account creation timestamp
    meta:
      dimension:
        type: timestamp
        label: Account Created
        description: When the customer first signed up
        time_intervals:
          - DAY
          - WEEK
          - MONTH
          - QUARTER
          - YEAR
        groups:
          - Dates

  - name: status
    description: Account status
    meta:
      dimension:
        type: string
        label: Account Status
        colors:
          Active: "#22c55e"
          Inactive: "#94a3b8"
          Churned: "#ef4444"
        groups:
          - Account Info

  - name: revenue
    description: Total customer spend
    meta:
      dimension:
        type: number
        label: Revenue
        format: usd
        round: 2
        compact: thousands
        urls:
          - label: View Details
            url: "/customers/${row.customer_id}/revenue"
```

---

## METRICS REFERENCE

Metrics are aggregated calculations on data.

### Basic Structure

```yaml
columns:
  - name: order_amount
    meta:
      metrics:
        total_amount:
          type: sum
          label: "Total Amount"
          format: "usd"
          round: 2
```

### Metric Types

| Type | SQL Aggregation | Use Case |
|------|-----------------|----------|
| `count` | `COUNT(*)` | Row count |
| `count_distinct` | `COUNT(DISTINCT column)` | Unique values |
| `sum` | `SUM(column)` | Total |
| `sum_distinct` | `SUM(DISTINCT on keys)` | Total with deduplication |
| `average` | `AVG(column)` | Mean |
| `average_distinct` | `AVG(DISTINCT on keys)` | Mean with deduplication |
| `min` | `MIN(column)` | Minimum |
| `max` | `MAX(column)` | Maximum |
| `median` | Percentile(50) | Middle value |
| `percentile` | Percentile(N) | Custom percentile |
| `number` | Custom SQL | Custom numeric result |
| `string` | Custom SQL | Custom string result |
| `date` | Custom SQL | Custom date result |
| `timestamp` | Custom SQL | Custom timestamp result |
| `boolean` | Custom SQL | Custom boolean result |

### Configuration Keys

| Path | Type | Description |
|------|------|-------------|
| `type` | string | Required: metric type (see above) |
| `label` | string | Human-readable display name |
| `description` | string | Detailed documentation |
| `hidden` | boolean | Hide from UI (default: `false`) |
| `format` | string | Format: `usd`, `gbp`, `eur`, `percent`, `km`, `mi`, `id` |
| `round` | number | Decimal places |
| `compact` | string | Compact display: `thousands`, `millions`, `billions`, `trillions`, `kilobytes`, etc. |
| `percentile` | number | For `percentile` type only (0-100) |
| `distinct_keys` | string or array | For `sum_distinct`/`average_distinct`: dimension(s) to deduplicate by |
| `sql` | string | For custom SQL metrics and model-level metrics |
| `filters` | array | Metric-level filters (status: "completed", amount: "> 1000", created_at: "inThePast 30 days") |
| `show_underlying_values` | array | Field IDs for drill-down |
| `groups` | array | Organizational groups (hierarchical) |
| `urls` | array | Array of clickable links |
| `required_attributes` | object | Access control |
| `ai_hint` | string or array | AI guidance |
| `tags` | array | Categorization tags |
| `default_time_dimension` | object | Associated time dimension |
| `default_time_dimension.field` | string | Time dimension field |
| `default_time_dimension.interval` | string | Interval: `DAY`, `WEEK`, `MONTH`, `QUARTER`, `YEAR` |

### Metric Filter Operators

| Operator | Values Example | Notes |
|----------|----------------|-------|
| (value) | `"completed"` | Equality (string) |
| (value) | `"1000"` | Equality (number) |
| `"> X"` | `"> 1000"` | Greater than |
| `"< X"` | `"< 50"` | Less than |
| `">= X"` | `">= 10"` | Greater than or equal |
| `"<= X"` | `"<= 100"` | Less than or equal |
| (array) | `["value1", "value2"]` | Multiple values (OR) |
| `"null"` | (no value) | Is null |
| `"!null"` | (no value) | Is not null |
| `"inThePast N days"` | `"inThePast 30 days"` | Last N periods |
| `"inTheNext N days"` | `"inTheNext 7 days"` | Next N periods |

**NOT supported in metric filters:** `inTheCurrent` (only works in chart/dashboard filters)

### Complete Examples

**Column-Level Metrics (E-commerce):**

```yaml
columns:
  - name: order_amount
    meta:
      metrics:
        total_revenue:
          type: sum
          label: Total Revenue
          description: Sum of all order amounts
          format: usd
          round: 2
          show_underlying_values:
            - order_id
            - customer_name
            - amount
          groups:
            - Revenue

        average_order_value:
          type: average
          label: Average Order Value
          format: usd
          round: 2
          groups:
            - Revenue

  - name: order_id
    meta:
      metrics:
        order_count:
          type: count
          label: Total Orders
          groups:
            - Volume

        unique_customers:
          type: count_distinct
          sql: "${TABLE}.customer_id"
          label: Unique Customers
          groups:
            - Volume
```

**Model-Level Metrics (SaaS):**

```yaml
models:
  - name: subscriptions
    meta:
      metrics:
        mrr:
          type: sum
          sql: "${TABLE}.monthly_amount"
          label: MRR
          format: usd
          compact: thousands

        arr:
          type: number
          sql: "SUM(${TABLE}.monthly_amount) * 12"
          label: ARR
          format: usd
          compact: millions

        churn_rate:
          type: number
          sql: |
            COUNT(CASE WHEN ${TABLE}.status = 'churned' THEN 1 END)::float
            / NULLIF(COUNT(*), 0) * 100
          label: Churn Rate
          round: 2
```

**Sum Distinct with Keys:**

```yaml
columns:
  - name: order_total
    meta:
      metrics:
        revenue:
          type: sum_distinct
          distinct_keys: order_id
          label: Revenue (Order Deduplicated)

        # Multiple keys (composite):
        revenue_by_currency:
          type: sum_distinct
          distinct_keys:
            - order_id
            - currency_code
```

---

## TABLES & JOINS REFERENCE

Tables (Explores) define queryable entities and their relationships.

### Model-Level Configuration

```yaml
version: 2

models:
  - name: orders
    description: All order transactions
    meta:
      label: Orders
      primary_key: order_id
      order_fields_by: label
      group_label: Sales
      sql_where: "is_deleted = false"
      default_time_dimension:
        field: created_at
        interval: DAY
      joins: []
      metrics: {}
```

### Configuration Keys

| Path | Type | Description |
|------|------|-------------|
| `meta.label` | string | Display name in UI |
| `meta.description` | string | Table documentation |
| `meta.hidden` | boolean | Hide from explore list (default: `false`) |
| `meta.sql_table` | string | Override table reference (e.g., "schema.table_name") |
| `meta.sql_where` or `meta.sql_filter` | string | Always-applied WHERE clause |
| `meta.order_fields_by` | string | Sort fields by: `"label"` or `"index"` |
| `meta.group_label` | string | Sidebar grouping for explores |
| `meta.primary_key` | string or array | Single or composite key |
| `meta.default_time_dimension` | object | Default time dimension for analysis |
| `meta.default_time_dimension.field` | string | Time dimension field |
| `meta.default_time_dimension.interval` | string | Interval: `DAY`, `WEEK`, `MONTH`, `QUARTER`, `YEAR` |
| `meta.default_show_underlying_values` | array | Default drill-down fields |
| `meta.required_filters` or `meta.default_filters` | array | Mandatory filters |
| `meta.required_attributes` | object | Access control |
| `meta.ai_hint` | string or array | AI guidance |
| `meta.spotlight` | object | AI search visibility |
| `meta.joins` | array | Join configurations |
| `meta.metrics` | object | Model-level metrics |
| `meta.explores` | object | Additional explores with filters |

### Joins Configuration

```yaml
meta:
  joins:
    - join: customers
      sql_on: "${orders.customer_id} = ${customers.customer_id}"
      type: left
      label: Customer
      relationship: many-to-one
      hidden: false
      alias: null
      fields:
        - customer_name
        - email
      always: false
      description: Customer who placed the order
```

### Join Configuration Keys

| Path | Type | Required | Description |
|------|------|----------|-------------|
| `join` | string | Yes | Model name to join |
| `sql_on` | string | Yes | Join condition (can be multi-line) |
| `type` | string | No | `left`, `inner`, `right`, `full` (default: `left`) |
| `label` | string | No | Display label |
| `alias` | string | No | Alias for self-joins |
| `relationship` | string | No | `one-to-one`, `one-to-many`, `many-to-one`, `many-to-many` |
| `hidden` | boolean | No | Hide from UI (default: `false`) |
| `fields` | array | No | Limit included fields |
| `always` | boolean | No | Always include in queries (default: `false`) |
| `description` | string | No | Join documentation |

### Complete Table Example

```yaml
version: 2

models:
  - name: orders
    description: All order transactions
    meta:
      label: Orders
      order_fields_by: label
      group_label: Sales
      primary_key: order_id
      default_time_dimension:
        field: created_at
        interval: DAY
      default_show_underlying_values:
        - order_id
        - customer_name
        - amount
        - status
      sql_where: "is_test = false AND is_deleted = false"

      joins:
        - join: customers
          sql_on: "${orders.customer_id} = ${customers.customer_id}"
          type: left
          label: Customer
          relationship: many-to-one

        - join: products
          sql_on: "${orders.product_id} = ${products.product_id}"
          type: left
          label: Product

      metrics:
        revenue_per_customer:
          type: number
          sql: "SUM(${TABLE}.amount) / NULLIF(COUNT(DISTINCT ${TABLE}.customer_id), 0)"
          label: Revenue per Customer
          format: usd
          round: 2

    columns:
      - name: order_id
        description: Unique order identifier
        meta:
          dimension:
            type: string
            label: Order ID
            hidden: true
          metrics:
            order_count:
              type: count
              label: Total Orders

      - name: customer_id
        meta:
          dimension:
            type: string
            label: Customer ID
            hidden: true

      - name: amount
        description: Order total in USD
        meta:
          dimension:
            type: number
            label: Order Amount
            format: usd
          metrics:
            total_revenue:
              type: sum
              label: Total Revenue
              format: usd

      - name: created_at
        meta:
          dimension:
            type: timestamp
            label: Order Date
            time_intervals:
              - DAY
              - WEEK
              - MONTH
              - QUARTER
              - YEAR
```

---

## DASHBOARD REFERENCE

Dashboards combine charts and content tiles in a grid layout.

### Base Structure

```yaml
contentType: dashboard
config:
  isAddFilterDisabled: false
  isDateZoomDisabled: false
description: Dashboard description
filters:
  dimensions: []
name: Dashboard Name
slug: dashboard-slug
spaceSlug: space-name
tabs: []
tiles: []
verified: true  # Optional: only admins
version: 1
```

### Configuration Keys

| Path | Type | Description |
|------|------|-------------|
| `contentType` | string | Required: `"dashboard"` |
| `name` | string | Dashboard name |
| `slug` | string | URL-safe slug |
| `description` | string | Dashboard documentation |
| `spaceSlug` | string | Space for dashboard |
| `verified` | boolean | Optional: admin-only verification flag |
| `config.isDateZoomDisabled` | boolean | Disable date zoom feature |
| `config.isAddFilterDisabled` | boolean | Disable add filter button |
| `config.dateZoomGranularities` | array | Available date zoom levels |
| `config.defaultDateZoomGranularity` | string | Default zoom level |
| `config.pinnedParameters` | array | Parameter names to pin |
| `tabs[].name` | string | Tab name |
| `tabs[].order` | number | Tab order (0-indexed) |
| `tabs[].uuid` | string | Tab UUID (must be valid UUID) |
| `filters.dimensions` | array | Dashboard-level filters |

### Grid Layout (36-Column System)

**Important:** The grid uses 36 columns. Use `w: 36` for full width.

| Layout | Width (w) | Tiles per row |
|--------|-----------|---------------|
| Full width | 36 | 1 |
| Half width | 18 | 2 |
| Third width | 12 | 3 |
| Quarter width | 9 | 4 |
| Sixth width | 6 | 6 |

### Tile Types

**Saved Chart Tile:**

```yaml
tiles:
  - h: 6
    properties:
      chartSlug: chart-slug
      hideTitle: false
      title: "Chart Title"  # Independent of chart name — update manually
    type: saved_chart
    w: 36
    x: 0
    y: 0
```

**Markdown Tile:**

```yaml
tiles:
  - h: 4
    properties:
      content: |
        ## Markdown Content
        - Bullet points
        - Links: [text](url)
      title: "Notes"
    type: markdown
    w: 12
    x: 0
    y: 0
```

**Heading Tile:**

```yaml
tiles:
  - h: 1
    properties:
      text: "Section Title"
    type: heading
    w: 36
    x: 0
    y: 0
```

**Loom Video Tile:**

```yaml
tiles:
  - h: 6
    properties:
      title: "Video"
      url: "https://www.loom.com/share/abc123"
    type: loom
    w: 12
    x: 24
    y: 0
```

**SQL Chart Tile:**

```yaml
tiles:
  - h: 6
    properties:
      chartSlug: sql-chart-slug
      savedSqlUuid: "uuid"
    type: sql_chart
    w: 12
    x: 12
    y: 0
```

### Dashboard Filters

```yaml
filters:
  dimensions:
    - disabled: false
      label: "Date Range"
      operator: inThePast
      required: false
      singleValue: false
      settings:
        completed: true
        unitOfTime: months
      target:
        fieldId: orders_created_at_month
        tableName: orders
      tileTargets: {}  # Map filter to different fields per tile
      values: [12]
```

### Filter Operators

| Operator | Values | Notes |
|----------|--------|-------|
| `equals` | `["value"]` | Exact match |
| `notEquals` | `["value"]` | Not equal |
| `isNull` | `[]` | Is null |
| `notNull` | `[]` | Is not null |
| `startsWith` | `["prefix"]` | String prefix |
| `endsWith` | `["suffix"]` | String suffix |
| `include` | `["substring"]` | Contains |
| `doesNotInclude` | `["substring"]` | Does not contain |
| `lessThan` | `[100]` | Less than number |
| `lessThanOrEqual` | `[100]` | Less than or equal |
| `greaterThan` | `[0]` | Greater than |
| `greaterThanOrEqual` | `[100]` | Greater than or equal |
| `inThePast` | `[30]` | Last N periods (requires `settings.unitOfTime`) |
| `notInThePast` | `[30]` | Not in last N |
| `inTheNext` | `[7]` | Next N periods |
| `inTheCurrent` | `[1]` | Current period |
| `notInTheCurrent` | `[1]` | Not in current period |
| `inBetween` | `["2024-01-01", "2024-12-31"]` | Date range |
| `notInBetween` | `[0, 100]` | Outside range |

### Cross-Explore Filters (tileTargets)

When dashboard tiles use different explores, map filters to different fields:

```yaml
filters:
  dimensions:
    - label: "Date Range"
      operator: inThePast
      settings:
        completed: false
        unitOfTime: days
      target:
        fieldId: orders_created_at    # Default for orders tiles
        tableName: orders
      tileTargets:
        customer-metrics:             # Override for customers tiles
          fieldId: customers_signup_date
          tableName: customers
        revenue-summary:              # Explicit match for orders
          fieldId: orders_created_at
          tableName: orders
        company-overview: false       # Exclude this tile
      values: [30]
```

### Complete Dashboard Example

```yaml
contentType: dashboard
config:
  dateZoomGranularities:
    - Day
    - Week
    - Month
    - Quarter
    - Year
  defaultDateZoomGranularity: Month
  isAddFilterDisabled: false
  isDateZoomDisabled: false
description: Executive sales dashboard
filters:
  dimensions:
    - label: Time Period
      operator: inThePast
      settings:
        completed: false
        unitOfTime: months
      target:
        fieldId: orders_created_at
        tableName: orders
      tileTargets: {}
      values: [12]

    - label: Region
      operator: equals
      target:
        fieldId: orders_region
        tableName: orders
      tileTargets: {}
      values: []

name: Executive Sales Dashboard
slug: executive-sales-dashboard
spaceSlug: leadership
tabs:
  - name: Overview
    order: 0
    uuid: e6f4d5a7-b8c9-4201-def0-345678901234
  - name: By Region
    order: 1
    uuid: f7a5e6b8-c9d0-4312-ef01-456789012345

tiles:
  # KPI Row
  - h: 3
    properties:
      chartSlug: total-revenue-kpi
      title: Total Revenue
    tabUuid: e6f4d5a7-b8c9-4201-def0-345678901234
    type: saved_chart
    w: 9
    x: 0
    y: 0

  - h: 3
    properties:
      chartSlug: total-orders-kpi
      title: Total Orders
    tabUuid: e6f4d5a7-b8c9-4201-def0-345678901234
    type: saved_chart
    w: 9
    x: 9
    y: 0

  - h: 3
    properties:
      chartSlug: new-customers-kpi
      title: New Customers
    tabUuid: e6f4d5a7-b8c9-4201-def0-345678901234
    type: saved_chart
    w: 9
    x: 18
    y: 0

  - h: 3
    properties:
      chartSlug: avg-order-value-kpi
      title: Avg Order Value
    tabUuid: e6f4d5a7-b8c9-4201-def0-345678901234
    type: saved_chart
    w: 9
    x: 27
    y: 0

  # Main Charts
  - h: 8
    properties:
      chartSlug: revenue-trend
      title: Revenue Trend
    tabUuid: e6f4d5a7-b8c9-4201-def0-345678901234
    type: saved_chart
    w: 24
    x: 0
    y: 3

  - h: 8
    properties:
      content: |
        ## Key Insights

        - Revenue tracking **+12%** vs target
        - Strong growth in Enterprise
        - APAC on track
      title: Key Insights
    tabUuid: e6f4d5a7-b8c9-4201-def0-345678901234
    type: markdown
    w: 12
    x: 24
    y: 3

  # Regional Tab
  - h: 1
    properties:
      text: Regional Performance
    tabUuid: f7a5e6b8-c9d0-4312-ef01-456789012345
    type: heading
    w: 36
    x: 0
    y: 0

  - h: 8
    properties:
      chartSlug: revenue-by-region
    tabUuid: f7a5e6b8-c9d0-4312-ef01-456789012345
    type: saved_chart
    w: 18
    x: 0
    y: 1

  - h: 8
    properties:
      chartSlug: regional-trend
    tabUuid: f7a5e6b8-c9d0-4312-ef01-456789012345
    type: saved_chart
    w: 18
    x: 18
    y: 1

version: 1
```

---

## CLI REFERENCE

### Authentication

```bash
# Login to Lightdash
lightdash login https://app.lightdash.cloud

# Login with API token
lightdash login https://app.lightdash.cloud --token YOUR_API_TOKEN

# List available projects
lightdash config list-projects

# Show current project
lightdash config get-project

# Set active project
lightdash config set-project --name "Project Name"
lightdash config set-project --uuid project-uuid
```

### Compilation & Deployment

```bash
# Compile dbt models
lightdash compile --project-dir ./dbt --profiles-dir ./profiles

# Deploy semantic layer (metrics, dimensions, tables)
lightdash deploy --project-dir ./dbt --profiles-dir ./profiles

# Deploy and create new project
lightdash deploy --create "New Project Name"

# Deploy ignoring validation errors
lightdash deploy --ignore-errors

# Skip dbt compilation (use existing manifest)
lightdash deploy --skip-dbt-compile

# Pure Lightdash (no dbt)
lightdash deploy --no-warehouse-credentials
```

### Charts & Dashboards

```bash
# Download all content
lightdash download

# Download specific charts
lightdash download --charts chart-slug-1 chart-slug-2

# Download dashboards
lightdash download --dashboards dashboard-slug

# Download with nested folders
lightdash download --nested

# Upload content
lightdash upload

# Force upload (ignore timestamps)
lightdash upload --force

# Upload specific items
lightdash upload --charts my-chart --dashboards my-dashboard

# Upload dashboard with referenced charts
lightdash upload --dashboards my-dashboard --include-charts

# Delete charts/dashboards
lightdash delete -c chart-slug
lightdash delete -d dashboard-slug
lightdash delete -c chart1 chart2 -d dashboard1 --force
```

### Preview Environments

```bash
# Create preview (watches for changes)
lightdash preview --name "feature-preview"

# Start preview without watching
lightdash start-preview --name "my-preview"

# Stop preview
lightdash stop-preview --name "my-preview"
```

### SQL Execution

```bash
# Run SQL query
lightdash sql "SELECT * FROM orders LIMIT 10" -o results.csv

# With row limit
lightdash sql "SELECT * FROM customers" -o customers.csv --limit 1000

# With custom page size
lightdash sql "SELECT * FROM events" -o events.csv --page-size 2000

# Verbose output
lightdash sql "SELECT COUNT(*) FROM users" -o count.csv --verbose
```

### Chart Execution

```bash
# Run chart YAML query
lightdash run-chart -p ./lightdash/charts/monthly-revenue.yml

# Save results to CSV
lightdash run-chart -p chart.yml -o results.csv

# Limit rows
lightdash run-chart -p chart.yml -o results.csv -l 100

# Custom page size
lightdash run-chart -p chart.yml -o results.csv --page-size 2000
```

### Validation & Linting

```bash
# Lint YAML locally
lightdash lint --path ./lightdash

# Validate against server
lightdash validate --project project-uuid

# Output as JSON
lightdash lint --path ./lightdash --format json
```

### Warehouse Connection

```bash
# Update warehouse connection from profiles.yml
lightdash set-warehouse --project-dir ./dbt --profiles-dir ./profiles

# With target override
lightdash set-warehouse --project-dir ./dbt --profiles-dir ./profiles --target prod

# Non-interactive
lightdash set-warehouse --project-dir ./dbt --profiles-dir ./profiles --assume-yes

# Specific project
lightdash set-warehouse --project-dir ./dbt --profiles-dir ./profiles --project project-uuid
```

### Code Generation

```bash
# Generate YAML from dbt models
lightdash generate

# For specific models
lightdash generate -s my_model

# With tags
lightdash generate -s tag:sales

# Include parents
lightdash generate -s +my_model
```

---

## WORKFLOWS & BEST PRACTICES

### Recommended End-to-End Loop

**dbt-first philosophy:**

1. **Define tables** in `models/*.yml` with joins and primary keys
2. **Define dimensions** on columns with types and time intervals
3. **Define metrics** on columns or at model level
4. **Deploy semantic layer:** `lightdash deploy`
5. **Create charts** in UI or as code
6. **Download charts:** `lightdash download --charts chart-slug`
7. **Manage as code:** Edit YAML, validate, upload
8. **Build dashboards:** Combine charts, add filters and markdown
9. **Deploy to production:** `lightdash upload --dashboards dashboard-slug`

### File Structure & Naming

**dbt project:**
```
dbt/
├── dbt_project.yml
├── models/
│   ├── staging/
│   ├── marts/
│   └── schema.yml              # Metrics, dimensions, joins
└── profiles.yml

lightdash/
├── charts/
│   ├── sales/
│   │   ├── monthly-revenue.yml
│   │   └── sales-by-category.yml
│   └── marketing/
│       └── campaign-performance.yml
└── dashboards/
    ├── sales/
    │   └── executive-summary.yml
    └── marketing/
        └── campaign-overview.yml
```

**Pure Lightdash project:**
```
lightdash/
├── lightdash.config.yml
├── models/
│   ├── orders.yml
│   └── customers.yml
├── charts/
└── dashboards/
```

### Period-over-Period (PoP) Comparisons

**Add comparison metrics to a chart:**

1. Define additional metric with `generationType: periodOverPeriod`
2. Include in `metricQuery.additionalMetrics`
3. Add to `metricQuery.metrics`
4. Use in chart layout (cartesian, table, etc.)

**Example: Year-over-Year:**

```yaml
metricQuery:
  additionalMetrics:
    - baseMetricId: orders_total_revenue
      generationType: periodOverPeriod
      granularity: YEAR
      hidden: true
      label: Revenue (Previous Year)
      name: total_revenue__pop__year_1__hash
      periodOffset: 1
      sql: "${TABLE}.amount"
      table: orders
      timeDimensionId: orders_order_date_year
      type: sum

  metrics:
    - orders_total_revenue
    - orders_total_revenue__pop__year_1__hash

  dimensions:
    - orders_order_date_year
```

**Granularities:** `DAY`, `WEEK`, `MONTH`, `QUARTER`, `YEAR`

### Dashboard Design Best Practices

1. **KPIs at top** — Critical metrics first
2. **5-10 charts per view/tab** — Avoid cognitive overload
3. **Full-width charts** — Use `w: 36` on 36-column grid
4. **Consistent colors** — Same metric = same color across dashboard
5. **Use tabs** — Split large dashboards
6. **Add headings** — Organize sections (type: `heading`)
7. **Use markdown** — Explain context and insights
8. **Filter defaults** — Set sensible initial values
9. **Cross-explore filters** — Use `tileTargets` when tiles use different explores
10. **Test on mobile** — Verify responsive layout

### CI/CD Deployment Examples

**GitHub Actions:**

```yaml
name: Deploy Lightdash
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    env:
      LIGHTDASH_API_KEY: ${{ secrets.LIGHTDASH_API_KEY }}
      LIGHTDASH_URL: https://app.lightdash.cloud
      LIGHTDASH_PROJECT: ${{ secrets.PROJECT_UUID }}
      CI: "true"
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm install -g @lightdash/cli
      - run: pip install dbt-core dbt-postgres
      - run: lightdash deploy --target prod
      - run: lightdash upload --force
```

### Common Mistakes to Avoid

1. **Guessing filter values** — Always run `lightdash sql` to verify exact values
2. **Not updating dashboard tile titles** — Chart name and tile title are independent
3. **Unused dimensions in metricQuery** — Every dimension must be used in layout or pivot
4. **Unsorted YAML keys** — Always sort alphabetically (CLI requires this)
5. **Wrong grid widths** — Use `w: 36` for full-width, not 24 or 30
6. **Too many tiles** — 8-10 per view maximum
7. **Missing time dimension defaults** — Set sensible defaults for better UX
8. **Not filtering test data** — Use `sql_where` to exclude test records

### Warehouse Credential Management

```bash
# Update connection from profiles.yml
lightdash set-warehouse \
  --project-dir ./dbt \
  --profiles-dir ./profiles \
  --assume-yes

# For specific project
lightdash set-warehouse \
  --project-dir ./dbt \
  --profiles-dir ./profiles \
  --project project-uuid
```

### Key Lightdash Concepts

**Explores:** Queryable entities (tables + joins)
**Dimensions:** Attributes for grouping/filtering
**Metrics:** Aggregated calculations
**Joins:** Cross-table relationships
**Charts:** Saved visualizations
**Dashboards:** Collections of tiles (charts, markdown, etc.)
**Filters:** Dashboard/chart-level filtering
**Space:** Container for organizing charts and dashboards

---

## QUICK REFERENCE TABLES

### Chart Type → Use Case Matrix

| Chart | Trends | Comparisons | Part-of-Whole | Detail | Correlation | Status | Progress |
|-------|--------|-------------|---------------|--------|-------------|--------|----------|
| **Line/Area** | ✓ | ✓ |  |  | ✓ |  |  |
| **Bar** |  | ✓ | ✓ | ✓ |  | ✓ |  |
| **Pie/Donut** |  |  | ✓ |  |  |  |  |
| **Table** |  |  |  | ✓ |  | ✓ |  |
| **Funnel** |  |  |  |  |  | ✓ |  |
| **Gauge** |  |  |  |  |  | ✓ | ✓ |
| **Treemap** |  |  | ✓ |  |  |  |  |
| **Map** |  | ✓ |  |  |  |  |  |
| **Big Number** |  |  |  |  |  | ✓ | ✓ |

### Metric Type → SQL Translation

| Metric Type | SQL |
|-------------|-----|
| `count` | `COUNT(*)` |
| `count_distinct` | `COUNT(DISTINCT column)` |
| `sum` | `SUM(column)` |
| `average` | `AVG(column)` |
| `min` | `MIN(column)` |
| `max` | `MAX(column)` |
| `median` | Percentile(50) |
| `percentile` | Percentile(N) |
| `sum_distinct` | `SUM(DISTINCT on keys) / COUNT(DISTINCT keys)` |
| `average_distinct` | `AVG(DISTINCT on keys)` |

### Format Presets

| Format | Example | Use For |
|--------|---------|---------|
| `usd` | $1,234.56 | US Dollar amounts |
| `gbp` | £1,234.56 | British Pound |
| `eur` | €1,234.56 | Euro |
| `percent` | 45.2% | Percentages |
| `km` | 123.4 km | Kilometers |
| `mi` | 76.6 mi | Miles |
| `id` | 12345 | IDs (no formatting) |
| `thousands` | 1.2K | Compact thousands |
| `millions` | 1.2M | Compact millions |
| `billions` | 1.2B | Compact billions |

### Dimension Time Intervals

**Auto-generate bins:**

```
RAW, YEAR, QUARTER, MONTH, WEEK, DAY, HOUR, MINUTE, SECOND, MILLISECOND,
YEAR_NUM, QUARTER_NUM, MONTH_NUM, WEEK_NUM, DAY_OF_YEAR_NUM, 
DAY_OF_MONTH_NUM, DAY_OF_WEEK_INDEX, HOUR_OF_DAY_NUM, MINUTE_OF_HOUR_NUM,
QUARTER_NAME, MONTH_NAME, DAY_OF_WEEK_NAME
```

---

**End of Cheatsheet**

This comprehensive cheatsheet provides every YAML/JSON key, enum value, configuration pattern, and best practice needed to generate Lightdash content from another BI tool's exports. All examples are syntactically complete and ready to adapt to your data model.

# DASHBOARD TRANSLATION — TABLEAU ZONES → LIGHTDASH GRID

Tableau dashboards use **pixel-positioned zones** (free layout) or **flex containers** (`layout-basic horizontal/vertical`). Lightdash dashboards use a **36-column responsive grid** with row heights in units (`x`, `y`, `w`, `h`).

This file is the algorithm for converting one to the other.

---

## 1. Lightdash grid quick math

- Grid is **36 columns** wide. `w: 36` = full width.
- Row height unit ≈ ~50px (varies). `h: 6` ≈ a typical chart, `h: 3` for KPI tiles, `h: 1` for headings.
- Tile coords (`x`, `y`) are 0-indexed integers.
- Tiles can overlap technically, but don't — Lightdash layouts are dense and reflow on small screens.

Common widths:

| Tiles per row | `w` |
|---|---|
| 1 | 36 |
| 2 | 18 |
| 3 | 12 |
| 4 | 9 |
| 6 | 6 |

---

## 2. Mapping zone types

| Tableau `zone type-v2` | Lightdash tile `type` |
|---|---|
| `worksheet` | `saved_chart` (`properties.chartSlug`) |
| `text` | `markdown` or `heading` |
| `image` | `markdown` with `![]()` (image must be hosted externally) |
| `web` | none — drop, log, document. |
| `filter` | promote to dashboard `filters.dimensions[]`; do not emit a tile |
| `parameter` | promote to dashboard parameter or filter |
| `legend` | drop — each chart shows its own legend |
| `blank` | drop — leave grid empty |
| `layout-basic horizontal` | the children become tiles arranged side-by-side |
| `layout-basic vertical` | children stacked |

---

## 3. Algorithm: pixel zones → grid tiles

When `<zones is-pixels="true">` and zones have absolute `x`,`y`,`w`,`h` in pixels:

1. Compute dashboard pixel width `W` from `<size minwidth="…">` (fallback 1280).
2. For each zone, compute fractional bounds:
   - `xFrac = zone.x / W`
   - `wFrac = zone.w / W`
3. Map to grid cols:
   - `xGrid = round(xFrac * 36)`
   - `wGrid = max(1, round(wFrac * 36))`
   - Clamp `xGrid + wGrid ≤ 36`.
4. Pick a row unit `R` (e.g. 50px). Compute:
   - `yGrid = round(zone.y / R)`
   - `hGrid = max(1, round(zone.h / R))`
5. Sort tiles by `yGrid`, then `xGrid`. Compact `y` values to 0-based row numbers (collapse vertical gaps).
6. Resolve overlaps: shift tiles right/down until non-overlapping.
7. Snap small tiles to common widths (9, 12, 18, 36) — Lightdash dashboards look better with consistent column counts.

For **flex container** zones (`layout-basic horizontal/vertical`), recurse into children: a horizontal container of N children with equal weights ⇒ each gets `w = floor(36 / N)` (last child takes the remainder).

Pseudocode:

```python
def convert_zone(zone, W=1280, R=50):
    if zone.type in ("filter", "parameter", "legend", "blank", "web"):
        return None  # promote or drop
    children = zone.findall("./zone")
    if "layout-basic" in zone.type:
        # arrange children
        if "horizontal" in zone.type:
            n = len(children); w = 36 // n
            return [convert_child(c, x=i*w, y=0, w=w, h=zone.h_units) for i,c in enumerate(children)]
        else:
            return [convert_child(c, x=0, y=i*c.h_units, w=36, h=c.h_units) for i,c in enumerate(children)]
    return single_tile(zone, W, R)
```

---

## 4. Worksheet zone → saved_chart tile

```xml
<zone id="1" x="0" y="0" w="500" h="400" type-v2="worksheet" name="Sales Chart"/>
```

→ assuming dashboard width 1000px, row unit 50:

```yaml
- type: saved_chart
  x: 0
  y: 0
  w: 18           # 500/1000 * 36 = 18
  h: 8            # 400/50 = 8
  properties:
    chartSlug: sales-chart       # the migrated chart's slug
    title: Sales Chart           # carry forward zone name; Lightdash tile titles are independent
    hideTitle: false
```

---

## 5. Text zone → markdown / heading

Tableau:

```xml
<zone type-v2="text">
  <formatted-text>
    <run bold="true" fontsize="14">Key Insights</run>
    <run> Q4 revenue +12%.</run>
  </formatted-text>
</zone>
```

Render `<run>` into markdown:
- `bold="true"` → `**…**`
- `italic="true"` → `*…*`
- font-size > 12 → wrap in `## ` heading
- A single short heading (no other text) → use `type: heading` with `text:`

```yaml
- type: markdown
  x: 24
  y: 3
  w: 12
  h: 4
  properties:
    title: Insights
    content: |
      ## Key Insights
      Q4 revenue **+12%**.
```

---

## 6. Filter zone → dashboard filter

Tableau:

```xml
<zone type-v2="filter" param="[Region]" name="Region"/>
```

Don't emit a tile. Add to dashboard YAML:

```yaml
filters:
  dimensions:
    - id: region-filter
      label: Region
      operator: equals
      target:
        fieldId: orders_region
        tableName: orders
      tileTargets: {}
      values: []
      disabled: false
      required: false
      singleValue: false
```

Look up the underlying field in the worksheet that the filter zone references (Tableau filters are tied to a field on a sheet). Map that field to its Lightdash `<table>_<column>` ID.

---

## 7. Parameter zone → dashboard parameter

```xml
<zone type-v2="parameter" param="[Selected Year]" name="Year"/>
<column name="[Selected Year]" param-domain-type="list" value="2024">
  <parameter-list>
    <value>2022</value><value>2023</value><value>2024</value>
  </parameter-list>
</column>
```

Lightdash parameters are dashboard-scoped key/value pairs that can be referenced via `${ld.parameters.<name>}` in metric SQL. Where the parameter is only used to filter the date dim, prefer to translate it into a date filter and skip the parameter object.

---

## 8. Filter / highlight actions

Tableau dashboard `<action>` with `type="filter"` linking source sheet → target sheet via a field becomes a dashboard-wide filter on that field with `tileTargets` constraining which tiles it applies to.

Highlight actions and select actions don't have a 1:1 in Lightdash — drop and document.

URL actions become `meta.dimension.urls[]` on the relevant dimension at the dbt level, not at the dashboard level.

---

## 9. Device layouts

Tableau `<devicelayout name="Phone">` defines a separate layout for small screens. Lightdash auto-stacks tiles on narrow viewports. **Drop** device-layout zones; do not try to mirror them.

---

## 10. End-to-end mini example

Tableau:

```xml
<dashboard name="Sales Overview">
  <size sizing-mode="automatic" minwidth="1200" minheight="800"/>
  <zones is-pixels="true">
    <zone id="header" x="0" y="0" w="1200" h="50" type-v2="text">
      <formatted-text><run fontsize="16" bold="true">Sales Overview</run></formatted-text>
    </zone>
    <zone id="kpi1" x="0" y="50" w="300" h="120" type-v2="worksheet" name="Total Revenue"/>
    <zone id="kpi2" x="300" y="50" w="300" h="120" type-v2="worksheet" name="Total Orders"/>
    <zone id="kpi3" x="600" y="50" w="300" h="120" type-v2="worksheet" name="AOV"/>
    <zone id="kpi4" x="900" y="50" w="300" h="120" type-v2="worksheet" name="New Customers"/>
    <zone id="trend" x="0" y="170" w="800" h="400" type-v2="worksheet" name="Revenue Trend"/>
    <zone id="region-filter" x="800" y="170" w="400" h="400" type-v2="filter" param="[Region]" name="Region"/>
  </zones>
</dashboard>
```

Lightdash:

```yaml
contentType: dashboard
name: Sales Overview
slug: sales-overview
spaceSlug: sales
description: Migrated from Tableau "Sales Overview"
config:
  isAddFilterDisabled: false
  isDateZoomDisabled: false
filters:
  dimensions:
    - id: region-filter
      label: Region
      operator: equals
      target: { fieldId: orders_region, tableName: orders }
      tileTargets: {}
      values: []
tiles:
  - type: heading
    x: 0
    y: 0
    w: 36
    h: 1
    properties:
      text: Sales Overview
  - type: saved_chart
    x: 0
    y: 1
    w: 9
    h: 3
    properties: { chartSlug: total-revenue, title: Total Revenue }
  - type: saved_chart
    x: 9
    y: 1
    w: 9
    h: 3
    properties: { chartSlug: total-orders, title: Total Orders }
  - type: saved_chart
    x: 18
    y: 1
    w: 9
    h: 3
    properties: { chartSlug: avg-order-value, title: AOV }
  - type: saved_chart
    x: 27
    y: 1
    w: 9
    h: 3
    properties: { chartSlug: new-customers, title: New Customers }
  - type: saved_chart
    x: 0
    y: 4
    w: 36
    h: 8
    properties: { chartSlug: revenue-trend, title: Revenue Trend }
version: 1
```

Notes:
- Region filter zone became a dashboard filter; the freed space was reabsorbed by widening the trend chart to full width.
- KPI strip became a quarter-width row (9 each).

# CONCEPT MAPPING — TABLEAU → LIGHTDASH

One-screen map. For deep details follow the linked references.

| Tableau concept | Lightdash concept | Notes |
|---|---|---|
| Workbook (`.twb`/`.twbx`) | A collection of dbt model YAML + chart YAML + dashboard YAML | One workbook → many files. |
| Datasource (`<datasource>`) | dbt model (`models/<name>.yml`) + warehouse table | One Tableau datasource ≈ one Lightdash explore. Inline blends → pre-join in dbt. |
| Connection (`<connection class="snowflake"/>`) | Lightdash warehouse + dbt `profiles.yml` | Map class → warehouse type. Hyper extracts must be re-pointed at the live warehouse. |
| Physical column (`<column>` no calc) | dbt model column with `meta.dimension` | Type maps: `string→string`, `integer/real→number`, `boolean→boolean`, `date→date`, `datetime→timestamp`. |
| Measure (`role="measure"`) | `meta.metrics.<name>` on the column | Aggregation maps directly: `sum→sum`, `average→average`, `count→count`, `count-d→count_distinct`, `min→min`, `max→max`, `median→median`. |
| Dimension (`role="dimension"`) | `meta.dimension` on the column | Set `time_intervals` for date/timestamp. |
| Calculated field (`<calculation class="tableau">`) | Custom dimension / metric / dbt SQL | See `calculation-translation.md`. Push complex logic to dbt. |
| Group (`<group>`) | `CASE WHEN` dimension in dbt or `meta.dimension.colors` for label-only groups | Group of values → CASE expression. |
| Bin (`<bin>`) | Custom dimension with `CASE WHEN` or `WIDTH_BUCKET` | Or compute as a dbt column. |
| Aliases (`<alias>`) | `meta.dimension.colors` keys (display only) **or** CASE WHEN dimension | Lightdash has no value-alias feature; rename in SQL. |
| Parameter (`param-domain-type`) | Lightdash dashboard filter, dashboard parameter, or chart-level filter | List → `equals` filter with predefined values; range → `inBetween` filter. |
| Worksheet (`<worksheet>`) | Chart YAML (`lightdash/charts/<slug>.yml`) | One sheet → one chart. |
| Mark class | `chartConfig.type` + series type | See `chart-type-mapping.md`. |
| Rows shelf | Y-axis: `chartConfig.config.layout.yField` (cartesian) or rows in table | Continuous measures vs discrete dims drive choice. |
| Cols shelf | X-axis: `chartConfig.config.layout.xField` | |
| Color encoding (dim) | Pivot → `pivotConfig.columns`; or per-series color | Discrete color split = pivot the dim. |
| Color encoding (measure) | `chartConfig.config.colorMetricId` (treemap/map), or conditional formatting (table) | Continuous color rarely 1:1. |
| Size encoding | Map: `sizeFieldId`. Treemap: `sizeMetricId`. Cartesian scatter: not directly. | |
| Shape encoding | Not natively supported — drop or substitute color. | |
| Detail encoding | Add to `metricQuery.dimensions` | Granularity-only field. |
| Label encoding | `showValue: true` on chart, or table column display. | |
| Tooltip override | Limited — map to `meta.dimension.urls` for click-throughs, otherwise drop. | |
| Filter `categorical` | `metricQuery.filters.dimensions` with `equals`/`isAnyOf` | |
| Filter `quantitative` (range) | Filter operator `inBetween` on metric or dimension | |
| Filter `relative-date` | `inThePast` / `inTheNext` / `inTheCurrent` operators | See `filter-translation.md`. |
| Context filter | Apply at chart level — Lightdash has no separate context layer; ordering doesn't matter for pure SQL filters. | Pre-filter in dbt if performance-critical. |
| Top-N filter | `metricQuery.limit` + sort | If "Top 10 by SUM(Sales)" → set sort + limit. |
| Set | Boolean dimension in dbt (`is_top_customer`) | |
| Hierarchy / drill | Manual drill — group dims via `meta.dimension.groups`. | |
| Table calc (`WINDOW_SUM`, `RUNNING_*`, `RANK`) | dbt window function → dimension/metric. Or `metricQuery.tableCalculations` for simple post-query SQL. | |
| LOD `{FIXED}` | Pre-compute in dbt as a separate model and join, or use `sum_distinct` with `distinct_keys`. | |
| LOD `{INCLUDE/EXCLUDE}` | Re-model in dbt; not directly expressible. | |
| Reference line | `chartConfig.config.eChartsConfig.series[].markLine` (cartesian) | Static values only. |
| Trend / forecast line | Pre-compute in dbt. Not native. | |
| Dashboard (`<dashboard>`) | Dashboard YAML (`lightdash/dashboards/<slug>.yml`) | |
| Dashboard zone (`type-v2="worksheet"`) | Tile `type: saved_chart` | See `dashboard-translation.md` for grid math. |
| Dashboard zone (`text`) | Tile `type: markdown` or `type: heading` | |
| Dashboard zone (`image`) | Tile `type: markdown` with `![alt](url)` (image must be hosted) | |
| Dashboard zone (`web`) | No native iframe tile — drop or document. | |
| Dashboard zone (`filter`) | Dashboard `filters.dimensions` entry | |
| Dashboard zone (`parameter`) | Dashboard parameter or filter | |
| Dashboard zone (`legend`) | Each chart shows its own legend; remove the dedicated legend zone. | |
| Dashboard zone (`blank`) | Skip — Lightdash uses grid empty-cells. | |
| Dashboard `layout-basic horizontal/vertical` container | Sibling tiles arranged via x/y/w/h on the 36-col grid. | |
| Filter action | Dashboard filter on the relevant dimension — Lightdash filters apply to all matching tiles. Use `tileTargets` to scope. | |
| Highlight action | Not native. Drop. | |
| URL action | `meta.dimension.urls` on the dimension | Click-through URL templates support `${value}` and `${row.field}`. |
| Parameter action | Dashboard parameter; mutate via filter UI. | |
| Story / Story Points | Out of scope — split into multiple dashboards or document. | |
| Device layouts (mobile) | Lightdash auto-stacks on small screens; manual mobile layouts not supported. | |

For the full Lightdash key reference see `lightdash-yaml-reference.md`. For Tableau XML element details see `tableau-twb-anatomy.md`.

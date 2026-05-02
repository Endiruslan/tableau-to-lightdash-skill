# MIGRATION WORKFLOW — TABLEAU → LIGHTDASH

End-to-end checklist. Walk top-to-bottom for each Tableau workbook.

---

## Phase 0 — Prereqs

- [ ] A target **Lightdash project** exists, connected to the warehouse.
- [ ] A **dbt project** is present and compiles (`dbt compile`).
- [ ] `lightdash` CLI installed (`npm i -g @lightdash/cli`) and authenticated (`lightdash login <url>`).
- [ ] Source workbook is a `.twb` (unzip `.twbx` first).

---

## Phase 1 — Discovery (read-only)

Build a catalog before generating anything.

1. **Unzip** the `.twbx` if needed; locate the `.twb`.
2. **Parse** the workbook (see `parsing-tableau.md`).
3. For every datasource, list:
   - connection class + server/database
   - all `<column>` (caption, name, role, type, datatype, aggregation, hidden flag, default format)
   - all calculated fields (formula, kind: physical/formula/lod/table_calc/rawsql/bin/group)
   - all parameters
4. For every worksheet, list:
   - name, mark class
   - rows / cols pills (resolved field references)
   - encodings (color/size/shape/text/detail/tooltip/label)
   - filters (class, field, values)
   - sorts and Top-N filters
   - referenced datasource(s)
5. For every dashboard, list:
   - size, sizing-mode
   - all zones (recursive), with type, geometry, target sheet (for worksheet zones), target field (for filter zones)
   - actions (filter / highlight / URL / parameter)

Write the catalog to `discovery.json`. The rest of the pipeline reads from it.

---

## Phase 2 — Data layer (dbt)

Goal: every Tableau datasource is backed by a dbt model in the target warehouse.

1. **Map connection classes** to your dbt warehouse. If the workbook uses a Hyper extract or local CSV, you must first land that data in the warehouse.
2. For each Tableau datasource, **create or identify a dbt model** with the same logical shape. If the Tableau datasource is just a single SQL table, use the existing dbt model that wraps it. If it's a multi-table blend, create a new dbt model that performs the join in SQL.
3. **Translate calculated fields**:
   - Physical/formula calcs → dbt columns (see `calculation-translation.md`).
   - Aggregate ratio metrics → leave them for `meta.metrics` with custom SQL.
   - LODs → separate dbt models (joined in YAML).
   - Table calcs → either dbt window columns or chart-local table calculations.
4. **Compile** dbt (`dbt compile`) to verify SQL.

---

## Phase 3 — Semantic layer (Lightdash YAML on dbt)

For each dbt model emit `models/<name>.yml` with:

```yaml
version: 2
models:
  - name: <model>
    meta:
      label: <Caption>
      primary_key: <pk>
      group_label: <area>
      default_time_dimension:
        field: <date col>
        interval: DAY
      joins: []                # promoted from Tableau blends
      metrics: {}              # ratio / model-level metrics
    columns:
      - name: <col>
        meta:
          dimension:
            type: <string|number|boolean|date|timestamp>
            label: <Caption>
            hidden: <bool>
            format: <usd|percent|...>
            time_intervals: [DAY, WEEK, MONTH, QUARTER, YEAR]   # date/timestamp only
          metrics:
            <metric_name>:
              type: <sum|count|count_distinct|...>
              label: <Caption>
              format: <...>
```

Validation:
- Run `lightdash compile --project-dir <dbt>` to catch syntax/refs.
- Run `lightdash deploy --project-dir <dbt>` to publish.

---

## Phase 4 — Charts

For each Tableau worksheet emit `lightdash/charts/<slug>.yml`.

1. Decide chart type: see `chart-type-mapping.md` decision tree.
2. Resolve field references on rows/cols/encodings to Lightdash field IDs (`<table>_<column>`). Date dims get an `_<interval>` suffix when bucketed (e.g. `orders_order_date_month`).
3. Fill `metricQuery`:
   - `dimensions` ← all dim refs from rows/cols + detail/color discrete encodings
   - `metrics` ← measure refs (after aggregation)
   - `filters.dimensions.and` ← from `<filter>` (see `filter-translation.md`)
   - `sorts` ← Tableau pill sort or Top-N field
   - `limit` ← Top-N units, else 500 default
   - `tableCalculations` ← any chart-local window functions
4. Fill `chartConfig` per type. Reference `lightdash-yaml-reference.md` for keys.
5. Set top-level metadata: `name`, `slug` (kebab-case worksheet name), `spaceSlug`, `tableName` (= dbt model), `version: 1`, `contentType: chart`.
6. **Sort YAML keys alphabetically** at every level (Lightdash CLI requires this).

Validate:
- `lightdash lint --path lightdash/charts/<slug>.yml`
- `lightdash run-chart -p lightdash/charts/<slug>.yml -o /tmp/preview.csv -l 5` (smoke test)

Upload:
- `lightdash upload --charts <slug>`

---

## Phase 5 — Dashboards

For each Tableau dashboard emit `lightdash/dashboards/<slug>.yml`.

1. Promote `filter` and `parameter` zones into top-level `filters.dimensions[]` (see `dashboard-translation.md`).
2. Convert remaining zones to tiles using the grid algorithm.
3. Resolve worksheet zones → `chartSlug` of the migrated chart.
4. For tabbed dashboards (rare in Tableau; usually achieved with multiple dashboards) optionally split into Lightdash tabs via `tabs[]`.
5. Translate filter actions into dashboard filters with `tileTargets`.
6. Drop unsupported features (highlight actions, web zones, device layouts) — log to report.

Validate:
- `lightdash lint --path lightdash/dashboards/<slug>.yml`

Upload:
- `lightdash upload --dashboards <slug> --include-charts`

---

## Phase 6 — Verification

For each migrated chart compare:

| Check | How |
|---|---|
| Same number of rows | `lightdash run-chart -p chart.yml -o lightdash.csv -l 0` vs Tableau "View Data" export |
| Same totals | sum the metric column in both CSVs |
| Same date range | inspect min/max of date dim |
| Same filter behavior | apply filters in both; counts should match |
| Visual parity | manual eyeball — Lightdash chart vs Tableau screenshot |

Document discrepancies in `MIGRATION_REPORT.md`.

---

## Phase 7 — Cleanup

- [ ] Remove temporary dbt models that turn out unused.
- [ ] Hide internal IDs (`hidden: true`) that Tableau showed but Lightdash users don't need.
- [ ] Add `description` to every model, dimension, metric (Tableau captions and `<desc>` blocks).
- [ ] Set `default_time_dimension` on each model with a primary date.
- [ ] Pin reusable charts to a Space and Project.
- [ ] Optional: schedule deliveries / alerts.

---

## Phase 8 — Handover

Deliver:
1. The dbt PR with the new/modified models.
2. The `lightdash/charts/` and `lightdash/dashboards/` PR.
3. `MIGRATION_REPORT.md` (see `unsupported-features.md` template).
4. A side-by-side comparison doc (Tableau screenshot vs Lightdash URL) for the top dashboards.

---

## Anti-patterns to avoid

- **Don't migrate by hand-typing YAML from screenshots.** Always parse the `.twb` XML — it's the source of truth.
- **Don't preserve Tableau worksheet titles verbatim** in the chart `name` if they include emoji/version suffixes ("Sales 📊 v2"). Clean to a Lightdash-friendly slug.
- **Don't blindly translate `Automatic` mark to `bar`.** Apply the resolution rules in `chart-type-mapping.md`.
- **Don't try to replicate dashboard pixel layouts cell-for-cell.** Lightdash grids look better when snapped to common widths (9, 12, 18, 36).
- **Don't migrate stale workbooks.** Confirm with the dashboard owner the workbook is still in use.
- **Don't forget `NULLIF` on ratio denominators.**
- **Don't sort YAML keys manually** — pipe through `yq -y -S` or use the `lightdash` CLI which auto-sorts on download.

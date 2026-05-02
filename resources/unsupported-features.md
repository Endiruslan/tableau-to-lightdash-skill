# UNSUPPORTED & PARTIALLY-SUPPORTED FEATURES

Features that don't survive a Tableau→Lightdash migration without manual rework. Always include this list in the migration report so the dashboard owner sees what was dropped.

---

## Hard-stop (will not be migrated)

| Tableau feature | Why | Recommendation |
|---|---|---|
| **Story / Story Points** | Narrative slideshow, no analogue | Split into separate dashboards or document. |
| **Pages shelf** | Frame-by-frame animation | Replace with a date filter or date-zoom on dashboard. |
| **Trend lines / Forecast / Reference distributions** | Inline statistical models tied to viz state | Pre-compute in dbt as a forecast / trend model, expose as a metric. |
| **Ad-hoc Clustering / Explain Data** | ML driven, opaque | Drop. |
| **Custom shapes / shape palettes** | Image assets bound to mark values | Use color encoding instead. |
| **`<devicelayout>` per-device layouts** | Distinct mobile layouts | Lightdash auto-stacks; drop. |
| **Web zone (`type-v2="web"`)** | iframe inside dashboard | Drop or move to external doc. |
| **Tableau Pulse/Slack subscriptions** | Tableau-server feature | Replace with Lightdash scheduled deliveries. |
| **Hyper extracts** | Snapshot of data | Re-point at the live warehouse via dbt. |

## Partial — needs design discussion

| Tableau feature | Lightdash partial path | Caveat |
|---|---|---|
| **LOD `{FIXED}`** | dbt aggregate model + join, or `sum_distinct` with `distinct_keys` | Doesn't fully preserve dim-context override semantics. |
| **LOD `{INCLUDE/EXCLUDE}`** | Re-model in dbt | No 1:1 syntax. |
| **Sets (combined / computed)** | Boolean dimension | Combination logic must be re-expressed as a CASE WHEN. |
| **Parameters used in formulas** | Lightdash dashboard parameters | Reference via `${ld.parameters.x}` in metric SQL. Limited UI vs Tableau. |
| **Hierarchies / drill-paths** | Manual drill via `meta.dimension.groups` | No auto-drill on click. |
| **Reference lines (static)** | `markLine` in cartesian chart config | Static value only; no per-pane formulas. |
| **Dual-axis with synced scales** | `cartesian` with two `yAxisIndex` | Scales independent unless you set explicit axis bounds. |
| **Filter actions (sheet→sheet)** | Dashboard-level filter with `tileTargets` | Doesn't preserve "click-to-filter on this mark"; user picks from filter UI. |
| **Highlight actions** | None | Drop. |
| **Tooltip viz-in-tooltip** | None | Drop the embedded viz; keep static tooltip text only. |
| **Custom number formats with conditional formatting in tooltips** | Limited | Use chart `format` + `conditionalFormattings`. |
| **RAWSQL / RAWSQLAGG** | Lift inner SQL into dbt or metric `sql:` | Re-test against the warehouse adapter. |
| **Aliases (value relabels)** | None first-class | Either CASE WHEN in SQL or `meta.dimension.colors` (label-only via the colors map keys, but doesn't change displayed value). |
| **Geographic auto-geocoding** | Pre-resolve in warehouse | Ship lat/lon columns. |

## Things worth migrating but easy to miss

- **Default formats** on `<column default-format="…"/>` → set Lightdash `format` (`usd`, `percent`, etc.).
- **Captions** on columns → `meta.dimension.label` / `meta.metrics.<name>.label`.
- **Hidden flag** (`hidden="true"`) → `meta.dimension.hidden: true` / metric `hidden: true`.
- **Field descriptions** (when present in `<desc>`) → `meta.dimension.description`.
- **Color encodings on dimensions** with explicit color palettes → `meta.dimension.colors: { value: "#hex", … }`.
- **Default sorts** on a dimension → `metricQuery.sorts` on each chart that uses it.

---

## Reporting template

For every workbook produce `MIGRATION_REPORT.md`:

```markdown
# Migration report — <Workbook Name>

## Migrated
- 12 worksheets → 12 charts
- 2 dashboards → 2 dashboards

## Manual review needed
- Worksheet "Forecast 2025": Tableau forecast disabled. **Action:** add forecast model in dbt.
- Worksheet "Customer Lifetime Value": uses `{FIXED [Customer]: SUM([Sales])}`.
  **Action:** added dbt model `customer_lifetime_value`. Verify results match.
- Dashboard "Exec Overview": filter action `Sales Chart → Detail Table` on `[Region]`.
  **Action:** translated to dashboard filter; verify tile scoping.

## Dropped
- Story "2024 Highlights" (5 slides)
- Custom shape palette on "Status" worksheet
- Phone device layout

## Files generated
- `lightdash/models/orders.yml` (1 model, 12 dimensions, 8 metrics)
- `lightdash/charts/sales-by-region.yml` …
- `lightdash/dashboards/exec-overview.yml`
```

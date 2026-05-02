---
name: migrating-tableau-to-lightdash
description: Migrate Tableau workbooks (.twb / .twbx) to Lightdash (dbt models, charts, dashboards). Use when the user asks to convert Tableau dashboards or worksheets to Lightdash, generate dbt YAML from Tableau datasources, or translate Tableau calculated fields, filters, parameters, or actions into Lightdash equivalents. Operates on the Tableau document schema 2026.1 (`twb_2026.1.0.xsd`) and the Lightdash content-as-code schemas (chart 1.0, dashboard 1.0, model 1.0).
---

# Migrating Tableau to Lightdash

You have been asked to migrate Tableau content (`.twb` / `.twbx` workbooks) into Lightdash content (dbt model YAML, chart YAML, dashboard YAML). This skill bundles the official Tableau document schema and the Lightdash content-as-code schemas plus a translation playbook.

**Stay in this skill the whole time.** Don't switch to "general dashboard help" — every step has Tableau-specific gotchas and Lightdash-specific shape requirements that the user is relying on you to handle.

---

## When to use

Trigger this skill when the user asks for any of:

- "Convert this Tableau workbook / dashboard / worksheet to Lightdash"
- "Generate dbt YAML from this Tableau datasource"
- "Translate this Tableau calculated field" (LOD, table calc, IF/CASE, ratio…)
- "Migrate the dashboard layout" (zones, filter actions, parameters)
- "Compare Tableau workbook vs the Lightdash project" / parity check
- "What can't be migrated from Tableau to Lightdash?"
- "Walk me through how to do a Tableau → Lightdash migration"

Don't use this skill for: pure Lightdash development with no Tableau source, generic BI advice, Tableau Server administration, or `.tflx`/`.twfl` flow files (Tableau Prep — out of scope).

---

## Inputs you should ask for (if not already provided)

1. The `.twb` file (or `.twbx` — you'll unzip it). If only screenshots are shared, refuse the migration and ask for the workbook XML — screenshot-driven migration is unreliable and will produce wrong YAML.
2. The target dbt project directory and warehouse adapter (Snowflake / BigQuery / Postgres / Redshift / Databricks / etc.).
3. The target Lightdash project / space slug.
4. The target dbt model name(s) the user wants Tableau datasources mapped to (or permission to invent them).

If 2–4 aren't known yet, propose sensible defaults and proceed; flag them in the migration report.

---

## Workflow (always follow this order)

1. **Discovery (read-only).** Parse the `.twb` and produce a catalog of every datasource, column, calc, parameter, worksheet (with mark/shelves/encodings/filters), and dashboard zone tree. Use `resources/parsing-tableau.md`.
2. **Plan.** Decide the dbt model split and chart-type mapping per worksheet **before generating YAML**. Surface unsupported features and get acknowledgement (`resources/unsupported-features.md`).
3. **Data layer.** Translate datasources → dbt models. Translate calculated fields per `resources/calculation-translation.md`. LODs and table calcs typically become new dbt models or window columns.
4. **Semantic layer.** Emit `models/*.yml` with `meta.dimension` and `meta.metrics`. Reference: `resources/lightdash-yaml-reference.md` ("Tables & Joins", "Dimensions", "Metrics" sections).
5. **Charts.** For every worksheet emit `lightdash/charts/<slug>.yml`. Map mark + shelves to chart type per `resources/chart-type-mapping.md`. Translate filters per `resources/filter-translation.md`.
6. **Dashboards.** For every dashboard emit `lightdash/dashboards/<slug>.yml`. Convert zones to grid tiles per `resources/dashboard-translation.md`.
7. **Validation.** `lightdash lint` + `lightdash run-chart` smoke tests + side-by-side row-count comparison vs Tableau "View Data". Document discrepancies.
8. **Report.** Always produce `MIGRATION_REPORT.md` summarizing what migrated, what needs review, what was dropped (template in `resources/unsupported-features.md`).

The full per-phase checklist is `resources/migration-workflow.md`.

---

## Key invariants (don't violate these)

- **Always parse the `.twb` XML.** Never reconstruct from screenshots, descriptions, or PDF exports — the XML is the source of truth and contains formulas, filter values, and field IDs you can't see in the UI.
- **Sort YAML keys alphabetically at every nesting level.** Lightdash CLI requires this and will warn on upload otherwise.
- **Use Lightdash field IDs, not Tableau field names.** A column `sales` in dbt model `orders` is referenced as `orders_sales` in Lightdash. A timestamp dimension with `time_intervals: [..., MONTH]` exposes additional IDs like `orders_order_date_month`.
- **Wrap denominators in `NULLIF(..., 0)`** for every ratio metric. Tableau silently shows null on divide-by-zero; warehouses raise.
- **Unzip `.twbx`** before parsing — it's a ZIP with the `.twb` inside.
- **Don't emit Lightdash YAML for unsupported Tableau features.** Drop, log, and put in the migration report. See `resources/unsupported-features.md`.
- **Federated datasources** (`federated.<hash>`) are virtual blends — resolve to the underlying datasource(s) and pre-join in dbt before emitting charts.
- **Tableau pill expressions** like `[federated.x].[sum:Sales:qk]` need parsing — see `resources/parsing-tableau.md` §5.
- **Tableau `mark class="Automatic"`** must be resolved to a concrete chart type using the shelf-composition rules in `resources/chart-type-mapping.md` §1.

---

## Resources index

Bundled in this skill — read them when the topic comes up.

| File | Read it when … |
|---|---|
| `resources/tableau-twb-anatomy.md` | You need to remember a Tableau XML element's structure or attribute enum. |
| `resources/lightdash-yaml-reference.md` | You need to remember a Lightdash YAML/JSON key, its enum, or a chart-type config. |
| `resources/concept-mapping.md` | One-line "what does X become?" lookup. |
| `resources/chart-type-mapping.md` | Decide which Lightdash `chartConfig.type` matches a given mark + shelf composition. |
| `resources/calculation-translation.md` | Translate a Tableau formula (LOD, table calc, ratio, CASE) to dbt SQL or a Lightdash metric. |
| `resources/filter-translation.md` | Translate a Tableau `<filter>` to a Lightdash filter operator. |
| `resources/dashboard-translation.md` | Convert pixel-positioned Tableau zones to the Lightdash 36-column grid. |
| `resources/parsing-tableau.md` | Open a `.twb` / `.twbx`, walk the XML, parse pill refs and calc kinds. |
| `resources/unsupported-features.md` | Decide whether to drop a feature; template for the migration report. |
| `resources/migration-workflow.md` | End-to-end checklist; phase-by-phase. |
| `resources/examples/01-bar-chart.md` | Worked: Bar chart (mark `Bar`, dim/measure shelves). |
| `resources/examples/02-line-with-pivot.md` | Worked: Line chart with color-pivot dim and relative-date filter. |
| `resources/examples/03-kpi-bignum.md` | Worked: KPI / `big_number` with comparison. |
| `resources/examples/04-table-conditional-formatting.md` | Worked: Highlight table → Lightdash `table` with gradient. |
| `resources/examples/05-calculated-fields.md` | Worked: Profit ratio metric, FIXED LOD via dbt model + join, CASE bucket dimension. |
| `resources/examples/06-dashboard.md` | Worked: Full dashboard zone tree → Lightdash 36-col grid + promoted filters. |
| `resources/schemas/twb_2026.1.0.xsd` | Authoritative Tableau schema (2026.1.0). Grep here when an attribute looks unfamiliar. |
| `resources/schemas/chart-as-code-1.0.json` | Authoritative Lightdash chart schema. |
| `resources/schemas/dashboard-as-code-1.0.json` | Authoritative Lightdash dashboard schema. |
| `resources/schemas/model-as-code-1.0.json` | Authoritative Lightdash model schema. |

When emitting code, validate against the JSON schemas (e.g. `ajv validate -s schemas/chart-as-code-1.0.json -d chart.yml --json`) before declaring the file ready.

---

## Output expectations

For a migration job produce, in order:

1. A short **plan** (≤ 1 screen) listing target dbt models, target chart slugs, target dashboard slugs, and known unsupported features per dashboard.
2. The **dbt model YAML** files (`models/<name>.yml`).
3. Any **new dbt SQL models** for LODs / window calcs (`models/*.sql`).
4. The **chart YAML** files (`lightdash/charts/<slug>.yml`).
5. The **dashboard YAML** files (`lightdash/dashboards/<slug>.yml`).
6. The **`MIGRATION_REPORT.md`** — what migrated, what needs review, what dropped.

Each YAML file should be runnable through `lightdash lint` without warnings. If the user asks for a single file at a time, still complete the plan first.

---

## House style for generated YAML

- Two-space indent.
- Alphabetically-sorted keys at every level.
- Strings unquoted unless they contain special characters (`:`, `#`, leading `-`, etc.) or whitespace at the boundaries.
- Field IDs use the `<table>_<column>` shape (snake_case throughout).
- Slugs: kebab-case, derived from the cleaned worksheet/dashboard name.
- `version: 1` on chart and dashboard YAML; `version: 2` on dbt model YAML.
- Always set `description` on dimensions and metrics if the source had a `<desc>` or caption with extra context.
- Set `format` (`usd`, `gbp`, `eur`, `percent`, etc.) when Tableau had a `default-format`.

---

## How to start a migration session

When the user shares a workbook (or asks to start), respond with:

> "Got it — I'll migrate `<file>`. First I'll do a read-only discovery pass and come back with a plan: target dbt models, chart slugs, and any unsupported features that need a call. Then we'll generate the YAML."

Then immediately:

1. Unzip `.twbx` if needed.
2. Run a discovery script (or do it inline): list datasources, worksheets, dashboards, calc fields with kinds, filters, actions.
3. Print the **plan** as a checklist; ask for confirmation on anything ambiguous.
4. Only after confirmation, start emitting YAML.

Don't generate YAML in the same turn as discovery on a non-trivial workbook (>5 worksheets) — it almost always produces drift between the plan and the output.

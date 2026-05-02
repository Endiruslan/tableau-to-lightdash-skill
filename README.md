# Tableau to Lightdash Migration Skill

Agent skill for migrating Tableau workbooks (`.twb` / `.twbx`) into Lightdash-as-code — dbt models, metrics, dimensions, charts, and dashboards.

## Install

```bash
npx skills add Endiruslan/tableau-to-lightdash-skill
```

Works with any agent runner that supports the Agent Skills format.

## What it does

Lets AI agents convert a Tableau workbook into Lightdash YAML. The skill parses Tableau XML, maps calculated fields, parameters, filters, and dashboard zones to their Lightdash equivalents, and emits dbt + Lightdash content-as-code (chart 1.0 / dashboard 1.0 / model 1.0 schemas).

Built on two upstream specs:

- [`tableau/tableau-document-schemas`](https://github.com/tableau/tableau-document-schemas) — official Tableau workbook XML schema (2026.1.0).
- [`lightdash/lightdash` — `skills/developing-in-lightdash`](https://github.com/lightdash/lightdash/tree/main/skills/developing-in-lightdash) — official Lightdash development skill + JSON schemas.

## Use when

- You have a `.twb` or `.twbx` and want equivalent Lightdash YAML.
- You're porting Tableau calculated fields, parameters, or LOD expressions to Lightdash metrics / dbt models.
- You're rebuilding a Tableau dashboard as a Lightdash dashboard YAML on the 36-column grid.
- You need a parity report — what migrated, what needs review, what was dropped.

**Don't use for:** authoring net-new Lightdash content from scratch (use `developing-in-lightdash` instead), Tableau Server administration, or `.tflx` / `.twfl` Tableau Prep flows.

## How it works

1. **Discovery** — parse the `.twb` (unzip `.twbx` first), catalog every datasource, column, calc, parameter, worksheet, and dashboard zone.
2. **Plan** — pick the dbt model split, chart-type mapping per worksheet, and surface unsupported features.
3. **Translate** — datasources → dbt models, calcs → dbt SQL or Lightdash metrics, filters → Lightdash filter operators, dashboards → 36-col grid tiles.
4. **Emit** — `models/*.yml`, `lightdash/charts/<slug>.yml`, `lightdash/dashboards/<slug>.yml`, `MIGRATION_REPORT.md`.
5. **Validate** — `lightdash lint` + `lightdash run-chart` smoke tests.

Full workflow, mapping tables, and worked examples in [`SKILL.md`](./SKILL.md) and `resources/`.

## Layout

```
SKILL.md                             # entrypoint with frontmatter
resources/
├── tableau-twb-anatomy.md           # Tableau XML structure
├── lightdash-yaml-reference.md      # Lightdash YAML/JSON cheatsheet
├── concept-mapping.md               # Tableau → Lightdash lookup
├── chart-type-mapping.md            # mark + shelves → chartConfig.type
├── calculation-translation.md       # formulas → dbt SQL / metrics
├── filter-translation.md            # <filter> → Lightdash operators
├── dashboard-translation.md         # zones → 36-col grid tiles
├── parsing-tableau.md               # how to walk a .twb / .twbx
├── unsupported-features.md          # what doesn't migrate
├── migration-workflow.md            # phase-by-phase checklist
├── examples/                        # 6 worked examples
└── schemas/                         # vendored authoritative schemas
```

## Contributing

PRs welcome. Especially valued: more worked examples (maps, scatter, gantt, treemap with multi-level groups), Vega-Lite recipes for visuals that don't map to Lightdash built-ins, test fixtures (small `.twb` paired with expected output), and a Python parser/codegen automating discovery + emit.

## License

MIT — see [`LICENSE`](./LICENSE).

Tableau schema by Salesforce / Tableau (Apache 2.0). Lightdash schemas by Lightdash Ltd (MIT).

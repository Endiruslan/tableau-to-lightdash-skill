# tableau-to-lightdash-skill

A Claude / agent skill for migrating Tableau workbooks (`.twb` / `.twbx`) to Lightdash (dbt model YAML + chart YAML + dashboard YAML).

Built from two upstream sources:

- [`tableau/tableau-document-schemas`](https://github.com/tableau/tableau-document-schemas) — the official Tableau workbook XML schema (2026.1.0).
- [`lightdash/lightdash` — `skills/developing-in-lightdash`](https://github.com/lightdash/lightdash/tree/main/skills/developing-in-lightdash) — the official Lightdash development skill, including the JSON schemas for chart-as-code, dashboard-as-code, and model-as-code.

This skill cross-references both and adds a translation layer: concept mapping, calculation translation, filter translation, dashboard layout conversion, parsing guide, and worked examples.

## Layout

```
.
├── SKILL.md                                # main skill entry (frontmatter + workflow)
├── resources/
│   ├── tableau-twb-anatomy.md              # Tableau XML structure reference
│   ├── lightdash-yaml-reference.md         # Lightdash YAML/JSON cheatsheet
│   ├── concept-mapping.md                  # one-line Tableau→Lightdash lookup
│   ├── chart-type-mapping.md               # mark+shelves → chartConfig.type
│   ├── calculation-translation.md          # formulas → dbt SQL / metrics
│   ├── filter-translation.md               # <filter> → metricQuery.filters / dashboard filters
│   ├── dashboard-translation.md            # zones → 36-col grid tiles
│   ├── parsing-tableau.md                  # how to open + walk a .twb / .twbx
│   ├── unsupported-features.md             # what doesn't migrate; report template
│   ├── migration-workflow.md               # end-to-end phase-by-phase checklist
│   ├── examples/                           # worked examples
│   │   ├── 01-bar-chart.md
│   │   ├── 02-line-with-pivot.md
│   │   ├── 03-kpi-bignum.md
│   │   ├── 04-table-conditional-formatting.md
│   │   ├── 05-calculated-fields.md
│   │   └── 06-dashboard.md
│   └── schemas/                            # authoritative schemas (vendored)
│       ├── twb_2026.1.0.xsd
│       ├── chart-as-code-1.0.json
│       ├── dashboard-as-code-1.0.json
│       └── model-as-code-1.0.json
└── README.md
```

## Use as a Claude Code skill

Drop the folder into your project's `.claude/skills/` (or your personal `~/.claude/skills/`) and invoke via `/migrating-tableau-to-lightdash` once registered. The frontmatter `name` / `description` in `SKILL.md` makes it auto-trigger when a user mentions Tableau-to-Lightdash migration.

## Use as a Claude Agent SDK skill

Mount the directory and pass `SKILL.md` as a system / orchestrator prompt; load resources on demand based on the workflow phase.

## Use as reading material

Each file under `resources/` is self-contained and human-readable. Start with `migration-workflow.md` for the big picture, `concept-mapping.md` for the cheat sheet, and the `examples/` folder for concrete patterns.

## Contributing

PRs welcome. Areas that need help:

- Additional worked examples (geographic maps, Gantt, scatter, treemap with multi-level groups).
- Vega-Lite recipes for Tableau visuals that don't map to Lightdash built-ins.
- Test fixtures — small `.twb` files paired with the expected Lightdash YAML output.
- A Python parser/codegen that automates the discovery + YAML emit phases.

## License

MIT — see `LICENSE`.

## Credits

Tableau schema by Salesforce / Tableau (Apache 2.0). Lightdash schemas and skill by Lightdash Ltd (MIT).

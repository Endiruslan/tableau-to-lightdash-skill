# PARSING TABLEAU WORKBOOKS

Practical guide for opening and walking a `.twb` / `.twbx`.

---

## 1. File formats

- **`.twb`** — plain XML. Open with any XML parser.
- **`.twbx`** — ZIP archive. Contains:
  - the `.twb` (XML)
  - `Data/` — `.hyper` extracts (binary), CSVs, etc.
  - `Image/` — embedded images
  - `Shapes/` — custom shape palettes

Unzip first:

```bash
unzip -o workbook.twbx -d workbook_unpacked/
ls workbook_unpacked/
# → workbook.twb  Data/  Image/
```

If migrating, you only need the `.twb`. Note paths to embedded images if the dashboard references them.

---

## 2. Recommended parser stack

| Language | Library |
|---|---|
| Python | `lxml` (XPath) — preferred for the migration tool |
| Python (stdlib) | `xml.etree.ElementTree` — no XPath superset, but enough for shallow walks |
| TypeScript / JS | `fast-xml-parser` or `xml2js` |
| CLI | `xmlstarlet sel -t -v "//worksheet/@name" workbook.twb` |

Tableau XML doesn't use a default namespace at the workbook level (some sub-elements may), so plain XPath works on the whole tree.

---

## 3. Walk order

For a migration tool process the workbook in this order:

1. **Datasources** — build the metadata catalog first (fields, calcs, parameters). Charts reference these.
2. **Worksheets** — convert each to a Lightdash chart YAML.
3. **Dashboards** — convert each to a Lightdash dashboard YAML, referencing chart slugs.
4. **Actions** — promote to dashboard filters/URLs after dashboards exist.

---

## 4. Python skeleton (`lxml`)

```python
from lxml import etree

def load_workbook(path: str) -> etree._Element:
    return etree.parse(path).getroot()

def datasources(root):
    return root.xpath("/workbook/datasources/datasource")

def columns(ds):
    return ds.xpath("./column")

def is_calculated(col):
    return col.find("calculation") is not None

def aggregation(col):
    return col.get("aggregation")  # may be None for dimensions

def worksheets(root):
    return root.xpath("/workbook/worksheets/worksheet")

def mark_class(ws):
    m = ws.find(".//mark")
    return m.get("class") if m is not None else "Automatic"

def shelves(ws):
    rows = (ws.findtext(".//rows") or "").strip()
    cols = (ws.findtext(".//cols") or "").strip()
    return rows, cols

def filters(ws):
    return ws.findall(".//view/filter")

def encodings(ws):
    enc = {}
    for el in ws.findall(".//encodings/*"):
        enc[el.tag] = el.get("column")
    return enc

def dashboards(root):
    return root.xpath("/workbook/dashboards/dashboard")

def zones(dash):
    # Recursive — zones can nest under layout containers
    return dash.xpath(".//zone")
```

---

## 5. Field reference parsing

Tableau pill expressions on shelves look like:

```
[federated.0c1n2k3m].[sum:Sales:qk]
```

Parse with regex:

```python
import re
PILL = re.compile(
    r"\[(?P<ds>[^\]]+)\]\.\[(?P<agg>[a-z\-]+:)?(?P<field>[^:\]]+)(?::(?P<typekey>[a-z]+))?\]"
)
m = PILL.match("[federated.x].[sum:Sales:qk]")
m.groupdict()  # {'ds': 'federated.x', 'agg': 'sum:', 'field': 'Sales', 'typekey': 'qk'}
```

For plain `[Field]` references (calculated field bodies, simple shelves), match `\[([^\]]+)\]`.

Resolve `field` against the datasource's `<column>` list to recover caption, role, datatype, calc body.

---

## 6. Calculations

Tableau formulas in `<calculation formula="…"/>` are not XML-escaped beyond the basics — they're stored as a single attribute string. Escapes you'll see: `&amp;`, `&quot;`, `&lt;`, `&gt;`. Most parsers unescape automatically when reading attributes.

Detect calc kinds:

```python
def calc_kind(col):
    c = col.find("calculation")
    if c is None:                      return "physical"
    if c.get("class") == "passthrough": return "rawsql"
    if c.get("class") == "bin":         return "bin"
    if c.get("class") == "categorical-bin": return "group"
    if c.find("table-calc") is not None:return "table_calc"
    f = c.get("formula", "")
    if "{FIXED" in f or "{INCLUDE" in f or "{EXCLUDE" in f: return "lod"
    return "formula"
```

Use `calculation-translation.md` to map each kind.

---

## 7. Datasource ID resolution

Inline blends create `federated.<hash>` virtual datasources. The real columns live on the underlying datasources referenced via `<datasource-dependencies>`. For migration:

- Treat each `federated.*` as a query that needs to be modeled as a dbt model joining the underlying datasources. Don't emit a chart pointing at `federated.*` directly.
- Single-source workbooks: ignore the federated wrapper; resolve fields to the source datasource.

---

## 8. Sanity checks before generating output

For each worksheet emit a log line:

```
[ws] "Sales by Region"  mark=Bar  rows=[Region]  cols=[sum:Sales]  filters=2  encodings=color,detail
```

For unsupported features add `WARN`:

```
[ws] "Forecast 2025"  WARN: trend line + forecast — not migratable, skipped
```

A migration always leaves a written report. See `migration-workflow.md`.

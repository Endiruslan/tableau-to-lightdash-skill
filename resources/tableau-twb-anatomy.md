# TABLEAU WORKBOOK (.twb / .twbx) ANATOMY

Reference for parsing Tableau workbooks during migration to Lightdash. Based on the official Tableau Document Schema 2026.1.0 (`twb_2026.1.0.xsd`, see `schemas/`).

A `.twb` file is plain XML. A `.twbx` file is a ZIP archive that contains the `.twb` plus extracts and resources — unzip first, then parse the `.twb`.

---

## 1. Top-level workbook structure

Root element is `<workbook>`. Main children:

```xml
<workbook version="2026.1" source-build="20261.26.0211.1127" source-platform="mac">
  <preferences><!-- locale, tooltip --></preferences>
  <style-theme name="seattle"/>
  <style/>
  <datasources>
    <datasource name="Orders" inline="true">…</datasource>
  </datasources>
  <worksheets>
    <worksheet name="Sales by Region">…</worksheet>
  </worksheets>
  <dashboards>
    <dashboard name="Executive">…</dashboard>
  </dashboards>
  <windows/>
  <actions/>
</workbook>
```

Key attributes on `<workbook>`: `version` (required, e.g. `2026.1`), `source-build`, `source-platform` (`win` | `mac` | `linux`).

---

## 2. Datasource model

### Container

```xml
<datasource name="Orders" caption="OrderData" inline="true" version="2026.1">
  <connection class="sqlserver" .../>
  <column name="[Sales]" role="measure" type="quantitative" datatype="real" aggregation="sum"/>
  <column name="[Order Date]" role="dimension" type="ordinal" datatype="date"/>
  <column name="[Profit Ratio]" caption="Profit %">
    <calculation class="tableau" formula="[Profit] / [Sales]"/>
  </column>
  <group name="[Region Group]">…</group>
  <bin name="[Sales Bin]" column="[Sales]" size="100"/>
  <aliases enabled="true">
    <alias key="2023" value="FY2023"/>
  </aliases>
</datasource>
```

### `<column>` — field definition

```xml
<column name="[Sales]"
        role="dimension|measure"
        type="nominal|ordinal|quantitative|unknown"
        datatype="integer|real|string|datetime|date|boolean|spatial"
        aggregation="sum|average|min|max|count|count-d|median|std-dev|var|..."
        caption="Total Sales"
        default-format="#,##0.00"
        hidden="false"
        semantic-role="[Geographic].[Country]">
  <calculation class="tableau" formula="SUM([Sales])"/>
</column>
```

Enumerations:

- **role**: `dimension`, `measure`, `unknown`
- **type** (semantic, pill color): `nominal` (categorical, blue), `ordinal` (ordered cat.), `quantitative` (continuous, green), `unknown`
- **datatype**: `integer`, `real`, `string`, `datetime`, `date`, `boolean`, `spatial`, `table`, `unknown`
- **aggregation**: `sum`, `average` (avg), `min`, `max`, `count`, `count-d` (distinct), `median`, `percentile`, `std-dev`, `std-dev-p`, `var`, `var-p`, `quart1`, `quart3`, `attr`, `concatenate`

### `<calculation>` — derived columns / metrics

```xml
<calculation class="tableau|passthrough|bin|categorical-bin"
             formula="SUM([Sales] * (1 - [Discount]))"
             scope-isolation="false"
             solve-order="1"/>
```

`class` values:
- `tableau` — Tableau formula language (default)
- `passthrough` — raw warehouse SQL (RAWSQL/RAWSQLAGG, dialect-specific)
- `bin` — numeric bins
- `categorical-bin` — discrete groupings

Table calc decoration (window functions) lives inside:

```xml
<calculation class="tableau" formula="RANK()">
  <table-calc type="rank" ordering-type="table-across" ordering-field="[Sales]">
    <order field="[Sales]" direction="DESC"/>
  </table-calc>
</calculation>
```

`type` values include `running-sum`, `running-avg`, `running-max`, `running-min`, `window-sum`, `window-avg`, `rank`, `rank-desc`, `rank-dense`, `percentile`, `index`, `first`, `last`, `lookup`, `offset`.

### `<group>` and `<bin>`

```xml
<group name="[Region Group]">
  <groupfilter>
    <groupfilter-item value="East"/>
    <groupfilter-item value="West"/>
  </groupfilter>
</group>

<bin name="[Sales (bin)]" column="[Sales]" size="500" new-bin="true"/>
```

### `<aliases>` — value relabels

```xml
<aliases enabled="true">
  <alias key="East" value="Eastern Region"/>
</aliases>
```

### `<connection>` — physical source

```xml
<!-- Live -->
<connection class="sqlserver" server="sql.example.com" database="Sales"
            username="analyst" extract="false">
  <relation type="table" name="[dbo].[Orders]" table="Orders"/>
</connection>

<!-- Hyper extract (.hyper file inside .twbx) -->
<connection class="hyper" directory="Data Sources" extract="true">
  <relation type="table" name="Orders"/>
</connection>
```

Connection `class` examples: `sqlserver`, `postgres`, `redshift`, `snowflake`, `bigquery`, `mysql`, `oracle`, `excel-direct`, `textscan`, `hyper`, `federated` (multi-source blend).

---

## 3. Worksheet model

```xml
<worksheet name="Sales Analysis">
  <table type="self-service">
    <view>
      <datasources>
        <datasource name="Orders"/>
      </datasources>
      <datasource-dependencies datasource="Orders">
        <column name="[Sales]"/>
        <column name="[Region]"/>
      </datasource-dependencies>
      <filter column="[Order Date]" class="relative-date"
              period-type-v2="last-n-days" first-period="-30" last-period="0"/>
      <filter column="[Sales]" class="quantitative" included-values="in-range">
        <min>1000</min><max>50000</max>
      </filter>
      <slices>
        <column>[Region]</column>
      </slices>
      <aggregation value="true"/>
    </view>
    <rows>[Region]</rows>
    <cols>[Year([Order Date])]</cols>
    <encodings>
      <color column="[federated.x].[sum:Profit:qk]"/>
      <size column="[federated.x].[sum:Sales:qk]"/>
      <text column="[federated.x].[sum:Sales:qk]"/>
      <detail column="[Order ID]"/>
      <tooltip column="[Customer Name]"/>
      <label column="[Sales]"/>
      <shape column="[Segment]"/>
    </encodings>
    <mark class="Bar"/>
    <style>
      <style-rule element="mark">
        <format attr="mark-color" value="#5b9bd5"/>
      </style-rule>
    </style>
  </table>
</worksheet>
```

Key children:

- **`<rows>` / `<cols>`** — text content is the shelf expression (a `<field>` node or inline pill ref).
- **`<filter>`** — see §6.
- **`<encodings>`** — children: `color`, `size`, `shape`, `text`, `detail`, `tooltip`, `label`, `page`, `focus`. Each carries a `column` attr referencing a datasource field.
- **`<mark class="…"/>`** — viz type. See §5.
- **`<slices>`** — additional dimensional partitioning (small multiples).
- **`<style>`** — formatting overrides.

### Pill / shelf field reference syntax

```
[federated.0c1n2…].[sum:Sales:qk]
   datasource id           agg : field : type-key
```

- `agg` — `sum`, `avg`, `min`, `max`, `count`, `count-d`, `attr`, `mdy` (date trunc), `none`
- type-key suffix — `qk` quantitative continuous, `nk` nominal, `ok` ordinal

Discrete (blue) vs continuous (green) is implied by the type-key and `derivation` attribute.

---

## 4. Calculation language

Formula syntax cheatsheet:

```
[Field]                      -- field reference
[Datasource].[Field]         -- qualified field reference
SUM([Sales])                 -- aggregate
IF cond THEN x ELSE y END
CASE [Status] WHEN 'A' THEN 1 WHEN 'B' THEN 2 ELSE 0 END
{FIXED [Region] : SUM([Sales])}        -- LOD: ignore other dims
{INCLUDE [Customer] : AVG([Sales])}    -- LOD: add a dim
{EXCLUDE [Date] : SUM([Sales])}        -- LOD: drop a dim
WINDOW_SUM(SUM([Sales]), -2, 0)        -- table calc
RUNNING_SUM(SUM([Sales]))
RANK(SUM([Sales]))
LOOKUP(SUM([Sales]), -1)
INDEX()
FIRST() / LAST()
```

Common functions:

| Group | Functions |
|------|-----------|
| Aggregate | `SUM`, `AVG`, `MIN`, `MAX`, `COUNT`, `COUNTD`, `MEDIAN`, `PERCENTILE`, `STDEV`, `STDEVP`, `VAR`, `VARP`, `ATTR` |
| String | `LEFT`, `RIGHT`, `MID`, `LEN`, `UPPER`, `LOWER`, `TRIM`, `LTRIM`, `RTRIM`, `CONTAINS`, `STARTSWITH`, `ENDSWITH`, `FIND`, `REPLACE`, `REGEXP_MATCH`, `REGEXP_REPLACE`, `REGEXP_EXTRACT`, `SPLIT` |
| Date | `YEAR`, `QUARTER`, `MONTH`, `WEEK`, `DAY`, `HOUR`, `MINUTE`, `SECOND`, `DATEPART`, `DATETRUNC`, `DATEADD`, `DATEDIFF`, `DATEPARSE`, `DATENAME`, `MAKEDATE`, `TODAY`, `NOW` |
| Math | `ABS`, `ROUND`, `CEILING`, `FLOOR`, `MOD`, `POWER`, `SQRT`, `LOG`, `LN`, `EXP`, `SIGN` |
| Logical | `AND`, `OR`, `NOT`, `IF/THEN/ELSE/END`, `IIF`, `IFNULL`, `ISNULL`, `ZN` (zero if null) |
| Cast | `INT`, `FLOAT`, `STR`, `DATE`, `DATETIME`, `BOOL` |
| Window | `WINDOW_*`, `RUNNING_*`, `LOOKUP`, `INDEX`, `FIRST`, `LAST`, `RANK`, `RANK_DENSE`, `RANK_PERCENTILE`, `TOTAL`, `SIZE` |
| LOD spec | `{FIXED}`, `{INCLUDE}`, `{EXCLUDE}` |
| RAW SQL | `RAWSQL_INT`, `RAWSQL_REAL`, `RAWSQL_BOOL`, `RAWSQL_STR`, `RAWSQL_DATE`, `RAWSQLAGG_*` |

---

## 5. Mark types (`<mark class="…"/>`)

`Automatic`, `Bar`, `Line`, `Area`, `Square`, `Circle`, `Shape`, `Text`, `Map`, `Pie`, `Gantt`, `Polygon`, `Density`.

`Automatic` resolves at render time based on shelf composition (e.g. one continuous date on cols + one measure on rows ⇒ line).

Properties:

```xml
<mark class="Bar">
  <property name="mark_color" value="#5b9bd5"/>
  <property name="mark_opacity" value="0.8"/>
  <property name="mark_border_color" value="#000000"/>
  <property name="mark_size" value="5"/>
</mark>
```

Stacking is implicit when multiple measures share an axis with the same encoding. Dual-axis is encoded by two `<pane>` elements with separate axes — look for `dual="true"` and matching axis sync.

---

## 6. Filters

### Categorical

```xml
<filter column="[Region]" class="categorical" included-values="all">
  <preset type="all-values" applied="true"/>
  <groupfilter>
    <groupfilter-item value="East"/>
    <groupfilter-item value="West"/>
  </groupfilter>
</filter>
```

`included-values` ∈ `all`, `non-null`, `null`, `in-range`, `top-n`.

### Quantitative range

```xml
<filter column="[Sales]" class="quantitative" included-values="in-range">
  <min>1000</min>
  <max>50000</max>
</filter>
```

### Relative date

```xml
<filter column="[Order Date]" class="relative-date"
        period-type-v2="last-n-days" first-period="-30" last-period="0"
        period-anchor="2024-12-31" include-future="false" include-null="false"/>
```

`period-type-v2` ∈ `last-n-days`, `last-n-weeks`, `last-n-months`, `last-n-quarters`, `last-n-years`, `next-n-days|weeks|months|years`, `year-to-date`, `quarter-to-date`, `month-to-date`, `last-day`, `last-week`, `last-month`, `last-quarter`, `last-year`, `today`, `yesterday`.

### Top-N

```xml
<filter column="[Region]" class="categorical" filter-group="2">
  <groupfilter function="end" user:ui-top-by-field="true" direction="TOP"
               expression="[sum:Sales:qk]" units="10"/>
</filter>
```

### Context filter

Same XML shape, with `context="true"` attribute.

### Formula (condition) filter

Filter has a child `<groupfilter function="filter">` whose body is a Tableau expression returning boolean.

---

## 7. Parameters

A parameter is a `<column>` with `param-domain-type` attribute:

```xml
<column name="[Selected Year]"
        role="dimension" type="nominal" datatype="integer"
        param-domain-type="list" value="2024" caption="Select Year">
  <parameter-list>
    <value>2022</value>
    <value>2023</value>
    <value>2024</value>
  </parameter-list>
</column>

<column name="[Sales Threshold]"
        role="measure" type="quantitative" datatype="real"
        param-domain-type="range" value="5000">
  <range min="0" max="100000" granularity="100"/>
</column>
```

`param-domain-type` ∈ `list`, `range`, `any`. Referenced in formulas as `[Selected Year]`.

---

## 8. Dashboard model

```xml
<dashboard name="Executive Summary">
  <size sizing-mode="automatic" minwidth="1000" minheight="700"/>
  <datasources>
    <datasource name="Orders"/>
  </datasources>
  <zones is-pixels="true">
    <zone id="1" x="0" y="0" w="500" h="400"
          type-v2="worksheet" name="Sales Chart"/>
    <zone id="2" x="500" y="0" w="200" h="400"
          type-v2="filter" param="[Region]" name="Region Selector"/>
    <zone id="3" type-v2="layout-basic vertical" x="0" y="400" w="700" h="300">
      <zone id="4" type-v2="worksheet" name="Details Table"/>
      <zone id="5" type-v2="text" show-title="false">
        <formatted-text><run bold="true">Footnote</run></formatted-text>
      </zone>
    </zone>
  </zones>
  <devicelayouts>
    <devicelayout name="Phone" auto-generated="true">
      <zones>…</zones>
    </devicelayout>
  </devicelayouts>
</dashboard>
```

`<size sizing-mode="…">` ∈ `automatic`, `fixed`, `range` (resizable within bounds).

`<zone type-v2="…">`:
- `worksheet` — embed a sheet (referenced by `name`)
- `filter` — quick filter UI for a field (`param="[Field]"`)
- `parameter` — parameter UI control
- `text` — formatted text (child `<formatted-text>`)
- `image` — `image-file-url`/`image-name`
- `web` — URL iframe
- `legend` — color/size/shape legend (auto from a worksheet)
- `blank` — spacer
- `layout-basic horizontal` / `layout-basic vertical` — flex container of child zones
- `layout-flow` — modern flex container

Zone geometry: `x`, `y`, `w`, `h` in pixels (when `is-pixels="true"` on parent `<zones>`). For flex containers, child geometry is computed; persisted layout cache lives in `<layout-cache>`.

### Dashboard actions

```xml
<actions brushing-enabled="true">
  <action name="Filter by Region">
    <activation method="on-select"/>
    <source sheet="Sales Chart" datasource="Orders"/>
    <target sheet="Regional Details">
      <target-field column="[Region]"/>
    </target>
  </action>
  <action name="Highlight Segment" type="highlight">…</action>
  <action name="Open URL" type="url">
    <url>https://example.com?id=[Product ID]</url>
  </action>
  <edit-parameter-action name="Set Year">
    <source sheet="Dashboard"/>
    <target-parameter name="[Selected Year]" value="2023"/>
  </edit-parameter-action>
</actions>
```

`activation method` ∈ `on-select`, `on-hover`, `explicit` (menu).

---

## 9. Field encoding details (encodings on shelves)

```xml
<encodings>
  <!-- discrete dim on color -->
  <color column="[federated.x].[Region]"/>

  <!-- continuous measure on color (with range/palette set elsewhere) -->
  <color column="[federated.x].[sum:Profit:qk]" type="continuous"/>

  <!-- size by measure -->
  <size column="[federated.x].[sum:Sales:qk]"/>

  <!-- text (label on Marks card / Text mark) -->
  <text column="[federated.x].[sum:Sales:qk]"/>

  <!-- detail: extra granularity but no visual encoding -->
  <detail column="[federated.x].[Order ID]"/>

  <!-- tooltip override fields (no axis position) -->
  <tooltip column="[federated.x].[Customer Name]"/>
</encodings>
```

Color/size palette config sits in companion `<style>` blocks at the worksheet or dashboard level, e.g. `color-palette`, `range-min`, `range-max`.

---

## 10. Things that DO NOT translate cleanly to Lightdash

These need warehouse-side rewrite, design changes, or are simply not mappable. Flag in the migration report and discuss before attempting.

| Tableau feature | Why hard | Lightdash workaround |
|---|---|---|
| **Table calculations** (`WINDOW_*`, `RUNNING_*`, `LOOKUP`, `RANK`) | Operate post-aggregation per-pane in client | Push into dbt as window functions, expose result as a metric or dimension. Or use Lightdash table calculations (subset of SQL post-query). |
| **LOD `{FIXED}` / `{INCLUDE}` / `{EXCLUDE}`** | Non-standard nested aggregation semantics | Materialize as a separate dbt model + join, or use `sum_distinct` with `distinct_keys` for FIXED-style cardinality safety. |
| **Sets** (combined / computed) | First-class objects in Tableau | Encode as boolean dimensions in dbt (`is_high_value`). |
| **Hierarchies / drill-paths** | Multi-level OLAP hierarchies | Lightdash supports manual drill via dimension groups; declare each level as a separate dimension and group them via `meta.dimension.groups`. |
| **Custom shapes & shape palettes** | Bound to image assets | Use color encoding instead, or supply a single fallback color. |
| **Story points** | Narrative slideshow | Out of scope. Document separately or split into multiple dashboards. |
| **Dual-axis charts** | Two independent y-axes per pane | Possible: cartesian with two series on different `yAxisIndex`. |
| **Reference / trend / forecast lines** | Inline statistical models | Pre-compute in dbt (forecast tables). For static reference lines use chart `markLine` config. |
| **Pages shelf / animated viz** | Frame-by-frame animation | Translate to a dimension filter or dashboard date zoom. |
| **Geographic role auto-geocoding** | Tableau's lat/lon lookup | Pre-resolve in warehouse; ship lat/lon columns. |
| **Blended data sources** | Multi-source post-aggregation join | Pre-join in dbt. |
| **Hyper extracts** | Snapshot of warehouse | Replace with dbt model + scheduled run. |
| **Ad-hoc clustering / "Explain Data"** | ML-driven | Not portable. Document and drop. |
| **Pages shelf**, **Trend lines**, **Reference distribution**, **Story** | UX features | Out of scope; document. |

---

## 11. Pattern-matching cheatsheet (for parsers)

| Element | XPath-ish | Use |
|---|---|---|
| All datasources | `/workbook/datasources/datasource` | iterate sources |
| All raw fields | `/workbook/datasources/datasource/column[not(calculation)]` | physical columns |
| All calc fields | `/workbook/datasources/datasource/column[calculation]` | derived metrics/dimensions |
| All worksheets | `/workbook/worksheets/worksheet` | sheets to convert to charts |
| Sheet rows shelf | `/workbook/worksheets/worksheet/table/rows` | y-axis dim/measure |
| Sheet cols shelf | `/workbook/worksheets/worksheet/table/cols` | x-axis dim/measure |
| Sheet mark type | `/workbook/worksheets/worksheet/table/view//mark/@class` | viz family |
| Sheet filters | `/workbook/worksheets/worksheet/table/view/filter` | dimension/measure filters |
| Sheet color encoding | `/workbook/worksheets/worksheet/table/encodings/color/@column` | color split / metric |
| All dashboards | `/workbook/dashboards/dashboard` | dashboards to convert |
| Dashboard zones | `/workbook/dashboards/dashboard//zone` | recursive (zones can nest) |
| Dashboard actions | `/workbook/dashboards/dashboard/../actions/action` | filter/highlight/URL actions |
| Parameters | `/workbook/datasources/datasource/column[@param-domain-type]` | parameter columns |

---

## 12. Connection class → Lightdash warehouse hint

| Tableau `connection class` | Likely Lightdash warehouse |
|---|---|
| `snowflake` | snowflake |
| `bigquery` | bigquery |
| `redshift` | redshift |
| `postgres` | postgres |
| `sqlserver` | sqlserver / databricks (verify) |
| `mysql` | mysql |
| `databricks` | databricks |
| `hyper` (extract) | none — replace with the underlying warehouse |
| `excel-direct`, `textscan`, `csv` | none — load to warehouse first |
| `federated` | resolve underlying connections, model in dbt |

Lightdash needs a real warehouse + dbt project. Workbooks backed only by extracts/csv require data engineering work before charts can move.

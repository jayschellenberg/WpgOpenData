# Implementation Guide: Assessment Parcel 3D Map

Reference for recreating this approach with different datasets (e.g. City of Winnipeg assessment parcels).

---

## Architecture Overview

```
Data Source (ArcGIS REST API / Socrata / CSV)
  → Download Script (.qmd, eval: false by default)
  → Date-stamped GeoParquet file in data/
  → Visualization Script (.qmd)
      → Reads parquet, builds sf object
      → Simplifies geometry for embedding
      → Dissolves to area-level polygons for choropleth
      → Renders interactive MapLibre map with toggle + legend
      → Outputs summary statistics table
  → docs/index.html (GitHub Pages)
```

### Key design decisions

| Decision | Choice | Why |
|---|---|---|
| Data format | GeoParquet (`.parquet`) | Fast columnar reads, compact, preserves geometry |
| Geometry library | `sf` with `geoarrow` bridge | `read_parquet()` returns geoarrow; convert with `st_as_sfc(geometry)` then `st_as_sf()` |
| Map renderer | `mapgl` (R bindings for MapLibre GL JS) | 3D extrusions, smooth interactivity, no Mapbox token needed |
| Deployment mode | `use_local_data <- TRUE` embeds GeoJSON (~284MB HTML); `FALSE` uses PMTiles (~17MB HTML) | Local works offline for testing; PMTiles for production deployment |
| PMTiles hosting | `docs/` folder via Git LFS, relative URL | Same-origin avoids CORS; GitHub Pages serves LFS files |
| Output | Quarto → `docs/index.html` via `_quarto.yml` `output-dir: docs` | Direct GitHub Pages deployment from `docs/` |

---

## File Structure

```
project-root/
├── _quarto.yml              # output-dir: docs
├── .gitattributes           # Git LFS tracking for docs/*.pmtiles
├── DownloadScript.qmd       # Data download (eval: false)
├── map-visualization.qmd    # Map + analysis (the main deliverable)
├── README.md
├── .gitignore               # data/, *_files/, but !docs/**
├── data/
│   └── {dataset}-{YYYY-MM-DD}.parquet
└── docs/
    ├── index.html                        # Rendered map (~17 MB)
    ├── {dataset}.pmtiles                 # Vector tiles (Git LFS)
    └── map-visualization_files/libs/     # Quarto JS/CSS dependencies
```

### `_quarto.yml`

```yaml
project:
  type: default
  output-dir: docs
```

### `.gitignore` key patterns

```gitignore
data/
*_files/
!docs/**
```

The `!docs/**` exception ensures rendered output is tracked for GitHub Pages even though `*_files/` is ignored.

### `.gitattributes` — Git LFS for PMTiles

```gitattributes
docs/*.pmtiles filter=lfs diff=lfs merge=lfs -text
```

PMTiles files are typically 100–300 MB, exceeding GitHub's 100 MB push limit. Git LFS handles large files and GitHub Pages serves them normally. Set up once with:

```bash
git lfs install
git lfs track "docs/*.pmtiles"
# This creates .gitattributes — commit it alongside the PMTiles file
```

---

## Step 1: Download Script

Separate `.qmd` with `eval: false` so it doesn't re-download on every render.

### ArcGIS REST API pattern (Manitoba GeoPortal)

```r
base_url <- "https://services.arcgis.com/{org}/arcgis/rest/services/{layer}/FeatureServer/0"
batch_size <- 2000  # ArcGIS max per request

# Get total count
count_resp <- request(glue("{base_url}/query")) |>
  req_url_query(where = "1=1", returnCountOnly = "true", f = "json") |>
  req_perform()
total_rows <- resp_body_json(count_resp)$count

# Paginate with resultOffset
offsets <- seq(0, by = batch_size, length.out = ceiling(total_rows / batch_size))
batches <- lapply(seq_along(offsets), function(i) {
  resp <- request(glue("{base_url}/query")) |>
    req_url_query(
      where = "1=1", outFields = "*", f = "geojson",
      resultOffset = offsets[i], resultRecordCount = batch_size,
      orderByFields = "OBJECTID"
    ) |>
    req_retry(max_tries = 3, backoff = ~10) |>
    req_throttle(rate = 1/2) |>
    req_perform()
  st_read(resp_body_string(resp), quiet = TRUE)
})
combined_sf <- bind_rows(batches)
```

### Winnipeg (Socrata) adaptation

Winnipeg's open data is on Socrata, not ArcGIS. Key differences:

- **API**: `https://data.winnipeg.ca/resource/{dataset-id}.geojson`
- **Pagination**: uses `$limit` and `$offset` (default limit 1000, max 50000)
- **Auth**: App token recommended but not required (throttled without it)
- **Format**: GeoJSON directly, or CSV + separate geometry

```r
# Socrata example
base_url <- "https://data.winnipeg.ca/resource/{dataset-id}.geojson"
batch_size <- 50000

resp <- request(base_url) |>
  req_url_query(`$limit` = batch_size, `$offset` = offset, `$order` = ":id") |>
  req_headers(`X-App-Token` = Sys.getenv("SOCRATA_APP_TOKEN")) |>
  req_perform()
```

### Write GeoParquet

```r
output_file <- fs::path("data", glue("{dataset}-{Sys.Date()}.parquet"))
write_parquet(combined_sf, output_file)
```

Date-stamping the filename lets you keep historical snapshots and the visualization script always picks the most recent:

```r
input_file <- sort(fs::dir_ls("data", glob = "*/{dataset}-*.parquet"), decreasing = TRUE)[1]
file_date  <- stringr::str_extract(basename(input_file), "\\d{4}-\\d{2}-\\d{2}")
```

---

## Step 2: Visualization Script — Structure

### YAML front matter

```yaml
---
title: "Title"
author: "Your Name"
date: today
date-format: "MMMM D, YYYY"
format:
  html:
    toc: true
output-file: index.html
execute:
  echo: false       # Hide all code from public
  warning: false
  message: false
---
```

Key points:
- `date: today` + `date-format` gives you the render date stamp automatically
- `echo: false` globally hides code; no need for `code-fold`
- `output-file: index.html` is required for GitHub Pages

### Code block visibility

| Block purpose | Chunk option | Shows in HTML |
|---|---|---|
| Library loading, config | `#\| include: false` | Nothing |
| Data loading, computation | `#\| include: false` | Nothing |
| Inline R stats in markdown | (none needed) | Prose with computed values |
| Map widget | (default) | Map only (echo: false hides code) |
| Summary table | (default) | Table only |

**Ordering rule**: Inline R expressions (`` `r variable` ``) in markdown headings/text only work if the variable was defined in a *prior* code chunk. Plan chunk order accordingly:

```
setup chunk (libraries, config)     ← include: false
find-data chunk (locate parquet)    ← include: false
  ↓ inline R available here ↓
## Heading with `r asmt_year`       ← uses variables from above
compute chunk (stats)               ← include: false
  ↓ inline R available here ↓
Narrative paragraph with `r n_parcels`, `r dollar(total_assessed)`
map chunk                           ← shows widget only
table chunk                         ← shows table only
```

---

## Step 3: Assessment Roll Metadata

### Extracting the assessment year from data

For Manitoba, the `Asmt_Roll` field contains text like `"2026 Tax Assessment Roll"`:

```r
asmt_year <- read_parquet(input_file, col_select = "Asmt_Roll") |>
  pull(Asmt_Roll) |> na.omit() |> head(1) |>
  stringr::str_extract("\\d{4}")
```

For Winnipeg, look for equivalent fields. The Socrata dataset metadata or column names may include year information. Check with:

```r
df <- read_parquet(input_file) |> as.data.frame()
names(df)          # List all columns
str(df, max = 1)   # Quick structure check
```

### Base value date

Hardcoded — set by legislation, not in the data:

```r
base_value_date <- "April 1, 2023"   # For 2026 assessment roll
```

Manitoba's base value dates by roll:
- 2026 roll → April 1, 2023
- 2022 roll → April 1, 2019
- 2018 roll → April 1, 2015

Winnipeg follows the same base value dates as the province.

### Using in headings

```markdown
## Provincial Overview — `r asmt_year` Assessment Roll (Base Value Date: `r base_value_date`)
```

---

## Step 4: Geometry Handling

### Reading GeoParquet with geoarrow geometry

`arrow::read_parquet()` returns geoarrow geometry (not sf). Must convert:

```r
parcels_sf <- read_parquet(input_file) |>
  mutate(geometry = st_as_sfc(geometry)) |>
  st_as_sf()
```

### Simplifying for embedded HTML

With 400K+ parcels, geometry simplification is essential to keep file size manageable:

```r
sf_use_s2(FALSE)   # Required — s2 chokes on invalid geometries in this data
parcels_sf <- st_make_valid(parcels_sf) |>
  st_simplify(dTolerance = 0.0002, preserveTopology = TRUE)
sf_use_s2(TRUE)
```

- `dTolerance = 0.0002` ≈ 22m at Manitoba latitudes. Parcels stay recognizable at street zoom.
- `st_make_valid()` **before** `st_simplify()` is critical. Without it, you get: `"Edge 1 has duplicate vertex with edge 6"`.
- `sf_use_s2(FALSE)` is required because the raw geometries have self-intersections that break s2's strict validation.

### Dissolving to area-level polygons (for choropleth)

```r
sf_use_s2(FALSE)
area_sf <- parcels_sf |>
  group_by(Area_Name) |>           # Municipality, neighbourhood, ward, etc.
  summarise(
    Total_Assessed = sum(Value_Col, na.rm = TRUE),
    Parcel_Count   = n(),
    Median_Value   = median(Value_Col, na.rm = TRUE),
    .groups = "drop"
  ) |>
  st_make_valid() |>
  st_simplify(dTolerance = 0.001, preserveTopology = TRUE) |>
  st_make_valid() |>
  mutate(
    Total_Assessed_Fmt = dollar(Total_Assessed, accuracy = 1),
    Median_Value_Fmt   = dollar(Median_Value, accuracy = 1),
    Value_Rank = paste(rank(-Total_Assessed, ties.method = "min"), "of", n())
  )
sf_use_s2(TRUE)
```

**Always simplify dissolved polygons.** The dissolve inherits full-resolution parcel boundaries. Without simplification, 186 municipality polygons ballooned the HTML from 17 MB to 71 MB. `dTolerance = 0.001` (~110m) is fine for area outlines viewed at lower zoom — they don't need parcel-level precision.

For Winnipeg, the grouping column would be neighbourhood or ward instead of `Muni_Name_With_Typ`.

---

## Step 5: Map Construction (mapgl)

### Base map

```r
m <- maplibre(
  style = carto_style("dark-matter"),
  center = c(lon, lat),    # -97.14, 49.89 for Winnipeg
  zoom = 11,               # 5 for province-wide, 11 for city
  pitch = 30,
  bearing = 0,
  projection = "mercator"
) |>
  add_navigation_control(position = "top-right") |>
  add_geocoder_control(
    position = "top-left",
    placeholder = "Search for an address..."
  )
```

`add_geocoder_control()` uses OpenStreetMap/Nominatim by default — no API key needed.

### Layer 1: Area choropleth (hidden by default)

```r
q_breaks <- as.numeric(quantile(area_sf$Total_Assessed, probs = c(0, 0.2, 0.4, 0.6, 0.8, 1.0), na.rm = TRUE))

m <- m |>
  add_fill_layer(
    id = "area-choropleth",
    source = area_sf,
    fill_color = interpolate(
      column = "Total_Assessed",
      values = q_breaks,
      stops = c("#2166ac", "#67a9cf", "#d1e5f0", "#fddbc7", "#ef8a62", "#b2182b")
    ),
    fill_opacity = 0.8,
    visibility = "none",
    tooltip = concat(
      "<strong>", get_column("Area_Name"), "</strong><br>",
      "Total Assessed Value: ", get_column("Total_Assessed_Fmt"), "<br>",
      "Rank: ", get_column("Value_Rank"), "<br>",
      "Parcels: ", get_column("Parcel_Count"), "<br>",
      "Median Value: ", get_column("Median_Value_Fmt")
    ),
    hover_options = list(fill_color = "#ffffff", fill_opacity = 1)
  )
```

Color scale: blue (#2166ac) = lowest, red (#b2182b) = highest. Quantile breaks ensure even color distribution regardless of skew.

### Layer 2: 3D parcel extrusions (visible by default)

```r
m <- m |>
  add_fill_extrusion_layer(
    id = "parcels-3d",
    source = parcels_sf,
    fill_extrusion_height = interpolate(
      column = "Value_Col",
      values = c(0, 100000, 500000, 2000000, 10000000),
      stops  = c(0, 50, 200, 600, 2000)
    ),
    fill_extrusion_color = match_expr(
      column = "Area_Name",
      values = area_names,
      stops  = area_colors,
      default = "#95a5a6"
    ),
    fill_extrusion_opacity = 0.8,
    tooltip = concat(
      "<strong>", get_column("Property_Address"), "</strong><br>",
      "Assessed Value: ", get_column("Total_Value"), "<br>",
      "Area Median: ", get_column("Area_Median_Fmt"), "<br>",
      "Roll No: ", get_column("Roll_No_Txt")
    ),
    hover_options = list(fill_extrusion_color = "#ffffff")
  )
```

Adjust the height `values`/`stops` to match the value distribution of your dataset. Manitoba ranges from $0 to ~$50M. Winnipeg residential will have a different range.

### Layer toggle + legend (combined single control)

**Critical**: `mapgl::add_control()` only keeps ONE custom HTML control. Multiple calls overwrite each other. Combine the radio toggle and legend into a single HTML string.

```r
# Format breakpoint labels with B/M/K abbreviations
fmt_short <- function(x) {
  dplyr::case_when(
    x >= 1e9  ~ paste0("$", trimws(format(round(x / 1e9, 1), nsmall = 1)), "B"),
    x >= 1e6  ~ paste0("$", trimws(format(round(x / 1e6, 0))), "M"),
    x >= 1e3  ~ paste0("$", trimws(format(round(x / 1e3, 0))), "K"),
    TRUE      ~ paste0("$", trimws(format(round(x, 0))))
  )
}
legend_labels <- fmt_short(q_breaks)

# Build color swatch rows
legend_colors <- c("#2166ac", "#67a9cf", "#d1e5f0", "#fddbc7", "#ef8a62", "#b2182b")
legend_rows <- paste0(sapply(seq_along(legend_colors), function(i) {
  label <- if (i == 1) {
    paste0("&lt; ", legend_labels[i + 1])
  } else if (i == length(legend_colors)) {
    paste0("&ge; ", legend_labels[i])
  } else {
    paste0(legend_labels[i], " \u2013 ", legend_labels[i + 1])
  }
  paste0(
    '<div style="display:flex;align-items:center;margin-bottom:3px;">',
    '<span style="display:inline-block;width:18px;height:12px;background:', legend_colors[i],
    ';border-radius:2px;margin-right:6px;flex-shrink:0;"></span>',
    '<span>', label, '</span></div>'
  )
}), collapse = "")

# Single combined HTML control
combined_control_html <- paste0(
  '<div style="background:rgba(0,0,0,0.75);padding:10px 12px;border-radius:6px;',
  'color:#fff;font-family:sans-serif;font-size:13px;line-height:1.6;">',
  '<div style="font-weight:bold;margin-bottom:4px;">View</div>',
  '<label style="display:block;cursor:pointer;">',
  '<input type="radio" name="map-layer" value="3d" checked style="margin-right:5px;">',
  '3D Parcels</label>',
  '<label style="display:block;cursor:pointer;">',
  '<input type="radio" name="map-layer" value="area" style="margin-right:5px;">',
  'Total Value by Area</label>',
  '<div id="area-legend" style="display:none;margin-top:8px;padding-top:8px;',
  'border-top:1px solid rgba(255,255,255,0.3);font-size:12px;line-height:1.5;">',
  '<div style="font-weight:bold;margin-bottom:5px;">Total Assessed Value</div>',
  legend_rows,
  '</div></div>'
)
```

### JavaScript toggle via onRender

```r
m <- m |>
  add_control(combined_control_html, position = "bottom-left") |>
  htmlwidgets::onRender("
    function(el, x) {
      var poll = setInterval(function() {
        if (!el.map || !el.map.isStyleLoaded()) return;
        clearInterval(poll);
        var map = el.map;
        var legend = document.getElementById('area-legend');
        var radios = el.querySelectorAll('input[name=\"map-layer\"]');
        radios.forEach(function(radio) {
          radio.addEventListener('change', function() {
            if (this.value === '3d') {
              map.setLayoutProperty('parcels-3d', 'visibility', 'visible');
              map.setLayoutProperty('area-choropleth', 'visibility', 'none');
              if (legend) legend.style.display = 'none';
            } else {
              map.setLayoutProperty('parcels-3d', 'visibility', 'none');
              map.setLayoutProperty('area-choropleth', 'visibility', 'visible');
              if (legend) legend.style.display = 'block';
            }
          });
        });
      }, 200);
    }
  ")
```

Key details:
- **`el.map`** is how mapgl's htmlwidget binding exposes the MapLibre map instance (found in `maplibregl-binding.js` line ~2616: `el.map = map`)
- **Polling with `setInterval`** is needed because the map may not be ready when `onRender` fires
- Layer IDs in `setLayoutProperty()` must exactly match the `id` parameter in `add_fill_layer()` / `add_fill_extrusion_layer()`
- Radio button `name` attribute must match the `querySelectorAll` selector

---

## Step 6: Summary Table

```r
summary_table <- parcels_all |>
  group_by(Area_Name) |>
  summarise(
    count        = n(),
    total_value  = sum(Value_Col, na.rm = TRUE),
    median_value = median(Value_Col, na.rm = TRUE),
    mean_value   = mean(Value_Col, na.rm = TRUE),
    max_value    = if (all(is.na(Value_Col))) NA_real_ else max(Value_Col, na.rm = TRUE),
    .groups = "drop"
  ) |>
  arrange(desc(total_value))

summary_table |>
  mutate(across(c(total_value, median_value, mean_value, max_value),
                ~dollar(.x, accuracy = 1))) |>
  knitr::kable(
    col.names = c("Area", "Parcels", "Total Assessed Value", "Median", "Mean", "Max"),
    align = c("l", "r", "r", "r", "r", "r")
  )
```

Remove `head(N)` to show all areas. The table is after the map in the document flow.

---

## Gotchas and Lessons Learned

### 1. Pandoc OOM with `embed-resources: true`
Do NOT use `embed-resources: true` when the HTML contains hundreds of MB of embedded GeoJSON. Pandoc tries to base64-encode everything and runs out of memory. Just omit it — the HTML is self-contained enough with inline GeoJSON.

### 2. `sf_use_s2(FALSE)` is required for geometry operations
Both `st_simplify()` and `group_by() |> summarise()` (dissolve) fail with s2 enabled on this data due to self-intersecting geometries. Always bracket geometry operations:
```r
sf_use_s2(FALSE)
# ... geometry operations ...
sf_use_s2(TRUE)
```
Always call `st_make_valid()` before `st_simplify()` or dissolve.

### 3. `add_control()` overwrites — only one custom control
`mapgl::add_control()` stores custom HTML under a single key. Calling it twice means only the last one renders. Combine all custom controls into one HTML string.

### 4. Inline R ordering in Quarto
Inline R (`` `r var` ``) in markdown text evaluates in document order. Variables must be defined in a code chunk that appears *before* the markdown that references them. If your heading uses `` `r asmt_year` ``, the chunk that sets `asmt_year` must come earlier in the file.

### 5. File size trade-offs

| Mode | HTML size | PMTiles | Total to host | Requirements |
|---|---|---|---|---|
| `use_local_data = TRUE` | ~284 MB | None | ~284 MB | None (works offline) |
| `use_local_data = FALSE` | ~17 MB | ~203 MB | ~220 MB | PMTiles in `docs/` via Git LFS |

For development and testing, use `TRUE` (embedded, works offline). For production deployment on GitHub Pages, use `FALSE` — the browser streams only the visible tiles on demand, so the user never downloads the full 203 MB.

### 6. GitHub Releases does NOT support CORS

**Do NOT host PMTiles on GitHub Releases.** GitHub Releases redirects to Azure Blob Storage which does not return `Access-Control-Allow-Origin` headers. The browser silently blocks PMTiles range requests, and the 3D parcel layer shows nothing — no error, just an empty map.

**The fix:** Serve PMTiles from the same origin as the HTML by placing it in `docs/` and using a **relative URL**:

```r
# WRONG — CORS blocked, parcels won't load
pmtiles_url <- "https://github.com/user/repo/releases/download/v1/data.pmtiles"

# CORRECT — same-origin, no CORS issue
pmtiles_url <- "mb-assessment-parcels.pmtiles"  # Relative to docs/index.html
```

Use Git LFS to push the file (typically >100 MB). GitHub Pages serves LFS-tracked files normally.

### 7. Winnipeg vs. Manitoba differences to expect

| Aspect | Manitoba (current) | Winnipeg (to adapt) |
|---|---|---|
| Data source | ArcGIS REST API (GeoPortal) | Socrata API (data.winnipeg.ca) |
| Dataset URL | `services.arcgis.com/.../ROLL_ENTRY/FeatureServer/0` | `data.winnipeg.ca/resource/{id}` |
| Pagination | `resultOffset` / `resultRecordCount` (max 2000) | `$offset` / `$limit` (max 50000) |
| Parcel count | ~437K | ~240K (estimate) |
| Area grouping | `Muni_Name_With_Typ` (186 municipalities) | Neighbourhood or ward |
| Assessment year field | `Asmt_Roll` ("2026 Tax Assessment Roll") | Check column names |
| Value field | `Total_Value` (string "$359,500") | Check — may be numeric already |
| Geometry format | GeoJSON from ArcGIS (WGS84) | GeoJSON from Socrata (WGS84) |
| Map center/zoom | `c(-98.0, 55.0)`, zoom 5 | `c(-97.14, 49.89)`, zoom 11 |
| Base value date | April 1, 2023 (same for both) | April 1, 2023 (same for both) |

---

## Rendering and Deployment

### Render

```bash
# Path to Quarto (bundled with RStudio on Windows)
"/c/Program Files/RStudio/resources/app/bin/quarto/bin/quarto.exe" render map-visualization.qmd

# Or if Quarto is on PATH
quarto render map-visualization.qmd
```

Output goes to `docs/index.html` per `_quarto.yml`.

### Deploy to GitHub Pages

Full workflow from render to live site:

```bash
# 1. Render with PMTiles mode
#    (set use_local_data <- FALSE in the QMD first)
quarto render map-visualization.qmd

# 2. Copy PMTiles to docs/ (if regenerated)
cp data/my-dataset.pmtiles docs/my-dataset.pmtiles

# 3. Stage, commit, push
git add docs/ .gitattributes map-visualization.qmd
git commit -m "Update map"
git push origin main
```

Then enable GitHub Pages in repo Settings → Pages → Source: `main` branch, `/docs` folder.

### Updating the data

When you re-download the source data:

1. Run the download script to get a new date-stamped parquet
2. Set `use_local_data <- TRUE` temporarily for testing (optional)
3. Render and verify the map locally
4. Switch back to `use_local_data <- FALSE`
5. Re-render (this regenerates PMTiles if the parquet is newer)
6. Copy the new PMTiles to `docs/`
7. Commit and push

---

## Required R Packages

```r
install.packages(c(
  "httr2",       # HTTP requests (download script)
  "readr",       # CSV parsing, parse_number()
  "dplyr",       # Data manipulation
  "sf",          # Spatial data
  "arrow",       # Parquet I/O
  "geoarrow",    # Geometry in parquet
  "cli",         # Console messages
  "fs",          # File system operations
  "glue",        # String interpolation
  "scales",      # dollar(), comma() formatting
  "stringr",     # String extraction
  "duckdb",      # (loaded but optional)
  "DBI",         # (loaded but optional)
  "mapgl",       # MapLibre GL JS bindings
  "htmlwidgets", # onRender for custom JS
  "knitr"        # kable() for tables
))

# freestiler (PMTiles generation) — from r-universe, not CRAN
install.packages("freestiler",
  repos = c("https://walkerke.r-universe.dev", "https://cloud.r-project.org"))
```

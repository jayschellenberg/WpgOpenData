# Open Data Spatial Pipeline Reference

Patterns, code templates, and lessons learned from the Winnipeg Assessment Parcels project.
Reusable for any Socrata-hosted open data with spatial geometry.

---

## 1. R Package Setup

### Install (run once)

```r
# CRAN packages
install.packages(c(
  "httr2", "readr", "dplyr", "sf", "arrow", "geoarrow",
  "cli", "fs", "glue", "ggplot2", "scales", "stringr",
  "duckdb", "DBI", "mapgl"
))

# freestiler from r-universe (not on CRAN)
install.packages("freestiler",
  repos = c("https://walkerke.r-universe.dev", "https://cloud.r-project.org"))
```

### Load order and known conflicts

```r
# Download script packages
library(httr2)    # HTTP requests with retry/throttle
library(readr)    # Fast CSV parsing
library(dplyr)    # Data manipulation
library(sf)       # Spatial features
library(arrow)    # Parquet I/O
library(geoarrow) # GeoParquet geometry support
library(cli)      # Progress messages
library(fs)       # File system operations
library(glue)     # String interpolation
library(ggplot2)  # Validation plots

# Visualization script packages (in addition to above)
library(duckdb)     # Required backend for freestile_query()
library(DBI)        # DuckDB dependency
library(freestiler) # Vector tiling (GeoParquet -> PMTiles)
library(mapgl)      # MapLibre GL JS 3D maps
library(scales)     # Dollar formatting
# WARNING: scales::number_format masks mapgl::number_format
# Use mapgl::number_format() explicitly in tooltip expressions
```

---

## 2. Socrata SODA API Patterns

### Endpoint URL structure

```
https://{domain}/resource/{dataset-id}.{format}
```

Formats: `.csv` (recommended for download), `.json`, `.geojson`

### Key query parameters

| Parameter | Purpose | Example |
|-----------|---------|---------|
| `$limit` | Max rows per request (default 1000, max ~50000) | `$limit=50000` |
| `$offset` | Skip N rows for pagination | `$offset=50000` |
| `$order` | Sort order (**critical for consistent pagination**) | `$order=:id` |
| `$select` | Column selection or aggregation | `$select=count(*) as total` |
| `$where` | Filter rows | `$where=year > 2020` |

### Get total row count first

```r
base_url <- "https://{domain}/resource/{dataset-id}"

count_resp <- request(glue("{base_url}.json")) |>
  req_url_query(`$select` = "count(*) as total") |>
  req_retry(max_tries = 3, backoff = ~10) |>
  req_perform()

total_rows <- count_resp |>
  resp_body_json() |>
  purrr::pluck(1, "total") |>
  as.integer()
```

### Pagination strategy

```r
batch_size <- 50000
n_batches <- ceiling(total_rows / batch_size)
offsets <- seq(0, by = batch_size, length.out = n_batches)
# Request each batch at: ?$limit=50000&$offset={0,50000,100000,...}&$order=:id
```

**Critical**: Always include `$order=:id` — without it, the API does not guarantee
consistent row ordering across paginated requests, leading to duplicates or gaps.

### CSV vs JSON vs GeoJSON tradeoffs

| Format | Wire size | Geometry format | Best for |
|--------|-----------|-----------------|----------|
| CSV | Smallest | WKT text column | Bulk download (recommended) |
| JSON | Medium | Nested object | API queries, small results |
| GeoJSON | Largest | Full FeatureCollection | Direct mapping (small datasets only) |

---

## 3. Download Script Template

### Batch downloader with retry/throttle

```r
download_batch <- function(base_url, batch_size, offset, batch_num, n_batches) {
  cli_alert_info(
    "Downloading batch {batch_num}/{n_batches} (offset {format(offset, big.mark = ',')})"
  )

  resp <- request(glue("{base_url}.csv")) |>
    req_url_query(
      `$limit` = batch_size,
      `$offset` = offset,
      `$order` = ":id"
    ) |>
    req_retry(max_tries = 3, backoff = ~10) |>
    req_throttle(rate = 1 / 2) |>  # 1 request per 2 seconds
    req_perform()

  raw_csv <- resp_body_string(resp)
  batch_df <- read_csv(raw_csv, show_col_types = FALSE,
                       col_types = cols(
                         # Force ID-like columns to character to prevent
                         # numeric parsing of leading-zero IDs
                         roll_number = col_character()
                         # Add other ID columns here as needed
                       ))

  # Check for parsing issues
  probs <- problems(batch_df)
  if (nrow(probs) > 0) {
    cli_alert_warning("Batch {batch_num}: {nrow(probs)} parsing issue(s)")
  }

  cli_alert_success(
    "Batch {batch_num}: {format(nrow(batch_df), big.mark = ',')} rows"
  )
  batch_df
}
```

### Date-stamped output filenames

```r
output_file <- fs::path("data", glue("{dataset_name}-{Sys.Date()}.parquet"))
```

### Skip-if-exists pattern

```r
force_download <- FALSE

if (!fs::file_exists(output_file) || force_download) {
  # ... download logic ...
} else {
  cli_alert_info("Skipping download -- file exists: {output_file}")
}
```

---

## 4. GeoParquet Pipeline

### Writing GeoParquet (after downloading CSV with WKT geometry)

```r
library(arrow)
library(geoarrow)  # MUST be loaded for geometry column support

# Convert WKT text to sf geometry
parcels_sf <- st_as_sf(combined_df, wkt = "geometry", crs = 4326)

# Write -- geoarrow handles the geometry column automatically
write_parquet(parcels_sf, output_file)
```

### Reading GeoParquet back (GOTCHA)

```r
# WRONG -- st_as_sf() alone fails on geoarrow_vctr columns:
# parcels_sf <- read_parquet(file) |> st_as_sf()

# CORRECT -- must convert geoarrow geometry to sfc first:
parcels_sf <- read_parquet(file) |>
  mutate(geometry = st_as_sfc(geometry)) |>
  st_as_sf()
```

### Handling missing geometry

```r
has_geom <- !is.na(df$geometry) & df$geometry != ""
df_with_geom <- df[has_geom, ]
df_no_geom <- df[!has_geom, ]

parcels_sf <- st_as_sf(df_with_geom, wkt = "geometry", crs = 4326)

if (nrow(df_no_geom) > 0) {
  cli_alert_warning("{nrow(df_no_geom)} rows have missing geometry")
  write_csv(df_no_geom, "data/no-geometry-records.csv")
}
```

---

## 5. freestiler Vector Tiling

### Use `freestile()` with sf objects (RECOMMENDED on Windows)

```r
library(freestiler)

# Select only the columns needed for visualization (keeps tiles lean)
parcels_sf <- read_parquet(input_file) |>
  mutate(geometry = st_as_sfc(geometry)) |>
  st_as_sf() |>
  select(id_col, value_col, category_col, label_col)

freestile(
  input = parcels_sf,
  output = "data/output.pmtiles",
  layer_name = "my_layer",   # becomes source_layer in mapgl
  min_zoom = 8L,
  max_zoom = 14L
)
```

### Why NOT `freestile_query()` with geoarrow parquet

`freestile_query()` uses DuckDB to read the parquet file. DuckDB sees the geoarrow
geometry column as `STRUCT(x DOUBLE, y DOUBLE)[][][]` instead of recognizing it as
geometry. Error: "No geometry column found in query result."

**Workaround**: Load parquet into R as sf, then pass to `freestile()`.
This uses more memory but works reliably.

### `serve_tiles()` not available on Windows binary

As of freestiler 0.1.0, the Windows r-universe binary does not export `serve_tiles()`.
**Workaround**: Pass the sf object directly to `add_fill_extrusion_layer(source = sf_object)`
in mapgl. This embeds GeoJSON in the HTML widget (large file, but works for local viewing).

### Tile size guidelines

For the Winnipeg dataset (245k multipolygon features):
- GeoParquet input: ~35 MB
- PMTiles output: ~47 MB
- Generation time: ~51 seconds
- Zoom 8-14, MLT format (default), 359 tiles

---

## 6. mapgl 3D Visualization

### Basic 3D map setup

```r
library(mapgl)

maplibre(
  style = carto_style("dark-matter"),  # dark basemap makes extrusions pop
  center = c(lon, lat),
  zoom = 11,
  pitch = 45,           # 3D tilt angle (0-60)
  bearing = -15,         # rotation
  projection = "mercator" # REQUIRED for fill-extrusion (globe has artifacts)
) |>
  add_fill_extrusion_layer(
    id = "layer-3d",
    source = sf_object,  # pass sf directly (workaround for serve_tiles)
    fill_extrusion_height = interpolate(
      column = "numeric_column",
      values = c(min_val, mid_val, max_val),
      stops = c(0, 200, 2000)        # pixel heights
    ),
    fill_extrusion_color = match_expr(
      column = "category_column",
      values = category_vector,
      stops = color_vector,
      default = "#95a5a6"
    ),
    fill_extrusion_opacity = 0.8,
    tooltip = concat(
      "<strong>", get_column("label_col"), "</strong><br>",
      "Value: $", get_column("value_col"), "<br>",
      "Category: ", get_column("category_col")
    ),
    hover_options = list(
      fill_extrusion_color = "#ffffff"
    )
  )
```

### Expression helpers (from mapgl, not MapLibre JSON)

| Function | Use | Example |
|----------|-----|---------|
| `interpolate()` | Continuous numeric mapping | Height from value |
| `match_expr()` | Categorical mapping | Color from category |
| `step_expr()` | Stepped/binned mapping | Discrete height classes |
| `concat()` | Build tooltip HTML | Combine text + columns |
| `get_column()` | Reference a data column | Inside `concat()` |
| `mapgl::number_format()` | Format numbers in tooltips | Currency, decimals |

### Dynamic color palette for many categories

```r
palette <- c("#e74c3c", "#3498db", "#2ecc71", "#f39c12", "#9b59b6",
             "#1abc9c", "#e67e22", "#ec407a", "#26c6da", "#ffca28",
             "#8d6e63", "#78909c")

use_codes <- category_counts$category_column
use_colors <- rep_len(palette, length(use_codes))
```

---

## 7. Gotchas & Lessons Learned

### Issue: `roll_number` parsed as numeric
**Symptom**: Leading zeros stripped from ID columns.
**Fix**: `col_types = cols(roll_number = col_character())` in `read_csv()`.

### Issue: `read_parquet() |> st_as_sf()` fails
**Symptom**: Error about geoarrow_vctr not being recognized.
**Fix**: `mutate(geometry = st_as_sfc(geometry)) |> st_as_sf()` — convert geoarrow to sfc first.

### Issue: `freestile_query()` says "No geometry column found"
**Symptom**: DuckDB reads geoarrow geometry as nested structs, not geometry type.
**Fix**: Use `freestile(input = sf_object)` instead. Load parquet into R as sf first.

### Issue: `serve_tiles()` not found
**Symptom**: `could not find function "serve_tiles"` on Windows.
**Fix**: Pass sf object directly to `add_fill_extrusion_layer(source = sf_object)`.

### Issue: Fill-extrusion rendering artifacts
**Symptom**: Warning about globe projection.
**Fix**: `projection = "mercator"` in `maplibre()`.

### Issue: `scales::number_format` masks `mapgl::number_format`
**Symptom**: Tooltip formatting doesn't work as expected.
**Fix**: Use `mapgl::number_format()` explicitly, or load `scales` before `mapgl`.

### Issue: `max()` returns `-Inf` for all-NA groups
**Symptom**: Warning in `summarise()` for groups with no non-missing values.
**Fix**: `max_value = if (all(is.na(col))) NA_real_ else max(col, na.rm = TRUE)`.

---

## 8. Project Structure Template

```
MyProject/
  .gitignore
  download-{dataset}.qmd     # Data pipeline (eval: false by default)
  map-{dataset}.qmd          # 3D visualization
  data/
    {dataset}-YYYY-MM-DD.parquet  # Downloaded GeoParquet (date-stamped)
    {dataset}.pmtiles              # Generated vector tiles
```

### .gitignore

```gitignore
# Data files (too large for git)
data/*.parquet
data/*.geojson
data/*.csv
data/*.pmtiles
data/temp/

# Quarto render output
_site/
_freeze/
*.html

# R artifacts
.Rhistory
.RData
.Rproj.user/

# OS files
Thumbs.db
.DS_Store
```

### Quarto YAML for download script (eval: false by default)

```yaml
---
title: "Download {Dataset Name}"
format:
  html:
    toc: true
    code-fold: show
    embed-resources: true
execute:
  eval: false
  echo: true
---
```

### Quarto YAML for visualization script (eval: true)

```yaml
---
title: "{Dataset Name} -- 3D Map"
format:
  html:
    toc: true
    code-fold: true
execute:
  echo: true
---
```

---

## 9. Winnipeg Project Specifics (for reference)

- **Portal**: data.winnipeg.ca (Socrata)
- **Dataset ID**: `d4mq-wa44`
- **API base**: `https://data.winnipeg.ca/resource/d4mq-wa44`
- **Rows**: 245,136 (as of 2026-03-10)
- **Columns**: 71 (including MultiPolygon geometry)
- **Key columns**: `roll_number`, `total_assessed_value`, `property_use_code`, `full_address`, `building_type`, `year_built`, `total_living_area`, `neighbourhood_area`, `geometry`
- **Property use codes**: 98 distinct values (e.g., "RESSD - DETACHED SINGLE DWELLING")
- **GeoParquet size**: ~35 MB
- **PMTiles size**: ~47 MB (zoom 8-14, MLT format)
- **Map center**: `c(-97.1384, 49.8951)`, zoom 11

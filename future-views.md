# Future Map Views — Saved for Later

## Assessment Volatility by Neighbourhood

Choropleth of IQR (interquartile range) of % assessment change within each neighbourhood.
Higher IQR = greater spread — some parcels saw large increases while others stayed flat.
Uses a sequential fill scale.

### Data

The `change_by_neighbourhood` dataframe (computed in `compute-change` chunk) already contains `iqr_pct`, `q25_pct`, and `q75_pct`. The `change_neighbourhood_sf` spatial object has these joined with dissolved neighbourhood geometries.

### Interactive Map (mapgl)

```r
iqr_vals <- change_neighbourhood_sf$iqr_pct[!is.na(change_neighbourhood_sf$iqr_pct)]
q_breaks_b <- unique(as.numeric(quantile(iqr_vals, probs = c(0, 0.25, 0.5, 0.75, 1.0))))
if (length(q_breaks_b) < 2) {
  q_breaks_b <- c(min(iqr_vals), max(iqr_vals))
}
iqr_colors <- c("#fef0d9", "#fdcc8a", "#fc8d59", "#d7301f")[seq_len(length(q_breaks_b))]

maplibre(
  style = carto_style("dark-matter"),
  center = c(-97.14, 49.89),
  zoom = 11,
  pitch = 0,
  bearing = 0,
  projection = "mercator"
) |>
  add_navigation_control(position = "top-right") |>
  add_fill_layer(
    id = "iqr-change",
    source = change_neighbourhood_sf,
    fill_color = interpolate(
      column = "iqr_pct",
      values = q_breaks_b,
      stops = iqr_colors
    ),
    fill_opacity = 0.85,
    tooltip = concat(
      "<strong>", get_column("neighbourhood_area"), "</strong><br>",
      "IQR of Change: ", get_column("iqr_pct_fmt"), "<br>",
      "Q25: ", get_column("q25_pct"), "% \u2013 Q75: ", get_column("q75_pct"), "%<br>",
      "Median Change: ", get_column("median_pct_fmt"), "<br>",
      "Parcels: ", get_column("change_parcel_count")
    ),
    hover_options = list(
      fill_color = "#ffffff",
      fill_opacity = 1
    )
  ) |>
  add_line_layer(
    id = "neighbourhood-borders-b",
    source = change_neighbourhood_sf,
    line_color = "#ffffff",
    line_width = 0.5,
    line_opacity = 0.3
  )
```

### Static Map (ggplot2 — for PNG export)

```r
#| fig-width: 10
#| fig-height: 10
#| dpi: 150

p_b <- ggplot(change_neighbourhood_sf) +
  geom_sf(aes(fill = iqr_pct), color = "grey30", linewidth = 0.15) +
  scico::scale_fill_scico(
    palette = "lajolla",
    direction = 1,
    name = "IQR of % Change",
    labels = function(x) paste0(round(x, 0), "%")
  ) +
  annotation_scale(
    location = "bl",
    width_hint = 0.25,
    style = "ticks",
    text_col = "grey30",
    line_col = "grey30"
  ) +
  annotation_north_arrow(
    location = "tr",
    which_north = "true",
    height = unit(1, "cm"),
    width = unit(1, "cm"),
    style = north_arrow_minimal(text_col = "grey30", line_col = "grey30")
  ) +
  labs(
    title = "Assessment Volatility by Neighbourhood",
    subtitle = glue("IQR of % change in assessed value (2026 \u2192 2027 proposed)\nHigher IQR = greater spread within neighbourhood"),
    caption = glue("Source: City of Winnipeg Open Data \u2022 Downloaded {file_date}")
  ) +
  theme_minimal(base_size = 12) +
  theme(
    axis.text = element_blank(),
    axis.title = element_blank(),
    panel.grid = element_blank(),
    plot.title = element_text(face = "bold", size = 16),
    plot.subtitle = element_text(size = 11, color = "grey40"),
    plot.caption = element_text(size = 8, color = "grey50"),
    legend.position = "bottom",
    legend.key.width = unit(2.5, "cm"),
    legend.key.height = unit(0.4, "cm")
  )

p_b

ggsave("exports/assessment-volatility-iqr.png", p_b,
       width = 10, height = 10, dpi = 150, bg = "white")
```

### Chunk options for QMD

```yaml
#| label: map-b-mapgl   (interactive)
#| label: map-b-ggplot  (static)
#| fig-width: 10
#| fig-height: 10
#| dpi: 150
```

### Prerequisites

The `iqr_pct`, `q25_pct`, `q75_pct` columns are already computed in the `compute-change` chunk's `change_by_neighbourhood` summary and joined to `change_neighbourhood_sf`. No additional data loading needed — just paste the map chunks back in after the "Where Assessments Jumped Most" section.

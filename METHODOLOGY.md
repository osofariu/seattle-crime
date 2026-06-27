# Methodology

How each figure in the dashboard is computed. The aim is a transparent, reproducible
neighborhood picture from the City of Seattle's open crime data; the definitions below
let you judge the numbers for yourself.

## Data preparation

- **Source:** [SPD Crime Data, 2008–Present](https://data.seattle.gov/Public-Safety/SPD-Crime-Data-2008-Present/tazs-3rd5/about_data),
  classified with the FBI's [NIBRS](https://www.fbi.gov/how-we-can-help-you/more-fbi-services-and-information/ucr/nibrs) scheme.
- **Time window:** the last `YEARS_BACK` calendar years (default 10), snapped to
  January 1 of `current_year − YEARS_BACK` so the first year is complete. The current
  year is partial and is labeled as such.
- **Coordinates:** `Latitude`/`Longitude` are coerced to numbers and clipped to a
  Seattle bounding box. Rows that are `REDACTED` (~14.5%) or fall outside the box are
  dropped. The retained set is referred to below as `geo`.
- Locations are published at **block level** (snapped to block centroids), which is why
  the maps look gridded rather than continuous.

## Proximity filter

For a geocoded address `(lat₀, lon₀)`, the great-circle distance to every incident is
computed with the haversine formula (Earth radius 3958.8 mi). Incidents within
`RADIUS_MILES` (default 0.5) form the working set, `nearby`. This is an exact
point-to-point distance centered on the address, not a coarse beat / neighborhood /
precinct bucket.

At this radius the choice of formula barely matters — a flat-plane (Pythagorean)
distance would differ by inches. Haversine is just the standard, correct-by-default
way to measure it, which also avoids hand-tuning a latitude correction (a degree of
longitude covers less ground than a degree of latitude this far north).

## Benchmark — "N× the citywide average"

The benchmark answers: *how does the incident count near this address compare to a
typical Seattle area of the same size?*, assuming the city's overall incident density.

- `N_city` = total incidents citywide over the window — every row in `geo`
  (after the date filter and coordinate cleanup described above).
- `N_local` = incidents near the address — every row in `nearby` (within the radius).
- Circle area: `A = π · r²` (e.g. ~0.79 mi² for a 0.5-mi radius).
- `S` = Seattle land area ≈ **83.8 mi²**.
- Citywide density: `N_city / S` (incidents per square mile).
- Expected count for an average circle this size: `E = (N_city / S) · A`.
- Ratio: `N_local / E`.

Both `N_local` and `N_city` are counted over the **same** time window, so the window
cancels and the ratio is time-independent. Equivalently:

```
ratio = (N_local / N_city) · (S / A)
```

The `incidents/yr` figure shown alongside is `N_local / span_years`, where
`span_years` is the date range covered by `geo`.

**Assumptions / limits.** This treats incidents as if they were spread uniformly
across the city (they are not — surfacing that difference is the whole point), and it
divides by land area even though a given circle may include water or parkland. Read the
ratio as a rough "how does this compare," not a precise percentile.

## Crimes per year (violent vs non-violent)

- **Bars:** incidents in `nearby` grouped by offense year, stacked into violent
  (`Offense Category == "VIOLENT CRIME"`) and non-violent (everything else). The full
  bar height is the yearly total, so totals stay comparable across years.
- **% violent line:** `violent / total × 100` per year, drawn on a second axis that is
  auto-scaled to the data (not 0–100 %), color-matched to the violent bars.

## Top offenses

Categories are the 10 most common `Offense Sub Category` values **in the past year**.

- **Left (counts):** each category's incident count over the past year.
- **Right (trend):** for each window `W ∈ {10, 5, 1}` years, the average
  `incidents/year = (count in the last W years) / W`, for the same categories in the
  same order. Using per-year **rates** (not raw counts) makes the windows comparable:
  if the 1-year bar exceeds the 5- and 10-year bars, that offense is rising.

Two `Offense Sub Category` values are non-specific catch-alls and should not be
over-read: **`999`** ("Not Reportable to NIBRS") and **`ALL OTHER`** (NIBRS code `90Z`).

## By hour of day

For each period (past 5 years, past year), the **share** of that period's incidents in
each hour is `count_in_hour / total × 100`. Each curve is smoothed with a centered
**3-hour circular moving average** (it wraps around midnight) so a single sparse hour
does not spike the line. Plotting percentages — rather than counts — lets the two
periods be compared despite very different totals.

## Location map (proportional symbols)

Incidents are collapsed to **unique block points**; each point's rate is
`count / period_years`. Marker **area** is proportional to that rate, scaled by a single
maximum shared across both periods:

```
size = clip( rate / rate_max × 170 , min = 4 )
```

So a dot of a given size means the same incidents/year in either column, and area (not
overplotted opacity) encodes how busy each spot is.

## Density heatmap (incidents per year)

A hexbin (grid size 40) over a fixed extent. Each incident contributes `1 / period_years`
to its cell, summed, so every hexagon reads as **incidents per year**. Both periods share
one color scale, and its maximum is **clipped at the 97th percentile** of all cell values
so a single hotspot does not wash out everything else (the colorbar's arrow marks the
clip). Cells with no incidents are left blank.

## Gun-related incidents

Incidents whose `Shooting Type Group` is not `-` (shots-fired, non-fatal injury, fatal
injury), counted per year and stacked by severity. The summary line flags an **uptick**
when the last-year count is ≥ 2 **and** more than 1.5× the average annual count over the
prior years — a deliberately conservative rule, because gun counts within one small area
are noisy.

## Known limitations

- **Counts are a lower bound.** ~14.5 % of records have no usable coordinates and are
  excluded from everything spatial.
- **Block-snapped coordinates** mean the maps show approximate, gridded locations, not
  exact addresses.
- **Recent reporting lag.** The latest weeks/months are typically under-reported (records
  are still being entered), so the past-year and current-year figures can read low.
- **The benchmark assumes uniform citywide density** (see above), so it is an intuition
  aid, not a statistic.

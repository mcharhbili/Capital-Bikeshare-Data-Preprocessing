# Capital Bikeshare Data Preprocessor

Notebooks and utilities for cleaning, mapping, and aggregating [Capital Bikeshare](https://capitalbikeshare.com/system-data) trip data — reconciling historical trip records against the current live station network, and producing analysis-ready pickup/dropoff datasets.

## Background

The raw trip/station data isn't part of this repo — it's produced by a separate project, [**Capital-Bikeshare-Data-Extractor**](https://github.com/mcharhbili/Capital-Bikeshare-Data-Extractor) (bundled here locally as [`capital_bikeshare_extractor/`](capital_bikeshare_extractor/)), which syncs trip data from Capital Bikeshare's public S3 archive and live station info from GBFS into a Parquet layout. Trip data spans **2010–2026**; the station reference table is a **live snapshot** of the current GBFS station feed — it only reflects stations that exist *today*.

Because of that gap, a large share of historical trips reference stations (by name or by ID) that have since been renamed, renumbered, or retired and no longer appear in the live snapshot. This project's notebooks quantify that gap, close as much of it as practical, and turn the result into clean per-station time series.

## Project structure

```
notebooks/                       # the actual pipeline — run in order, see below
  01_station_mapping_check.ipynb
  02_station_id_mapping_check.ipynb
  03_station_id_crosswalk.ipynb
  04_station_map.ipynb
  05_bike_movement_aggregates.ipynb

capital_bikeshare_extractor/     # sibling tool that syncs raw trip/station data from S3 + GBFS (own README)
capital_bikeshare_data_preprocessing/  # reserved for a future installable package (currently empty)
```

Two more folders are used by the notebooks but **do not exist in this repo** — see [Data folders](#data-folders) below:

- `raw_data/` — inputs: `raw_trips/` (partitioned parquet), `stations.parquet`, and `station_id_crosswalk.csv` (generated + hand-edited, see notebook 03).
- `processed_data/` — outputs of notebook 05: `daily_bike_movement.parquet`, `hourly_bike_movement.parquet`.

## The station-mapping problem

Trips carry two ways to reference a station: a **name** (`start_station_name` / `end_station_name`) and an **ID** (`start_station_id` / `end_station_id`). Matching either against `stations.parquet` is not a straightforward join:

- The station table has two ID-like columns — `station_id` (a GBFS UUID, unrelated to trip data) and `short_name` (a numeric/alphanumeric code that *is* the real join key for trip IDs).
- `start_station_id` / `end_station_id` are stored as `int64` in older trip partitions but switch to alphanumeric strings from 2021-02 onward (e.g. `MTL-ECO5-03`), so reading the full partitioned dataset requires an explicit schema override.
- Matching by **name** resolves ~68% of trip rows; matching by **ID** (against `short_name`) resolves ~81%, plus another small increment from manually crosswalking historical IDs to their current equivalents.

## Notebooks

Run in order — each one builds on outputs from the previous:

| # | Notebook | What it does |
|---|---|---|
| 01 | `01_station_mapping_check.ipynb` | Checks trip **station names** against `stations.name`: exact-match coverage, whitespace/casing normalization, `difflib`-based closest-match suggestions for true mismatches, and the row-level match rate (~67.8%). |
| 02 | `02_station_id_mapping_check.ipynb` | Same check using **station IDs** instead of names (`start_station_id`/`end_station_id` ↔ `stations.short_name`) — establishes this as the better join key (~81.3% row match rate) and profiles the unmatched IDs (first/last appearance, age, row volume, average trip duration) to gauge which ones are worth chasing down. |
| 03 | `03_station_id_crosswalk.ipynb` | Generates fuzzy-match candidates for unmatched IDs and maintains a **persistent, human-reviewable crosswalk CSV** (`station_id_crosswalk.csv`, in your `raw_data/` folder). Re-running it preserves any manually-resolved rows, refreshes only what's still unresolved, and reports the resulting match-rate improvement. |
| 04 | `04_station_map.ipynb` | Interactive [folium](https://python-visualization.github.io/folium/) map of all current reference stations, sized/colored by capacity. Also saves a static `notebooks/station_map.html`. |
| 05 | `05_bike_movement_aggregates.ipynb` | Applies the ID + crosswalk resolution to build the final analysis-ready datasets: per-station **daily** and **hourly** pickup/dropoff counts, written to `processed_data/`. |

### The crosswalk workflow (notebook 03)

`station_id_crosswalk.csv` (written into your `raw_data/` folder) is a durable, editable mapping — not a one-off computation:

1. Run the notebook. Unmatched IDs get written out with up to 3 fuzzy-match candidates (name + similarity score) and a blank `chosen_short_name` column.
2. Open the CSV and fill in `chosen_short_name` (copy a candidate, enter one you found yourself, or type `RETIRED` for stations with no current equivalent). Rows are sorted by row volume, so resolving the top few has the biggest impact.
3. Save, then re-run the notebook's later cells. Your manual edits are preserved — only still-unresolved rows get their candidates refreshed, and any newly-discovered unmatched IDs get appended.

## Output datasets

Both outputs from notebook 05 share the same shape and resolution logic (station IDs resolved via identity match + the crosswalk; trip rows that don't resolve on a given endpoint are excluded from that endpoint's counts, not bucketed as "unknown"):

- **`daily_bike_movement.parquet`**: `station, date, n_pickups, n_dropoffs`
- **`hourly_bike_movement.parquet`**: `station, datetime, n_pickups, n_dropoffs`

`n_pickups` counts trips *starting* at that station/time; `n_dropoffs` counts trips *ending* there.

## Setup

Requires Python 3.10+ (pandas, pyarrow, folium):

```bash
pip install pandas pyarrow folium
```

## Data folders

**`raw_data/` and `processed_data/` are not part of this repo** — they're gitignored and every user must point their own copies at the notebooks:

- `raw_data/` — you populate this yourself by running [Capital-Bikeshare-Data-Extractor](https://github.com/mcharhbili/Capital-Bikeshare-Data-Extractor) (bundled locally at [`capital_bikeshare_extractor/`](capital_bikeshare_extractor/) — see its own README) against wherever you want the extracted trip/station Parquet files to live.
- `processed_data/` — created automatically by notebook 05 the first time it runs, at whatever path you configure.

**Before running any notebook, edit its path-setup cell near the top** (the `ROOT` / `RAW_DATA` / `RAW_TRIPS_PATH` / `STATIONS_PATH` / `PROCESSED_DATA` assignments, depending on the notebook) to point at your actual `raw_data/` and `processed_data/` locations — the defaults assume they're plain siblings of `notebooks/`, which won't be true for every setup. Each notebook resolves these paths independently (there's no shared config), so update the cell in every notebook you run.

## Data provenance

- Trip data: [Capital Bikeshare System Data](https://capitalbikeshare.com/system-data), synced via [Capital-Bikeshare-Data-Extractor](https://github.com/mcharhbili/Capital-Bikeshare-Data-Extractor).
- Station data: Capital Bikeshare's live [GBFS](https://gbfs.org/) `station_information` feed.

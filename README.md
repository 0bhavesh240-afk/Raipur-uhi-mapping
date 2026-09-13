# Raipur Urban Heat Island (UHI) Mapping

Map and quantify the Urban Heat Island effect in **Raipur, Chhattisgarh, India**
by fusing free satellite thermal imagery (Landsat 8/9 TIRS and Sentinel-3 SLSTR,
both available via the **Copernicus Data Space Ecosystem** / USGS Earth
Explorer / Google Earth Engine) with your **own ground-based temperature
readings** for calibration and validation.

> Built as a portfolio / application project. Fully modular Python package —
> clone, drop in your credentials and ground data, and run the pipeline.

---

## What this project does

1. **Downloads satellite thermal data** for a bounding box around Raipur
   (default AOI: `81.45–81.80°E, 21.10–21.35°N`) for a chosen date range:
   - Landsat 8/9 Collection-2 Level-2 **Surface Temperature** product (`ST_B10`)
   - Sentinel-3 SLSTR **Level-2 LST** product
   - Sentinel-2 L2A optical bands (for NDVI / NDBI / land-cover context)
2. **Derives Land Surface Temperature (LST)** in °C, cloud-masks scenes, and
   mosaics/composites them over the AOI.
3. **Computes spectral indices** — NDVI (vegetation), NDBI (built-up), and a
   simple Urban Index — used to explain *why* some pixels run hotter.
4. **Ingests your ground-truth readings** (a simple CSV of lat, lon,
   timestamp, temperature °C that you collect with a thermometer / data
   logger / weather-station app) and matches each point to the nearest
   satellite pixel in space and time.
5. **Validates** satellite-derived LST against ground truth (MAE, RMSE, R²,
   bias) and optionally fits a local calibration (linear regression).
6. **Quantifies the UHI intensity**: mean LST of urban pixels − mean LST of
   a rural/vegetated reference ring around the city, plus zonal statistics
   per ward/land-use class if you supply a shapefile.
7. **Visualizes** everything: LST heatmaps, NDVI/NDBI maps, UHI intensity
   map, ground-vs-satellite scatter/validation plots, and an interactive
   Folium map (`outputs/uhi_map.html`).

---

## Project structure

```
raipur-uhi-mapping/
├── config/
│   └── config.yaml                # AOI, date range, paths, thresholds
├── data/
│   ├── raw/                       # downloaded satellite scenes (gitignored)
│   ├── processed/                 # LST / NDVI / NDBI rasters (gitignored)
│   └── ground_truth/
│       └── ground_temperature_template.csv
├── src/
│   ├── data_acquisition/
│   │   ├── copernicus_download.py   # Sentinel-2/3 via Copernicus Data Space
│   │   ├── gee_download.py          # Landsat 8/9 + Sentinel via Earth Engine
│   │   └── landsat_stac_download.py # Landsat via USGS/AWS STAC (no GEE needed)
│   ├── preprocessing/
│   │   ├── lst_calculation.py       # DN -> radiance -> brightness temp -> LST
│   │   └── spectral_indices.py      # NDVI, NDBI, Urban Index
│   ├── ground_truth/
│   │   └── ground_data_processor.py # load + spatiotemporal match to pixels
│   ├── analysis/
│   │   ├── validation.py            # MAE/RMSE/R2, calibration regression
│   │   └── uhi_calculator.py        # UHI intensity, zonal stats
│   └── visualization/
│       └── mapping.py               # static + interactive (Folium) maps
├── scripts/
│   └── run_pipeline.py              # CLI: end-to-end orchestration
├── notebooks/
│   └── 01_exploratory_analysis.ipynb
├── tests/
│   └── test_indices.py
├── .github/workflows/ci.yml         # lint + unit tests on push
├── requirements.txt
├── .gitignore
├── LICENSE
└── README.md
```

---

## Data sources (all free)

| Source | Sensor | What you get | Access |
|---|---|---|---|
| [Copernicus Data Space Ecosystem](https://dataspace.copernicus.eu/) | Sentinel-2 L2A | 10 m optical (NDVI/NDBI) | Free account, OAuth2 token |
| [Copernicus Data Space Ecosystem](https://dataspace.copernicus.eu/) | Sentinel-3 SLSTR L2 LST | 1 km thermal LST, already geophysical product | Free account |
| [USGS EarthExplorer](https://earthexplorer.usgs.gov/) / [Google Earth Engine](https://earthengine.google.com/) | Landsat 8/9 Collection-2 L2 | 30 m (resampled from 100 m) surface temperature `ST_B10` | Free account |
| Your field work | Handheld IR thermometer / logger / weather stations | Ground-truth °C at known lat/lon/time | — |

Landsat is a USGS/NASA mission (not Copernicus) but is just as free and is the
best 30 m-resolution thermal option for a city-scale study — the project
supports both so you can cross-check.

---

## Setup

```bash
git clone https://github.com/<your-username>/raipur-uhi-mapping.git
cd raipur-uhi-mapping
python -m venv venv
source venv/bin/activate        # venv\Scripts\activate on Windows
pip install -r requirements.txt
cp config/config.yaml.example config/config.yaml   # if you rename the example
```

### Credentials

Create a `.env` file in the project root (never commit this — it's already
gitignored):

```
COPERNICUS_USERNAME=your_email@example.com
COPERNICUS_PASSWORD=your_password
EARTHENGINE_PROJECT=your-gee-project-id   # only if using gee_download.py
```

Sign up (free, instant) at:
- Copernicus: https://dataspace.copernicus.eu/
- Google Earth Engine: https://earthengine.google.com/signup/ (needed only for `gee_download.py`)

---

## Add your ground-truth readings

Fill in `data/ground_truth/ground_temperature_template.csv`:

```csv
station_id,latitude,longitude,datetime_utc,temperature_c,notes
GT001,21.2514,81.6296,2026-04-15T07:30:00,32.4,near Marine Drive, shaded
GT002,21.2100,81.6600,2026-04-15T07:45:00,29.8,outskirts / green cover
...
```

Collect these as close as possible to the satellite overpass time (Landsat:
~10:30 local time; Sentinel-3: varies, check the metadata) for a fair
comparison — LST and 2 m air temperature are related but not identical, so
note this caveat in your write-up.

---

## Run the full pipeline

```bash
python scripts/run_pipeline.py \
    --start-date 2026-03-01 --end-date 2026-05-31 \
    --source landsat \
    --ground-truth data/ground_truth/ground_temperature_template.csv \
    --output-dir outputs/
```

This will:
1. Query & download the least-cloudy scene(s) in the date range
2. Compute LST, NDVI, NDBI over the Raipur AOI
3. Match ground-truth points to pixels and compute validation metrics
4. Compute UHI intensity and zonal stats
5. Write PNG maps + an interactive `outputs/uhi_map.html` + a `outputs/report.md` summary

Each module can also be run/imported independently — see the docstrings.

---

## Methodology notes (for your write-up / report)

- **LST from Landsat**: uses the USGS Collection-2 Level-2 Science Product
  (`ST_B10`), which already applies the single-channel algorithm with ASTER
  GED emissivity — scale/offset only, no manual radiative-transfer step
  needed (kept in `lst_calculation.py` for transparency, plus an optional
  manual mono-window implementation from raw Band 10 if you want to show the
  full math in a report).
- **Urban Heat Island Intensity (UHI)**: defined here as
  `UHI = mean(LST_urban_core) − mean(LST_rural_reference)`, where the rural
  reference is a buffer ring of predominantly vegetated/agricultural pixels
  (NDVI > 0.4) outside the built-up core (NDBI > NDVI).
- **Validation**: satellite LST (skin temperature) vs. ground sensor
  readings (often near-surface air temperature) will not match 1:1 — report
  MAE/RMSE/R² and discuss the physical difference rather than treating it as
  pure "error."

---

## License

MIT — see `LICENSE`. Attribution to Copernicus (Sentinel data, contains
modified Copernicus Sentinel data) and USGS/NASA (Landsat) as required by
their data policies.

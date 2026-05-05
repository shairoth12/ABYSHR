# AbyssMorph — Abyssal Hill Shape Analysis

Pipeline for detecting and characterizing **abyssal hill morphology** from multibeam sonar data. Implements the Ridgelet Transform method (Downey & Clayton, *G3*, 2007) extended with cross-section shape analysis (Roth et al., 2019).

## Requirements

| Tool | Version |
|---|---|
| GMT (Generic Mapping Tools) | 5 or later |
| MB-System (`mbgrid3`, `mbdatalist`) | 5.5 or later |
| MATLAB | 2015a or later |
| Python | 2.7 |
| gcc | any recent version |

Python packages: `numpy`, `matplotlib`, `subprocess`, `bokeh`, `jupyter`

## Repository Layout

```
Codes/     Working directory — run the pipeline from here
Example/   Pre-populated outputs for a Mid-Atlantic Ridge test track
           Run here first to validate your environment
```

## Quick Start

Run the Example first to confirm your environment is set up correctly, then apply the same steps to your own data in `Codes/`.

---

## Stage 1 — Radon Transform

**Script:** `Codes/radon.pl`

### Setup

- `MB_files/` — directory of multibeam data files (`.mbxxx` format, not `.fbt`)
- `file_list` — full paths to MB data files, one per line
- `blacklist.asc` — regions to exclude (`lon_min lon_max lat_min lat_max`, one per line). Must exist even if empty.
- `track1`, `track2`, … — lat/lon waypoints along the analysis track. Can be the actual ship track or a synthetic flow-line track orthogonal to abyssal hill fabric.
- `multibeam.cpt` — GMT color palette (e.g. `makecpt -Z -Chaxby -T-4650/-1850/200`)
- `output/` — directory for Stage 1 outputs (created automatically on first run)

### Run

1. Edit the user-defined variables at the top of `radon.pl`:

   ```perl
   @xlow = (-43.9);   # longitude lower bound(s)
   @xhig = (-43.6);   # longitude upper bound(s)
   @ylow = (28.1);    # latitude lower bound(s)
   @yhig = (28.3);    # latitude upper bound(s)
   @orientation = (1); # 0 = N-S track, 1 = E-W track
   ```

   Also set `$region`, `$radius`, `$res`, and `$num_tracks`.

2. Run:

   ```bash
   ./radon.pl
   ```

> **Note:** If `output/*.multibeam.asc` already exists, gridding is skipped. Delete it to force re-gridding.

### Key outputs (`output/`)

- `*.multibeam.asc` — gridded bathymetry (xyz ASCII)
- `*.multibeam.asc.bl` — same, after blacklist masking
- `*.track{N}.radon.out` — Radon transform output (columns: azimuth, lat, radon value, lon)
- `*.track{N}.stats` — per-track-point bathymetric statistics
- PostScript/EPS maps of the multibeam and Radon data

---

## Stage 2 — Ridgelet Transform (locate hills)

**Script:** `Codes/Maxima/maxima.m`

### Run

1. Edit variables in `maxima.m`:

   | Variable | Meaning |
   |---|---|
   | `azimin` / `azimax` | Azimuth search range (degrees) |
   | `threshold` | Minimum Ridgelet amplitude to keep |
   | `scale_min` / `scale_max` | Hill width search range (km) |
   | `orient` | Plotting axis: `2` = N-S track, `4` = E-W track |
   | `track` | Array of track numbers to process |

2. Run `maxima.m` from MATLAB.

3. Output: `local_maxima.track1` (columns: lon, lat, width km, Ridgelet value, azimuth°)

4. Inspect results visually:
   - Edit and run `plotter_maxima.pl` to plot all detected maxima.
   - Optionally run `../Shape_analysis/sort_maxima.pl` + `plotter_srt.pl` to preview filtered results.

5. Adjust parameters and repeat until the maxima look clean.

---

## Stage 3 — Shape Analysis

**Script:** `Codes/Shape_analysis/maxima_to_morph.pl`

### Run

1. Edit variables in `maxima_to_morph.pl`:

   | Variable | Meaning |
   |---|---|
   | `$pre` | Region name prefix for output files |
   | `$hill_height_trsh_low` / `$hill_height_trsh_high` | Minimum hill height thresholds (meters) — one per flank |
   | `$min_dist` | Minimum distance between adjacent hills (match value used in `sort_maxima.pl`) |
   | `$min_pro_num` | Minimum cross-sections required to report shape parameters |
   | `$plot` | Enable/disable PNG cross-section plots |

2. Run from `Shape_analysis/`:

   ```bash
   ./maxima_to_morph.pl
   ```

3. Outputs go to `Output2/`:
   - `*_hill_shape.track*.txt` — hill shape parameters
   - `stat_*_hill_shape.track*.txt` — ancillary statistics
   - PNG cross-section plots for each hill

4. Visually inspect the PNGs alongside `plot*.pdf` to catch duplicated hills or bad bathymetry.

### Final Processing

1. Copy `Output2/*.hill_shape.track*.txt` to `Shape_analysis/Results/`.
2. Open the appropriate Jupyter notebook:
   - `shape_data_analysis_1_MOR_side.ipynb` — single flank
   - `shape_data_analysis_both_MOR_sides.ipynb` — both flanks
3. Adjust the filename variable to point to your results file, then run.

Outputs: a CSV of track-averaged shape parameters and a text file with per-hill statistics.

---

## References

- Downey, N. J., & Clayton, R. W. (2007). A ridgelet transform method for constraining tectonic fabric orientations from seismic and bathymetric data. *Geochemistry, Geophysics, Geosystems*, 8(12). https://doi.org/10.1029/2007GC001723
- Roth, S., Cormier, M.-H., et al. (2019). Abyssal hill morphology at the fast-spreading East Pacific Rise. *Journal of Geophysical Research: Solid Earth*.

---

*Developed by Shai Roth — shairot12@gmail.com (2019)*

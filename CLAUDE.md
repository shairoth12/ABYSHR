# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

AbyssMorph is a scientific pipeline for detecting and analyzing **abyssal hill morphology** on the ocean floor. It implements the Ridgelet Transform method (Downey & Clayton, G3, 2007) extended with shape analysis (Roth et al., 2019). Input is multibeam sonar data; output is hill geometry statistics (width, height, asymmetry) along a survey track.

## Required External Dependencies

- **GMT** (Generic Mapping Tools) v5+
- **MB-System** v5.5+ (`mbgrid3`, `mbdatalist`)
- **MATLAB** 2015a+ (for wavelet transform step)
- **Python 2.7** with `numpy`, `matplotlib`, `bokeh`, `jupyter`
- **gcc** (to compile `radon.c`)

## Pipeline Architecture

The analysis runs in three sequential stages, each in its own directory:

### Stage 1 — Radon Transform (`radon.pl`)
Orchestrates multibeam gridding and Radon transform computation.
1. Edit user-defined variables at the top of `radon.pl`: lon/lat bounds (`@xlow`, `@xhig`, `@ylow`, `@yhig`), `$region`, `$radius`, `$res`, `$num_tracks`
2. Run: `./radon.pl`
3. Outputs land in `output/` as `*.radon.out` files (one per track segment)

Key behavior: if `output/*.multibeam.asc` already exists, gridding is skipped. Delete it to force regridding.

### Stage 2 — Wavelet Maxima (`Maxima/maxima.m`)
Computes the wavelet (Ridgelet) transform of Radon output to locate hill positions, azimuths, and estimated widths.
1. Edit variables in `maxima.m`: `azimin`/`azimax` (azimuth range), `threshold`, `scale_min`/`scale_max` (hill width range in km), `orient`, `track`
2. Run `maxima.m` from MATLAB
3. Output: `local_maxima.track1` (columns: Lon, Lat, Width km, Ridgelet value, Azimuth°)
4. Inspect visually: edit and run `plotter_maxima.pl`, then optionally run `../Shape_analysis/sort_maxima.pl` + `plotter_srt.pl` to check filtered results

Iterate steps 1–4 until maxima look clean.

### Stage 3 — Shape Analysis (`Shape_analysis/maxima_to_morph.pl`)
Samples bathymetric cross-sections perpendicular to each detected hill, fits hill boundaries, and computes shape parameters.
1. Edit variables in `maxima_to_morph.pl`: `$pre` (region name prefix), `$hill_height_trsh_low`/`$hill_height_trsh_high`, `$min_dist`, `$min_pro_num`, `$plot`
2. Run: `./maxima_to_morph.pl` from `Shape_analysis/`
3. Outputs go to `Output2/`: `*.hill_shape.track*.txt` + PNG cross-section plots
4. Visually inspect PNGs alongside `plot*.pdf` to catch duplicate-sampled hills or bad bathymetry

Final statistics: copy `Output2/*.hill_shape.track*.txt` into `Shape_analysis/Results/` and run the appropriate Jupyter notebook:
- `shape_data_analysis_1_MOR_side.ipynb` — single flank
- `shape_data_analysis_both_MOR_sides.ipynb` — both flanks

## File Layout

```
Codes/           # Working directory — run the pipeline from here
  radon.pl       # Stage 1 master script
  radon.c / radon.h  # C implementation of Radon transform (compile with gcc)
  blacklist.asc  # Regions to exclude (lon_min lon_max lat_min lat_max, one per line)
  file_list      # Full paths to input *.mbxxx multibeam data files
  track1..N      # Ship-track or synthetic flow-line coordinates
  output/        # Stage 1 outputs (created automatically)
  Maxima/        # Stage 2: MATLAB wavelet scripts
  Shape_analysis/ # Stage 3: Python cross-section analysis + Perl orchestration

Example/         # Same structure as Codes/ but pre-populated with expected outputs
                 # Run here first to validate your environment
```

## Key Configuration Files

| File | Purpose |
|------|---------|
| `blacklist.asc` | Regions to mask (seamounts, bad data patches); must exist even if empty |
| `file_list` | Paths to input MB data (*.mbxxx format, not *.fbt) |
| `track1`, `track2`… | Lat/lon waypoints defining the analysis track |
| `multibeam.cpt` | GMT color palette for bathymetry plots |
| `radon.cpt` | GMT color palette for Radon transform plots |

## Python Scripts

`hill_cross_analysis.py` and `funct_cross.py` are Python 2.7. They are called by `maxima_to_morph.pl` via `subprocess`; they are not run directly. The `.pyc` (`funct_cross.pyc`) is a pre-compiled cache and can be deleted safely.

## Agent skills

### Issue tracker

Issues live in GitHub Issues (`shairoth12/AbyssMorph`). See `docs/agents/issue-tracker.md`.

### Triage labels

Default label vocabulary (`needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`). See `docs/agents/triage-labels.md`.

### Domain docs

Single-context layout — `CONTEXT.md` and `docs/adr/` at the repo root. See `docs/agents/domain.md`.

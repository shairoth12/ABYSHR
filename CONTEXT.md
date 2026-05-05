# AbyssMorph — LLM Context Document

**Purpose of this file:** Dense reference for language models working in this repo. Covers the scientific method, every code file, all data formats, configuration variables, and non-obvious implementation details.

---

## 1. Scientific Problem

Abyssal hills are elongate ridges covering ~80% of the ocean floor. Their shape (width, height, asymmetry, azimuth) encodes the conditions at the mid-ocean ridge where they formed: spreading rate, spreading direction, and magma supply. This repo implements a pipeline to automatically detect individual abyssal hills from multibeam sonar bathymetry and measure their cross-sectional shape.

The pipeline implements the method of **Roth et al. (2019)**, which extends **Downey & Clayton (2007)**. The canonical citations are:
- Downey, N.J. & Clayton, R.W. (2007). A ridgelet transform method for constraining tectonic models via abyssal-hill morphology. *Geochem. Geophys. Geosyst.* 8, Q03004.
- Roth, S., Granot, R. & Downey, N.J. (2019). Discrete characterization of abyssal hills: Unraveling temporal variations in crustal accretion processes at the 10°30′N segment, East Pacific Rise. *Earth Planet. Sci. Lett.* 525, 115762.

---

## 2. Mathematical Method

### 2.1 Stage 1 — Radon Transform

The Radon transform maps 2-D bathymetry f(x₁,x₂) to a 2-D location–azimuth domain:

    Ra_f(θ, s) = ∫∫ f(x₁,x₂) · δ(x₁·sin θ + x₂·cos θ − s) dx₁ dx₂

Implementation: for each **track point** (a point along the ship track defining "location"), the code:
1. Subtracts the local mean depth within a 10 km radius to get a bathymetric anomaly.
2. Computes the average anomaly along lines at each azimuth θ from −89° to +89° (179 angles), restricted to within a 10 km integration radius.
3. Normalises by line length through defined data to handle swath edges and NaN gaps.

This yields one row of the Radon output per track point: 179 (azimuth, location, radon_value) triplets.

**Key geometry:** θ is measured clockwise from north. The C code stores slopes as tan(θ), so θ=0° (N–S line) is slope=0, θ=90° is ±∞. Azimuth in the output is 90−atan(slope)/D2R (converts back to compass bearing). Azimuths range 1°–179°; 90° is east–west.

### 2.2 Stage 2 — Wavelet (Ridgelet) Transform

A 1-D Mexican Hat (MH) wavelet transform is applied along lines of constant azimuth in the Radon domain. The MH wavelet is the second derivative of a Gaussian:

    ψ(x) = (1 − x²) · e^(−x²/2)

Implemented in the **frequency domain** for efficiency:

    W_f(a, b) = IFFT( FFT(signal) · ω² · e^(−ω²/2) · √a )
    where ω = ξ · a  (scaled angular frequency)

The scale parameter `a = 2^b` where `b` is log₂(scale). The width parameter in physical units is the distance between the zero crossings of ψ at that scale, which equals `2π/ω₀ · a` ≈ `2^b · Δx` where Δx is the spatial sample interval.

Maxima in the 3-D (location × scale × azimuth) ridgelet domain mark individual abyssal hills. Each maximum gives: geographic position (lon, lat), estimated width (km), ridgelet magnitude, and azimuth (°).

**Why the MH wavelet:** symmetric → preserves sign of peaks; zero mean → insensitive to local depth trends; zero linear moment → measures small hills superimposed on large ones independently.

### 2.3 Stage 3 — Shape Analysis

For each detected hill (position + azimuth from Stage 2):
1. Project N equally spaced points along the hill's **strike direction** (azimuth_s), spanning ±s_length km.
2. At each strike point, extract a bathymetric cross-section **perpendicular to strike** (azimuth_d = azimuth_s ± 90°), using GMT's `project` command.
3. On each cross-section profile:
   - Find the **crest** (shallowest point) via convolution-based zero-crossing of dz/dx.
   - Find the **flanks' minima** (deepest troughs on each side) via slope sign changes.
   - Compute: left/right **width** (m), **height** (m), **slope** (° by two methods), **area** (km²).
4. Average and compute std across all valid cross-sections to get per-hill statistics.

**Inward vs. outward:** an abyssal hill "faces inward" when its steeper flank points toward the ridge axis. Determined post-hoc in the Jupyter notebooks by comparing the flank that faces toward the spreading center.

---

## 3. Pipeline Architecture

```
Input: multibeam *.mbxxx files  +  track waypoint file  +  blacklist.asc
         ↓
[Stage 1]  radon.pl  →  radon.c  →  output/*.radon.out
                                     output/*.multibeam.asc (cached grid)
         ↓
[Stage 2]  Maxima/maxima.m  →  Maxima/local_maxima.trackN
         ↓
[Stage 3]  Shape_analysis/maxima_to_morph.pl
              → Shape_analysis/filt_maxima.py      (filters duplicates)
              → Shape_analysis/hill_cross_analysis.py  (per-hill)
                   → project_strike.pl, project_cross.pl  (GMT sampling)
                   → funct_cross.py                (shape algorithms)
              → Shape_analysis/Output2/*_hill_shape.track*.txt
         ↓
[Analysis]  Shape_analysis/Results/*.ipynb  (Jupyter, statistics + figures)
```

Temp files (tempx, tempy, tempz, tempt, temp_prof.txt, tempxy.txt, tempxyz.txt) are communication scratch files; they are moved to `Temp_files/` at end of each stage run. Do not rely on their presence between stages.

---

## 4. File-by-File Reference

### 4.1 Stage 1 — Radon

#### `Codes/radon.pl`
Master orchestrator for Stage 1. **Run this file to start the pipeline.**

User-configurable variables at the top (must edit before running):

| Variable | Meaning |
|---|---|
| `$no_seg` | Number of geographic sub-segments to process |
| `@xlow`, `@xhig` | Longitude bounds for each segment |
| `@ylow`, `@yhig` | Latitude bounds for each segment |
| `@orientation` | 0 = N–S track, 1 = E–W track (affects MATLAB sorting) |
| `$region` | Name prefix for output files (e.g. `MAR_28N`) |
| `$radius` | Grid bounds padding radius in **meters** (default 5000 = 5 km); expands the gridding extent so enough data surrounds the track. Not the Radon integration radius — that is `INC_RADIUS` hard-coded in `radon.c` (10 km). Set `$radius` ≥ `INC_RADIUS` to avoid edge truncation of Radon integrals. |
| `$res` | Bathymetry grid resolution in **meters** (default 150) |
| `$num_tracks` | How many `track1`, `track2`, … files exist |

Logic flow:
1. Computes extended bounds `$xmin2/$xmax2/$ymin2/$ymax2` = bounds + `$radius` padding (to include data around the track edges for Radon integration).
2. If `$pre.multibeam.asc` does not exist: runs `mbdatalist` + `mbgrid3` to create it. **Delete the .asc file to force re-gridding.**
3. Calls `blacklist.pl` to mask blacklisted rectangles → `.asc.bl`.
4. Reads the `.asc.bl` file into `@datax`, `@datay`, `@dataz` arrays, detecting grid dimensions from x-value changes.
5. Writes `tempx`, `tempy`, `tempz` (grid coordinate/depth files) and `tempt` (filtered track points within bounds).
6. Writes `radon.h` dynamically with `NUMX`, `NUMY`, `NUMTRACK`, `STAT_FILE` as C preprocessor constants.
7. Compiles `radon.c` → `radon` binary.
8. Runs `./radon > $pre.$track_file.radon.out`.
9. Calls `radon_image.pl` for GMT visualization.
10. Moves temp files to `Temp_files/`.

#### `Codes/radon.c`
C implementation of the Radon transform. Reads temp files, calls math routines for each track point.

Array dimensions are compile-time constants from `radon.h`: `NUMX`, `NUMY`, `NUMTRACK`, `NUMSLOPE=179`.

Key constants in `radon.c`:

| Constant | Value | Meaning |
|---|---|---|
| `AVE_RADIUS` | 10000.0 m | Radius for mean depth calculation |
| `RMS_RADIUS` | 20000.0 m | Radius for RMS amplitude calculation |
| `INC_RADIUS` | 10000.0 m | Radius for Radon line integration |
| `NUMSLOPE` | 179 | Number of azimuth samples |
| `MINANG`/`MAXANG` | −89°/+89° | Azimuth range |

Key functions:

- `data_stats(data, refx, refy, datax, datay, stats)`: Computes local mean depth and RMS within 10 km and 20 km radius. Stats array layout: `[0]=n_10km, [1]=mean_10km, [2]=std_10km (corrected), [3]=rms_10km, [4]=rms_err_10km, [5]=n_20km, [6]=mean_20km, [7]=std_20km, [8]=rms_20km, [9]=rms_err_20km]`. The mean (`stats[1]`) is used as the bathymetric anomaly baseline.

- `proj_cart(refx, refy, datax, datay, xdom, ydom)`: Converts lon/lat grid to Cartesian meters relative to the current track point. Uses equirectangular projection with cosine correction for x. Returns distances in meters.

- `make_grid(dom, del, grid, num)`: Creates cell-edge grid from cell-center array (adds half-cell at each end). Used for bilinear-style index lookup.

- `linesum(xgrid, ygrid, slope, data, data_av)`: Computes the Radon integral along a line of given slope through the origin. Calls `combine_sort_purge` to find all grid-cell crossings, then `line_int` to integrate.

- `line_int(pointsx, pointsy, num_points, xgrid, ygrid, data, data_av)`: Piecewise integration: for each segment between adjacent grid crossings, takes midpoint, finds nearest grid cell, adds `(depth − mean_depth) × segment_length`. Normalises by total line length.

- **`data_val < 99000` check** in `line_int`: Skips cells with no sonar data (returned as large positive numbers, not NaN, by mbgrid3).

Output to stdout (redirected to `.radon.out`): `azimuth latitude radon_value longitude` (one row per track_point × azimuth combination).

Output to STAT_FILE: `lon lat n_10km mean_10km std std_corrected rms n_20km mean_20km std std_corrected rms` — one row per track point.

#### `Codes/radon.h`
Auto-generated by `radon.pl`. Contains only:
```c
#define NUMX <int>
#define NUMY <int>
#define NUMTRACK <int>
#define STAT_FILE "<path>"
```
Do not edit manually; it is overwritten each run.

#### `Codes/blacklist.pl` / `Codes/blacklist2.pl`
Perl scripts that read `blacklist.asc` (lon_min lon_max lat_min lat_max per line) and set grid points within those rectangles to a large positive value (effectively NaN for the Radon code). Called as `./blacklist.pl grid.asc blacklist.asc`.

#### `Codes/mbgrid3`
Pre-compiled MB-System wrapper. Produces xyz ASCII output with a 4-line header. Do not confuse with the standard `mbgrid` binary.

#### `Codes/radon_image.pl`
GMT plotting script for Stage 1 visualization. Creates multibeam map and Radon transform image in EPS/PS format. Called as `./radon_image.pl $pre $xmin $xmax $ymin $ymax $track_file $orientation`.

---

### 4.2 Stage 2 — Wavelet Maxima

#### `Codes/Maxima/maxima.m`
MATLAB script (run manually). **Must edit user variables before running.**

User-configurable variables:

| Variable | Meaning |
|---|---|
| `azimin` | Minimum azimuth index to process (≥ 2, because wavelet needs neighbors) |
| `azimax` | Maximum azimuth index (exclusive upper bound) |
| `filtscale` | Gaussian filter σ for denoising (increase to suppress false positives) |
| `filtsize` | Gaussian filter kernel size in pixels (integer) |
| `threshold` | Minimum ridgelet magnitude to keep a maxima |
| `scale_min` | Minimum hill half-width in km |
| `scale_max` | Maximum hill half-width in km |
| `nscales` | Number of wavelet scale levels |
| `orient` | 2 = N–S track (sort by latitude), 4 = E–W track (sort by longitude) |
| `track` | Track numbers to process (array, e.g. `[1]` or `[1 2 3]`) |

Logic:
1. Reads all `../output/*.trackN.radon.out` files into a matrix.
2. Sorts by lat (orient=2) or lon (orient=4) and reshapes into (location × azimuth) matrix `z`.
3. Applies Gaussian denoise filter to `z`.
4. Zeros out blacklisted coordinates in `z`.
5. Computes cumulative along-track distance `d` via `get_dist.m`.
6. Interpolates `z` and spatial coordinates to next power-of-2 length (FFT efficiency).
7. For each azimuth `bindex` from `azimin` to `azimax`:
   - Extracts row `z(:, bindex)` as 1-D signal.
   - Calls `wtrans()` → CWT matrix (location × scale).
   - Calls `find_maxima(cwt_prev, cwt_curr, cwt_next, ...)` → local maxima in 3-D (comparing curr against neighbors in both scale and azimuth dimensions).
   - Appends azimuth column.
8. Filters by `threshold` on ridgelet magnitude.
9. Converts scale from log₂ units back to km: `scale_km = 2^b × Δx`.
10. Saves `local_maxima.trackN` with columns: lon lat width_km ridgelet_value azimuth_degrees.

**Common failure mode:** if `azimax` is set too high, it runs off the end of the 179-column azimuth array and the while loop crashes. Set `azimax` ≤ 179 − 1 = 178.

#### `Codes/Maxima/wtrans.m`
Function `wtrans(bathy, b_min, b_max, nscales)`. Computes the continuous wavelet transform of a 1-D signal using the Mexican Hat wavelet in the frequency domain.

```matlab
xhat = fft(bathy);                          % frequency domain
xi = [0:n/2  -n/2+1:-1] * 2π/n;           % angular frequencies
for b in linspace(b_min, b_max, nscales):
    a = 2^b
    ω = xi * a                              % scaled frequencies
    window = ω² · exp(-ω²/2) · √a          % MH wavelet in freq domain
    out(:, k) = real(IFFT(xhat * window))
```

Returns (n × nscales) matrix. Column k is the wavelet transform at scale `2^linspace(b_min,b_max,nscales)[k]`.

#### `Codes/Maxima/find_maxima.m`
Function `find_maxima(mat1, mat2, mat3, x, x2, y2, y, quads)`.

- `mat1`, `mat2`, `mat3` = CWT matrices for three consecutive azimuths.
- Finds indices where `mat2 > all 26 neighbors` across scale (rows) and along-track (columns), including comparisons to `mat1` and `mat3` neighbors (azimuth dimension).
- Returns (N × 4) matrix: [lon, lat, scale_log2, magnitude].
- Filters blacklisted (lon_min, lon_max, lat_min, lat_max) rectangles.

#### `Codes/Maxima/get_dist.m`
Computes cumulative great-circle distances between consecutive (lon, lat) points using the haversine formula. Returns distances in km.

#### `Codes/Maxima/plotter_maxima.pl`
GMT plotting script for visualizing `local_maxima.trackN` over the bathymetry map. Edit `$pre`, `$xmin/$xmax/$ymin/$ymax` to match your region.

---

### 4.3 Stage 3 — Shape Analysis

#### `Codes/Shape_analysis/maxima_to_morph.pl`
Master script for Stage 3. **Run from `Shape_analysis/` directory.**

User-configurable variables:

| Variable | Meaning |
|---|---|
| `$pre` | Region prefix string (must match Stage 1/2 filenames) |
| `$track_dirct` | 0 = E–W track (sort maxima by lon descending), 1 = N–S (sort by lat descending) |
| `$res` | Grid resolution in meters (match Stage 1 `$res`) |
| `$plot` | 0 = no plots, 1 = ~10 evenly spaced cross-section PNGs per hill, 2 = all cross-sections |
| `$s_length` | Half-length (km) of along-strike sampling window. Total = 2 × s_length |
| `$hill_height_trsh_low` | Min height of lower flank (m) for cross-section to be valid |
| `$hill_height_trsh_high` | Min height of higher flank (m) for cross-section to be valid |
| `$min_dist` | Minimum spacing between hills in km (used by filt_maxima.py) |
| `$min_pro_num` | Minimum number of valid cross-sections for a hill to count |

Logic:
1. Reads `../Maxima/local_maxima.track*` files.
2. Sorts by lon (descending) or lat (descending) based on `$track_dirct`.
3. Calls `filt_maxima.py` to remove closely-spaced duplicate hills.
4. For each hill in `maxima.srt.fl.trackN`:
   - Parses lon, lat, width, ridgelet_value, azimuth.
   - **Azimuth convention adjustment:** ensures cross-sections go from west→east (E–W track) or south→north (N–S track) by rotating azimuth_d by ±90°.
   - **Width scaling** (empirical, from Roth 2019): `m_width × 5` if magnitude < 60, `× 2.5` if < 90, `× 2` otherwise. This corrects the ridgelet width estimate for use as cross-section half-width.
   - Calls `hill_cross_analysis.py` with all parameters as CLI arguments.

#### `Codes/Shape_analysis/filt_maxima.py`
Filters the sorted maxima file to remove hills closer than `$min_dist` km, keeping the hill with the highest ridgelet magnitude. Reads `maxima.srt.trackN`, outputs `maxima.srt.fl.trackN`.

#### `Codes/Shape_analysis/hill_cross_analysis.py`
Python 2.7 script. Called by `maxima_to_morph.pl` once per hill with 16 positional CLI arguments.

**Arguments (sys.argv[1..16]):** pre, track, num_hill, fig, lon, lat, dist, m_width, s_length, azimuth_s, azimuth_d, hill_height_trsh_low, hill_height_trsh_high, res, pro_num, min_pro_num.

Logic per hill:
1. Calls `project_strike.pl` → `temp_prof.txt`: list of (lon, lat) points spaced along hill's strike, centered on hill.
2. For each strike point:
   - Calls `project_cross.pl` → `tempxy.txt` (projected distances), `tempxyz.txt` (lon, lat, depth).
   - Loads depth `z` and projected distance `x` (in meters).
   - Drops NaN values.
   - Skips profile if < 5 valid points or > 40% NaN.
   - Calls `get_max(dzdx, len_x)` → crest index.
   - Calls `get_mins(z, dzdx, max_i)` → left and right flank minima.
   - Calls `get_top(l_min_p, r_min_p, z)` → shallowest point between minima.
   - Validates height thresholds: both flanks must exceed `hill_height_trsh_low`, at least one must exceed `hill_height_trsh_high`.
   - Calls `get_shapes(x, z, dzdx, res, real_top, l_min_p, r_min_p)` → 10 shape metrics.
   - Optionally saves PNG of profile with annotated crest/minima.
3. Averages all valid cross-sections.
4. Appends rows to `Output2/{pre}_hill_shape.track{N}.txt` and `Output2/stat_{pre}_hill_shape.track{N}.txt`.

**Output file structure** (`*_hill_shape.track*.txt`):
- One row per cross-section: `{hill_no}_{profile_no}  lon lat dist l_width r_width l_height r_height l_minmax_slope r_minmax_slope l_max_slope r_max_slope l_area r_area`
- Two summary rows per hill: `avg_{hill_no}` and `std_{hill_no}` with per-column mean and standard deviation.
- Units: widths in meters, heights in meters, slopes in degrees, areas in km².
- Zero rows = invalid profiles.

**`no_mins` flag:** 0 = success, 1 = failed to find minima on both flanks.

#### `Codes/Shape_analysis/funct_cross.py`
Pure Python 2.7 functions. No I/O.

- **`get_max(dzdx, len_x)`**: Finds the single zero-crossing of dz/dx (from positive to negative) in the central half of the profile, indicating the crest. Uses a box-car convolution that grows until only one crossing remains (or convolution exceeds half the profile length). Returns (index_list, convolution_width). The convolution width is a quality metric: small = sharp, clear crest.

- **`get_mins(bathy, dzdx, max_indx)`**: Finds the deepest local minimum on each flank (left and right of crest). Two-pass algorithm: first finds zero-crossings of dz/dx from negative to positive; second finds near-flat inflection points (sum of 3-point slope ≥ −0.1, abs neighboring slopes < 0.1). Applies a refinement scan inward from the outer minimum to find the true boundary at the 33rd percentile height. Returns (l_min, r_min, flag) where flag=0 is success.

- **`get_top(l_min_pnt, r_min_pnt, bathy)`**: Returns the index of the shallowest point (argmax of bathymetry = least negative depth) between the two flank minima.

- **`get_shapes(x_dist, bathy, dxdz, grd_spc, top_pnt, l_min_pnt, r_min_pnt)`**: Computes 10 shape metrics:
  - Widths: `l_wdt = x[top] − x[l_min]`, `r_wdt = x[r_min] − x[top]` (in meters)
  - Heights: `l_hgt = bath[top] − bath[l_min]`, `r_hgt = bath[top] − bath[r_min]` (in meters, negative bathymetry so difference is positive)
  - Slope method 1 (min-max): `arctan(height/width)` in degrees
  - Slope method 2 (max local): `arctan(max(|dz/dx|))` along each flank in degrees
  - Areas: trapezoidal integration of (depth − flank_minimum_depth) × grid_spacing, in km²

#### `Codes/Shape_analysis/project_strike.pl`
GMT-based script. Projects N equally spaced points along a bearing `azimuth_s` for distance ±`s_length` km from (lon, lat). Writes to `temp_prof.txt`. Called as:
```
perl project_strike.pl <lon> <lat> <azimuth_s> <s_length> <pro_num> <pre>
```

#### `Codes/Shape_analysis/project_cross.pl`
GMT-based script. Extracts a bathymetric profile perpendicular to the hill axis (bearing `azimuth_d`) using `gmt project` + `gmt grdtrack` on the cached multibeam grid. Width = `m_width`. Writes `tempxy.txt` (projected distance + coordinates) and `tempxyz.txt` (lon lat depth). Called as:
```
perl project_cross.pl <lon> <lat> <m_width> <azimuth_d> <res> <pre>
```

#### `Codes/Shape_analysis/sort_maxima.pl`
Manual utility: re-sorts and filters `local_maxima.track1` with custom thresholds. Use when iterating on `maxima.m` parameters to visually check whether the detected hills look correct before running the full shape analysis.

#### `Codes/Shape_analysis/plotter_srt.pl`
GMT script: plots the sorted+filtered maxima (`maxima.srt.fl.track1`) as line segments over the bathymetry, annotated with hill width. Essential for QC between Stage 2 and Stage 3.

#### `Codes/Shape_analysis/Results/shape_data_analysis_1_MOR_side.ipynb`
Jupyter notebook for tracks covering a single flank of a mid-ocean ridge. Reads `*_hill_shape.track*.txt`, computes inward/outward facing fractions, plots height/width/slope vs. along-track distance.

#### `Codes/Shape_analysis/Results/shape_data_analysis_both_MOR_sides.ipynb`
Same as above but for flow-line transects spanning both ridge flanks (Pacific + Cocos plates in the paper's EPR analysis).

---

## 5. Data Formats

### 5.1 Input

**`file_list`**: One absolute path per line to multibeam data files in MB-System format (`*.mbXXX`, not `*.fbt`).

**`trackN`**: Two-column whitespace-delimited text. Column 1 = longitude, column 2 = latitude. One waypoint per row, ordered along the ship track.

**`blacklist.asc`**: Four-column whitespace-delimited text: `lon_min lon_max lat_min lat_max`. Rectangles to exclude. Must exist even if empty.

### 5.2 Stage 1 Outputs

**`output/{pre}.multibeam.asc`**: Gridded bathymetry in xyz ASCII format with a 4-line mbgrid header. Columns: lon, lat, depth (meters, negative = ocean). NaN cells written as a large positive value (≥99000).

**`output/{pre}.multibeam.asc.bl`**: Same as above after blacklist masking.

**`output/{pre}.track{N}.radon.out`**: Four-column whitespace-delimited text, one row per (track_point × azimuth):
```
azimuth  latitude  radon_value  longitude
```
Azimuth ranges 1°–179° (compass bearing, clockwise from north). Radon value is the mean bathymetric anomaly (meters) along lines at that azimuth through the track point.

**`output/{pre}.track{N}.stats`**: 12-column text, one row per track point:
```
lon  lat  n_10km  mean_10km  std_10km  rms_10km  rms_err_10km  n_20km  mean_20km  std_20km  rms_20km  rms_err_20km
```
Depths in meters. n = count of valid depth samples.

### 5.3 Stage 2 Outputs

**`Maxima/local_maxima.trackN`**: Five-column space-delimited, one row per detected hill:
```
longitude  latitude  width_km  ridgelet_magnitude  azimuth_degrees
```
- width_km: estimated half-width of the abyssal hill in km (zero-crossings of MH wavelet at this scale).
- ridgelet_magnitude: amplitude of the ridgelet maxima; larger = stronger/clearer hill signal.
- azimuth_degrees: compass bearing of the hill's long axis (0–180°).

**`Maxima/infiles.trackN`**: Temp file listing which `.radon.out` files were loaded.

### 5.4 Stage 3 Outputs

**`Shape_analysis/maxima.srt.trackN`**: Sorted version of `local_maxima.trackN` with an appended distance-along-track column (6 columns total).

**`Shape_analysis/maxima.srt.fl.trackN`**: Filtered version (duplicates within `$min_dist` km removed).

**`Shape_analysis/Output2/{pre}_hill_shape.track{N}.txt`**: Main shape output. 14-column fixed-width text. Units: widths/heights in meters, slopes in degrees, areas in km².

```
Column header: no.  longitude  latitude  distance  l-width  r-width  l-height  r-height  l-minmax-slope  r-minmax-slope  l-max-slope  r-max-slope  l-area  r-area
```

Row labels:
- `{hill_no}_{profile_no}`: individual cross-section (0 = invalid/masked)
- `avg_{hill_no}`: mean across valid cross-sections
- `std_{hill_no}`: standard deviation

**`Shape_analysis/Output2/stat_{pre}_hill_shape.track{N}.txt`**: Ancillary stats per cross-section:
```
no.  longitude  latitude  distance  profile-len  min-depth  max-depth  conv
```
`conv` = convolution window size used by `get_max` (larger = noisier profile / ambiguous crest).

**PNG cross-section plots**: `Output2/{pre}_track{N}_hill_{HH}_{PP}.png` — profile with annotated crest (red dot), convolution max (orange), and flank minima (green). Created only if `$plot > 0`.

---

## 6. Key Configuration Choices and Gotchas

### Radon radius vs. grid resolution
`$radius` in `radon.pl` controls grid bounds padding — it widens the area gridded around the track so that Radon integration lines don't run off the edge of the data. It is **not** the Radon integration radius; that is `INC_RADIUS = 10000 m` hard-coded in `radon.c` and cannot be changed without recompiling.

Rule of thumb: set `$radius` ≥ `INC_RADIUS` (i.e., ≥ 10 km) to ensure integration lines always land inside the grid. The default 5 km is undersized relative to `INC_RADIUS` and can cause truncated integrals near the track endpoints. Setting `$radius` much larger than `INC_RADIUS` wastes memory and gridding time without improving results.

### The radon.h recompile pattern
Every run of `radon.pl` overwrites `radon.h` and recompiles `radon.c`. This is by design — NUMX and NUMY are compile-time array sizes. If you abort mid-run, `radon.h` may be inconsistent with the grid. Always let `radon.pl` run to completion or delete `radon.h` before re-running.

### Data values are negative
Ocean depths are stored as negative numbers (e.g., −3500 m). The `data < 0` checks in `radon.c` are the "is this a real data point" test. Empty cells from mbgrid3 have values ≥ 0 (or ≥ 99000).

### Multibeam caching
`radon.pl` skips `mbdatalist` + `mbgrid3` if `$pre.multibeam.asc` exists. Delete this file to force re-gridding after changing bounds or resolution.

### Power-of-2 interpolation in maxima.m
The signal is resampled to length `2^ceil(log2(N))` for FFT efficiency. The wavelet transform in `wtrans.m` assumes all azimuths have the same length, so this happens once and all azimuths share the same `dom` grid.

### Azimuth convention
- In `radon.c` output: azimuth = 90 − atan(slope)/D2R. East–west lines = 90°, north–south = 0° or 180°.
- In `maxima.m` output column 5: same convention (compass bearing, 0–180°).
- In `maxima_to_morph.pl`: `azimuth_s` = hill's long-axis bearing; `azimuth_d` = perpendicular (for cross-sections). The adjustment logic ensures cross-sections always go east (E–W track) or north (N–S track).

### Width scaling in maxima_to_morph.pl
The raw `m_width` from local_maxima represents the ridgelet's scale estimate. It is scaled empirically before use as the cross-section sampling half-width:
- `m_val < 60`: × 5 (weak detections tend to underestimate width)
- `60 ≤ m_val < 90`: × 2.5
- `m_val ≥ 90`: × 2

These factors come from the Roth et al. (2019) calibration.

### Python 2.7 dependencies
`hill_cross_analysis.py` and `funct_cross.py` are Python 2.7. They use:
- `print` as a statement (not function)
- Integer division without `//`
- `range()` returns a list
- `numpy`, `matplotlib` (pylab), `subprocess.call`

### Minimum cross-section count
A hill is zeroed (all-zero averages) if fewer than `$min_pro_num` cross-sections are valid. This is intended — the Jupyter notebooks filter out zero-averaged hills.

### Duplicate sampling
If two detected maxima in `local_maxima.trackN` are close, `filt_maxima.py` keeps the one with higher ridgelet magnitude. But if maxima from different azimuths overlap, they can still result in duplicate hills. The README recommends visual inspection of `Output2/` PNGs and the `plotter_srt.pl` map.

---

## 7. Example Dataset

`Example/` mirrors the `Codes/` layout with pre-populated inputs and outputs for **MAR 28°N** (Mid-Atlantic Ridge, 28°N, a small longitude segment from −43.9° to −43.6°). This is the fastest way to verify the environment is working. Run the pipeline in `Example/` before `Codes/`.

Expected outputs after a successful Example run:
- `Example/Output/MAR_28N_-43.9_to_-43.6.track1.radon.out`
- `Example/Maxima/local_maxima.track1` (5 hills detected)
- `Example/Shape_analysis/Output2/MAR_28N_-43.9_to_-43.6_hill_shape.track1.txt` (4 hills with shape data)

---

## 8. External Tool Versions and Dependencies

| Tool | Version | Notes |
|---|---|---|
| GMT | ≥ 5 | Used in radon_image.pl, project_strike.pl, project_cross.pl, plotter_*.pl |
| MB-System | ≥ 5.5 | `mbdatalist`, `mbgrid3` |
| MATLAB | ≥ 2015a | For wavelet transform step; needs Image Processing Toolbox (fspecial, imfilter) |
| Python | 2.7 | numpy, matplotlib, pylab |
| gcc | any | To compile radon.c |

---

## 9. Scientific Context for Output Interpretation

- **l-/r- convention:** "left" and "right" are defined by the direction of the cross-section as sampled (west-to-east for E–W tracks, south-to-north for N–S tracks). Which side is "inward" (toward the ridge axis) must be determined from the ridge geometry, not from the file itself.

- **l-minmax-slope vs. l-max-slope:** The min-max slope measures the average gradient from flank minimum to hill crest; the max-slope measures the steepest two-point gradient along the flank. For tectonically dominated hills (slow spreading), these differ significantly. For magmatically dominated hills (fast spreading), slopes are gentler and more uniform, so they are closer.

- **Area:** Cross-sectional area of the flank (m × m → km²), computed as the trapezoidal area between the depth profile and the flank minimum depth. Larger area = more material above the adjacent trough.

- **Spreading rate proxies:** Height and width both decrease with increasing spreading rate up to ~100 km/Myr, then plateau. Fraction of inward-facing hills decreases roughly linearly from ~90% (slow) to ~55% (ultra-fast). These trends are from Table 2 of Roth et al. (2019).

- **Temporal signal:** In flow-line transects (both flanks of a ridge), the along-track dimension maps to geological time via the spreading rate. Cyclic patterns in hill height reflect temporal changes in magma supply.

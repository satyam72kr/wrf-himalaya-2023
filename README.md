# Cumulus vs microphysics sensitivity, Western Himalaya, July 2023

WRF-ARW physics-sensitivity experiments for the 6–13 July 2023 Western Himalayan
heavy-rainfall event, with the verification code used in the accompanying paper.

Nine simulations were run over one common domain, differing in **exactly two**
namelist entries — `mp_physics` and `cu_physics`. Six of them form a complete
3 × 2 factorial in microphysics × cumulus, which is what allows the two effects to
be separated cleanly. The other three extend the cumulus sampling.

---

## Experiments

| Exp | `mp_physics` | Microphysics | `cu_physics` | Cumulus | In factorial |
|-----|-----|--------------|-----|-------------------|-----|
| 1 | 6  | WSM6     | 1  | Kain–Fritsch      | yes |
| 2 | 8  | Thompson | 6  | Tiedtke           | yes |
| 3 | 8  | Thompson | 3  | Grell–Freitas     | no  |
| 4 | 16 | WDM6     | 6  | Tiedtke           | yes |
| 5 | 6  | WSM6     | 6  | Tiedtke           | yes |
| 6 | 8  | Thompson | 1  | Kain–Fritsch      | yes |
| 7 | 16 | WDM6     | 1  | Kain–Fritsch      | yes |
| 8 | 6  | WSM6     | 16 | New Tiedtke       | no  |
| 9 | 3  | WSM3     | 5  | Grell 3D ensemble | no  |

All 75 remaining namelist options are identical across the nine runs: RRTM longwave,
Dudhia shortwave, YSU PBL, MM5 similarity surface layer, Noah land surface model.
No nesting.

## Model configuration

| | |
|---|---|
| Model | WRF-ARW 4.5 |
| Domain | single, 81 × 81 mass points, 9 km |
| Projection | Lambert conformal, centre 31.0°N 78.5°E, true latitudes 30°N / 34°N |
| Vertical | 45 terrain-following levels, top 50 hPa |
| Forcing | NCEP FNL 1°, 6-hourly, 34 metgrid levels |
| Relaxation zone | `spec_bdy_width = 5` |
| Integration | 0000 UTC 5 Jul → 1800 UTC 13 Jul 2023, 54 s time step, hourly output |
| Spin-up | 5–6 July discarded |
| Verification | 7–12 July 2023, six days |

---

## Repository layout

```
namelists/   namelist.wps, namelist1.input … namelist9.input
scripts/     preprocess_himalaya.sh     CDO pipeline: de-accumulate, shift, daily sum, regrid
             metrix_himalaya_v2.py      bias, RMSE, correlation, POD; factorial main effects
             fss_himalaya_v2.py         Fractions Skill Score over neighbourhood widths
figures/     fig_study_region.py        Fig. 1  domain, terrain, verification box
             figs12_himalaya.py         Fig. 2  factorial bias;  Fig. 3  FSS
             fig3_spatial_himalaya.py   Fig. 4  difference maps; Fig. S1  event totals
             fig4_daywise_himalaya.py   Fig. 5  day-wise rainfall
data/        verification_stats.csv     per-experiment scores
             fss_results.csv            FSS by experiment, threshold, neighbourhood
             daily_*.nc                 derived daily rainfall on the 0.25° grid
```

Raw `wrfout` files are not included — they are far too large, and every number in the
paper is reproducible from the derived daily fields in `data/`.

## Requirements

CDO 2.x for the pre-processing, and Python 3.9+ with:

```
xarray  netCDF4  numpy  pandas  scipy  matplotlib  cartopy  geopandas
```

`fig_study_region.py`, `fig3_spatial_himalaya.py` and `fig4_daywise_himalaya.py`
draw administrative boundaries from an Indian states shapefile. Set `SHAPEFILE`
at the top of each script to your own copy. If the path is wrong the scripts print
a warning and carry on without boundaries rather than failing silently.

---

## Reproducing the results

Edit the `BASE` path at the top of each script, then run in this order:

```bash
bash preprocess_himalaya.sh     # wrfout -> daily rainfall on the IMD grid
python metrix_himalaya_v2.py    # -> verification_stats.csv
python fss_himalaya_v2.py       # -> fss_results.csv
python fig_study_region.py      # Fig. 1
python figs12_himalaya.py       # Fig. 2, Fig. 3
python fig3_spatial_himalaya.py # Fig. 4, Fig. S1
python fig4_daywise_himalaya.py # Fig. 5
```

Figures are written to `plots/` as 600 dpi JPEG at 174 mm width.

---

## Two processing choices that matter

These are documented here because they changed the verified skill by as much as the
physics choice itself, and a reader reproducing the analysis will get different
numbers without them.

### Temporal alignment: +20 hours

WRF accumulates on the UTC calendar day. IMD does not — each daily value is the
24 hours ending 0830 IST (0300 UTC), and that accumulation is **labelled with the
later date**. Comparing the two by date misaligns them by close to a day.

The shift is derived, not fitted. Hourly output is de-accumulated with hour-ending
timestamps, so the value at 0100 UTC is the rainfall in 0000–0100 UTC and a plain
daily sum bins 2300–2300 UTC. A forward shift of *N* hours makes the bin labelled *X*
cover (*X* 00:00 − *N* − 1 h, *X* 23:00 − *N*]. Setting that equal to the IMD window,
0300 UTC of *X*−1 to 0300 UTC of *X*, gives **N = 20 h**.

Tested against 0, +22, +24 and −24 h: RMSE and daily correlation both optimise at
exactly +20 h and degrade either side of it.

```bash
cdo shifttime,20hour  in.nc  shifted.nc
cdo daysum            shifted.nc  daily.nc
```

### Regridding: bilinear, not bicubic

Bicubic weights are not sign-constrained and overshoot across sharp gradients. In
this domain that produced accumulations as low as **−3.89 mm day⁻¹** — physically
impossible, and implying compensating positive overshoot in exactly the orographic
maxima that carry the signal. Bilinear weights are non-negative and sum to unity, so
they cannot leave the range of the surrounding source points; after switching, the
minimum across all nine experiments was of order 10⁻¹⁰ mm day⁻¹.

Conservative remapping would be preferable in principle, but the WRF output is a
curvilinear grid without cell-corner coordinates and `cdo genbounds` was unavailable
in the build used.

---

## Analysis settings

| | |
|---|---|
| Verification box | 28.25–33.75°N, 75.25–81.75°E |
| Grid | IMD 0.25°, 447 land points, 2682 point-days |
| Period | 7–12 July 2023 |
| Thresholds | 1, 10, 20, 35, 50 mm day⁻¹ |
| Neighbourhood widths | 1, 3, 5, 7, 9 grid points (≈ 28–250 km) |
| FSS useful-skill target | 0.5 + *f*₀/2 (Roberts and Lean 2008) |

The box is trimmed one grid row and column inside the domain to keep the analysis
out of the lateral boundary relaxation zone. At the southern end of its eastern and
western edges it still reaches about 3.7 km — 0.4 of a model grid point — into the
innermost relaxation row, where the nudging is weakest. `fig_study_region.py` prints
this overlap when it runs.

---

## Data sources

| Dataset | Source |
|---|---|
| IMD 0.25° gridded daily rainfall | India Meteorological Department, Pune — [download](https://www.imdpune.gov.in/cmpg/Griddata/Rainfall_25_NetCDF.html) |
| GPM IMERG Final Run daily V07 | NASA GES DISC — `10.5067/GPM/IMERGDF/DAY/07` |
| NCEP FNL 1° operational analyses | NSF NCAR RDA — `10.5065/D6M043C6` |

Redistribution of these datasets is not permitted here; only the derived daily fields
on the verification grid are included.

---

## Citation

<!-- Replace once the paper is accepted. -->

> Author(s) (year) Cumulus parameterization dominates microphysics in controlling
> simulated rainfall at 9 km grid spacing during a Western Himalayan extreme event:
> a 3 × 2 factorial WRF study. *Modeling Earth Systems and Environment*.

## Licence

Code released under the MIT Licence. The derived data files are released under
CC BY 4.0.

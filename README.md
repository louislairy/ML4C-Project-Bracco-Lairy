# Weather Reconstruction over Paris

Can we reconstruct the weather in Paris when its weather station breaks down, using only the stations of other European cities?

This project was carried out as part of the **Machine Learning for Climate and Energy** course at École Polytechnique (Sep. - Dec. 2025), supervised by Dr. Bruno Deremble. The analysis notebook is written in French.

## Problem

Five weather stations monitor surface conditions in **Paris, Brest, London, Marseille and Berlin**. We assume the Paris station fails and try to infer its measurements from the four remaining stations. Starting with 2 m temperature, we extend the approach to every available surface variable, including precipitation, which turns out to be the hardest to reconstruct.

## Data

The dataset comes from the **ERA5** reanalysis and contains hourly surface data for the five cities. It is not included in this repository (it was provided by the course supervisors).

- One NetCDF file per variable per city, stored as `weather_data/<city>/<variable>.nc`
- 40 years of data per city (41 years for Paris), starting in 1980
- Some variables are *analysis* variables, others are *forecast* variables shifted by 3 h relative to the analysis ones, so the timestamps stored in each NetCDF file are used for alignment

| Variable | Description | Unit |
| --- | --- | --- |
| `t2m` | 2 m air temperature | K |
| `skt` | Skin temperature | K |
| `d2m` | 2 m dewpoint temperature | K |
| `u10`, `v10` | 10 m wind components (east-west, north-south) | m/s |
| `tcc` | Total cloud cover | 0-1 |
| `sp` | Surface pressure | Pa |
| `tp` | Total precipitation | m |
| `ssrd` | Surface solar radiation downwards | J/m² |
| `blh` | Boundary layer height | m |

Files are loaded with `xarray`, converted to `pandas` DataFrames, and merged into a single table with one column per variable and city (e.g. `t2m_london`). Incomplete rows are dropped.

## Methodology

All models are trained on data **before 2010** and evaluated on data **from 2010 onwards**. Hyperparameters are selected with time-series cross-validation (`TimeSeriesSplit`) on the training period, so that the chronology of the data is respected.

1. **Exploratory analysis**: descriptive statistics, correlation matrices (hourly and daily), and day-of-year correlations to reveal seasonal changes in how each neighbouring city relates to Paris.
2. **Linear regression** of Paris temperature, first from neighbouring temperatures only, then from all standardized neighbouring variables.
3. **PCA** to test whether a compact set of principal components keeps the predictive information.
4. **Ridge regression** to handle strongly correlated predictors, first as a single model, then as **12 monthly models** to capture seasonal dynamics.
5. **Lagged features** (neighbouring temperature and wind at H-1 to H-24) and recursive use of past Paris predictions, with a sensitivity analysis on outage duration (1 to 30 days).
6. **Reconstruction of all variables** with the monthly Ridge approach, compared using RMSE, R² and normalized RMSE (RMSE divided by the standard deviation of the observed variable).
7. **Precipitation**: rain/dry classification with LDA and QDA on daily data, then a **hurdle model** (QDA classifier followed by a Ridge regression on log(1 + precipitation)).

## Key results

**2 m temperature in Paris** (test period 2010 onwards)

| Model | Resolution | RMSE (K) | R² |
| --- | --- | --- | --- |
| Linear regression, neighbouring temperatures | Hourly | 2.29 | 0.899 |
| Linear regression, neighbouring temperatures | Daily | 1.75 | 0.928 |
| Linear regression, all neighbouring variables | Hourly | 2.12 | 0.914 |
| Linear regression, all neighbouring variables | Daily | 1.56 | 0.943 |
| Ridge, single model | Hourly | 2.11 | 0.914 |
| Monthly Ridge with lags and recursive Paris predictions | Hourly | 1.92 | 0.929 |

- London temperature is the strongest single predictor of Paris temperature in most months.
- Monthly models perform best in spring and worst in early winter (November, December), where winter weather regimes are harder to capture from neighbouring stations.
- Reusing past Paris predictions helps for short outages (RMSE 1.61 K for a 1-day outage) but the error accumulates for longer ones (1.91 K for 30 days).
- PCA with 6 components performs slightly worse than regression on the raw variables, since the components maximize variance in the predictors without regard to the target.

**Other variables**: large-scale variables are reconstructed best. Surface pressure reaches R² ≈ 0.99, temperatures (`t2m`, `skt`) R² ≈ 0.93, dewpoint and wind R² ≈ 0.81-0.86. Cloud cover is poorly reconstructed (R² ≈ 0.27), because it depends strongly on local conditions.

**Precipitation**: a linear regression fails to reproduce rain events, which are mostly zero with sudden peaks and often local. Framing the problem as rain/dry classification works much better:

| Model | Accuracy | Rain recall | Rain precision |
| --- | --- | --- | --- |
| LDA | 0.82 | 0.65 | 0.79 |
| QDA | 0.78 | 0.71 | 0.68 |

LDA is more reliable overall, while QDA detects more rainy days. The hurdle model (QDA + Ridge on log-transformed amounts) reaches R² = 0.31 (RMSE 2.98 mm/day) using same-day data from neighbouring cities. Predicting rain amounts remains the main limitation of the approach.

## Repository structure

```
.
├── project_weather_vf.ipynb   # Full analysis (in French)
├── images/                    # Figures used in the notebook
└── weather_data/              # ERA5 data (not included)
    ├── paris/
    │   ├── t2m.nc
    │   └── ...
    ├── brest/
    ├── london/
    ├── marseille/
    └── berlin/
```

## Running the notebook

1. Place the ERA5 NetCDF files in `weather_data/<city>/<variable>.nc`.
2. Install the dependencies:
   ```bash
   pip install numpy pandas xarray netCDF4 scikit-learn matplotlib seaborn plotly
   ```
3. Open `project_weather_vf.ipynb` in Jupyter and run the cells in order.

## Authors

- Nicolas Bracco
- Louis Lairy

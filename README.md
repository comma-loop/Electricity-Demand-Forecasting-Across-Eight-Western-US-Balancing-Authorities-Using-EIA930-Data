# Electricity Demand Forecasting Across Eight Western US Balancing Authorities Using EIA-930 Data
 
24-hour-ahead hourly electricity demand forecasting for eight Western US balancing authorities (BAs), built on public EIA-930 data. The project covers data quality checks, SQL-based exploratory analysis, machine-learning forecasts compared against naive baselines and EIA's own day-ahead forecast, and an assessment of whether the forecasts are usable operationally (prediction intervals, extreme-hour alerts, stability over time, and whether CO2 data helps).
 
| | |
|---|---|
| **Region** | Western US: Pacific Northwest, California, Nevada, Arizona |
| **Balancing authorities** | AZPS, BPAT, CISO, LDWP, NEVP, PACW, PGE, SRP |
| **Data window** | 2022-01-01 to 2025-01-01 (hourly, UTC) |
| **Forecast task** | Demand at hour *t + 24* (MW), issued at the end of every hour *t* |
| **Test period** | Targets from 2024-07-01 to 2025-01-01 (34,872 hourly forecasts across the eight BAs) |
| **Selected model** | LightGBM (one global model for all eight BAs) |
 
## Results at a glance
 
- The two EIA-930 sources agree exactly on demand for all eight BAs, so there is no source ambiguity for the target variable.
- Demand scale differs about 10x between the largest and smallest BA, and seasonality ranges from mild (21% of mean) to extreme (72% of mean).
- LightGBM reaches a test MAE of **271 MW**, versus **335 MW** for persistence and **370 MW** for EIA's day-ahead forecast. It beats persistence in all 8 BAs and in all 6 rolling-origin windows.
- On a BA that was never seen in training, the model beats persistence in 6 of 8 cases. It is unreliable for BAs whose scale falls outside the training range unless demand is normalised.
- The 90% prediction intervals cover 88.4% overall, but coverage is much lower in CISO (39.5%) and in extreme-demand hours (about 75%).
- Adding CO2 features gave no improvement (MAE +0.94%, confidence interval includes zero).
## Data sources
 
Both files are EIA-930 (Hourly and Daily Balancing Authority Operations) data published by the U.S. Energy Information Administration. They are U.S. Government works in the public domain. Official page: <https://www.eia.gov/electricity/gridmonitor/>.
 
| Source | File | Rows | Coverage | Used for |
|---|---|---|---|---|
| **Primary** | `data/raw/EIA930_BALANCE_2022_2024.csv` | 1,626,431 | 63 BAs, 2022 to 2024, hourly | Target demand and EIA's official day-ahead forecast (used only as a comparator) |
| **Secondary** | `data/raw/EIA930_8BA_2022_2025.csv` | 210,456 | The eight study BAs, 2022 to early 2025, hourly | CO2 emissions intensity (generated and consumed electricity), used only in the CO2 analyses |
 
Notes on how the sources are used:
 
- The study scope is the set of BAs in the secondary file. The 55 other BAs in the primary file are deliberately excluded.
- All timestamps are UTC time at end of hour. Local time is derived only for calendar features and for plots (Arizona's `America/Phoenix` does not observe daylight saving time; the other six BAs are on `America/Los_Angeles`).
- CO2 intensity is computed by EIA from the generation mix, so it is a derived variable and not a driver of demand. The file does not document units, so results are reported in "source units".
- The raw CSV files are not stored in this repository. Download them from EIA and place them in `data/raw/` before running the notebooks.
 
## Research questions and results
 
> **RQ1.** How consistent are the two EIA-930 sources, and what structure (scale, seasonality, CO2 link) does hourly demand have?
>
> **RQ2.** How well do models forecast demand 24 h ahead against naive baselines, and do they hold up over time and on unseen balancing authorities?
>
> **RQ3.** Can the forecast be used operationally: interval coverage, warning of extreme hours, regime change, and does CO2 help?
 
### RQ1: Source consistency and structure of demand (Notebook A)
 
**Source consistency.** The two sources were joined on `(BA, timestamp)` with a full outer join, before any cleaning.
 
- For every BA, correlation between the sources is 1.00, bias is 0.0 MW, MAE is 0.000 MW, and 100% of comparable hours match within 0.5 MW.
- No hours exist only in the primary file. Three hours per BA (24 in total) exist only in the secondary file, at the edges of the window.
- Conclusion: the sources are consistent, and the primary file is used as the source of the target variable.
**Data quality.** There were no duplicated `(BA, timestamp)` keys. Cleaning set 6 non-positive demand values and 87 implausible values (below 0.3x or above 3x a BA's training median) to missing, and added 8 rows for hours absent from the source. Missing hours are left missing (no interpolation, no percentile clipping). The final series has 210,188 valid hours; the share of missing hours is between 0.004% and 0.45% per BA, and the longest gap is 49 hours.
 
**Scale and seasonality.**
 
| BA | Average demand (MW) | Peak demand (MW) | Seasonal range (% of mean) | Peak month | Low month | Weekend / weekday | Peak hour (local, hour-ending) |
|---|---:|---:|---:|:---:|:---:|---:|:---:|
| CISO | 25,317 | 51,104 | 36.3 | Aug | Apr | 0.914 | 20 |
| BPAT | 6,656 | 11,496 | 27.1 | Jan | Sep | 0.951 | 19 |
| NEVP | 4,528 | 9,797 | 61.9 | Jul | Mar | 0.969 | 18 |
| SRP | 3,882 | 9,283 | 72.4 | Jul | Mar | 0.969 | 18 |
| AZPS | 3,863 | 9,854 | 71.5 | Jul | Mar | 0.965 | 19 |
| LDWP | 2,747 | 6,170 | 30.4 | Aug | Nov | 0.927 | 17 |
| PGE | 2,567 | 4,504 | 21.0 | Dec | May | 0.937 | 19 |
| PACW | 2,424 | 4,005 | 26.3 | Jan | May | 0.928 | 19 |
 
- Average demand differs 10.4x between the largest BA (CISO) and the smallest (PACW).
- The Southwest BAs (NEVP, SRP, AZPS) peak in summer with very large seasonal swings (62% to 72%). The Pacific Northwest BAs (BPAT, PGE, PACW) peak in winter. California BAs (CISO, LDWP) peak in August.
- Demand peaks in the late afternoon or evening (hour-ending 17 to 20) and is 3% to 9% lower on weekends than on weekdays.
**Link with CO2.** Within each BA, the correlation between demand and CO2 intensity of generated electricity ranges from **-0.17 (LDWP) to +0.44 (SRP)**, and from -0.01 to +0.55 after removing each BA's average daily profile. The sign and strength differ across BAs. This is a statistical association only; it is not evidence of causation or of forecasting value (see RQ3).
 
**Temporal dependence.** Demand is strongly correlated with its own recent past: correlation with the value 24 hours earlier is 0.90 to 0.98, and with the value one week earlier 0.74 to 0.92. This is why persistence is a strong baseline.
 
### RQ2: Forecast accuracy against baselines (Notebook B)
 
**Setup.**
 
- One target per hourly origin: demand at *t + 24*. This is a rolling point forecast evaluated at every hour, not a once-a-day submission of 24 values.
- Features available at the issue hour: current demand, lags (1, 2, 3, 24, 48, 168 h), 24-hour rolling mean and standard deviation, 1-hour difference, calendar features, and a BA identifier.
- Time-ordered split by target time: fit before 2024-04-01, validation 2024-04-01 to 2024-06-30 (used to pick the model), final training before 2024-07-01, test from 2024-07-01. The test set is scored once.
- Models: Ridge, Random Forest, and LightGBM (global model). Random models use 5 seeds.
- Baselines: **Persistence** (demand now; at a 24 h horizon this is identical to "same hour yesterday"), **WeeklyNaive** (demand at the same hour one week before the target), and **EIA day-ahead** (EIA's official forecast, compared by target hour; it is issued at a different time, so it does not use the same information).
- Metrics: MAE, RMSE, nMAE (% of mean demand), sMAPE, MASE (scaled by persistence error on each BA's training data), and bias.
**Model selection (validation, MAE in MW).** LightGBM 240.1, Random Forest 245.6, Ridge 302.7, Persistence 318.7, WeeklyNaive 460.3. LightGBM was selected.
 
**Overall test results (34,872 hours, 8 BAs).**
 
| Method | MAE (MW) | RMSE (MW) | nMAE (%) | MASE | Bias (MW) |
|---|---:|---:|---:|---:|---:|
| **LightGBM (global)** | **271.1** | **554.8** | **3.87** | **0.925** | -34.3 |
| Random Forest | 271.1 | 566.5 | 3.87 | 0.926 | -21.5 |
| Ridge | 335.2 | 719.6 | 4.79 | 1.080 | 7.8 |
| Persistence | 335.3 | 746.4 | 4.79 | 1.065 | 4.0 |
| EIA day-ahead | 370.4 | 865.6 | 5.29 | 1.017 | -167.0 |
| WeeklyNaive | 604.9 | 1,331.4 | 8.64 | 1.970 | 58.5 |
 
Day-level bootstrap (185 test days) of the MAE difference for LightGBM: **-64 MW versus persistence** (95% CI -80 to -49), -99 MW versus EIA day-ahead (CI -115 to -84), and -333 MW versus WeeklyNaive (CI -383 to -287). The seed-to-seed spread of the MAE is about 0.3 MW, so the result is not seed noise.
 
**By balancing authority (test).**
 
| BA | LightGBM MAE (MW) | Persistence MAE (MW) | EIA MAE (MW) | nMAE (%) | MASE | Beats persistence | Beats EIA |
|---|---:|---:|---:|---:|---:|:---:|:---:|
| AZPS | 200.3 | 203.9 | 180.8 | 4.37 | 1.06 | Yes | No |
| BPAT | 192.4 | 238.1 | 121.5 | 2.89 | 0.71 | Yes | No |
| CISO | 1,014.7 | 1,406.2 | 1,824.0 | 3.63 | 0.82 | Yes | Yes |
| LDWP | 180.2 | 203.7 | 255.1 | 6.59 | 1.08 | Yes | Yes |
| NEVP | 189.4 | 193.8 | 353.7 | 3.74 | 1.05 | Yes | Yes |
| PACW | 91.8 | 118.8 | 95.9 | 3.72 | 0.75 | Yes | Yes |
| PGE | 112.1 | 134.8 | 31.4 | 4.23 | 0.85 | Yes | No |
| SRP | 212.2 | 219.4 | 155.1 | 4.72 | 1.12 | Yes | No |
 
- The model beats persistence in **8 of 8** BAs. Relative error ranges from 2.89% (BPAT, the easiest BA) to 6.59% (LDWP). By combined nMAE and MASE rank, SRP is the hardest BA.
- The gains over persistence are large for BPAT, CISO, and PACW and small for AZPS, NEVP, and SRP.
- The model beats EIA's forecast in 4 of 8 BAs. The overall comparison is dominated by CISO, the largest BA.
**Stability over time (rolling origin).** Six consecutive test windows, refitting the model with an expanding training window before each:
 
| Window | LightGBM MAE (MW) | Persistence MAE (MW) |
|---|---:|---:|
| Jul 2024 | 364.0 | 418.0 |
| Aug 2024 | 334.6 | 405.5 |
| Sep 2024 | 335.1 | 384.5 |
| Oct 2024 | 230.0 | 303.6 |
| Nov 2024 | 190.8 | 232.6 |
| Dec 2024 | 190.5 | 258.3 |
 
The model beats persistence in **6 of 6** windows. Absolute error falls from summer to winter for both methods, so the change mainly reflects seasonality and cannot be separated from drift with a single test year.
 
**Unseen balancing authorities (leave-one-BA-out).** Each BA was held out in turn; the model was trained on the other seven and scored on the held-out BA, then compared with the same model configuration trained with that BA included.
 
- *Raw (MW) features:* MAE on an unseen BA is 1.01x to 18.88x the MAE when the BA is seen (median 1.02x). The model beats persistence in **6 of 8** held-out BAs and fails for BPAT (1.61x) and CISO (18.88x). Tree models cannot extrapolate beyond the MW range seen in training, and CISO is far larger than any other BA.
- *Normalised features* (demand divided by each BA's own training mean): the ratio is 1.01x to 1.07x (median 1.04x). The model again beats persistence in **6 of 8** held-out BAs and fails for AZPS and SRP. This variant assumes a new BA has some history available to compute its mean.
**What the model relies on.** Permutation importance is highest for current demand and the 24-hour rolling mean (MAE increases by about 1,790 MW when either is shuffled), followed by lag 1 (461 MW), lag 48 (255 MW), and lag 168 (232 MW). Calendar features matter much less. The model mostly corrects the current demand level using time-of-week and seasonal information, which is consistent with persistence being a strong baseline.
 
### RQ3: Operational usability (Notebook B)
 
**Prediction intervals (split conformal, 90% nominal).** Calibrated on the April to June 2024 validation window and evaluated on the test period.
 
| Method | Overall coverage | Ordinary hours | Extreme hours | Coverage range across BAs | Mean width (MW) |
|---|---:|---:|---:|---:|---:|
| Pooled (one quantile for all BAs) | 88.4% | 91.6% | 74.9% | 39.5% (CISO) to 99.9% (PACW) | 1,050 |
| Per-BA (one quantile per BA) | 86.9% | 90.7% | 70.8% | 82.1% to 91.5% | 1,090 |
 
Overall coverage is close to 90%, but the pooled intervals are badly miscalibrated by BA (CISO under-covered, PACW and PGE over-covered). Per-BA calibration fixes the BA imbalance but coverage still falls short of 90% in the hours that matter most operationally: about 71% to 75% in extreme-demand hours.
 
**Extreme-hour alerts.** An hour is "extreme" if actual demand exceeds the 90th percentile of that BA's training data. An alarm is raised when the forecast exceeds that threshold; the rule was fixed in advance and not tuned. Because the test period (summer to winter) contains more high-demand hours than the training distribution, **19.3% of test hours are extreme (6,726 hours)** instead of the nominal 10%.
 
| Method | Precision | Recall | F1 | PR-AUC |
|---|---:|---:|---:|---:|
| **LightGBM** | 0.884 | 0.863 | 0.873 | 0.949 |
| Persistence | 0.861 | 0.862 | 0.861 | 0.937 |
| EIA day-ahead | 0.894 | 0.859 | 0.876 | 0.941 |
| WeeklyNaive | 0.693 | 0.727 | 0.710 | 0.762 |
 
- LightGBM is clearly better than WeeklyNaive and only slightly better than persistence; it is on par with EIA's forecast.
- Recall by BA ranges from 0.70 (BPAT) to 0.94 (AZPS, NEVP).
- Errors are larger in extreme hours: MAE is 1.7x to 2.3x the ordinary-hour MAE, and the forecast is biased low in every BA (-35 to -791 MW; -208 MW overall). Under-forecasting the peak is the more worrying direction for operations.
- In extreme hours the model beats persistence on MAE in 5 of 8 BAs (BPAT, CISO, LDWP, PACW, PGE) but not in AZPS, NEVP, or SRP.
**Stability by month (regime change).** The final model beats persistence in 6 of 7 test months. The ratio of model MAE to persistence MAE is 0.74 to 0.87 from July to December 2024, and monthly MAE falls from 364 MW (July) to 188 MW (December). The only month where the model loses is January 2025, which contains just 70 hours. This is a performance-over-time analysis; no formal regime-change detection was performed.
 
**Does CO2 help?** The same model and the same rows were used with and without four CO2 features (CO2 intensity at *t* and 24 hours earlier, for generated and consumed electricity), over 5 seeds.
 
| Feature set | MAE (MW) | MASE |
|---|---:|---:|
| Without CO2 | 271.9 | 0.917 |
| With CO2 | 274.5 | 0.923 |
 
Adding CO2 changed MAE by **+0.94%**. A day-level bootstrap of the paired difference gives +1.9 MW with a 95% CI of -2.0 to +5.8 MW, which contains zero. **No evidence was found that CO2 features improve the forecast** in this setup. This does not mean CO2 is unrelated to demand; it applies only to this model and test period.
 
## Limitations
 
**Forecast design**
- The evaluation is a rolling point forecast of one target hour (*t + 24*) at every hourly origin. It is not a once-a-day day-ahead submission of 24 values, and no other horizon was evaluated.
- The comparison with EIA's day-ahead forecast is by target hour only. EIA forecasts are issued at different times with different information, so this is not a like-for-like contest.
- Hyperparameters were carried over from the original project and not re-tuned.
**Data and features**
- There is **no weather data** (temperature is the main driver of load variation) and no holiday calendar. This is the most likely reason the error is higher on the hot Southwest BAs and on the most variable BA.
- Missing hours are not imputed. Forecast origins whose lags cross a gap are dropped, which removes 4.3% of CISO and 7.6% of LDWP test origins (under 0.1% for the other BAs).
- CO2 intensity is derived from the generation mix and has no documented units, so it was only analysed with scale-free methods.
- BA locations on the map are approximate city locations; the project has no BA boundary data, so the map shows region only, not service-territory size.
**Evaluation**
- There is a single test period of about six months (July to December 2024, summer to winter). Seasonal effects cannot be separated from drift, and spring is not represented in the test set.
- Extreme-hour prevalence in the test set (19.3%) is almost double the nominal 10%, so precision should be read together with prevalence.
- The conformal interval guarantee assumes calibration and test data are exchangeable. Here calibration is April to June while the test covers July to December, so under-coverage is expected, as the CISO and extreme-hour results show.
- Leave-one-BA-out results apply only to the eight BAs in this study and do not show that the model generalises to other BAs. The raw-MW variant fails for BAs outside the training scale (notably CISO).
- The day-level bootstrap does not account for dependence between BAs, and there are only eight BAs, so BA-level rankings and correlations are suggestive.
- Only three model families were compared (Ridge, Random Forest, LightGBM). Deep learning and foundation models were not evaluated.
- No monetary value, reserve savings, grid reliability, or dispatch impact was measured, and none is claimed.
**Scope**
- Results describe these eight Western US BAs and should not be generalised to other interconnections without separate evidence.
## How to reproduce
 
```bash
# 1. Install dependencies
pip install -r requirements.txt   # core packages: pandas, numpy, duckdb, pyarrow,
                                  # scikit-learn, lightgbm, matplotlib, plotly, shap
 
# 2. Download the two EIA-930 CSV files and place them in data/raw/
#    data/raw/EIA930_BALANCE_2022_2024.csv
#    data/raw/EIA930_8BA_2022_2025.csv
 
# 3. Run the notebooks in order
jupyter nbconvert --to notebook --execute "Notebook A.ipynb"   # RQ1, writes data/processed/ and report/
jupyter nbconvert --to notebook --execute "Notebook B.ipynb"   # RQ2 and RQ3, writes report/
```
 
Notebook B has a `FAST_MODE` flag (smaller models, fewer seeds) for quick test runs. Keep it `False` to reproduce the reported numbers. Random seeds are fixed (`SEED = 42`; model seeds 0 to 4).
 
## Main outputs
 
| File | Contents |
|---|---|
| `report/table_sql1_source_consistency.csv` | RQ1: agreement between the two sources |
| `report/table_sql2_scale_and_seasonality.csv` | RQ1: scale, seasonality, daily and weekly shape per BA |
| `report/table_sql3a_demand_co2_correlation.csv` | RQ1: demand and CO2 correlation |
| `report/table_nb2_overall_comparison.csv` | RQ2: model and baseline comparison on test |
| `report/table_nb2_per_ba_main.csv` | RQ2: per-BA results |
| `report/table_nb2_rolling_origin.csv`, `table_nb2_leave_one_ba_out.csv` | RQ2: stability over time and on unseen BAs |
| `report/table_nb2_conformal.csv` | RQ3: interval coverage |
| `report/table_nb2_extreme_alarms.csv`, `table_nb2_extreme_by_ba.csv` | RQ3: extreme-hour alerts |
| `report/table_nb2_error_by_month.csv` | RQ3: error by month |
| `report/table_nb2_co2_ablation.csv` | RQ3: CO2 ablation |
| `report/table_nb2_forecast_timeseries.csv` | Actual versus forecast for every test hour |
 

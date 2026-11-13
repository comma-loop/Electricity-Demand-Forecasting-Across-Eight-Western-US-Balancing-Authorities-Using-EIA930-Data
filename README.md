# EIA-930 Hourly Electricity Demand Forecasting

**24-hour-ahead demand forecasts for eight western U.S. balancing authorities (BAs), using one global LightGBM model, benchmarked against simple baselines and the official EIA day-ahead forecast.**

Balancing authorities must commit generation before the next day's demand is known. This project asks how much a machine-learning model actually adds over rules that simply reuse past demand, where it fails, and how far its uncertainty estimates can be trusted.

> **Status:** v1 complete. Ongoing improvements are tracked in the [roadmap](#roadmap) and [changelog](#changelog).
>
> 📄 **Full write-up: [Read the paper (PDF)](docs/EIA930_demand_forecasting_paper.pdf)**

---

## Headline results (v1)

Test set: 34,872 hourly forecast origins, with target hours from 1 July 2024 to 1 January 2025 (08:00 UTC).

| Method | MAE (MW) | RMSE (MW) | nMAE (%) |
|---|---:|---:|---:|
| **LightGBM (global)** | **271.05** | **554.8** | **3.87** |
| Random Forest | 271.09 | 566.5 | 3.87 |
| Ridge | 335.19 | 719.6 | 4.79 |
| Persistence (same hour yesterday) | 335.30 | 746.4 | 4.79 |
| EIA day-ahead forecast | 370.41 | 865.6 | 5.29 |
| Weekly naive | 604.89 | 1,331.4 | 8.64 |

- LightGBM lowers MAE by **19.2%** vs. persistence (day-block bootstrap 95% CI of the difference: −79.9 to −49.0 MW) and by **26.8%** vs. the EIA forecast.
- LightGBM and Random Forest are effectively **tied** (the 0.04 MW gap is far below the across-seed standard deviation).
- The EIA forecast has the **lower MAE in four of eight BAs** (AZPS, BPAT, PGE, SRP). The pooled gain over EIA comes mostly from CISO.

### Results by balancing authority

| BA | Test rows | LightGBM MAE | Persistence MAE | EIA MAE | LightGBM nMAE (%) |
|---|---:|---:|---:|---:|---:|
| AZPS | 4,424 | 200.3 | 203.9 | **180.8** | 4.37 |
| BPAT | 4,425 | 192.4 | 238.1 | **121.5** | 2.89 |
| CISO | 4,233 | **1,014.7** | 1,406.2 | 1,824.0 | 3.63 |
| LDWP | 4,091 | **180.2** | 203.7 | 255.1 | 6.59 |
| NEVP | 4,425 | **189.4** | 193.8 | 353.7 | 3.74 |
| PACW | 4,425 | **91.8** | 118.8 | 95.9 | 3.72 |
| PGE | 4,425 | 112.1 | 134.8 | **31.4** | 4.23 |
| SRP | 4,424 | 212.2 | 219.4 | **155.1** | 4.72 |

All MAE values are in MW; the lowest in each row is in bold. LightGBM beats persistence in all eight BAs, but the margin ranges from 1.8% (AZPS) to 27.8% (CISO). The EIA MAE for PGE (31.4 MW) is suspiciously low and was not investigated.

---

## Research questions and findings

### 1. Does a global model beat simple baselines and the official forecast?

Yes on average, with caveats. LightGBM has the lowest pooled MAE, beats persistence in all eight BAs and in all six rolling-origin windows (error ratio 0.74 to 0.87), but the margin is small at AZPS and NEVP and the official forecast wins in four BAs.

### 2. Is it reliable on new BAs, on extreme hours, and in its uncertainty estimates?

Only partly.

- **Leave-one-BA-out (unseen BA).** With raw MW inputs the model fails to beat persistence at two of eight BAs (BPAT, CISO), and the CISO error reaches **19,017 MW**. With inputs normalised by each BA's mean demand it beats persistence in six of eight (it fails at AZPS and SRP), but it needs some history for the new BA.
- **Prediction intervals.** One pooled 90% split-conformal interval covers **88.4%** of all test hours, but only **74.9%** of extreme hours and **39.5%** of CISO hours (PACW: 99.9%). Per-BA calibration narrows the spread across BAs (82.1% to 91.5%) but lowers overall coverage to 86.9%.
- **Extreme-hour alarm** (demand above each BA's own 90th training percentile):

  | Method | Precision | Recall | F1 | PR-AUC |
  |---|---:|---:|---:|---:|
  | LightGBM | 0.884 | 0.863 | 0.873 | 0.949 |
  | EIA day-ahead | 0.894 | 0.859 | 0.876 | 0.941 |
  | Persistence | 0.861 | 0.862 | 0.861 | 0.937 |
  | Weekly naive | 0.693 | 0.727 | 0.710 | 0.762 |

  The model adds little over a free rule for flagging high-demand hours. Its MAE on extreme hours is 2.4 times that on ordinary hours (511.8 vs. 213.4 MW), and peaks are under-forecast on average (−207.9 MW).

### 3. Do grid CO₂ intensity features help predict demand?

No clear evidence that they do. Adding four CO₂ features raised MAE from 271.90 to 274.47 MW (+0.94%). The day-level bootstrap 95% interval of the difference (−2.0 to +5.8 MW) includes zero. CO₂ and demand are correlated within BAs (up to 0.55), but the features add no accuracy to a model that already sees recent demand. This is an association test only, with no causal claim.

---

## Method overview

**Data.** Two public [EIA-930 Hourly Electric Grid Monitor](https://www.eia.gov/electricity/gridmonitor/) files: a primary file with hourly demand and each BA's day-ahead forecast (63 BAs, 2022 to 2024) and a secondary file for eight BAs with CO₂ intensity columns. Study BAs: AZPS, BPAT, CISO, LDWP, NEVP, PACW, PGE, SRP. Timestamps are UTC.

**Task.** Predict demand 24 hours ahead from information available at the forecast origin: ŷ(b, t+24) = f(x(b, t)), with one function fitted for all BAs.

**Features (15).** Current demand, lags at 1, 2, 3, 24, 48 and 168 hours, 24-hour rolling mean and standard deviation, one-hour difference, four calendar variables (UTC hour, day of week, month, day of year) and a BA code. Features are built with DuckDB window functions on a complete hourly grid, so a lag of *k* rows is exactly *k* hours.

**Models.** Ridge, Random Forest (300 trees) and LightGBM (800 rounds, learning rate 0.03, 31 leaves), all global and untuned. Random Forest and LightGBM are run with five seeds. Persistence, weekly naive and the EIA forecast are baselines or comparators. The EIA forecast is never a model input.

**Split.** By target time, never at random. Training targets run from 2022-01-09 to 2024-06-30 (the last three months serve as the validation window), and test targets from 2024-07-01 onward.

**Evaluation.** MAE, RMSE, nMAE, sMAPE and MASE, with day-level bootstrap confidence intervals, rolling-origin evaluation (six windows), leave-one-BA-out, split-conformal intervals, an extreme-hour alarm, and a CO₂ feature ablation. Permutation importance and SHAP show that current demand and the 24-hour mean dominate the model.

**Data-quality and leakage checks.** Invalid values are set to missing (no interpolation), and any row whose lag spans a gap is excluded. Five leakage checks confirm that the target equals the demand at t+24, that every lag matches, that features rebuilt from truncated history match the SQL, that the forward-looking function appears only in the target and the EIA comparator, and that training targets precede test targets.

---

## Repository structure

```
.
├── Notebook A.ipynb      # <!-- TODO: one-line description -->
├── Notebook B.ipynb      # <!-- TODO: one-line description -->
├── sql/                  # DuckDB SQL (features, target, comparator)
├── data/processed/       # Processed tables used by the notebooks and dashboard
├── tableau/              # Tableau dashboard and its source CSVs
├── report/               # <!-- TODO: describe contents -->
├── docs/                 # Project paper (PDF)
├── requirements.txt
└── README.md
```

---

## How To Run

```bash
git clone https://github.com/comma-loop/Electricity-Demand-Forecasting-Across-Eight-Western-US-Balancing-Authorities-Using-EIA930-Data.git
cd Electricity-Demand-Forecasting-Across-Eight-Western-US-Balancing-Authorities-Using-EIA930-Data

python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

1. Download the two EIA-930 files from the [EIA Grid Monitor](https://www.eia.gov/electricity/gridmonitor/) and place them in `data/raw/` <!-- TODO: exact file names and folder expected by the notebooks -->.
2. Run `Notebook A.ipynb`, then `Notebook B.ipynb`, from top to bottom. <!-- TODO: confirm order and what each produces -->

---

## Tableau dashboard

The `tableau/` folder holds two connected dashboards: one for demand characteristics (scale and seasonality across BAs) and one for forecasting performance (model vs. baselines vs. EIA). <!-- TODO: add screenshot (e.g. tableau/dashboard.png) or a Tableau Public link -->

---

## Limitations

- One half-year (July to December 2024) in one region, so season and drift cannot be separated.
- No weather or holiday data, and the learners were not tuned.
- The EIA forecast is issued at a different time with different information, and it was not scored on extreme hours alone.
- Conformal calibration covers April to June only, which is why coverage falls short on the summer test period.
- The CO₂ units are undocumented, and we did not verify that CO₂ values at hour *t* are available when a forecast is issued.
- Leave-one-BA-out results say nothing about BAs outside these eight.
- Software versions were only partly recorded (pandas 2.3.3, DuckDB 1.5.6).

## Future Improvements

- [ ] Add weather features (temperature) and a holiday calendar
- [ ] Extend to a second year of data
- [ ] Calibrate conformal intervals per BA over a longer window; try adaptive conformal inference
- [ ] Tune the models
- [ ] Score the EIA forecast on extreme hours and investigate its PGE series
- [ ] Run leave-one-BA-out on more BAs
- [ ] Add linear-lag and pretrained-forecaster baselines
- [ ] Small demo app showing next-day forecasts with intervals

## Changelog

- **v1.0** (tag `v1.0-paper`): eight BAs, global LightGBM, baselines, conformal intervals, extreme-hour alarm, CO₂ ablation, Tableau dashboard. Results as reported in the [paper](docs/EIA930_demand_forecasting_paper.pdf).

---

## Paper

The full methodology, tables and discussion are in the paper: [`docs/EIA930_demand_forecasting_paper.pdf`](docs/EIA930_demand_forecasting_paper.pdf).

*Hourly Electricity Demand Forecasting from EIA-930 Data with Global Gradient Boosting against the Official Day-ahead Forecast on Extreme Days, with a Test of Grid CO2 Features.*

The paper is an unpublished report on **v1** of this project, and all results in this README come from that version. Later improvements are documented in the [changelog](#changelog) and are not covered by the PDF. <!-- TODO: add preprint link (arXiv / Zenodo / SSRN) here if you post one -->

## Authors

Truong Huy An, Bui Le Khanh Dang and Le Vo Minh Thu, FPT University, Ho Chi Minh City, Viet Nam.

## Data source and license

Data: U.S. Energy Information Administration, [Hourly Electric Grid Monitor (EIA-930)](https://www.eia.gov/electricity/gridmonitor/). <!-- TODO: add a LICENSE file (e.g. MIT) and name it here -->

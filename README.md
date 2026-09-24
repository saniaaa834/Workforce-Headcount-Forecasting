# 📈 HR Workforce Headcount Forecasting: Time Series Analysis

![Python](https://img.shields.io/badge/Python-3.9%2B-blue)
![Jupyter](https://img.shields.io/badge/Notebook-Jupyter-orange)
![Models](https://img.shields.io/badge/Models-ARIMA%20%7C%20SARIMA%20%7C%20Holt--Winters-green)
![MAPE](https://img.shields.io/badge/Holdout%20MAPE-1.74%25-brightgreen)

A time series project that forecasts **monthly active workforce headcount 12 months ahead**, so hiring, retention and payroll can be planned against a quantified range instead of a single guess.

---

## 📌 Table of Contents
1. [Problem Statement](#-problem-statement)
2. [Dataset](#-dataset)
3. [Methodology](#-methodology)
4. [Key Results](#-key-results)
5. [12-Month Forecast and Operating Plan](#-12-month-forecast-and-operating-plan)
6. [Limitations & Honest Caveats](#-limitations--honest-caveats)
7. [Repository Structure](#-repository-structure)
8. [How to Run](#-how-to-run)
9. [Tech Stack](#-tech-stack)
10. [Author](#-author)

---

## 🎯 Problem Statement

Organisations that hire ahead of demand overspend on payroll, and those that hire behind it burn out teams and pay premium recruiting costs. This project answers:

- How many active employees will the company have over the **next 12 months**?
- How **uncertain** is that number, and how does the uncertainty grow with the horizon?
- What does the forecast mean in practice: **gross hires, recruiter capacity and payroll**?

## 📊 Dataset

- **Grain:** an employee-month panel (one row per employee per monthly snapshot), about 18,360 rows covering roughly 800 employees.
- **Period:** 48 monthly snapshots, **Apr-2021 to Mar-2025**.
- **Target (built in the notebook):** headcount at month *t* = number of employees with `employment_status == 'Active'` in that month's snapshot.
- **Data quality:** the two columns needed to build the target (`snapshot_date`, `employment_status`) are 100% complete, so no imputation was needed. There are no duplicate employee-month rows and no missing months.
- **Missing values, handled deliberately:**
  - `termination_date` (~78% blank) is *structurally* missing: blank means "still employed". It was **not** imputed, because that would invent terminations.
  - `manager_id` (100% empty) and `last_promotion_date` (~99% empty) were dropped.

> The raw dataset is **not included** in this repository because it contains employee-level fields such as birth dates and salaries. To reproduce the results, place your copy at `data/snapshots_updated.csv`.

## 🔬 Methodology

1. **Data preparation:** parsed dates day-first (`30-04-2021`) to avoid silent day/month swaps, then aggregated the panel into a monthly series with an explicit `MS` frequency.
2. **Outlier screening:** screened month-over-month change on both absolute and percentage scales. The March-2025 spike (+22) is flagged only on the absolute scale, is an ordinary month in percentage terms, and matches real hire records (22 hires, 5 exits), so it was **retained**. A sensitivity check confirmed that capping it would only delete 4 real employees from the baseline.
3. **Exploratory analysis:** headcount grew 3.6x (149 to 538), about 38% CAGR, with **no month of net decline**. Growth is accelerating, not linear.
4. **Decomposition:** classical additive, classical multiplicative and robust STL. Seasonal strength was below the 0.3 "negligible" threshold in all three (0.24 to 0.27), while trend strength was about 0.99.
5. **Stationarity:** ADF and KPSS run together. The series needs **d = 2** (two differences) before both tests agree, which is the statistical signature of accelerating growth.
6. **Seasonality checks:** year-overlay plot, decomposition strength, ACF at lag 12 and Seasonal Naive performance all point to little or no annual seasonality.
7. **Models compared (16 in total), all on one chronological 38/10 train-test split with static multi-step forecasts:**
   - *Baselines:* Naive, Seasonal Naive, Naive + Drift
   - *Moving averages:* SMA and EMA (windows 3 and 6)
   - *Exponential smoothing:* SES, Holt, damped Holt, Holt-Winters (additive and multiplicative), auto-ETS
   - *ARIMA:* grid search over p and q with **d fixed at 2**, so AIC values are comparable
   - *SARIMA and SARIMAX:* bounded grids, plus monthly hires as an exogenous regressor
8. **Selection:** the champion is chosen on **out-of-sample RMSE**, not AIC, because likelihoods of models fitted to differently transformed data are not comparable.
9. **Diagnostics and robustness:** Ljung-Box, Jarque-Bera, Shapiro-Wilk, Durbin-Watson, forecast bias, a 12-month alternative holdout, and outlier sensitivity.
10. **Forecast:** refit on all 48 months, forecast 12 months with **80% (planning) and 95% (risk)** prediction intervals, then convert to an operating plan and scenario analysis.

## 🏆 Key Results

**Top models on the 10-month holdout (lower is better):**

| Rank | Model | MAE | RMSE | MAPE (%) |
|---|---|---|---|---|
| 1 | **Holt-Winters (additive, additive)** | **8.79** | **13.17** | **1.74** |
| 2 | SARIMA(0,1,2)x(1,1,1)[12] | 10.15 | 14.34 | 2.02 |
| 3 | ETS(M,A,N) | 9.74 | 14.42 | 1.92 |
| 4 | Holt (linear trend) | 9.86 | 14.57 | 1.95 |
| 5 | SARIMAX (+ hires) | 10.68 | 14.63 | 2.13 |
| 9 | Naive + Drift | 22.90 | 29.85 | 4.58 |
| 10 | Naive (last value) | 61.40 | 72.73 | 12.49 |

- The champion cuts RMSE by **81.9%** versus the naive benchmark, with an average miss of about **8.8 employees per month (1.74% of headcount)**.
- **Modelling the trend was the single most valuable decision.** Adding drift to the naive model, or a slope to smoothing, delivers most of the accuracy gain.
- Level-only and moving-average models perform *worse* than naive on this steadily rising series, because they anchor to a lower past level.
- Adding hires as an exogenous variable did **not** improve accuracy (RMSE rose by 0.29), even though the coefficient was statistically significant and the back-test used actual future hires.

## 🔮 12-Month Forecast and Operating Plan

| | Value |
|---|---|
| Baseline headcount (Mar-2025) | 538 |
| Forecast at month 12 (Mar-2026) | **726** (+188, about +34.9%) |
| 80% planning range at month 12 | 713 to 791 |
| 95% risk range at month 12 | 693 to 811 |
| Interval widening (95%), month 1 to month 12 | about 10.6x |

**Derived plan:** with average monthly attrition of about 1.39%, the business needs roughly **295 gross hires** over 12 months (about 25 per month, peaking at 28), which is around **6 recruiter FTEs** at 4 hires per recruiter per month. Incremental annualised payroll at month 12 is roughly 18.8 million (in the dataset's currency).

**Planning rule from the interval widths:** commit firmly to months 1 to 4, where the band is tight, and treat months 5 to 12 as a range reviewed quarterly against actuals.

The notebook also includes scenario analysis (hiring freeze, conservative, base, aggressive, stretch), a risk register and an executive summary with recommendations.

## ⚠️ Limitations & Honest Caveats

- **The champion is not stable.** On a 12-month holdout the winner flips to SARIMA(0,1,2)x(1,1,1)[12]. The gap on the primary split is only about 9% on 10 test points, so the two should be treated as **co-equal validated alternatives**, not as a clear winner and loser.
- **The champion barely uses its seasonal terms.** The fitted Holt-Winters has γ = 0, so it behaves largely like a trend model. Smoothing models also did not go through the same Ljung-Box residual validation as ARIMA and SARIMA.
- **Short series.** 48 observations is only 4 annual cycles, so every seasonal conclusion, including "little or no seasonality", rests on limited evidence.
- **No downturn in the history.** Not one month of net decline appears, so no model can forecast a contraction. The forecast is a *momentum projection*, and because d = 2 extrapolates acceleration, the far horizon should be read as an upper bound.
- **Over-parameterised SARIMA.** Some SARIMA coefficients sit on the boundary with very large standard errors, a typical symptom of too many parameters for 38 training points.
- **Intervals cover estimation uncertainty only**, not structural-break risk such as a hiring freeze, which is the bigger practical risk.

## 📁 Repository Structure

```
HR-Workforce-Headcount-Forecasting/
├── README.md
├── LICENSE
├── requirements.txt
├── .gitignore
├── HR_Workforce_Headcount_Forecasting.ipynb
└── data/
    └── snapshots_updated.csv      (not included; add your own copy)
```

Running the notebook also writes these result files: `model_comparison.csv`, `headcount_forecast_12m.csv`, `workforce_operating_plan.csv`, `scenario_analysis.csv`, `residual_diagnostics.csv`.

## 🚀 How to Run

```bash
# 1. Clone the repository
git clone https://github.com/saniaaa834/HR-Workforce-Headcount-Forecasting.git
cd HR-Workforce-Headcount-Forecasting

# 2. (Optional) create a virtual environment
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

# 3. Install dependencies
pip install -r requirements.txt

# 4. Add the dataset at data/snapshots_updated.csv, then launch the notebook
jupyter notebook HR_Workforce_Headcount_Forecasting.ipynb
```

The notebook looks for the CSV in several common locations, including `./data/snapshots_updated.csv`, and works in Jupyter or Google Colab.

## 🛠️ Tech Stack

- **Language:** Python
- **Data handling:** Pandas, NumPy
- **Visualisation:** Matplotlib
- **Time series and statistics:** Statsmodels (STL, ADF, KPSS, ETS, ARIMA, SARIMAX), SciPy
- **Evaluation:** Scikit-learn (MAE, RMSE)

## 👤 Author

- Sania Sheikh

## 📄 License

Released under the [MIT License](LICENSE).

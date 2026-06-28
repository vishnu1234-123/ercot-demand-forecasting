# ERCOT Demand Forecasting & Distributional-Shift Detection

Short-term forecasting of ERCOT (Texas) hourly electricity demand, with a focus on a question that matters more than raw accuracy: **when a model trained on normal conditions meets a structural shock, can we detect that it has broken, and respond sensibly?** The shock studied here is Winter Storm Uri (February 2021).

## The question

Can a simple model forecast ERCOT hourly demand better than a naive baseline — and, when a regime break hits, do our monitors catch the model degrading in time to act?

## What it does

- Pulls hourly ERCOT demand (MWh) from the EIA API (June 2020 – June 2021).
- Builds a **seasonal-naive baseline** (demand = same hour one week ago) and a **linear forecaster** on time features plus a weekly lag.
- Evaluates **out-of-sample** with a time-ordered split (Uri sits in the test set, so the model is trained only on normal conditions).
- Detects drift two ways:
  - **Error monitor** (lagging): a causal, anomaly-aware threshold on rolling forecast error — no look-ahead, and the shock is excluded from its own baseline so it doesn't self-poison.
  - **PSI monitor** (leading-ish): tracks how far the recent demand distribution has drifted from a calm reference period.
- **Triage:** flags drift as CONFIRMED only when both signals agree, filtering benign seasonal drift down to a softer WATCH state.

## Key findings

- The linear model modestly beats the naive baseline in normal conditions (~10.0% vs ~10.5% MAPE); both roughly **double** their error during Uri (~24% vs ~26%).
- A causal error monitor would have alarmed at Uri onset using only past data.
- Triage correctly separates the real emergency (both signals firing, mid-February) from benign spring seasonal drift (PSI elevated, but error fine).
- During the rolling blackouts, measured demand stops meaning "demand" and becomes "served load" — the variable's *meaning* changes, not just its distribution. No model trained on normal demand should be trusted through such an event; the right response is to flag and fall back to conservative judgment, not to retrain on the shock.

## How to run

```bash
pip install -r requirements.txt
jupyter notebook analysis.ipynb   # or open in VS Code
```

Then **Run All**. Data is cached in `data/ercot_demand.csv`, so the notebook runs end-to-end with no setup. To re-pull fresh data, add a free EIA API key to `.env` (see `.env.example`) — the notebook uses the cached CSV if present and only calls the API otherwise.

## Project structure

ercot-demand-forecasting/

├── analysis.ipynb        Full analysis, runs top-to-bottom

├── README.md             This file

├── requirements.txt      Dependencies

├── .env.example          Template for the optional EIA API key

└── data/

└── ercot_demand.csv  Cached hourly demand (so the notebook runs key-free)


## Notebook structure

1. Data load (with CSV caching) and cleaning / index verification
2. Exploratory analysis + data-driven anomaly detection (run before modeling)
3. Time-ordered train/test split + seasonal-naive baseline
4. Feature engineering + linear forecaster
5. Error decomposition by regime (normal vs Uri)
6. Drift monitor #1 — causal, anomaly-aware forecast-error monitor
7. Drift monitor #2 — PSI input-distribution drift
8. Triage — combining both signals into CONFIRMED / WATCH states

## Limitations & next steps

- **No weather features.** Temperature is the dominant driver of electricity demand; adding it would shrink most non-Uri error spikes. This is the single highest-leverage improvement.
- **Single forward-chained split**, chosen to study the break cleanly. For a robust accuracy estimate, expanding-window walk-forward cross-validation would be the next step.
- **PSI reference is fixed at January**, so PSI reads elevated into spring as the season shifts. A production system would refresh the reference periodically and monitor feature-level drift, not just the demand distribution.
- Uses revised EIA data, not real-time vintages, so real-time accuracy would be modestly lower.




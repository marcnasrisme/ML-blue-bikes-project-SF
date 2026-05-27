# BlueBikes Demand Forecasting & Morning Rebalancing

A coupled learning-and-optimization pipeline for bike-share rebalancing: predict station-level hourly net demand, then optimize 5AM starting inventories to minimize the number of times stations go empty (no bikes) or full (no docks).

**MIT 15.095 — Machine Learning Under a Modern Optimization Lens**
Constantinos Barmpouris & Marc Nasr · Fall 2025

## Headline result

| Method | Empty events | Full events | Total |
|---|---|---|---|
| Baseline (real 5AM system) | 53 | 49 | 102 |
| **Point-prediction MILP** | **32** | **35** | **67** |
| Leaf-averaged MILP | 436 | 104 | 540 |

The point-prediction MILP **reduces total empty/full events by 34%** on a representative evaluation day — and it improves *both* dimensions simultaneously, not by trading one failure mode for the other.

The leaf-averaged variant fails spectacularly (5.3× more events) — a useful lesson in itself, see below.

## The approach

The project combines two ideas:

1. **Predict hourly net demand** `d̂[s,h]` (pickups minus returns) per station per hour using an **Optimal Regression Tree** (Bertsimas & Dunn, 2017). `R² ≈ 0.73` on held-out data. Dominant splits are lag features (previous-day same-hour, 3-day rolling mean) — confirming strong daily periodicity.

2. **Embed those forecasts into an allocation MILP** that decides 5AM bike inventories per station to minimize empty/full events over the day, subject to a fixed fleet size and station-by-station dock capacity.

```
                         ┌────────────────────────┐
   feature matrix ────►  │  Optimal Regression    │  ────► d̂[s,h]  (predicted net demand)
   (lags, weather,       │  Tree (depth=5)        │
   time, station)        └────────────────────────┘
                                                          │
                                                          ▼
                         ┌────────────────────────┐
   fleet B_tot, ──────►  │  MILP: choose x[s]     │  ────► 5AM allocation
   capacities C[s]       │  to minimize Σ events  │        + simulated trajectory
                         └────────────────────────┘
```

## The MILP

**Decision variables**

| Variable | Meaning |
|---|---|
| `x[s] ∈ Z+` | bikes staged at station `s` at 5AM |
| `b[s,h] ∈ Z+` | bikes available at station `s` at start of hour `h` |
| `z_empty[s,h] ∈ {0,1}` | indicator: station `s` empty at hour `h` |
| `z_full[s,h] ∈ {0,1}` | indicator: station `s` full at hour `h` |

**Objective**

```
min  Σ_{s,h} (z_empty[s,h] + z_full[s,h])
```

**Key constraints**

- Fleet conservation: `Σ_s x[s] = B_tot`
- Initial inventory: `b[s,5] = x[s]`
- Inventory dynamics: `b[s,h+1] = b[s,h] + d̂[s,h]`
- Capacity bounds: `0 ≤ b[s,h] ≤ C[s]`
- Event linking: `b[s,h] ≥ 1 - z_empty[s,h]`, `b[s,h] ≤ C[s] - 1 + z_full[s,h]`

Solved in JuMP + Gurobi.

## Why the leaf-averaged variant collapses

Replacing point forecasts with the mean of historical outcomes in the same ORT leaf is intuitively "more robust" — but it's actively harmful here. The MILP objective counts **boundary hits** (inventories crossing 0 or `C[s]`), which is a discontinuous objective. Smoothing out demand extremes removes precisely the signal needed to prevent stockouts.

This is the conceptual punchline of the project: **predictive smoothing is not a free lunch when your downstream objective is sensitive near feasibility boundaries.**

## What the optimizer actually does

The point-prediction MILP creates a more **polarized 5AM allocation** than the baseline — it concentrates bikes at predicted-depleting stations and starves predicted-receiving stations to free up the fleet. This donor/receiver structure is what drives the simultaneous reduction in both empty and full events.

If donor/receiver classification is wrong (e.g. due to censoring or atypical days), the prescription degrades sharply — because the same bikes that prevent stockouts at one station must be taken from another.

## Limitations

- **Censoring bias**: when a station is empty or full, additional demand attempts are unobserved, so the ORT systematically underestimates depletion/accumulation at exactly the stations we care most about.
- **One-shot 5AM plan**: no mid-day re-rebalancing. A rolling-horizon version is the obvious extension.
- **Deterministic forecasts**: prediction errors are correlated across stations (commute structure) and across time (weather), which a point-forecast plan can't hedge against. Scenario-based or quantile-based formulations are next steps.

## Data

[SF Bay Area Bike Share dataset (Kaggle)](https://www.kaggle.com/datasets/benhamner/sf-bay-area-bike-share) — used as a BlueBikes-style proxy. Station metadata, trip records, daily weather, station status (largest table). ~6 months of hourly station-panel data.

Raw CSVs are not committed (~2GB total). See notebooks for preprocessing.

## Repo contents

- `notebookml.ipynb` — main pipeline: feature engineering + ORT training + evaluation
- `JuliaNotebook.ipynb` — JuMP/Gurobi MILP formulation and solution
- See the project report PDF for the full 15-page write-up

## Team contributions

| | Marc Nasr | Constantinos Barmpouris |
|---|---|---|
| Data joins + cleaning | ✓ | |
| Feature engineering (time / weather / lags) | ✓ | |
| ORT training + tuning | ✓ | |
| Predictive performance analysis | ✓ | |
| MILP formulation + event-logic | | ✓ |
| JuMP+Gurobi optimization pipeline | | ✓ |
| Solution behavior analysis | | ✓ |
| Report writing | ✓ | ✓ |

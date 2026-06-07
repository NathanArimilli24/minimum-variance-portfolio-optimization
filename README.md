# Minimum Variance Portfolio Optimization

**Sample covariance vs. Ledoit-Wolf shrinkage, evaluated with a rolling-window out-of-sample backtest.**

FIN 684: Investment Theory / Advanced Corporate Finance (Prof. Sury) — UT Austin, McCombs School of Business
Team: Nathan Arimilli, Shahmir Javed, Joseph Bailey

---

## Problem

Minimum-variance portfolios are only as good as the covariance matrix they are built on. Estimating a
covariance matrix from historical returns is noisy: with 20 assets there are 210 unique covariance
parameters to estimate from a short history (here, 36 monthly observations per window), so the sample
estimate tends to overfit. That produces unstable, highly concentrated portfolios that trade heavily and
generalize poorly out of sample.

The **Ledoit-Wolf shrinkage estimator** pulls the noisy sample covariance toward a structured target
(a scaled identity matrix), with the shrinkage intensity chosen analytically to minimize expected error.
This project asks a practical question: **does shrinkage actually produce more stable, more implementable
minimum-variance portfolios out of sample, and at what cost to return?**

## Approach

- **Data:** monthly CRSP stock returns pulled from **WRDS** for 20 blue-chip stocks across six sectors
  (AAPL, MSFT, GOOG, IBM, NVDA, JPM, BAC, MS, GS, JNJ, PFE, MRK, KO, PG, WMT, HD, GE, CAT, MMM, XOM),
  2010-2024.
- **Optimization:** constrained quadratic program (minimize wᵀΣw subject to fully invested weights and
  long/short position limits) solved with `cvxpy`, with OSQP → ECOS → SCS solver fallbacks.
- **Covariance estimators:** sample covariance and Ledoit-Wolf shrinkage (`sklearn.covariance.LedoitWolf`).
- **Backtest:** 36-month rolling estimation window, monthly rebalancing. Each month, covariance is
  estimated on the trailing window and the resulting weights are applied to the **following** month's
  returns (genuine out-of-sample, no look-ahead). Demo runs Jan 2014 – Dec 2024 (132 months).
- **Diagnostics:** annualized return/volatility, Sharpe, Sortino, max drawdown, monthly turnover
  (Σ|Δw|), and weight stability (cross-sectional standard deviation of weights).

## Results (default demo)

| Metric | Sample covariance | Ledoit-Wolf |
|---|---:|---:|
| Total return (2014-2024) | 315.79% | 242.30% |
| Annualized return | 13.83% | 11.84% |
| Annualized volatility | 18.67% | 14.03% |
| Sharpe ratio | 0.570 | 0.576 |
| Sortino ratio | 0.777 | 0.930 |
| Maximum drawdown | −30.88% | −19.20% |
| Avg. monthly turnover | 106.7% | 25.2% |
| Median monthly turnover | 87.6% | 23.4% |

**Takeaway.** The sample method earned a higher gross return (~2%/year more), but only by taking on
much more risk: ~33% higher volatility, a far deeper drawdown (−30.9% vs −19.2%), and a worse Sortino
ratio. It also traded ferociously — average monthly turnover above 100% versus ~25% for Ledoit-Wolf —
with positions routinely flipping between large long and large short bets (the final-period sample
portfolio holds JPM at +50.9% against PG at −45.7%, whereas Ledoit-Wolf's largest positions are GOOG at
+20% and CAT at −18%). Shrinkage delivers a slightly better Sharpe with dramatically more stable,
implementable portfolios.

**Transaction costs.** Turnover this high is expensive in practice. Charging a cost of *c* per unit of
one-way turnover, the annual drag is roughly `12 × c × (avg turnover)`:

| Cost per unit turnover | Sample drag (≈) | Ledoit-Wolf drag (≈) | Differential |
|---|---:|---:|---:|
| 10 bps | ~1.3%/yr | ~0.3%/yr | ~1.0%/yr |
| 20 bps | ~2.6%/yr | ~0.6%/yr | ~2.0%/yr |

Even at a modest **10 bps**, costs erode roughly half of the sample method's ~2% gross return advantage.
The advantage disappears entirely near **~20 bps**, beyond which Ledoit-Wolf wins on *net* return as well
— on top of already winning on volatility, drawdown, turnover, and downside risk. For a real mandate, the
shrinkage portfolio is the more defensible choice.

Figures (in [`output/`](output/)): rolling-window performance, turnover analysis, weight stability, and
the final-period efficient frontier.

## Repository structure

```
.
├── README.md
├── code/
│   └── Project_2.ipynb      # full pipeline: data → optimization → backtest → diagnostics
├── data/
│   └── README.md            # how to obtain the WRDS/CRSP data (not redistributable)
├── output/                  # saved results from the demo run
│   ├── performance_summary.csv
│   ├── final_weights_sample.csv
│   ├── final_weights_ledoit_wolf.csv
│   ├── turnover_stability_diagnostics.csv
│   └── fig1..fig4_*.png
└── report/
    ├── Project2_Memo.pdf
    ├── Minimum Variance Portfolio Optimization.pptx
    └── Assignment - Minimum Variance Portfolio Optimization (FIN 684).pdf
```

## Reproducing the analysis

**Requirements:** Python 3.10+, a **WRDS account** (the data comes from WRDS/CRSP and is not
redistributable), and:

```bash
pip install wrds cvxpy ecos scs osqp scikit-learn pandas numpy matplotlib seaborn scipy tqdm
```

**Run it:**

- *Interactive* — open `code/Project_2.ipynb` and run all cells. The final cell calls
  `run_portfolio_optimization_backtest()`, which prompts for your WRDS login and each parameter
  (press Enter to accept the defaults that produced the results above).
- *Headless / reproducible* — connect once and pass an explicit config:

  ```python
  db = connect_to_wrds()
  config = make_config(db, save_results=True)        # defaults reproduce the demo
  results = run_portfolio_optimization_backtest(config=config,
                                                save_results=True,
                                                output_dir='../output')
  ```

## Honest caveats

- **WRDS is required to run the pipeline.** CRSP data is licensed and cannot be committed here, so the
  repo ships the saved demo outputs (figures + summary CSVs) but not the raw returns. The full 132-month
  monthly statistics CSV regenerates only when you re-run with WRDS access.
- **Results are dataset-specific.** They cover one 20-stock universe over 2014-2024. Other universes,
  periods, or constraint settings can shift the comparison; the program is built to let you re-run with
  any of those.
- **Costs are illustrative.** The transaction-cost figures above apply a simple per-unit-turnover charge,
  not a calibrated market-impact model.
- **CRSP ticker mapping.** Returns are joined to tickers via `crsp.msenames` over valid date ranges;
  historical ticker reuse is a known CRSP subtlety and is handled with date-bounded name matching and
  date/ticker de-duplication.

## Notes

Course project for FIN 684. 

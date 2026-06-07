# Data

This project uses **monthly CRSP stock returns retrieved from WRDS** (Wharton Research Data Services).

CRSP data is licensed and **cannot be redistributed**, so no return data is committed to this repo.
To reproduce the analysis you need your own WRDS account.

## What the code pulls

For each ticker in the universe, monthly returns (`ret`) from the CRSP monthly stock file, joined to
ticker names over their valid date ranges:

```sql
SELECT date, ticker, ret
FROM crsp.msf a
LEFT JOIN crsp.msenames b
  ON a.permno = b.permno
WHERE DATE_TRUNC('month', b.namedt)    <= DATE_TRUNC('month', a.date)
  AND DATE_TRUNC('month', a.date)      <= DATE_TRUNC('month', b.nameendt)
  AND a.date BETWEEN '<start>' AND '<end>'
  AND ticker IN (<tickers>)
  AND ret IS NOT NULL
ORDER BY date, ticker
```

The default demo universe is 20 blue-chip stocks (AAPL, MSFT, GOOG, IBM, NVDA, JPM, BAC, MS, GS, JNJ,
PFE, MRK, KO, PG, WMT, HD, GE, CAT, MMM, XOM) over 2010-2024.

## How to obtain it

1. Get a WRDS account: https://wrds-www.wharton.upenn.edu/
2. `pip install wrds`, then run `wrds.Connection()` and enter your credentials (the code does this for
   you via `connect_to_wrds()`).
3. Run the notebook in `code/` — it fetches, validates, and caches the returns in memory for the backtest.

If you save results (`save_results=True`), the run also writes the monthly backtest statistics and
portfolio weights as CSVs to `output/`.

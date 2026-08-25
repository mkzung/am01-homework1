# AM01 — Homework 1

Applied Statistics with Python (AM01), London Business School, MAM2027.
Author: Team 8

## Contents

| File | What it is |
|---|---|
| `team8-homework1.ipynb` | the assignment notebook |
| `STOCKS_EXPLAINED.md` | walkthrough of the stocks section: the finance, the pandas, and how to read each plot |
| `MOVIES_EXPLAINED.md` | walkthrough of the movies section: duplicates, aggregation, the three scatterplots |
| `PANDAS_DRILLS.md` | paper exercises for practising these patterns from memory |

## Status

- **Stocks (NYSE / yfinance)** — done: companies per sector, price download, daily & monthly returns, summary statistics, density plot, risk-return plot, both written answers.
- **Movies (IMDB)** — done: missing values & duplicates, counts by genre, return on budget, top 15 directors, ratings distribution, three scatterplots, all written answers.
- **San Francisco rents** — not started (deliberately left out for now).

## Running it

The notebook needs the AM01 course environment (uv, Python 3.12):

```bash
uv sync
uv run jupyter lab
```

Data loads from the local `../data/` folder when present and otherwise falls back to the
course repository over HTTPS, so the notebook also runs in Colab.

## Stocks analysed

AAPL, JPM, DIS, DPZ, ANF, XOM, plus SPY as the benchmark.
Window: five years to the run date, monthly returns.

## Collaboration

Submitted as **Team 8**. The write-up of the density plot in the stocks section came from a
teammate; two of its figures were checked against the data and corrected before use (see the
note at the end of `STOCKS_EXPLAINED.md`).

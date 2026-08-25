# Homework 1 — the stocks section, explained

A line-by-line walkthrough of what the code does and why, so you can defend every cell.
This file is **for you, not for submission** — Kostis asks for a clean stand-alone notebook.

---

## 1. The finance you need (five minutes)

**Adjusted close, not close.** A raw closing price lies to you. If a stock splits 2-for-1, the price halves overnight, but you are not half as rich — you own twice as many shares. Dividends do the same in reverse: the price drops on the ex-dividend date even though you received cash. The *adjusted* close rewrites history so that a change in the series equals the actual return to a shareholder. That is why the code passes `auto_adjust=False` and then selects `"Adj Close"` — it asks yfinance to keep the explicit adjusted column rather than silently overwriting the raw one.

**A return is relative.** Buy at 100, sell at 101.75, and your return is (101.75 − 100) / 100 = 1.75%. Prices are not comparable across stocks — a \$500 share is not "better" than a \$20 one — but returns are. Everything from here on works with returns.

**Monthly, not daily.** Daily returns are mostly noise: about 250 observations a year, nearly all hovering near zero, and the density plots collapse into identical spikes. Monthly returns keep enough observations to be statistically meaningful (67 here) while making the differences between stocks visible.

**Two numbers carry the whole analysis:**

| Statistic | What it means in finance |
|---|---|
| `mean` of monthly returns | **expected return** — what the stock delivers in an average month |
| `std` (standard deviation) | **risk**, also called volatility — how far a typical month strays from that average |

Every plot in this section is a different view of those two numbers. The density plot shows the full shape; the risk-return scatter compresses each stock to a single point.

---

## 2. Companies per sector

```python
sector_counts = (
    nyse["sector"]
    .value_counts()
    .rename_axis("sector")
    .reset_index(name="companies")
)
```

`value_counts()` counts how many times each value appears in a column and returns the result already sorted from most to least — which is exactly what the question asks for, so no explicit sort is needed. It hands back a Series whose *index* is the sector name, so `rename_axis` labels that index and `reset_index` turns it into an ordinary two-column DataFrame that seaborn can plot.

```python
ax = sns.barplot(data=sector_counts, x="companies", y="sector", color="#4C72B0")
```

Sectors go on the **y-axis** deliberately. Names like "Consumer Non-Durables" are long; on a horizontal axis they overlap or have to be rotated 45°, which is harder to read. One colour, not a rainbow — colour should encode information, and here it would only repeat what the axis already says.

`ax.bar_label(...)` prints the number at the end of each bar, so a reader does not have to trace back to the axis to get a value.

**Sanity check:** the counts must sum to 508, the number of rows in `nyse.csv`. The plot title prints that sum, so a mistake would be visible immediately.

---

## 3. Downloading the prices

```python
my_stocks = ["JPM", "XOM", "KO", "JNJ", "NKE", "CCL", "SPY"]
```

Six stocks from `nyse.csv` deliberately spread across six different sectors — finance, energy, consumer staples, healthcare, apparel and travel — plus **SPY**, the S&P 500 ETF, which acts as the benchmark. If all six came from one industry they would move together and the comparison would say nothing.

```python
end_date = pd.Timestamp.today().normalize()
start_date = pd.Timestamp(year=end_date.year - 5, month=1, day=1)
```

Dates are computed from today rather than hard-coded, so re-running the notebook next month still gives a five-year window. `normalize()` strips the time-of-day so the timestamp is a clean midnight.

```python
prices = yf.download(my_stocks, start=..., end=..., auto_adjust=False, progress=False)["Adj Close"]
prices = prices[my_stocks]
```

Passing a **list** downloads every ticker in one request instead of seven. The result has a two-level column index — `(field, ticker)` — and selecting `"Adj Close"` collapses it to a simple table: dates down the rows, one column per stock. yfinance returns tickers alphabetically, so the last line restores your own ordering, which keeps every later table and legend consistent.

---

## 4. Calculating returns

```python
daily_returns   = prices.pct_change()
monthly_returns = prices.resample("ME").last().pct_change()
yearly_returns  = prices.resample("YE").last().pct_change()
```

`pct_change()` computes (today − yesterday) / yesterday for **every column at once**. No loop over tickers is needed; this is the whole point of working with a DataFrame.

`resample("ME")` regroups the daily rows into month-end buckets and `.last()` takes the final trading day of each month. Only then does `pct_change()` run, so you get month-over-month returns rather than an average of daily ones. `"ME"` means month-end and `"YE"` year-end; older pandas versions spelled these `"M"` and `"Y"`.

**The first row is always `NaN`** — there is no earlier period to compare against. That is correct behaviour, not a bug, and it is why `.dropna()` appears later before plotting.

---

## 5. The summary table

```python
monthly_summary = monthly_returns.agg(["min", "max", "median", "mean", "std"]).T.round(4)
```

`.agg([...])` applies all five statistics to all seven stocks in one pass. Without `.T` the statistics are rows and the stocks are columns; transposing puts **one stock per row**, which reads far more naturally and is the shape the risk-return plot needs.

Values are decimals, so 0.0196 means 1.96% per month.

---

## 6. The density plot

```python
monthly_long = monthly_returns.melt(value_name="monthly_return", var_name="stock", ignore_index=False).dropna()
```

Seaborn wants **long (tidy) data**: one row per observation, with a column identifying which group it belongs to. `melt` converts the wide price-matrix shape (a column per stock) into (date, stock, return) triples. `ignore_index=False` keeps the dates on the index. This reshaping step is the single most common source of confusion with seaborn — if a plot refuses to colour by group, the data is almost always still in wide form.

```python
g = sns.displot(data=monthly_long, x="monthly_return", hue="stock", kind="kde", fill=True, alpha=0.3, ...)
```

A KDE is a smoothed histogram: high curve means many months landed near that return.

**Read the width, not the height.** A narrow spike means returns cluster tightly around the average — low risk. A wide, flat curve means anything can happen in a given month — high risk. Width here is a visual rendering of the `std` column in your summary table, so the two should agree; if they ever disagree, one of them is wrong.

---

## 7. The risk-return plot

```python
risk_return = monthly_summary.reset_index(names="stock")
sns.scatterplot(data=risk_return, x="std", y="mean", ...)
```

Nothing new is computed — the plot reads straight from the summary table. Risk on the x-axis, expected return on the y-axis, one dot per stock.

```python
for row in risk_return.itertuples():
    ax.annotate(row.stock, (row.std, row.mean), xytext=(6, 4), textcoords="offset points")
```

`itertuples()` walks the rows and `annotate` places the ticker next to its dot. `xytext=(6, 4)` with `textcoords="offset points"` nudges the label 6 points right and 4 up so it does not sit on top of the marker.

```python
ax.xaxis.set_major_formatter(mtick.PercentFormatter(xmax=1, decimals=1))
```

The numbers are decimals (0.044, not 4.4), so `xmax=1` tells matplotlib that 1.0 represents 100%. Without this the axis reads "0.04" instead of "4.4%".

**How to read it.** The expected pattern runs bottom-left to top-right: more risk, more reward. What matters are the exceptions — points far to the right that stay low. A useful trick is to imagine a line from the origin through **SPY**: anything below that line delivered less return per unit of risk than simply buying the index, which is a concrete argument for diversification rather than a slogan.

If you want it as a number, compute `mean / std` for each stock. That is a simplified Sharpe ratio (the real one subtracts the risk-free rate first) and it ranks the stocks on exactly that criterion.

---

## 8. What our numbers actually showed

Window: January 2021 – August 2026, 67 monthly observations.

| Stock | Mean | SD (risk) | mean/SD |
|---|---|---|---|
| XOM | 2.59% | 8.03% | 0.32 |
| SPY | 1.29% | 4.39% | 0.29 |
| JPM | 1.96% | 6.80% | 0.29 |
| KO | 1.32% | 4.68% | 0.28 |
| JNJ | 1.13% | 5.04% | 0.22 |
| CCL | 1.90% | 17.50% | 0.11 |
| NKE | **-1.21%** | 9.10% | **-0.13** |

Three things fall out of this, and they are what the written answers argue:

1. **SPY is the least volatile of all seven.** A basket of 500 companies is steadier than any one of them. That is diversification, visible in a single number.
2. **NKE is the clean counter-example to "risk pays".** Twice the market's volatility and a *negative* average return.
3. **CCL is the expensive one.** The highest risk in the set by far, for a return indistinguishable from JPM's at less than half the volatility.

And the honest caveat: 67 observations from one five-year window is thin. CCL's +67.7% month is a post-pandemic rebound, not a repeatable property. Risk rankings are far more stable across periods than return rankings — say so in the answer, because it is true and it is the kind of thing that separates a considered answer from a description of the graph.

---

## 9. Before you submit

- Cell 51 asks you to **delete the explanatory text and comments** and hand in a clean stand-alone document. That means Kostis's tutorial comments — keep short comments that explain a non-obvious choice.
- This assignment wants the **notebook itself** uploaded to Canvas, not an HTML export (the pre-programme one was HTML).
- Fill in the *Details* block at the end: collaborators, time spent, what gave you trouble.
- Still unfinished in this notebook: the **movies (IMDB)** section and the **San Francisco rents** section.

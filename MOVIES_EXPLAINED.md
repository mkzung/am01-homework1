# Homework 1 — the movies (IMDB) section, explained

Same format as `STOCKS_EXPLAINED.md`: what each cell does, why it is written that way, and how to check it.
**For you, not for submission.**

---

## 1. The one decision that shapes everything else: duplicates

The question sounds routine — *"Are all entries distinct or are there duplicate entries?"* — and the obvious answer is wrong.

```python
movies.duplicated().sum()      # -> 0
```

`duplicated()` compares **entire rows**. Two rows count as duplicates only if every one of the 11 columns matches. Here nothing does, so the answer looks like "no duplicates" and most people stop.

```python
movies.duplicated(subset=["title", "director", "year"], keep=False).sum()   # -> 107 rows, 53 films
```

Restrict the comparison to the columns that **identify a film** and 54 extra rows appear. Compare two copies of *Alice in Wonderland* and the only difference is `votes`: 306,320 against 306,336. The same IMDB page was scraped twice a few minutes apart.

`keep=False` marks *all* copies including the first, which is what you want when inspecting; `keep="first"` marks only the extras, which is what you want when dropping.

Why it is not pedantry: those 53 films would be counted twice in every genre count, every director total, every average below. So:

```python
movies_clean = movies.drop_duplicates(subset=["title", "director", "year"], keep="first")
# 2961 -> 2907 rows
```

**The general lesson:** `duplicated()` with no arguments answers "are any rows byte-identical", which is almost never the question you actually have. Decide first which columns identify a record, then pass them as `subset`.

---

## 2. Missing values

```python
movies.isna().sum()          # per column
movies.isna().sum().sum()    # grand total -> 0
```

`isna()` returns a table of True/False the same shape as the data; summing it once gives a count per column, twice gives a single number. Zero missing values across 2,961 rows is unusual for scraped data and tells you the file was cleaned before you got it.

---

## 3. Count of movies by genre

```python
genre_counts = movies_clean["genre"].value_counts().reset_index(name="movies")
assert genre_counts["movies"].sum() == len(movies_clean)
```

Identical to the NYSE-by-sector pattern. The `assert` is a cheap habit worth keeping: if the counts stop adding up to the row total, the cell fails loudly instead of quietly producing a wrong table.

What comes out: Comedy 844, Action 719, Drama 484 — about 70% of the sample — and a long tail where five genres have fewer than five films and Thriller has one.

---

## 4. Return on budget: the definition matters more than the code

The brief says: *"a variable return_on_budget which shows how many dollars did a movie make at the box office for each dollar of its budget"*. That sentence has **two valid readings**, and they give wildly different answers.

| Definition | Horror | What it answers |
|---|---|---|
| ratio of averages — `mean(gross) / mean(budget)` | **2.74** | for every dollar the genre spent, how much did the genre take? |
| average of ratios — `mean(gross / budget)` | **86.15** | what does the typical film in this genre return? |

The gap is not a rounding difference, it is a factor of thirty. It comes from a single film: *Paranormal Activity*, budget $15,000, gross $107.9m, ratio **7,195**. Averaging ratios lets that one film outvote the other 127 Horror films combined.

The notebook uses the ratio of averages, and says so in a comment. Either choice is defensible — **failing to notice there was a choice is not**. If Kostis asks one question about this section, it will be this one.

```python
genre_money = (
    movies_clean.groupby("genre")
    .agg(films=("title", "size"),
         mean_gross=("gross", "mean"),
         mean_budget=("budget", "mean"))
)
genre_money["return_on_budget"] = genre_money["mean_gross"] / genre_money["mean_budget"]
```

**Named aggregation** — `films=("title", "size")` — is worth learning: you say which column and which function, and the result column gets the name you chose. Cleaner than `.agg(["mean"])` followed by renaming.

The `films` column is deliberately included so the small-sample problem is visible in the output rather than hidden: Musical tops the ranking at 28.9 on **two films**.

---

## 5. Top 15 directors

```python
top_directors = (
    movies_clean.groupby("director")["gross"]
    .agg(["sum", "mean", "median", "std", "count"])
    .sort_values("sum", ascending=False)
    .head(15)
)
```

`.agg([...])` on a single column after `groupby` puts the statistics in columns and the directors in rows already — no `.T` needed here, unlike the stocks summary where `.agg()` was applied to a whole frame.

Reading it: Spielberg leads on total ($4.01bn) across 23 films, but Lucas averages $348m per film across five. **Ranking by total rewards volume.** The `std` column separates the two patterns — Cameron's $309m standard deviation exceeds his own median, because *Avatar* and *Titanic* dwarf the rest of his output, whereas Spielberg's mean and median nearly coincide.

---

## 6. Ratings by genre, and the `>= 5 films` filter

```python
genre_sizes = movies_clean["genre"].value_counts()
big_genres = genre_sizes[genre_sizes >= 5].index
ratings_plot_data = movies_clean[movies_clean["genre"].isin(big_genres)]
```

Three steps: count films per genre, keep the labels of the ones with at least five (`.index` gives the genre names), then filter with `.isin()`. Twelve of seventeen genres survive.

`isin()` is the "one of these values" filter, the alternative to chaining a dozen `|` comparisons.

The plot uses `sns.displot(..., col="genre", col_wrap=4, facet_kws={"sharey": False})`. `sharey=False` matters: Comedy has 844 films and Sci-Fi has 7, so a shared y-axis would flatten the small genres into invisibility.

Also notice the `NaN` in the `std` column for Thriller — one film, no dispersion to measure. Same lesson as the drill sheet.

---

## 7. The three scatterplots

All three answer the same shape of question: does X predict gross?

| Predictor | Correlation | Verdict |
|---|---|---|
| budget | **0.64** | the only one worth much |
| votes | 0.63 | (not asked, but comparable) |
| rating | 0.27 | weak |
| cast Facebook likes | 0.21 | weakest |

**Which variable goes on which axis.** The brief asks explicitly. Gross is the outcome you would want to predict, so it goes on **Y**; the candidate predictor goes on **X**. Convention, but the question is testing whether you know it.

**Why `regplot` rather than `scatterplot`.** `regplot` draws the points *and* fits a straight line with a 95% confidence band, so the strength and direction of the relationship are visible without computing anything.

**Why a log scale on Facebook likes.** They range from near zero to hundreds of thousands. On a linear axis every point is crushed against the left edge. `plt.xscale("log")` spreads them out. Do not do this silently — label the axis "(log scale)".

**The break-even line on the budget plot.**

```python
lims = [0, movies_clean["budget"].max()]
plt.plot(lims, lims, linestyle="--", color="grey")
```

A 45° line where gross equals budget. Everything below it lost money at the box office — **42% of the films here**. One extra line turns a scatterplot into a business statement.

**Correlation is not causation, and here the arrow may point backwards.** Facebook likes were counted years after most of these films were released, so a star's follower count partly reflects the success of the film. And studios assign big budgets to properties they already expect to sell, so "budget predicts gross" is partly "expected gross justifies budget".

---

## 8. "Is there anything strange in this dataset?"

The question at the end of the faceted plot is the one place the marker is looking for observation rather than technique. Five things, roughly in order of importance:

1. **One genre per film.** IMDB assigns several; this file picked one. Every genre comparison on the page is therefore cruder than it looks.
2. **Grossly unbalanced groups.** 844 Comedies, 1 Thriller. Five genres unusable, and Sci-Fi survives the filter with seven films.
3. **The 54 duplicate records** — differing only in vote counts, a scraping artefact.
4. **Gross is not inflation-adjusted** while the films span **1920 to 2016**. Cross-era comparisons are distorted and older films systematically understated.
5. **Gross looks like US box office only**, so films that earned mostly abroad are undercounted — which feeds straight back into the return-on-budget table.

---

## 9. New pandas moves in this section

Things that appear here but not in the stocks section:

| Method | What it does |
|---|---|
| `.isna().sum()` | count missing values per column |
| `.duplicated(subset=[...], keep=False)` | find duplicates on chosen columns |
| `.drop_duplicates(subset=[...], keep="first")` | remove them, keep one |
| `.agg(name=("col", "func"))` | named aggregation — control the output column names |
| `.isin([...])` | "value is one of these" filter |
| `.nlargest(n, "col")` | top n rows by a column, shorter than sort + head |
| `sns.regplot` / `sns.lmplot` | scatter with a fitted line; `lmplot` adds facets |
| `plt.xscale("log")` | log axis for values spanning orders of magnitude |
| `mtick.FuncFormatter` | custom axis labels, e.g. `$120m` instead of `120000000` |

---

## 10. Still to do

- **San Francisco rents** — three plots and a written answer.
- Fill in the *Details* block: collaborators, time spent, hardest part.
- Cell 51: strip Kostis's tutorial comments and submit a clean stand-alone notebook (the `.ipynb`, not HTML).

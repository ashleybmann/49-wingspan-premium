# Wingspan Premium

A small regression project that asks: **do NBA front offices draft prospects earlier for having long arms?**

It pulls NBA Draft Combine measurements and official draft slots, computes each player's excess length (wingspan minus barefoot height), and fits a simple OLS model to see whether extra length buys an earlier pick.

## Contents

| File | Description |
| --- | --- |
| `fetch_draft_data.py` | Pulls combine anthropometrics and draft history from the NBA Stats API (2015–2024) and writes the CSV |
| `nba_draft_combine_wingspan.csv` | 409 drafted combine participants with height, wingspan, and draft slot |
| `model_wingspan_premium.py` | Fits the OLS model, prints outlier tables, and saves the chart |
| `eda.ipynb` | Exploratory notebook |
| `wingspan_premium_scatter.png` | Chart exported by the model script |

### Data schema

| Column | Description |
| --- | --- |
| `draft_year` | Year of the draft class |
| `overall_pick` | Overall draft slot (1–60) |
| `round_number` / `round_pick` | Round (1 or 2) and pick within the round |
| `player_id` | NBA Stats player ID |
| `player_name` | Player's name (accents stripped) |
| `position` | Position listed at the combine |
| `team_abbreviation` | Drafting team |
| `organization` | College, club, or team the player came from |
| `height_wo_shoes` | Barefoot height (inches) |
| `wingspan` | Wingspan (inches) |
| `wingspan_diff` | `wingspan - height_wo_shoes` (inches) |

## Method

The project runs three steps:

1. **Fetch and merge.** `fetch_draft_data.py` calls the `nba_api` combine and draft-history endpoints for each year, matches combine rows to draft picks on player ID, and falls back to a cleaned name (lowercase, no punctuation or `Jr.`/`III` suffixes) within the same draft year. Rows missing height, wingspan, or a draft slot are dropped. Undrafted combine participants are not included.
2. **Model.** `model_wingspan_premium.py` restricts the data to **2015–2023**, the voluntary combine era. From 2024 on, attendance is mandatory for draft-eligible prospects, which changes who gets measured. It then fits `draft_pick ~ wingspan_diff` with OLS. A negative slope would mean longer players go earlier.
3. **Outliers and visualize.**
   - **Length Reaches:** players in the top quartile of `wingspan_diff` (≥ 6.5 in) who were drafted furthest ahead of the regression line.
   - **Skill/Craft Exceptions:** first-round picks with the shortest relative wingspans.
   - Plots a scatter of pick vs. `wingspan_diff`, colored by residual, with the fit line and both outlier groups labeled.

## Results

![Wingspan premium scatter, 2015–2023](wingspan_premium_scatter.png)

| Statistic | Value |
| --- | --- |
| Sample size | 352 |
| Slope | −0.07 picks per inch (95% CI −0.84 to +0.71) |
| p-value | 0.86 |
| R² | 0.0001 |

**There is no measurable wingspan premium.** Each extra inch of wingspan relative to height moves the predicted slot by less than a tenth of a pick, and the confidence interval comfortably includes zero. Wingspan alone explains essentially none of the variation in draft position.

The outliers show why. The top "Length Reaches" (D'Angelo Russell, Jaylen Brown, Jaren Jackson Jr., Scottie Barnes, Kris Dunn) are long, but they were top-five picks for many reasons beyond length. The "Skill/Craft Exceptions" (Tyler Herro, Frank Kaminsky, Ty Jerome, Luke Kennard, Tre Mann) went in the first round with average-or-worse length, mostly as shooters. See [Caveats](#caveats) before drawing conclusions.

## Getting started

Requirements: Python 3 with `pandas`, `nba_api`, `statsmodels`, `matplotlib`, and Jupyter.

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install pandas nba_api statsmodels matplotlib jupyter ipykernel
python -m ipykernel install --user --name wingspan   # kernel name the notebook expects
```

The CSV is already in the repo. To rebuild it from the API (takes a few minutes, with pauses between requests):

```bash
python fetch_draft_data.py
```

Fit the model, print the tables, and regenerate the chart:

```bash
python model_wingspan_premium.py
```

Both scripts use relative paths, so run them from this directory.

## Tuning the analysis

These constants are at the top of `model_wingspan_premium.py`:

- **`FIRST_YEAR` / `LAST_YEAR`**: change the draft classes in the model. Set `LAST_YEAR = 2024` to include the first mandatory-combine class.
- **`LENGTH_QUANTILE`**: how long a player must be to count as a Length Reach (default: top quartile).
- **`FIRST_ROUND_MAX_PICK`**: the pick cutoff for Skill/Craft Exceptions.
- **`TOP_N` / `ANNOTATE_N`**: how many players are printed in each table and labeled on the chart.

In `fetch_draft_data.py`, **`YEARS`** sets which draft classes are pulled.

## Caveats

- **Selection bias.** Only drafted players who attended the combine are measured. Many top prospects skipped the combine in the voluntary era, and undrafted players are excluded, so the sample is truncated on the outcome.
- **One predictor.** The model ignores everything scouts actually weigh: skill, production, age, athleticism, and position. A null result here means length doesn't predict draft slot *on its own*, not that teams ignore it.
- **Position confound.** Expected wingspan differs by position, and teams may value length relative to positional peers. A model with position controls, or `wingspan_diff` standardized within position, would test that.
- **Draft pick isn't linear.** The gap between picks 1 and 5 matters far more than between 45 and 50. A log or rank-based outcome may fit the question better.

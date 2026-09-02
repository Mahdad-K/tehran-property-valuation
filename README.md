# Tehran Property Valuation

An automated valuation model (AVM) for residential property in Tehran, built from public listings.

Three notebooks: a scraper that builds the dataset, an XGBoost model, and a hyperparameter study.

## Result

MAE 10.77 (sd 0.53) against a mean of 77.29 million toman per m2, about 14%. XGBoost with default
parameters, `RepeatedKFold` 10 splits x 3 repeats, 4,833 listings, 120 features.

The error is skewed, so the same set of predictions looks very different depending on which metric
you pick.

| Metric | Value |
|---|---|
| Median absolute percentage error | 10.3% |
| Mean absolute percentage error | 21.5% |
| R2 | 0.763 |
| Priced within 10% of actual | 48.6% |
| Priced within 20% of actual | 77.7% |

Median error is 10.3% and mean percentage error is more than double that. Cheap listings are the
reason. Missing by 5 million toman on something priced at 30 is a small absolute mistake and a huge
percentage one, and there are enough of those to drag the average up.

Predicting the median for every listing gives MAE 27.52, so this beats a naive guess by about 60%.

## Dataset

Source: [2nabsh.com](https://2nabsh.com), an Iranian listings portal, crawled street by street
across Tehran. 6,639 listings collected, 4,833 used after filtering. 53 raw fields per listing,
120 features after encoding.

The crawl is resumable. Streets are split into batches, each written to its own file, and batches
already on disk get skipped on a re-run. Street pages lazy-load, so it scrolls until the loading
element disappears instead of assuming a page size.

Iranian prices move fast. Training on the older listings and testing on the newest 20% gives MAE
17.24 against 11.09 on a random split. The training portion averages 73.2 million toman per m2 and
the test portion 93.8, a 28% shift across roughly 150 days. A model anchors to the price level it
learned, so this one is built to be re-run monthly, and prices are stored per square metre so runs
from different months stay comparable.

Fields captured: price per m2 (the target), floor area, land area, total floors, floor, units per
floor, bedrooms, building age, listing age, then unit type, deed type, orientation, facade,
flooring, cabinets and neighbourhood as categoricals, plus 33 amenity flags.

The amenity flags are where the interesting signal sits. Some are standard, like lift, parking,
storage and terrace. The rest are specification items: pool, sauna, jacuzzi, roof garden, gym,
lobby, function room, master bathroom, air handling unit, chiller. On the same street and at the
same size, that second group is most of what separates a high-end apartment from an ordinary one,
and it lives in the amenity list rather than in the structured fields, which makes it easy to miss
when you are deciding what to scrape.

Cleaning needs a bit of domain knowledge. Several fields arrive as free text. `نوساز` (newly built)
maps to building age 0, `همکف` (ground floor) to floor 0, `تک واحد` (single unit) to 1 unit per
floor. Relative dates convert to days. Unit suffixes get stripped off numbers, neighbourhood is
pulled out of the free-text address, and duplicates are dropped on listing URL. Nulls are relabelled
per column before one-hot encoding, otherwise a missing facade and a missing deed type collapse into
the same category.

## Tuning

Three search methods. None of them beat the defaults.

| Configuration | Method | MAE (sd) |
|---|---|---|
| Defaults | none | 10.77 (0.53) |
| Optuna best | 100-trial TPE study | 11.34 (0.43) |
| RandomizedSearchCV best | 25 iterations | 14.09 (0.64) |
| GridSearchCV best | 2,592 combinations | 15.03 (0.60) |

The CV configurations differ between runs, so this is not a controlled benchmark. The gaps are wide
enough that the conclusion holds anyway. Going from five obvious features to the full 120 cut error
by 40%. Searching the parameter space cut it by nothing. The feature set is the ceiling, not the
estimator.

## Limitations

These are asking prices. The portal publishes what sellers list at, and in a market with wide
negotiation margins that is not what a property transacts for.

The listings also carry free-text descriptions, which this project ignores. Quality runs from
detailed write-ups down to near-empty boilerplate, and sellers often describe renovation, view and
finish quality there while leaving the structured fields blank. Pulling that out needs NLP work that
was out of scope, but it is the obvious next thing to try.

Tails are not trimmed. 9 listings over 1,000 m2, 7 priced under 10 and 12 over 250, roughly 1% of
the data. Most of those are commercial units or data entry errors rather than homes, and cutting
them would probably tighten the error.

## Running

```bash
pip install -r requirements.txt
```

The version bounds are deliberate. `early_stopping_rounds` moved out of `.fit()` in xgboost 2.0 and
`mean_squared_error(squared=False)` was removed in scikit-learn 1.6, so newer releases break two
cells.

Notebook 01 needs Chrome and chromedriver, plus `street_links_raw.json`, a list of street names and
their portal URLs that is not included here. The portal's markup has changed since the crawl was
run, so the selectors would need updating to collect anything fresh. Notebooks 02 and 03 need
`data/dataset.xlsx`. That is not redistributed, since the listing data belongs to the portal, and
`data/` is gitignored.

```
notebooks/
  01_scrape_2nabsh.ipynb          build the dataset
  02_price_model.ipynb            train and cross-validate
  03_hyperparameter_tuning.ipynb  Optuna, grid and random search
```

Python, Selenium, BeautifulSoup, pandas, NumPy, scikit-learn, XGBoost, Optuna.

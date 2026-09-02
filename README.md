# Tehran Property Valuation

An automated valuation model (AVM) for residential property in Tehran, built from public listings.

Three notebooks: a scraper that builds the dataset, an XGBoost model, and a hyperparameter study.

## Result

MAE 10.77 (sd 0.53) against a mean of 77.29 million toman per m2, about 14%. XGBoost with default
parameters, `RepeatedKFold` 10 splits x 3 repeats, 4,833 listings, 120 features.

The error distribution is skewed, so one number is not enough:

| Metric | Value |
|---|---|
| Median absolute percentage error | 10.3% |
| Mean absolute percentage error | 21.5% |
| R2 | 0.763 |
| Priced within 10% of actual | 48.6% |
| Priced within 20% of actual | 77.7% |

The cheapest listings produce very large percentage errors from small absolute misses, which is
what pulls the mean up.

Predicting the median for every listing gives MAE 27.52, so this is roughly 60% better than a
naive guess.

## Dataset

Source: [2nabsh.com](https://2nabsh.com), an Iranian listings portal, crawled street by street
across Tehran. 6,639 listings collected, 4,833 used after filtering. 53 raw fields per listing,
120 features after encoding.

The crawl is resumable: streets are split into batches, each written to its own file, and batches
already on disk are skipped. Street pages lazy-load, so it scrolls until the loading element
disappears rather than assuming a page size.

That matters because Iranian prices move fast under high inflation. Training on the older listings
and testing on the newest 20% gives MAE 17.24 against 11.09 on a random split: the training
portion averages 73.2 million toman per m2 and the test portion 93.8, a 28% shift across about 150
days. A model anchors to the price level it was trained on, so the crawl is built to be re-run
monthly and prices are stored per square metre to keep runs comparable.

**Fields:** price per m2 (target), floor area, land area, floors, floor, units per floor,
bedrooms, building age, listing age, plus unit type, deed type, orientation, facade, flooring,
cabinets and neighbourhood as categoricals, plus 33 amenity flags.

The amenity flags cover standard items (lift, parking, storage, terrace) and specification items
(pool, sauna, jacuzzi, roof garden, gym, lobby, master bathroom, air handling unit, chiller). The
second group carries most of the difference between a high-end property and an average one on the
same street, and it sits in the amenity list rather than in the structured fields.

**Cleaning.** Several fields arrive as free text: `نوساز` (newly built) maps to age 0, `همکف`
(ground floor) to floor 0, `تک واحد` (single unit) to 1 unit per floor. Relative dates convert to
days. Unit suffixes are stripped from numbers, neighbourhood is extracted from the address, and
duplicates are dropped on URL. Nulls are relabelled per column before one-hot encoding, so a
missing facade and a missing deed type do not collapse into one category.

## Tuning

Three search methods, none of which improved on the defaults:

| Configuration | Method | MAE (sd) |
|---|---|---|
| Defaults | none | 10.77 (0.53) |
| Optuna best | 100-trial TPE study | 11.34 (0.43) |
| RandomizedSearchCV best | 25 iterations | 14.09 (0.64) |
| GridSearchCV best | 2,592 combinations | 15.03 (0.60) |

CV configurations differ between runs, so this is not a controlled benchmark, but the gaps are
wide enough to be clear. Going from five obvious features to the full 120 cut error by 40%, while
searching the parameter space cut it by nothing. The feature set is the ceiling, not the estimator.

## Limitations

**Asking prices, not sale prices.** The portal publishes what sellers list at. In a market with
wide negotiation margins that is not what a property transacts for.

**Descriptions are unused.** Listings carry free-text descriptions ranging from detailed to
near-empty. Sellers often describe renovation, view and finish quality there while leaving the
structured fields blank, but extracting it needs NLP work that was out of scope.

**Tails are not trimmed.** About 1% of rows are commercial units or data entry errors: 9 listings
over 1,000 m2, 7 priced under 10 and 12 over 250. Cutting them would probably tighten the error.

## Running

```bash
pip install -r requirements.txt
```

The version bounds matter: `early_stopping_rounds` moved out of `.fit()` in xgboost 2.0, and
`mean_squared_error(squared=False)` was removed in scikit-learn 1.6.

Notebook 01 needs Chrome and chromedriver plus `street_links_raw.json`, a list of street names and
portal URLs that is not included. The portal's markup has changed since the crawl, so the
selectors would need updating to collect fresh data. Notebooks 02 and 03 need `data/dataset.xlsx`,
which is not redistributed here, since the listing data belongs to the portal.

```
notebooks/
  01_scrape_2nabsh.ipynb          build the dataset
  02_price_model.ipynb            train and cross-validate
  03_hyperparameter_tuning.ipynb  Optuna, grid and random search
```

Python, Selenium, BeautifulSoup, pandas, NumPy, scikit-learn, XGBoost, Optuna.

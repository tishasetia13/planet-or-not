# Planet or Not

A classifier that looks at NASA Kepler "objects of interest" and predicts whether each one is a **confirmed exoplanet** or a **false positive** (something that looked like a planet but was not).

**Scope, stated honestly:** this version uses only the summary numbers in NASA's KOI table. It does **not** use the light curves (the brightness-over-time data). A light-curve neural network was planned but is not part of this version.

## Data
- NASA Exoplanet Archive, Kepler Objects of Interest (cumulative table): 9,564 rows.
- Trained on rows labelled CONFIRMED or FALSE POSITIVE only (7,587 rows). CANDIDATE rows have no final answer, so they were set aside.

## Avoiding leakage (the main care taken)
- Removed columns that record how NASA decided the label: the disposition score, the Kepler pipeline's own verdict, the four false-positive test flags, vetting comments and dates, and the Kepler name (only given to confirmed planets). On the labelled rows, the pipeline verdict matched the label in 99.9% of cases, so keeping it would have made the task meaningless.
- Also set aside: empty columns, uncertainty columns, pipeline bookkeeping columns, and image-based centroid columns.
- Final model uses 33 numeric inputs (orbit period, transit depth and duration, signal-to-noise, estimated object size, star properties and so on).

## Split
- Split **by star** (`kepid`), 70 / 15 / 15, fixed random seed. A star with several objects never appears in more than one pile.
- Rows: train 5,305, validation 1,132, test 1,150.
- The test pile was used once, after the model was chosen on validation.

## Results
Validation pile (used to choose the model):

| model | accuracy | precision | recall | f1 | auc |
|---|---|---|---|---|---|
| Always says false positive | 0.637 | 0.0 | 0.0 | 0.0 | 0.5 |
| Logistic regression | 0.851 | 0.747 | 0.891 | 0.812 | 0.93 |
| Gradient boosting | 0.939 | 0.919 | 0.912 | 0.916 | 0.985 |

Test pile, final model (**Gradient boosting**, retrained on train + validation):

| accuracy | precision | recall | F1 | AUC |
|---|---|---|---|---|
| 0.936 | 0.919 | 0.911 | 0.915 | 0.981 |

- A model that always answers "false positive" gets **0.621** accuracy on the test pile, so that is the floor to beat.
- Confusion matrix on test: 397 planets caught, 39 planets missed, 35 false positives wrongly called planets, 679 false positives correctly rejected. See `confusion_matrix.png`.

Inputs the chosen model relied on most (permutation importance on the validation pile: how much AUC drops when an input is scrambled):
- `koi_max_mult_ev`: 0.0293
- `koi_prad`: 0.0292
- `koi_dor`: 0.0215
- `koi_max_sngle_ev`: 0.0135
- `koi_ror`: 0.0106
- `koi_model_snr`: 0.0082
- `koi_smet`: 0.0053
- `ra`: 0.0035

## Limitations
- In the training-pile histograms, many false positives sit at very large sizes or very deep dips, where almost no planets are, so they are easy to spot. Harder look-alikes, which resemble small planets, are probably where most mistakes happen. This was not analysed separately.
- Only one train / validation / test split was used, so the scores have no error bars.
- No light curves were used, so the model cannot look at the actual shape of the dip.
- Not deployed: there is no web app yet. The saved model file can be loaded for that.
- Labels come from NASA's table, which includes decisions made by automated and human vetting, not independent ground truth.

## What I would do next
- Add light-curve features. A pre-folded Kepler dataset from the AstroNet project (Shallue & Vanderburg, 2018) is available and would avoid downloading raw files.
- Separate error analysis for the hard look-alikes.
- Repeat the split with several seeds to get error bars.

## Files
- `planet_model.joblib`: trained model and its list of input columns
- `results.json`: all scores
- `confusion_matrix.png`: test results picture

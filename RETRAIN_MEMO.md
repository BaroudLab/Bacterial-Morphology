# Bacterial-Morphology: mislabelled validation header and retraining

**Date:** 2026-11-05
**Repository:** [BaroudLab/Bacterial-Morphology, PR #1](https://github.com/BaroudLab/Bacterial-Morphology/pull/1) (commit `ab47938`, open)

## Summary

An external reviewer reported two reproducibility issues in the classifier training.

1. **Wrong column names in a saved table.** `ValidationPredictions_All.csv` had its ground-truth columns named in the wrong order. Only the names were wrong. The model was trained and scored against the correct labels, so the published scores stand.
2. **Validation on a random subset.** Each validation pass scored a random 384 of the 414 held-out cases, with no seed.

We fixed both in PR #1 and retrained on the complete held-out set. The new F1 scores are close to the published ones, with the largest drop being 0.028 (rods).

| Class | Published F1 (384 cases) | Retrained F1 (414 cases) | Change |
|---|---:|---:|---:|
| Positive | 0.968 | 0.962 | −0.006 |
| Planktonic | 0.962 | 0.966 | +0.004 |
| Clumped | 0.834 | 0.848 | +0.014 |
| Rods | 0.851 | 0.823 | −0.028 |
| Filaments | 0.837 | 0.820 | −0.017 |

## Issue 1: the header was wrong, the training was not

### What went wrong

The training labels come from `tables/LabelingSetAll.csv`, whose columns are in this order:

`positive, planktonic, clumped, rods, filaments`

The dataset reads them by position (`info_table.iloc[idx, 1:]`), so model output `i` is always trained against column `i` of that file.

After training, cell 24 of `Training.ipynb` wrote the results to a table. It named the label columns by hand, in a different order:

`positive, rods, planktonic, filaments, clumped`

So `y_true[:,1]` holds planktonic labels but was saved under the name `rods`, and so on for the other three label columns. Only `positive` was named correctly.

### Why the training and scores were not affected

The names were applied only when writing this output table. The training loop, the loss and the scoring in `Scores_All.csv` all work on arrays by position and never use these names.

### Evidence

We checked the original table (base commit `117c4d4`) against `LabelingSetAll.csv`, using the saved case IDs.

| Saved column name | Actually contains | Match by position | Match by name |
|---|---|---:|---:|
| `positive` | positive | 100% | 100% |
| `rods` | planktonic | 100% | 45% |
| `planktonic` | clumped | 100% | 36% |
| `filaments` | rods | 100% | 71% |
| `clumped` | filaments | 100% | 57% |

Recalculating F1 from the saved logits gives a clear result either way:

| How the labels are read | Positive | Planktonic | Clumped | Rods | Filaments |
|---|---:|---:|---:|---:|---:|
| **By position** | 0.968 | 0.962 | 0.834 | 0.851 | 0.837 |
| `Scores_All.csv` as published | 0.968 | 0.962 | 0.834 | 0.851 | 0.837 |
| By the wrong header | 0.968 | 0.450 | 0.135 | 0.603 | 0.565 |

Reading by position reproduces the published scores to three decimals. Reading by the header gives the poor scores a reader would get by trusting the column names in the original file.

### Where the wrong names did and did not reach

| Component | Result |
|---|---|
| `Training.ipynb`, training and scoring | Not affected: works by position |
| `Scores_All.csv` | Not affected: reproduced above |
| `ApplyTraining.ipynb` | Not affected: assigns classes to model outputs in the correct order |
| `ExtractClassProbs.ipynb` | Not affected: uses another order, but looks columns up by name in `ApplyTraining` output |
| `Figure5+SI2`, `Figure6a` | Not affected, based on their code: they select class columns by name in the correct order |
| Deployed inference (`awd.run`, HF Space) | Not affected: class order is correct |
| `ValidationPredictions_All.csv` | **Affected:** wrong header on the label columns |
| `Figure4.ipynb` | **Partly affected:** see below |

`Figure4.ipynb` paired `pred_logit_i` with label column `i`, so each panel's data was right, but the column names suggested the wrong class for panels 2 to 5. The real order is positive, planktonic, clumped, rods, filaments.

**Not yet checked:** the published Figure 4 and its caption. The notebook code doesn't set class titles on those panels, so any class names were added elsewhere, and they should be compared with the real order above.

### The fix

PR #1 corrects the names in `Training.ipynb` cell 24 and `Figure4.ipynb`, and changes only the header row of `ValidationPredictions_All.csv`. We confirmed that the data rows of that file are identical before and after.

## Issue 2: validation used a random 384 of 414 cases

The 1,242 labelled cases split into 828 for training and 414 for validation. Training and validation shared one loader with `shuffle=True, batch_size=32, drop_last=True`, which keeps 12 × 32 = 384 validation cases, a different random set on each pass. With no seed, the split couldn't be reproduced either.

One visible sign is that `LabelSplitInfoAll.csv` doesn't match the cases behind the scores. It lists 100 clumped and 154 rods cases, while the final saved predictions contain 105 and 150.

PR #1 adds a separate validation loader (`shuffle=False, drop_last=False`) and seeds NumPy and PyTorch before the split. Training keeps the original shuffled loader.

## Retraining

### Setup

| | |
|---|---|
| **Code** | `Training.ipynb` at PR commit `ab47938`, 100 epochs, seed 42 |
| **Hardware** | Hugging Face Jobs, `t4-small` (Nvidia T4) |
| **Software** | PyTorch 2.4.1, torchvision 0.19.1 |
| **Training time** | 832 s for the notebook, excluding startup |
| **Job** | [`6aec55b40e0338d1bb8068f4`](https://huggingface.co/jobs/aaristov/6aec55b40e0338d1bb8068f4) |

### Rebuilding the crops

`Crops/` isn't in the repository, so we rebuilt all 1,242 from the `day2` `.crops.zarr` stacks on `awd.run`, following `CreateTrainingSetFromDatabase.ipynb`: the TRITC channel (index 1), converted to 8-bit over the 400–600 range, with file names and labels taken from `LabelingSetAll.csv`. Together they take up about 112 MB.

To check them, we ran the deployed model on the rebuilt crops:

- On the 384 published validation cases, F1 was 0.962 / 0.959 / 0.810 / 0.853 / 0.860, close to the published scores.
- Correlation with the saved logits was 0.95, against 0.17 for shuffled crops.
- Of the conversion settings tried, the notebook's channel and range matched best.

The logits aren't identical (median difference about 0.7). This fits the deployed weights coming from a different training run than the saved logits, but we can't prove it without the original crops and weights. Strictly, the crops are checked as equivalent, not byte-identical.

### Changes needed to run on a GPU

The committed notebook can't run on a GPU: it never moves the model to the device, and it converts GPU tensors directly to NumPy. The job applied these changes at run time, and they are not yet in PR #1:

- Added `model = model.to(device)` in cell 16.
- Added `.cpu()` before NumPy and scikit-learn conversions in cells 17 and 23.

They don't change the method. The exact code that ran is saved as `executed_cells.py`.

### Results

All 414 validation cases were scored once each.

| Class | Cases | Precision | Recall | F1 |
|---|---:|---:|---:|---:|
| Positive | 265 | 0.973 | 0.951 | 0.962 |
| Planktonic | 361 | 0.962 | 0.970 | 0.966 |
| Clumped | 117 | 0.888 | 0.812 | 0.848 |
| Rods | 148 | 0.866 | 0.784 | 0.823 |
| Filaments | 95 | 0.852 | 0.789 | 0.820 |

Recalculating F1 from the saved predictions gives the same values, and the case counts match `LabelSplitInfoAll.csv`, which the original tables did not.

### How to read the comparison

- **Close, but not like-for-like.** The original split wasn't seeded, so the two runs use different train/validation splits and scored different cases.
- **One seed only.** The largest drop (rods, −0.028) is about the size of sampling noise for 148 cases (roughly ±0.03, a rough estimate). This run can't show whether rods and filaments are really slightly lower.
- **Overfitting appears in both runs.** Validation loss was lowest at 0.202 at epoch 27 (counting from 0) and ended at 0.223, with training accuracy at 0.998. The method saves the last epoch, not the best one. Choosing an earlier epoch would need a separate held-out test set.
- **Low exact-match accuracy is expected.** The final 0.693 (originally 0.659) counts a case only if all five labels are right, so it isn't comparable to per-class F1.

## Current status and open items

| Item | Status |
|---|---|
| Header and validation fixes | In PR #1, open |
| Retrained tables and GPU changes | **Not in PR #1.** The PR description still says training wasn't re-run |
| Deployed model | Unchanged. `awd.run`, the Space and the public model repo still use the old weights |

1. Add the four retrained tables and the GPU changes to PR #1, update its description, and regenerate the figures that use those tables.
2. Compare the published Figure 4 panel labels with the real class order, and tell the reviewer the outcome.
3. Keep the deployed model for now, as the retrained one isn't clearly better.
4. Optionally, repeat with several seeds to measure run-to-run variation.

## Where the files are

Private Hugging Face dataset: [`aaristov/bacterial-morphology-retrain`](https://huggingface.co/datasets/aaristov/bacterial-morphology-retrain)

| Path | Contents |
|---|---|
| `crops.tar` | The 1,242 rebuilt crops |
| `train_job.py` | The job script, including the GPU changes |
| `runs/full-100ep-seed42/` | Full run: weights, the four result tables, `executed_cells.py`, `run_meta.json` |
| `runs/smoke/` | 1-epoch test run; its scores are not meaningful |

Retrained weights SHA-256: `d6071c1ffb7e25992c8d39ff777146a58c5e4743e8b2ffbbb98140d340e6bd00`

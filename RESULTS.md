# RESULTS.md — Washing-Machine EDA and Disaggregation

## 1. Scope and current status

The analysis is designed around the preferred REFIT household cohort (Houses 1, 2, 5, 6, 7 and 8, subject to the notebook's README-based eligibility checks). The main validation design is Leave-One-Household-Out (LOHO): one complete household is held out for evaluation and the other selected households are used for training.

**Status at preparation time:** the current Colab run had completed the Seq2Point training stage for the House 1 fold, but the complete six-fold LOHO evaluation and final report-generation stage had not finished. The displayed Seq2Point training losses were finite, and the run selected epoch 14 as its best checkpoint (validation loss 0.01623). This is a training diagnostic only; it is not a held-out-house performance metric. No six-household macro score, final per-house model comparison, or overall model ranking is claimed in this document.

Update this report after the final LOHO run and use the saved metrics CSVs. Do not use partial console output as a substitute for the completed result tables.

## 2. Washing-machine EDA and event definition

The cleaned REFIT measurements are converted to one-minute mean power. The primary event rule identifies candidate ON minutes above 20 W, merges fully observed OFF gaps shorter than 13 minutes, and retains merged event envelopes of at least 27 minutes. This provides a consistent operational definition for comparing households. It is based on a published REFIT-specific parameter set and should not be interpreted as a universal physical definition of every wash cycle.

Events longer than 240 minutes are separately flagged as suspect-fill candidates under a project data-quality heuristic. Suspect intervals are kept visible in the event audit but excluded from supervised target labels and scoring under the notebook's policy. Q10 is a secondary sensitivity check.

The notebook is intended to produce household summaries, representative 24-hour traces, detected-cycle plots, cross-household comparisons and hour/day usage summaries. Insert the executed household tables and plots here after the full EDA/reporting stage. Do not infer cycle totals from the partial log lines.

## 3. Modelling and validation design

The model inputs are derived from whole-house aggregate demand only. The washing-machine sub-meter is used as the target and for final evaluation, not as a feature.

The planned comparison includes:

- **Always-off baseline:** predicts 0 W for every minute.
- **Hour-of-week profile:** a training-house-derived average profile, fitted without the held-out household's appliance labels.
- **HistGradientBoostingRegressor:** tabular model using aggregate current, lags, rolling statistics and calendar features.
- **Seq2Point CNN:** optional temporal model using a centred 121-minute aggregate window to predict the centre minute. Because it uses future context within the centred window, it is retrospective/offline rather than strictly causal.

The target household is held out completely during fitting and model selection. Seq2Point's validation tail is drawn only from training households. Metrics should be calculated on aligned valid timestamps, and coverage should be reported so missing inputs/targets are not hidden.

## 4. Metrics to report

Populate the tables below from `outputs/metrics/loho_results.csv` and `outputs/metrics/loho_macro.csv` after the completed evaluation.

### Macro-average comparison

| Model | MAE (W) | RMSE (W) | R² | Event-state F1 | Event-envelope MAE (W) | Absolute energy error (kWh) |
|---|---:|---:|---:|---:|---:|---:|
| Always-off | Pending final run | Pending | Pending | Pending | Pending | Pending |
| Hour-of-week profile | Pending final run | Pending | Pending | Pending | Pending | Pending |
| HistGradientBoosting | Pending final run | Pending | Pending | Pending | Pending | Pending |
| Seq2Point (if retained in final benchmark) | Pending final run | Pending | Pending | Pending | Pending | Pending |

### Per-household results

| Held-out household | Model | Valid scored minutes | Coverage (%) | MAE (W) | RMSE (W) | Event-state F1 | Energy error (kWh) |
|---|---|---:|---:|---:|---:|---:|---:|
| House 1 | To populate | — | — | — | — | — | — |
| House 2 | To populate | — | — | — | — | — | — |
| House 5 | To populate | — | — | — | — | — | — |
| House 6 | To populate | — | — | — | — | — | — |
| House 7 | To populate | — | — | — | — | — | — |
| House 8 | To populate | — | — | — | — | — | — |

Do not leave placeholder values in the final report: replace them from the completed CSV outputs or explicitly state that a model/fold was not run.

## 5. Metric interpretation

- MAE and RMSE measure power error in watts. R² is undefined when the scored target variance is zero.
- Event-state precision, recall and F1 measure ON/OFF agreement after applying the shared event rule to actual and predicted power.
- Event-envelope MAE measures power error within actual event envelopes, including short measured pauses merged into an event.
- Energy is calculated from aligned valid one-minute readings as `sum(power_W) / 60,000` kWh. Relative energy error is undefined when actual energy is zero.
- The always-off baseline may achieve deceptively low overall MAE because the target is zero-dominated. It is not evidence of successful appliance disaggregation by itself.

## 6. Failure analysis and visual evidence

After the final run, insert:

1. The held-out actual-versus-predicted day selected by the notebook's deterministic selection rule.
2. At least one missed actual event and one false activation, with their held-out timestamps and household IDs.
3. Per-household event-state F1 and power-error comparison.

Failure cases must be drawn from the saved held-out prediction files. Do not describe a hypothetical failure as an observed result.

## 7. Limitations

LOHO across a small cohort provides a more meaningful unseen-household test than a random row split, but it does not establish generalisation to every UK home or to India. Appliance signatures overlap, sub-meter and aggregate channels may be asynchronous, and data gaps or provider-filled intervals can affect labels. Results from the final completed benchmark should be interpreted as an offline research prototype, not a validated production disaggregator.

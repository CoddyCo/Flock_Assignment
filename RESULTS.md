# RESULTS — Washing-Machine EDA and Disaggregation

## Executive summary

The final notebook completed all six Leave-One-Household-Out folds for Houses 1, 2, 5, 6, 7 and 8. Each held-out household has four result rows (always-off, hour-of-week profile, HistGBR and Seq2Point), giving 24 model/household results. The notebook's final checks report four methods per household and confirm the frozen LOHO protocol checks passed.

**HistGradientBoosting (HistGBR) is the strongest overall compromise** on the common evaluation support. It has the best macro event-state F1, recall, MAE, RMSE, R² and relative energy error among the two learned models. Seq2Point has a slightly lower event-envelope MAE, but its event F1 and energy error are worse on average. The always-off baseline has the lowest overall MAE, which is expected with a zero-dominated target and is not evidence of useful disaggregation.

## EDA: event definition and usage

The same primary event rule is applied to all six homes:

1. Candidate ON minute: washing-machine power strictly greater than 20 W.
2. Merge consecutive ON runs separated by fewer than 13 fully observed OFF minutes.
3. Keep the merged event only if its elapsed envelope is at least 27 minutes.
4. Do not merge across missing-data intervals; flag incomplete events.
5. Flag event envelopes longer than 240 minutes as `suspect_fill` under a separate project QC rule.

The 20 W / 13-minute OFF / 27-minute ON settings are the REFIT-specific values reported in Table 2 of Manca and Massidda (2022). The 240-minute suspect threshold is a project quality-control decision, not part of that published setting.

Across the selected cohort, the notebook reports **2,450 complete non-suspect cycles**. Usage varies substantially across homes:

| House | Complete cycles | Valid weeks observed | Cycles per valid week | Median duration | Median energy | Median peak power |
|---:|---:|---:|---:|---:|---:|---:|
| 1 | 332 | 76.8 | 4.32 | 33 min | 0.322 kWh | 2,327 W |
| 2 | 287 | 64.0 | 4.48 | 98 min | 0.634 kWh | 2,093 W |
| 5 | 606 | 82.0 | 7.39 | 46 min | 0.289 kWh | 2,070 W |
| 6 | 95 | 67.0 | 1.42 | 131 min | 0.719 kWh | 2,052 W |
| 7 | 795 | 74.6 | 10.66 | 60 min | 0.542 kWh | 2,083 W |
| 8 | 335 | 68.3 | 4.91 | 61 min | 0.799 kWh | 2,337 W |

House 7 has the largest observed number of complete cycles and the highest cycle rate in this table. House 6 has the lowest rate and a long median event envelope. House 8 has the highest median energy per complete cycle. These are descriptive results for the selected observation periods; they are not population estimates for UK households.

The notebook includes a deterministic representative 24-hour aggregate/target plot and two example detected-cycle plots per selected house, plus cross-house comparisons, hour-of-day usage and day-of-week usage. The House 1 representative day (19 December 2013) shows two distinct washing-machine power bursts, while the whole-home aggregate contains several other large peaks—including one above 10 kW—that are not washer activity. This illustrates why aggregate peaks cannot be attributed to the washing machine by magnitude alone. The House 2 representative day (20 November 2014) similarly shows washer cycles alongside unrelated aggregate-load events.

The timing plots are household-specific rather than showing one universal schedule: House 6 has a pronounced concentration around 10:00–11:00 local time, while House 8 has a larger early-morning share. In the weekday/weekend comparison, House 5 is distinctive: 57.8% of its complete-cycle starts are on weekends, compared with 21.7%–36.8% in the other five homes. These are descriptive proportions of observed cycle starts, not evidence of a causal schedule preference.

## Model design and validation

### Inputs and models

All learned predictors use aggregate power only.

- **HistGBR features:** current aggregate power; lags of 1, 5, 10, 30 and 60 minutes; one- and five-minute aggregate deltas; rolling mean, standard deviation and maximum over 5, 15, 30 and 60 minutes; and Europe/London hour/day calendar encodings.
- **HistGBR training cap:** 200,000 labelled rows per training house (up to 1,000,000 rows across the five training houses in a fold).
- **Seq2Point:** a centred 121-minute aggregate window predicts the centre-minute target. The compact CNN has 1-D convolution blocks of 32, 64 and 64 channels, two max-pooling layers, adaptive average pooling to 10 positions and a 128/64-unit dense head. It uses weighted Huber loss, a training-only scaler, a time-separated validation tail from the training households, up to 8 epochs and patience 3. A fold uses up to 125,000 training windows and 25,000 validation windows across the five training houses.
- **Baselines:** always-off predicts zero; hour-of-week predicts training-house mean power for the matching weekday/hour.

HistGBR's features use current and past aggregate readings. Seq2Point uses a centred window, so it uses future aggregate context relative to the centre minute. Seq2Point is therefore an offline/retrospective estimate, not a causal real-time prediction.

### LOHO protocol and scoring support

Each fold holds out one entire household and trains on the other five. For Seq2Point, the training-house time tails are reserved for early stopping; held-out appliance labels are never used for training or epoch selection. Predictions are scored on identical timestamps across all four methods: the target must be measured and valid, and the complete 121-minute aggregate window required by Seq2Point must exist.

Mean common support is **71.14% of the full one-minute household grid** and **89.20% of valid target minutes**. Per-house coverage against valid target minutes ranges from 83.99% (House 1) to 92.86% (House 7). These coverage figures matter: the model comparison does not include minutes for which required aggregate context is unavailable or target labels are excluded.

### Macro LOHO results

Values below are household macro-averages on the common support. MAE/RMSE are watts; event-envelope MAE is calculated inside the merged actual event envelopes; energy relative error is the absolute energy error as a percentage of measured energy.

| Method | MAE (W) | RMSE (W) | R² | Event-state precision | Event-state recall | Event-state F1 | Per-minute F1 | Event-envelope MAE (W) | Relative energy error |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| Always-off | 16.03 | 152.82 | -0.010 | 0.000 | 0.000 | 0.000 | 0.000 | 446.25 | 100.00% |
| Hour-of-week profile | 30.47 | 153.10 | -0.022 | 0.052 | 0.348 | 0.085 | 0.082 | 430.51 | 103.79% |
| HistGBR | 23.36 | 136.35 | 0.178 | 0.244 | 0.770 | **0.356** | **0.342** | 355.54 | **31.46%** |
| Seq2Point | 25.85 | 144.00 | 0.069 | 0.203 | 0.646 | 0.294 | 0.285 | **339.91** | 44.91% |

The profile and always-off event-envelope errors are included for completeness; neither is an event-disaggregation model. The principal comparison is between HistGBR and Seq2Point.

HistGBR's macro fit time was 4.69 seconds and prediction time 0.55 seconds per fold on the recorded local run. Seq2Point's corresponding averages were 30.64 seconds and 6.76 seconds. These are observed times in the notebook execution environment, not hardware-independent performance guarantees.

### Per-household event-state F1

| House | HistGBR event F1 | Seq2Point event F1 | HistGBR absolute energy error (kWh) | Seq2Point absolute energy error (kWh) |
|---:|---:|---:|---:|---:|
| 1 | 0.186 | 0.135 | 50.36 | 26.86 |
| 2 | 0.326 | 0.283 | 11.13 | 6.35 |
| 5 | 0.375 | 0.207 | 9.79 | 65.19 |
| 6 | 0.397 | 0.397 | 18.02 | 54.95 |
| 7 | 0.527 | 0.540 | 107.68 | 50.72 |
| 8 | 0.326 | 0.201 | 30.60 | 151.58 |

The household table shows why the macro result matters. Seq2Point slightly leads on event F1 in House 7 and is nearly tied in House 6, but HistGBR leads in the other four homes. Energy error also varies substantially by house; neither learned model is uniformly best on every household.

## Interpretation

- The always-off baseline obtains the lowest all-minute MAE because washing-machine power is zero for most minutes. Its event-state F1 is zero and its energy relative error is 100%, so it is not a useful disaggregation solution.
- HistGBR is the most defensible model from this experiment: it has higher macro event-state F1 and per-minute F1, higher recall, lower overall RMSE and lower relative energy error than Seq2Point.
- Seq2Point has a slightly lower macro event-envelope MAE (339.91 W versus 355.54 W), but it does not translate that into better event detection or energy estimation.
- Both learned models have modest event-state F1 (0.356 and 0.294). They should be treated as exploratory household-insight estimates, not as ground truth or as the basis for billing, automated control or customer penalties.

A supplementary full-valid-target-support comparison is also exported for always-off, hour-of-week and HistGBR. It has higher coverage (79.75% of the full grid, 100% of valid target minutes), but it is not directly comparable with Seq2Point because the CNN requires a complete 121-minute aggregate window.

## Failure analysis

The notebook selects HistGBR for the illustrative failure report because it has the higher observed macro event-state F1. Two House 5 examples are retained:

1. **Missed actual event:** 16 February 2015, 08:00–10:12 UTC (133 minutes). The actual event contained 0.912 kWh, had 99.25% common scored support, and received zero predicted-event overlap.
2. **False activation:** 14 January 2015, 03:44–09:50 UTC (367 minutes). The predicted event had 99.46% common support, no actual-event overlap and 4.126 kWh predicted energy. It was also flagged as `predicted_event_suspect` because its envelope exceeded 240 minutes.

These examples are consistent with the central NILM challenge: other household loads can resemble washing-machine demand, and a model can assign washing-machine power to an overlapping or unrelated aggregate load. The long false activation also shows why event duration and data-quality flags matter alongside pointwise regression metrics.

## Figures generated by the notebook

- **Raw House 1:** `outputs/figures/data_quality/house1_daily_coverage.png`; `house1_before_after_zoom.png`.
- **EDA:** `outputs/figures/eda/house*_representative_day.png`; two `house*_cycle_*.png` examples per selected house; `cross_house_comparison.png`; `hour_of_day_usage.png`; `day_of_week_usage.png`.
- **Model evaluation:** `outputs/figures/modelling/house*_actual_vs_predicted.png`; `per_house_model_metrics.png`; `failure_missed_actual_event_house5.png`; `failure_false_activation_house5.png`.

## References

1. University of Strathclyde, **REFIT Electrical Load Measurements** dataset and official cleaned-data README: [dataset page](https://pureportal.strath.ac.uk/en/datasets/refit-electrical-load-measurements/) · [cleaned-data README](https://pure.strath.ac.uk/ws/portalfiles/portal/62090183/CLEAN_READ_ME_081116.txt).
2. Manca, M. M. and Massidda, L. (2022). “Deep learning based non-intrusive load monitoring with low resolution data from smart meters.” *Communications in Applied and Industrial Mathematics*, 13(1), 39–56. DOI: [10.2478/caim-2022-0004](https://doi.org/10.2478/caim-2022-0004).

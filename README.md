# Flock REFIT NILM — Washing-Machine Disaggregation

## Project overview

This project investigates whether one-minute whole-house electricity demand can be used to estimate the power consumed by a washing machine. It uses the REFIT domestic electricity dataset and follows the stages of a practical non-intrusive load monitoring (NILM) workflow: raw-data quality assessment, household and appliance selection, washing-machine usage analysis, aggregate-only modelling, household-level validation, and interpretation for an energy utility.

The washing-machine sub-meter is the **regression target**, not a model input. Predictive features are derived from the household aggregate signal and time information. Other appliance sub-meter channels are not used as model features.

## Main notebook

The analysis is implemented in one self-contained Google Colab notebook:

`flock_refit_final_colab_cuda_memory_fixed.ipynb`

The notebook contains the ingestion, cleaning, event-detection, feature-engineering, baseline, HistGradientBoosting, Seq2Point, evaluation and plotting code. It does not require a separate source-code ZIP to execute.

### Run in Google Colab

1. Upload the notebook to Google Colab.
2. Place the following source files in `MyDrive/flock-refit-archives/`:
   - `REFIT_RAW_081116.7z`
   - `CLEAN_REFIT_081116.7z`
   - `CLEAN_READ_ME_081116.txt`
3. Run the notebook sequentially from the beginning. The raw and cleaned CSV files are extracted to Colab local storage; processed files, figures and metrics are saved to Google Drive.
4. The notebook uses the following output locations:
   - Project outputs: `MyDrive/flock-refit-nilm-final/`
   - Processed Parquet files: `MyDrive/flock-refit-processed/final_v7/`
5. For a fresh, reproducible run, execute all upstream data and feature-preparation cells before running model evaluation. Do not combine metrics from different notebook runs.

The notebook installs `py7zr` and `pyarrow` when they are unavailable. HistGradientBoosting runs on CPU; the optional Seq2Point CNN uses CUDA when available.

## Analysis workflow

| Stage | Work performed |
|---|---|
| Raw-data audit | Inspect raw House 1, timestamps, duplicates, invalid measurements, coverage, gaps and outages; resample to one-minute values. |
| Household selection and EDA | Read the official cleaned-data README for appliance mappings; verify the selected households; detect washing-machine events and compare usage patterns. |
| Feature engineering | Construct aggregate-only lag, rolling-statistic and calendar features. Keep the washing-machine target separate. |
| Modelling | Compare always-off and training-derived hour-of-week baselines with HistGradientBoosting. A Seq2Point CNN is included as a supplementary temporal experiment. |
| Validation | Use Leave-One-Household-Out (LOHO): one entire household is held out while the other selected households provide training data. |
| Interpretation | Review event detection, power and energy errors, household variation, failure cases, and illustrative energy in evening time windows. |

## Key methodological decisions

- **Household cohort:** preferred candidates are Houses 1, 2, 5, 6, 7 and 8. The notebook verifies their dedicated `Washing Machine` channel and data eligibility against the official README and data files. If a candidate fails the documented eligibility rules, the selection code applies its deterministic replacement rule. `Washer Dryer` is not treated as equivalent to `Washing Machine`.
- **Time:** Unix timestamps are treated as UTC. `Europe/London` is used when calculating local hour/day summaries.
- **Resolution:** source readings are resampled to one-minute mean power. The primary minute-coverage criterion is at least 50% of the expected source readings.
- **Duplicates and invalid readings:** exact duplicates are removed; conflicting readings at the same timestamp are collapsed by per-channel median. Negative appliance readings are marked missing. Raw appliance readings above the documented 4,000 W limit are marked missing; unusually high aggregate readings are flagged for review rather than automatically discarded.
- **Gaps:** only complete internal gaps of one or two minutes are interpolated. Leading/trailing gaps and longer gaps remain missing. The measured washing-machine target is not interpolated for regression training or scoring.
- **Provider-filled data:** aggregate-based zero-fill and forward-fill patterns are flagged as potential provider filling. These are quality-control heuristics, not proof of a sensor outage.
- **Cycle definition:** candidate ON minutes exceed 20 W; OFF gaps shorter than 13 valid minutes are merged; the merged event must span at least 27 minutes. This operational rule is based on a REFIT-specific published reference and is not a universal definition of a wash cycle. Events longer than 240 minutes are separately flagged as suspect-fill candidates by a project quality-control rule.
- **Features:** aggregate current, aggregate lags, rolling statistics and calendar features. Appliance sub-meter channels and household identifiers are excluded from model inputs.
- **Training cap:** up to 200,000 sampled rows per training household per fold, preserving the observed ON/OFF distribution.
- **Validation:** the held-out household's target is not used for fitting, feature-scaler fitting, baseline construction or model selection. It is reserved for evaluation.

## Metrics and interpretation

The notebook reports power-estimation errors (MAE, RMSE and R² where defined), ON/OFF precision/recall/F1, active/event-period error, energy error and evaluation coverage. Per-household outcomes should be examined alongside macro averages. Overall MAE alone is not sufficient: washing machines are inactive for much of the timeline, so an always-off predictor can appear competitive on that metric.

The final benchmark values are generated from executed metric files. See `RESULTS.md`; values are intentionally not hard-coded in this README.

## Current submission status

The current working run successfully completed the Seq2Point training stage for the House 1 fold with finite training and validation losses. The six-household LOHO benchmark was not yet complete at the time these documents were prepared. Consequently, no final six-fold model metrics or claims of cross-household performance are made here. Update `RESULTS.md` and the recommendation after the completed benchmark CSVs have been saved.

## Limitations

REFIT contains a limited number of UK households. Aggregate and appliance sub-meter measurements may not be perfectly synchronised, appliance signatures can overlap, and the cleaned release may contain provider-filled periods whose original outages cannot be fully reconstructed. Results on REFIT do not establish performance in Indian households or across all washing-machine designs. A local labelled dataset and external validation would be required before deployment.

## AI assistance

AI assistance was used for implementation support, debugging, code review and documentation refinement. Data-quality findings, plots and quantitative model results must be generated by executing the notebook on the supplied REFIT data; unexecuted or partial model runs are not presented as final results.

## References

1. Murray, D., Stankovic, L. and Stankovic, V. (2017). “An electrical load measurements dataset of United Kingdom households from a two-year longitudinal study.” *Scientific Data*, 4, 160122. https://doi.org/10.1038/sdata.2016.122
2. Manca, M. M. and Massidda, L. (2022). “Deep learning based non-intrusive load monitoring with low resolution data from smart meters.” *Communications in Applied and Industrial Mathematics*, 13(1), 39–56. https://doi.org/10.2478/caim-2022-0004
3. Kelly, J. and Knottenbelt, W. (2015). “Neural NILM: Deep Neural Networks Applied to Energy Disaggregation.” *BuildSys 2015*. https://doi.org/10.1145/2821650.2821672
4. Zhang, C., Zhong, M., Wang, Z., Goddard, N. and Sutton, C. (2018). “Sequence-to-Point Learning With Neural Networks for Non-Intrusive Load Monitoring.” *Proceedings of AAAI*, 32(1), 2604–2611. https://doi.org/10.1609/aaai.v32i1.11873

## What I would do next

- Complete and save all LOHO folds, then regenerate the metric tables and plots from the saved outputs.
- Analyse at least two concrete held-out failure cases and verify that results are consistent with the event-detection audit.
- Evaluate the model on additional eligible households and on locally labelled data before considering deployment.
- If time permits, compare the optional Seq2Point model with HistGradientBoosting on exactly matched evaluation timestamps.

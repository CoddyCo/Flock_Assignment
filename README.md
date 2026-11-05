# Flock Energy — REFIT Washing-Machine Disaggregation

## Project overview

This project builds a reproducible, one-minute non-intrusive load-monitoring (NILM) workflow using the University of Strathclyde's REFIT Electrical Load Measurements dataset. It covers the four parts of the Flock Energy take-home assignment:

1. Audit and clean the **raw House 1** measurements, then resample them to one-minute intervals.
2. Explore washing-machine operation in the official cleaned data for six eligible households.
3. Estimate washing-machine power from whole-house aggregate power and evaluate the models with **Leave-One-Household-Out (LOHO)** validation.
4. Translate observed usage into a cautious utility-facing opportunity and document the limits of the analysis.

The complete implementation is in the Jupyter notebook `final_notebook.ipynb`. The numerical results in the companion documents are taken from its completed execution on 5 November 2026.

## Repository contents

The notebook expects the following local project layout:

```text
project/
├── final_notebook.ipynb
├── requirements-local.txt
├── README.md
├── DATA.md
├── RESULTS.md
├── Recommendation.md
├── data/
│   ├── archives/
│   │   ├── REFIT_RAW_081116.7z
│   │   ├── CLEAN_REFIT_081116.7z
│   │   └── CLEAN_READ_ME_081116.txt
│   ├── refit_data_final/       # extracted CSV files
│   └── processed/
│       └── final_v7/           # one-minute Parquet files
├── outputs/
│   ├── metrics/
│   └── figures/
│       ├── data_quality/
│       ├── eda/
│       └── modelling/
└── models/
```

The raw archive is used for the House 1 cleaning exercise. The official cleaned archive is used for multi-household EDA and model development. Do not substitute the raw House 1 target for the cleaned-data model target.

## Local setup and execution

The final notebook was executed in a local macOS environment using Python 3.11.15 and PyTorch's Apple MPS device. The notebook selects an available accelerator (MPS, CUDA or CPU) and retains CPU fallback.

1. Create and activate a virtual environment:

   ```bash
   python3 -m venv .venv
   source .venv/bin/activate
   ```

2. Install the dependencies listed in `requirements-local.txt`:

   ```bash
   python -m pip install --upgrade pip
   python -m pip install -r requirements-local.txt
   ```

   The notebook also checks the active Jupyter kernel and installs missing runtime dependencies into that interpreter. Main libraries include NumPy, pandas, scikit-learn, PyTorch, Matplotlib, PyArrow and py7zr.

3. Place the two REFIT `.7z` archives and `CLEAN_READ_ME_081116.txt` in `data/archives/`.

4. Start Jupyter from the project directory and run `final_notebook.ipynb` from top to bottom:

   ```bash
   jupyter lab
   ```

   The notebook uses the current working directory as the project root by default. Set `FLOCK_REFIT_HOME` before launching Jupyter if you need a different project root.

5. Outputs are written to `data/processed/final_v7/`, `outputs/metrics/`, `outputs/figures/` and `models/`.

**Fast restart:** when the cleaned one-minute Parquet files, `household_summary.csv` and `detected_cycles.csv` already exist, use the notebook's fast-resume route. It skips archive extraction, raw House 1 cleaning and the earlier EDA, rebuilds aggregate-only model features, and reruns all six LOHO folds. The fast-resume checks stop if any required file is absent.

## Data selection and preprocessing decisions

- **Household cohort:** Houses 1, 2, 5, 6, 7 and 8. Selection is based on a dedicated README-mapped `Washing Machine` channel, available files and sufficient usable coverage/cycles. Solar-affected Houses 3, 11 and 21 are excluded; House 4 has two washing-machine channels; House 13 has a documented channel-change note. A `Washer Dryer` channel is not treated as a washing-machine substitute.
- **Time:** Unix timestamps are converted to timezone-aware UTC. London local time is used only for hour/day-of-week analysis.
- **Resolution:** source readings are resampled to one-minute means. A minute is valid when at least 50% of the expected source readings are present.
- **Duplicates:** exact duplicate rows are removed. Conflicting rows at the same timestamp are collapsed using a per-channel median; the `Issues` flag uses the maximum so any flagged duplicate keeps the minute flagged.
- **Gaps/outliers:** negative readings and invalid numeric/infinite values are set to missing (except negative aggregate readings in solar-affected homes, which are retained by the cleaning function). Raw appliance/IAM readings above 4,000 W are set to missing. Aggregate readings above 23,000 W are flagged, not silently deleted. Only complete internal gaps of one or two minutes are interpolated in the cleaning pipeline; longer outages and edge gaps remain missing. Suspected provider-fill intervals are protected from interpolation.
- **Cycle rule:** strictly above 20 W; merge fully observed OFF gaps shorter than 13 minutes; then keep merged envelopes of at least 27 minutes. Events longer than 240 minutes are separately flagged as `suspect_fill` by this project's quality-control rule.
- **Targets and predictors:** model inputs contain aggregate-derived information only. Appliance sub-meter readings are used for the training target, labels and evaluation, not as predictors. Issue-flagged target minutes and suspect-fill event intervals are excluded from regression fitting/scoring. The measured target itself is not interpolated for regression.

Detailed decisions and raw-data audit results are in [DATA.md](DATA.md).

## Models and validation design

Four approaches are compared:

- **Always-off baseline:** predicts 0 W.
- **Hour-of-week profile:** predicts the training-house mean for the same weekday/hour, with a training-set global-mean fallback.
- **HistGradientBoostingRegressor (HistGBR):** uses current aggregate power, aggregate lags (1, 5, 10, 30 and 60 minutes), first differences, rolling mean/standard deviation/maximum features (5, 15, 30 and 60 minutes), and local calendar features. Configuration: 100 iterations, learning rate 0.1, 31 leaf nodes, L2 regularisation 1.0 and random seed 42. Training is capped at 200,000 labelled rows per training household.
- **Seq2Point CNN:** uses a centred 121-minute aggregate window (60 minutes on either side of the centre) to predict the centre-minute washing-machine power. It has three 1-D convolution blocks (32, 64 and 64 channels), two max-pooling layers, adaptive average pooling to 10 positions, and a small dense regression head. It uses weighted Huber loss, a training-only input scaler, a chronological validation tail from training households, a maximum of 8 epochs and patience 3. The training cap is 125,000 windows and 25,000 validation windows per fold across the five training households. Inference uses aggregate readings only.

LOHO holds out one complete household at a time: the other five households are used for fitting, and this is repeated for all six selected homes. The held-out household's appliance target is not used for training, preprocessing fit, early stopping or model selection. For direct comparison, all four approaches are scored on the common support where the 121-minute aggregate context exists and the measured target is valid. Coverage is reported against both the full household minute grid and valid target minutes.

The Seq2Point model uses future aggregate context because its window is centred. It is therefore a retrospective/offline estimate, not a causal real-time predictor. HistGBR's engineered features use current and past aggregate values.

## Results at a glance

On the common LOHO support, HistGBR has the strongest overall balance of event detection, regression and energy estimation:

- HistGBR macro event-state F1: **0.356**; Seq2Point: **0.294**.
- HistGBR macro MAE/RMSE: **23.36 W / 136.35 W**; Seq2Point: **25.85 W / 144.00 W**.
- Mean relative energy error: **31.46%** for HistGBR and **44.91%** for Seq2Point.
- Mean common-support coverage: **71.14% of the full minute grid**, or **89.20% of valid target minutes**.

The always-off baseline has the lowest all-minute MAE (16.03 W), but its event-state F1 is zero. This illustrates why zero-dominated appliance data should not be judged by MAE alone. HistGBR is the most defensible model from this experiment, but its event F1 and event-envelope errors are not strong enough for unsupervised operational decisions. Full results, per-house variation and failure examples are in [RESULTS.md](RESULTS.md).

## Assumptions, trade-offs and limitations

- The six-house cohort is a rule-based, manageable sample, not all 20 REFIT homes.
- The 20 W / 27-minute ON / 13-minute OFF rule is a published REFIT-specific operating definition, not a universal washing-machine definition.
- `suspect_fill` (>240-minute event envelope) is a project QC flag, not proof of a bad sensor value.
- One-minute data preserve more temporal structure than 15- or 30-minute interval readings. Lower-resolution data would make cycle boundaries and appliance attribution less precise.
- Household variation, missingness, asynchronous sub-meter readings and overlapping appliance signatures limit generalisation.
- Results from UK REFIT homes must not be assumed to transfer to Indian homes without local validation.

## AI assistance

ChatGPT was used during development for debugging, implementation review, performance troubleshooting and preparation of the submission documentation. The reported numerical results are taken from the executed notebook outputs, not generated estimates. The notebook itself contains the code, run outputs, synthetic checks and exported evidence manifest.

## Reproducibility outputs

After a complete run, the notebook exports 55 evidence files (metrics, audit outputs, figures and environment information) into `flock_refit_evidence_for_reports.zip`. Key numerical outputs include:

- `outputs/metrics/house1_raw_audit_summary.json`
- `outputs/metrics/household_selection.csv`
- `outputs/metrics/household_summary.csv`
- `outputs/metrics/detected_cycles.csv`
- `outputs/metrics/usage_summary.csv`
- `outputs/metrics/peak_window_energy.csv`
- `outputs/metrics/loho_results.csv`
- `outputs/metrics/loho_macro.csv`
- `outputs/metrics/failure_cases.csv`
- `outputs/metrics/environment_versions.json`

The notebook also runs synthetic checks for interpolation, provider-fill protection, event merging, missing-gap splitting, energy calculation, vectorised event masks and event-support coverage.

## References

1. University of Strathclyde, **REFIT Electrical Load Measurements** dataset and official cleaned-data README: [dataset page](https://pureportal.strath.ac.uk/en/datasets/refit-electrical-load-measurements/) · [cleaned-data README](https://pure.strath.ac.uk/ws/portalfiles/portal/62090183/CLEAN_READ_ME_081116.txt).
2. Manca, M. M. and Massidda, L. (2022). “Deep learning based non-intrusive load monitoring with low resolution data from smart meters.” *Communications in Applied and Industrial Mathematics*, 13(1), 39–56. DOI: [10.2478/caim-2022-0004](https://doi.org/10.2478/caim-2022-0004). Table 2 is the source used for the REFIT washing-machine 20 W, 13-minute OFF and 27-minute ON settings.

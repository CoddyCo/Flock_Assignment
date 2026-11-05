# DATA.md — REFIT Data Audit and Preprocessing

## 1. Data sources and scope

The analysis uses the REFIT domestic electricity dataset:

- **Raw REFIT:** House 1 is used for the detailed raw-data audit and cleaning exercise. The raw archive contains separate House 1 parts, which are combined before processing.
- **Cleaned REFIT:** the official cleaned release is used for household selection, washing-machine exploratory analysis, cycle detection and modelling.
- **Official mapping:** the cleaned-data README is parsed to map appliance names to household CSV columns. The mapping is checked against the actual CSV schema; channel numbers are not assumed to mean the same appliance across households.

The preferred modelling cohort is Houses 1, 2, 5, 6, 7 and 8. The selection logic verifies a dedicated `Washing Machine` channel, excludes documented solar-affected aggregate houses (3, 11 and 21), and applies the eligibility requirements for joint valid data and complete cycles. A `Washer Dryer` channel is not substituted for a washing-machine channel. The exact accepted cohort and channel/date manifest should be taken from the notebook's `household_selection.csv` and `household_summary.csv` outputs.

## 2. Time handling and one-minute resampling

The Unix timestamp is the canonical time axis and is interpreted as UTC. Local time is derived using `Europe/London` only for hour-of-day and day-of-week summaries. The source readings are approximately eight-second measurements; they are converted to one-minute mean power rather than summed power.

The primary minute-validity rule requires at least 50% of the expected source readings for that minute. Coverage sensitivity is also calculated using at least one reading and an 80% criterion. A complete one-minute UTC index is constructed so missing minutes remain visible in the time series.

## 3. Raw House 1 cleaning policy

The raw House 1 parts are concatenated before the audit. The processing policy is:

1. Parse and sort timestamps; record out-of-order values and duplicate counts.
2. Drop exact duplicate rows.
3. Where multiple non-identical rows share a timestamp, collapse numeric channels using the per-channel median.
4. Mark negative appliance readings as missing. Raw appliance readings above the documented 4,000 W sensor limit are marked missing. Aggregate values above 23,000 W are flagged for review rather than automatically removed.
5. Resample valid source readings to one-minute means under the 50% coverage criterion.
6. Interpolate only complete internal missing runs of one or two minutes. Do not interpolate leading or trailing gaps or any run of three or more minutes.
7. Retain missingness and imputation flags and report longer gaps/outages.

Coverage counts, duplicate counts, invalid-reading counts, timestamps and outage durations are execution-derived outputs. They are not reproduced as numbers here because the current submission run has not yet completed all report-generation cells. Use the saved `house1_raw_audit_summary.json`, `house1_raw_coverage.csv`, channel audit and outage outputs to populate the final quantitative audit.

## 4. Cleaned-house preprocessing

The same one-minute UTC grid and 50% coverage rule are applied consistently to each selected cleaned household. Missing appliance readings are not silently converted to zero. The washing-machine target is never interpolated for regression training or scoring.

The cleaned provider release may already contain zero-filled or forward-filled values. The notebook applies aggregate-based heuristics to flag potential provider-filled intervals, including consecutive exact-zero periods and long runs of identical aggregate values. These are potential-fill indicators rather than definitive evidence of a physical outage. A provider-filled target-only gap may not be recoverable from the cleaned release.

If a documented `Issues` flag is present, its schema and meaning are checked before use. Issue-flagged target minutes are excluded from model fitting and evaluation while remaining visible in EDA and cycle audit counts. Do not infer the existence or encoding of this field if it is absent.

## 5. Washing-machine event labels

The primary operational event rule is:

- measured washing-machine power strictly above **20 W** identifies candidate ON minutes;
- fully observed OFF gaps shorter than **13 minutes** are merged;
- the merged event envelope must be at least **27 minutes** long;
- missing-data gaps are not bridged as OFF time.

These parameters are a REFIT-specific benchmark reference from Manca and Massidda (2022, Table 2), not universal washing-machine settings. Q10 is treated as a sensitivity analysis rather than the primary detector.

Events longer than **240 minutes** are flagged as suspect-fill candidates by a separate project quality-control heuristic. This is not a published physical limit. Suspect intervals remain visible in the audit but are excluded from the model's target labels/training/scoring under the notebook's stated policy.

## 6. Known limitations

- REFIT aggregate and appliance sub-meter channels can be asynchronous, so their minute-level readings may not sum exactly.
- Some aggregate signals are affected by solar generation; these households are excluded from the primary cohort.
- Appliance channels can change or represent different appliances during a monitoring period. The official mapping and change notes must be respected.
- The cleaned release may contain historical provider-filled values, limiting recovery of the original missing-data pattern.
- The selected cohort is small and UK-specific; the dataset does not establish transferability to Indian homes.

## 7. Execution-derived outputs to retain

The following files should be saved from the completed notebook run and used to update this document:

- `house1_raw_audit_summary.json`
- `house1_raw_audit_channels.csv`
- `house1_raw_coverage.csv`
- `outages.csv` (or the notebook's equivalent outage summary)
- `household_selection.csv`
- `household_summary.csv`
- cycle/event audit tables and the data-quality figures

All counts, percentages, dates and coverage values in the final version should come from these outputs, not from console estimates.

## Reference

Manca, M. M. and Massidda, L. (2022). “Deep learning based non-intrusive load monitoring with low resolution data from smart meters.” *Communications in Applied and Industrial Mathematics*, 13(1), 39–56. https://doi.org/10.2478/caim-2022-0004

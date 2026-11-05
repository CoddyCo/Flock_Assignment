# DATA — REFIT Coverage, Quality and Cleaning

## Scope and data sources

The analysis uses the University of Strathclyde REFIT Electrical Load Measurements dataset. The raw archive is used only for the required House 1 cleaning exercise. The official cleaned archive is used for the six-household washing-machine EDA and disaggregation experiment.

The source measurements are approximately 7–8 seconds apart. All analysis is performed at one-minute resolution using mean power in watts. The notebook uses the Unix timestamp as canonical UTC time and converts to `Europe/London` only for calendar-based summaries.

## Raw House 1 audit

Files: `RAW_House1_Part1.csv` and `RAW_House1_Part2.csv`.

| Audit item | Finding |
|---|---:|
| Source rows across both files | 8,518,763 |
| Raw time span | 2013-10-09 14:06:17 to 2015-07-10 11:56:32 UTC |
| Timestamp parse failures | 0 |
| Out-of-order rows | 2 |
| Exact duplicate rows dropped | 595,940 |
| Conflicting duplicate rows collapsed | 1,187,067 |
| Timestamps with conflicting duplicates | 571,959 |
| Rows after duplicate handling | 7,307,715 |
| Median source interval | 7 seconds |
| 1st / 99th percentile interval | 1 / 15 seconds |
| Intervals longer than 60 seconds | 4,040 |
| Intervals longer than 10 minutes | 343 |
| Intervals longer than 1 hour | 21 |
| `Time` field agrees with Unix as UTC (within 1 second) | 100% |
| `Time` field agrees with London-local interpretation (within 1 second) | 40.721% |
| Resampled one-minute bins | 920,031 |
| One-minute output span | 2013-10-09 14:06 to 2015-07-10 11:56 UTC |
| Missing-run records of at least 3 minutes, across audited channels | 35,187 |
| Longest missing run | 78,221 minutes |

The timestamp audit supports interpreting the Unix timestamps as UTC. The notebook records four UK daylight-saving clock-change dates within this monitoring period: 2013-10-27 (25-hour local day), 2014-03-30 (23-hour day), 2014-10-26 (25-hour day) and 2015-03-29 (23-hour day).

After duplicate handling, 63 aggregate readings above the 23,000 W physical-bound alert were flagged but retained for review. The aggregate 99.9th percentile was approximately 10,680 W; values above the percentile were retained rather than automatically clipped. The sub-meter sum exceeded aggregate power by more than 10% on 43,087 rows; this was kept as a diagnostic because appliance and aggregate meters are not perfectly synchronous.

The raw House 1 CSV did **not** contain an `Issues` field, although the cleaned-data README documents the field in the cleaned release.

## Raw House 1 cleaning procedure

1. Parse the Unix timestamp, compare the accompanying `Time` field, and keep UTC as the canonical time axis.
2. Remove exact duplicate rows.
3. Collapse conflicting rows sharing a timestamp using a per-channel median. The `Issues` flag, when present, uses the maximum across duplicates.
4. Coerce power channels to numeric and set non-numeric/infinite values and negative non-solar readings to missing. Raw appliance/IAM values above 4,000 W are set to missing and logged.
5. Flag aggregate readings above 23,000 W; do not silently remove them.
6. Estimate the source interval from the observed timestamps and resample to complete UTC one-minute bins. A minute is valid when at least 50% of the expected source readings are present.
7. Interpolate only complete, internal missing runs of one or two minutes. Do not fill leading/trailing gaps, longer gaps or any run protected by a provider-fill flag.
8. Record missing runs of at least three minutes in the outage output.

For raw House 1, the aggregate channel had 19,937 one-minute bins filled by the short-gap interpolation rule. The notebook creates daily-coverage and before/after plots using Aggregate, Appliance5 (Washing Machine) and Appliance1 (Fridge). The 48-hour zoom is centred on 2014-08-29 00:00 UTC, selected from the highest concentration of recorded data-quality flags. In the zoom, the filled points are limited to short internal gaps; extensive missing sections remain visibly unfilled. The daily-coverage plots show that outages recur across the monitoring period and differ by channel, supporting the decision not to interpolate long gaps.

## Cleaned-house selection and coverage

The official README mapping was checked against the actual CSV columns. The selected cohort is Houses **1, 2, 5, 6, 7 and 8**. All six have one dedicated `Washing Machine` channel and passed the notebook's minimum eligibility checks (at least 30 joint-valid days and at least 10 complete events). Solar-affected Houses 3, 11 and 21 were excluded; House 4 has two washing-machine channels; House 13 has a documented channel-change note. `Washer Dryer` was not substituted for `Washing Machine`.

| House | Cleaned target channel / mapping | UTC coverage start | UTC coverage end |
|---:|---|---|---|
| 1 | Appliance5 — Beko WMC6140 | 2013-10-09 13:06 | 2015-07-10 11:56 |
| 2 | Appliance2 — LG F1289TD | 2013-09-17 22:08 | 2015-05-28 08:05 |
| 5 | Appliance3 — AEG L99695HWD | 2013-09-26 09:56 | 2015-07-06 17:48 |
| 6 | Appliance2 — Bosch Classixx 1200 Express | 2013-11-28 12:15 | 2015-06-28 22:14 |
| 7 | Appliance5 — Bosch (model unspecified) | 2013-11-01 22:01 | 2015-07-08 02:10 |
| 8 | Appliance4 — README label “Washing Machine” | 2013-11-01 22:13 | 2015-05-10 23:36 |

The table below reports the cleaned one-minute washing-machine target before the later model-specific exclusions. `Missing` is the percentage of the target channel that remains missing; `Zero` is the share of valid target readings equal to zero. `Issue minutes` are the cleaned-release `Issues=1` minutes. `Joint-valid days` require both aggregate and target readings and exclude issue-flagged minutes.

| House | One-minute grid rows | Valid target rows | Missing target | Zero among valid | Issue-flagged minutes | Joint-valid days | Complete non-suspect cycles |
|---:|---:|---:|---:|---:|---:|---:|---:|
| 1 | 920,091 | 774,005 | 15.88% | 97.55% | 16,686 | 526.2 | 332 |
| 2 | 889,078 | 645,191 | 27.43% | 94.31% | 13,458 | 439.0 | 287 |
| 5 | 933,593 | 826,711 | 11.45% | 52.45% | 70,888 | 527.4 | 606 |
| 6 | 831,480 | 675,755 | 18.73% | 97.42% | 14,524 | 459.4 | 95 |
| 7 | 882,970 | 751,828 | 14.85% | 74.64% | 35,837 | 497.8 | 795 |
| 8 | 799,284 | 688,377 | 13.88% | 84.13% | 12,331 | 469.8 | 335 |

The event totals are defined using the common 20 W / 27-minute ON / 13-minute OFF rule. The eligibility summary records 2,450 complete, non-suspect events across the six selected homes. The cycle table separately records incomplete and suspect events so that they are not silently counted as complete usage cycles.

## Cleaned-data quality handling

- **Resampling:** every selected cleaned household is resampled to one-minute mean power using the same 50%-of-expected-source-readings validity rule.
- **Provider-fill protection:** on cleaned data, aggregate patterns of exact zero for at least two consecutive minutes or an identical non-null value for at least ten consecutive minutes are flagged as possible provider fill. These are heuristics, not proof of an outage. Flagged intervals are masked across aggregate and appliance channels and protected from being interpolated back in.
- **Short gaps:** the cleaning pipeline interpolates only fully internal one- or two-minute gaps in non-target channels. The measured washing-machine regression target is not interpolated.
- **`Issues` field:** in the cleaned release, any source reading with `Issues=1` marks the corresponding one-minute bin. Issue-flagged target minutes are excluded from model fitting and scoring. They remain available during event labelling so that an asynchronous-meter consistency flag does not artificially split an event.
- **Suspect events:** an event envelope longer than 240 minutes is flagged `suspect_fill`. This is a project quality-control threshold, not a published REFIT threshold or proof that the event is invalid. Suspect event intervals remain in the audit, but are excluded from model targets and reference activity labels.
- **Event-label continuity:** internal one- or two-minute target gaps may be bridged for event labelling only when not protected by a provider-fill interval. The interpolated label-only values are not treated as measured regression targets or measured energy.
- **Outliers:** high aggregate readings are reported and retained unless a separate explicit invalidity rule applies. Sub-meter/aggregate discrepancies are diagnostics, not automatic row-deletion rules.

## Remaining data concerns

1. Missingness differs materially by household. House 2 has the highest target missingness (27.43%); its evaluation support is consequently narrower.
2. House 5 has the largest issue-flagged count (70,888 minutes, 7.59% of its one-minute grid) and 42 `suspect_fill` events; 20,387 target minutes inside suspect event envelopes are excluded from model labels/targets.
3. Washing-machine activity is sparse in several homes: zero readings account for 94–98% of valid target minutes in Houses 1, 2 and 6. A low overall MAE can therefore be achieved by predicting zero.
4. The `Issues` field is absent from the raw House 1 files, although present in the cleaned release. The raw and cleaned datasets are separate assignment inputs and should not be treated as identical.
5. The provider-fill detector is heuristic. Long constant power values may sometimes be real appliance plateaus; flags must be interpreted alongside the signal and not treated as confirmed sensor faults.
6. The selected six homes are a manageable cohort, not a census of REFIT's 20 households. Results may differ in other houses or countries.

## Data outputs and figures

The notebook writes data-quality tables and logs under `outputs/metrics/`, including `house1_raw_audit_summary.json`, `house1_raw_audit_channels.csv`, `house1_raw_coverage.csv`, `household_selection.csv`, `data_manifest.csv`, `household_summary.csv`, `detected_cycles.csv`, `issue_flags.csv`, `suspected_fill.csv` and outage/duplicate/modified-value logs.

Key figures:

- `outputs/figures/data_quality/house1_daily_coverage.png`
- `outputs/figures/data_quality/house1_before_after_zoom.png`

The processed one-minute Parquet files are written to `data/processed/final_v7/`. Original archives and extracted source CSVs are not included in the small evidence bundle.

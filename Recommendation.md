# Recommendation.md — Utility-Facing Summary

## Use case

One-minute whole-house smart-meter readings can be analysed to estimate washing-machine activity and energy use. This may support exploratory customer insights, such as identifying typical operating times and discussing whether some cycles could be scheduled outside a utility-defined peak period. It should not be used for billing, customer penalties or guaranteed cycle identification without further validation.

## Evidence and suitable use

The REFIT analysis uses aggregate-only inputs and holds out complete households during validation. The final six-household benchmark is still to be completed; therefore, this submission does not claim a measured level of predictive accuracy or quantified savings. A completed per-household LOHO evaluation is required before deciding whether the model is sufficiently reliable for customer-facing recommendations. Overall MAE alone is not enough because washing-machine demand is zero for much of the day.

## Potential energy insight

The notebook calculates observed washing-machine energy in illustrative local-time windows of 16:00–20:00 and 17:00–21:00. These are scenario windows, not official tariff periods. Energy observed in these windows is not automatically shiftable energy, and no tariff saving is inferred. A utility should substitute its actual peak window and establish which customers and cycles are flexible.

## How to measure an intervention

Run a pilot with a baseline period and an intervention period, preferably with a comparable control group. Measure peak-window kWh per household, energy shifted, recommendation engagement and persistence of behaviour change. REFIT is observational, so it cannot establish that a recommendation causes savings.

## Limitations and transfer to India

Appliance signatures overlap, the selected REFIT cohort is small and UK-specific, and aggregate and sub-meter sensors may not be perfectly synchronised. Washing-machine designs and programmes, internal water heating, semi-automatic operation, appliance mix (including air conditioners, geysers and inverters/UPS), supply-voltage variation and metering practices may differ in Indian homes. Local labelled data and external validation are necessary before deployment in India.

With 15- or 30-minute readings, short phases, ON/OFF transitions and cycle boundaries become less identifiable, and overlapping loads are harder to separate. Interval energy can still be estimated, but appliance-level timing and cycle detection are less reliable.

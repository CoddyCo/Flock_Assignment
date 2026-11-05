# Recommendation — Utility Use of REFIT Washing-Machine Insights

## Recommendation

Use appliance disaggregation as an **advisory customer-insight layer**, not as a billing meter or automated control signal. In this REFIT sample, House 7 records 795 complete washing-machine cycles (10.66 per valid week), compared with 95 cycles (1.42 per valid week) in House 6. House 7 also has the highest observed washing-machine energy in the illustrative 16:00–20:00 window: 87.14 kWh, or 1.168 kWh per valid week. These are historical measurements, not achievable savings.

A utility could use a locally validated model to identify households with frequent washing-machine operation during its actual peak period and offer opt-in scheduling prompts. The 16:00–20:00 and 17:00–21:00 windows used here are illustrative only, not REFIT tariff periods.

## How to measure impact

Run a time-bounded pilot with an opt-in treatment group receiving prompts and a comparable control group receiving none. Compare baseline and intervention-period washing-machine kWh in the utility's actual peak window. Track shifted kWh per household, cycle completion time, engagement and persistence after prompts stop. Account for seasonality and household differences; where feasible, validate estimates against directly metered washing machines. REFIT is observational and cannot establish that prompts cause behaviour change.

## Suitability and limitations

HistGradientBoosting is the stronger learned model in this six-household LOHO experiment: macro event-state F1 is 0.356 versus 0.294 for Seq2Point, and relative energy error is 31.46% versus 44.91%. However, event F1 is modest and common-support coverage is 71.14% of the full minute grid (89.20% of valid target minutes). Performance varies by household. Use predictions for exploratory segmentation or pilot targeting only—not for billing, penalties or unattended automation.

Aggregate signatures overlap: kettles, heaters and other loads can resemble washing-machine demand. Missing readings, issue flags, provider-filled intervals and household-specific programmes further limit identification. The cohort contains six UK homes; transfer to India is not established. Machine types and programmes, voltage and metering characteristics, and other household loads may differ. Collect representative Indian aggregate and appliance-labelled data and validate by household before deployment.

With 15- or 30-minute readings, short appliance phases are averaged together. ON/OFF transitions, cycle boundaries and overlapping signatures become harder to distinguish, although interval energy may remain measurable. Event-level appliance attribution would be less precise and require validation at the intended meter resolution.

**Decision:** proceed only with a monitored, opt-in insight pilot after local validation. Treat REFIT as evidence that aggregate-only washing-machine estimation is worth exploring—not proof of deployable accuracy or quantified savings.

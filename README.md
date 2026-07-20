# Starlink Constellation Analyzer
### Atmospheric Drag–Based Decay Risk Screening for the Starlink Satellite Fleet

**Author:** Lukman
**Tools:** Python (pandas, requests, matplotlib) · Jupyter Notebook · CelesTrak Public API

---

## Problem Statement

Starlink's active constellation exceeds 10,000 satellites, and that number grows every week. Each satellite is subject to atmospheric drag, which increases as a satellite loses altitude — one of the earliest available signals of orbital decay. At this scale, manually reviewing every satellite's tracking data is not feasible.

## Business Objective

Build a repeatable, automatable screening tool that flags satellites showing anomalous drag signatures, turning a 10,000+ row dataset into a short, prioritized watchlist an analyst can act on — the same risk-scoring approach used elsewhere in this portfolio for aircraft fleet and maintenance risk, applied here to orbital data.

## Dataset

- **Source:** [CelesTrak](https://celestrak.org) public GP (General Perturbations) API — live, freely available orbital element data for every tracked object
- **Endpoint:** `https://celestrak.org/NORAD/elements/gp.php?GROUP=starlink&FORMAT=json`
- **Size:** 10,785 active Starlink satellites (pulled July 2026)
- **Fields:** 17 orbital elements per satellite, including inclination, eccentricity, mean motion, and BSTAR (drag coefficient)
- **Note:** CelesTrak enforces query-frequency limits on this endpoint. The pipeline caches results locally (`starlink_data.json`) and only re-fetches when no cache is present, both for reliability and to respect the API's usage policy.

## Data Cleaning

- Verified 0 missing values and 0 duplicate rows across all 10,785 records
- Dropped 3 columns carrying zero analytical value (`EPHEMERIS_TYPE`, `CLASSIFICATION_TYPE`, `ELEMENT_SET_NO`) — confirmed constant across the entire fleet via `.nunique()`
- Converted `EPOCH` from string to a proper datetime type, enabling age-based analysis
- Engineered `epoch_age_days` (freshness of each satellite's orbital fit) as a validation check

## Methodology

**1. Risk signal.** Used `BSTAR` (the atmospheric drag term in the orbital model) as the core decay-risk indicator — a satellite losing altitude experiences increasing drag before that shows up anywhere else in the data.

**2. Threshold.** BSTAR is heavily right-skewed (mean 0.000505 vs. median 0.000170; standard deviation larger than the mean), which rules out a mean ± std threshold, since outliers would distort the very boundary meant to catch them. Used an IQR (Tukey) fence instead:
- Q1 = 0.000020, Q3 = 0.000352, IQR = 0.000332
- Upper threshold = Q3 + 1.5 × IQR ≈ **0.000850**
- Initial flag: **897 satellites (8.3% of the fleet)**

**3. Confound check — new satellites.** Investigation revealed 546 of the 897 flagged satellites were not decaying — they were newly launched and still climbing from a low parking orbit to their operational shell, which naturally produces high drag as a side effect of low altitude, not failure. Used `REV_AT_EPOCH` (orbit count since launch, a direct age proxy that a calendar date alone can't provide) to separate the two populations, filtering to satellites with at least 1,000 revolutions.

**4. Validation.** Before trusting the remaining 351 satellites, ruled out a second possible explanation — that the flag was really just catching stale tracking fits rather than real drag. Compared orbital-fit age between flagged and non-flagged groups (1.42 vs. 1.51 days average) — no meaningful difference, ruling this out.

**5. Final model.** A robust z-score — `(BSTAR − median) / IQR`, floored at zero — ranks the validated 351-satellite watchlist by severity rather than a flat yes/no flag.

## Key Insights

- Screened all 10,785 active Starlink satellites; an unrefined threshold would have wrongly prioritized 546 healthy, newly-launched satellites over genuine risk candidates
- Refined model produces a validated **351-satellite watchlist** — a 61% reduction from the naive first pass, with every exclusion backed by evidence, not intuition
- A secondary pattern emerged during validation: elevated drag concentrates in satellites roughly 9–12 months old, with two specific 2025 launch batches showing the strongest signal — a lead worth operational follow-up, not yet a conclusion
- Confirmed the risk signal isn't an artifact of outdated tracking data

## Visualizations

| Chart | Purpose |
|---|---|
| `chart1_bstar_distribution.png` | Fleet-wide BSTAR distribution with the risk threshold marked |
| `chart2_confound_reveal.png` | Why the model needed refining — orbit count by group |
| `chart3_top15_watchlist.png` | The final, ranked watchlist |

## Business Recommendations

1. **Prioritize manual/operational review of the 351-satellite watchlist**, starting with the top 15 by risk score — the highest-confidence drag anomalies in the established (non-new) fleet.
2. **Investigate the 2025-179 and 2025-211 launch batches specifically** — both are overrepresented among the most extreme drag values in the 9–12 month age band, though the sample size here is small enough that this is a lead, not a conclusion.
3. **Re-run this screening on a recurring schedule** rather than as a one-off — drag is a moving target, and a watchlist is only useful if it stays current.

## Future Work & Automation

- **n8n pipeline:** scheduled weekly trigger → pull latest CelesTrak data → re-run the scoring model → diff against last week's watchlist → Slack/email alert on any newly-flagged satellite
- **Power BI dashboard:** star-schema model built over the exported CSVs, with drill-through from fleet-level KPIs down to individual satellite risk scores
- **Deeper validation:** cross-check the launch-batch finding against `INCLINATION` (shared orbital shell?) and total fleet size per launch batch (is 7-of-11 actually unusual, or expected at that sample size?)

## Limitations

- BSTAR is a *modeled* drag coefficient, not a direct physical measurement — it can reflect fitting artifacts as well as genuine atmospheric effects
- The launch-batch pattern above is based on 22 satellites in one age band — not large enough to confirm causation
- This is a screening tool, not a collision or reentry predictor. It prioritizes attention; it does not replace full orbital propagation

## Suggested Repo Structure

```
starlink-decay-risk-analyzer/
├── starlink_constellation_analyzer.ipynb   # Main analysis notebook
├── starlink_data.json                      # Cached raw CelesTrak pull
├── starlink_cleaned_full.csv               # Full cleaned dataset
├── starlink_decay_watchlist.csv            # Final 351-satellite watchlist
├── chart1_bstar_distribution.png
├── chart2_confound_reveal.png
├── chart3_top15_watchlist.png
└── README.md
```

---

*Data source: [CelesTrak](https://celestrak.org). This project uses publicly available orbital data and is for educational/portfolio purposes.*

# Lab 4 — Written analysis: three traffic profiles

All numbers below come from my own runs (`docs/runs/*.txt|json`): 400 requests per
profile, the API restarted between runs so the monitoring window starts empty.
Screenshots of the Model Behaviour dashboard for each run are in `docs/screenshots/`.

| Profile | Score mean | REVIEW | DECLINE | Drift score (max PSI) | Fairness gap |
|---|---|---|---|---|---|
| normal | 0.2355 | 14.2% | 8.0% | 0.030 stable | 0.019 |
| drifted 0.05 | 0.2437 | 14.0% | 8.5% | 0.189 moderate | 0.015 |
| drifted 0.15 | 0.2609 | 16.2% | 9.8% | 1.152 significant | 0.036 |
| drifted 1.0 | 0.5897 | 29.8% | 54.8% | 4.251 significant | 0.038 |
| unfair | 0.3966 | 15.8% | 32.0% | 0.353 significant | 0.722 |

Per-feature PSI (the diagnosis, not just the alarm):

| Feature | normal | drift 0.05 | drift 0.15 | drift 1.0 | unfair |
|---|---|---|---|---|---|
| LIMIT_BAL | 0.015 | 0.051 | 0.057 | 2.043 | 0.015 |
| AGE | 0.030 | 0.037 | 0.045 | 1.708 | 0.030 |
| PAY_0 | 0.006 | 0.006 | 0.006 | 0.826 | 0.279 |
| utilisation_ratio | 0.029 | 0.067 | 0.261 | 3.092 | 0.029 |
| payment_ratio | 0.013 | **0.189** | **1.152** | **4.251** | 0.288 |
| max_delay | 0.004 | 0.004 | 0.004 | 0.782 | **0.353** |

## 1. Which signal moved first, and by how much before anything else did?

Input drift. At strength 0.05 the drift score went from 0.030 to 0.189 — a 6x
increase that crosses into the moderate band — driven by `payment_ratio`
(0.013 → 0.189). Over the same run the output signals barely moved: the mean
score rose by 0.008 (0.2355 → 0.2437), REVIEW actually fell (14.2% → 14.0%) and
DECLINE rose by half a point (8.0% → 8.5%), well inside run-to-run noise. The
decision mix only becomes visibly different at strength 0.15 (DECLINE 9.8%), by
which point PSI is already 1.15, four times the "significant" threshold.

So input drift led the output signals by at least one full step of the shift
magnitude. In production terms, if the population drifted gradually, PSI would
have flagged it while the business-visible decline rate still looked normal —
the weeks of "scoring the wrong population" the lab warns about.

Sensitivity is not uniform: `payment_ratio` and `utilisation_ratio` react to
small shifts, while `max_delay` and `PAY_0` stay at their baseline (0.004 /
0.006) until the full shift. A single global threshold is blunt; the per-feature
panel is what shows which feature is carrying the signal.

## 2. Which signal would have paged me, given my thresholds?

My rules (`monitoring/prometheus/alerts/ml_alerts.yml`):

| Rule | Condition | normal | drift 0.05 | drift 0.15 | drift 1.0 | unfair |
|---|---|---|---|---|---|---|
| ModerateFeatureDrift (warning) | drift > 0.10 for 15m | – | fires | fires | fires | fires |
| SignificantFeatureDrift (critical) | drift > 0.25 for 15m | – | – | **pages** | **pages** | **pages** |
| DecisionMixShift (warning) | DECLINE share > 20% for 30m | – | – | – | fires | fires |
| FairnessGapWidened (critical) | gap > 0.10 for 20m | – | – | – | – | **pages** |

(Assuming the traffic persisted beyond each `for:` window; in the short lab runs
these were observed in the *pending* state in Prometheus.)

- **drift 0.05** would only produce a warning ticket — correctly, since nothing
  the business sees has changed yet.
- **drift 0.15** is the first run that pages (critical), purely on input drift,
  while DECLINE is still under 10%.
- **drift 1.0** pages on drift and additionally trips DecisionMixShift (54.8%
  declines vs an 8% baseline), which is the alert underwriting would care about.
- **unfair** pages on *both* SignificantFeatureDrift and FairnessGapWidened. The
  fairness page is the one that tells the on-call engineer this is not a routine
  retraining situation.

No alert fired on the `normal` profile (drift 0.030, gap 0.019), which is the
negative case that keeps the alerts credible.

## 3. Which signal told me *what* was wrong?

The aggregate drift score says *that* something changed, not *what* or *to
whom*. The `drifted` and `unfair` runs are the clearest illustration: drift at
strength ≈0.1 and the `unfair` run both land in the 0.2–0.4 range, and on the
drift-score stat panel they look alike. Three panels separate them:

1. **Selection rate by group / fairness gap.** In every drifted run both groups
   move together (full shift: group 1 = 86.9%, group 2 = 83.1%, gap 0.038). In
   the unfair run group 1 goes to 93.8% while group 2 stays at 21.6% — exactly
   its normal-traffic value — and the gap jumps from 0.019 to 0.722. The change
   was absorbed by one group only.
2. **PSI per feature.** Drift moves the demographic and balance features
   (LIMIT_BAL 2.04, AGE 1.71, utilisation_ratio 3.09). In the unfair run
   LIMIT_BAL, AGE and utilisation_ratio are *identical* to the normal run
   (0.015, 0.030, 0.029); only repayment behaviour moved (max_delay 0.353,
   PAY_0 0.279, payment_ratio 0.288). A different population looks different on
   every axis; a change confined to repayment history for one group is a
   different kind of event.
3. **Decision mix.** Unfair traffic raised DECLINE to 32% while REVIEW stayed at
   15.8% — the extra risk lands in outright declines for one group, not in more
   manual review for everyone.

**Conclusion.** Input drift is the earliest signal and the right thing to page
on for population change, but it is a reason to look, not a diagnosis. The
per-feature PSI panel says *which* inputs moved, and the selection-rate panel
says *who* absorbed the change. Without the fairness gap as a separate metric,
the unfair run would have been triaged as "moderate-to-significant drift,
consider retraining" — and retraining on that data would have baked the
disparity into the next model.

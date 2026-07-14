# From Starlight to a Trustworthy Classifier
### Written Report — India High School Exoplanet Data Challenge (Celesta)

**Dataset:** NASA Exoplanet Archive — Kepler Objects of Interest (KOI) Cumulative Table · DOI 10.26133/NEA4
**Task:** predict `koi_disposition` ∈ {CONFIRMED, CANDIDATE, FALSE POSITIVE} from transit & stellar measurements.

---

**The problem.** NASA's Kepler telescope recorded the brightness of ~200,000 stars, hunting for the tiny,
repeating dips that betray a planet crossing in front of its star — a *transit*. But most dips are impostors:
eclipsing binary stars, instrument noise, or blended background sources. Each of the 9,564 "Kepler Objects of
Interest" in our dataset has been sorted by experts into **CONFIRMED** (a real planet), **CANDIDATE** (a
real-looking signal not yet confirmed), or **FALSE POSITIVE** (an artifact). Our goal was to reproduce that expert
triage automatically from 140 columns of transit geometry and stellar properties.

**Exploring and cleaning the data.** The classes are imbalanced — false positives make up ~51% — so we judged
models by **macro-F1** rather than accuracy, which stops a lazy "always guess false positive" model from looking
good. We found 19 completely empty columns and a great deal of missing data, some of it revealing: only confirmed
planets are ever given a `kepler_name`, for instance. Exploratory plots surfaced physics we could rely on —
confirmed planets sit at high signal-to-noise with sensible radii, while false positives spread into low-SNR,
implausibly-large-"planet" territory (the signature of eclipsing binaries).

**The leakage audit — our most important step.** A dataset like this is full of traps that let a model "cheat" by
reading the answer instead of learning the physics. The organisers had already removed `koi_score` (a pre-computed
disposition confidence). We removed several more: `kepler_name` (a dead giveaway that a planet is confirmed),
`koi_pdisposition` (the Kepler pipeline's *own* verdict on the same question), and a batch of vetting and
provenance metadata recorded *after* the disposition was decided. We then treated the four Robovetter
false-positive flags (`koi_fpflag_*`) as a special, **toggleable** group so we could *measure* their influence
rather than let them quietly dominate the result. Being disciplined here is the difference between a model that
merely scores well and one a scientist would actually trust.

**Feature engineering.** Raw columns describe the transit; we added features describing whether that transit is
*physically consistent with a real planet*. We converted ~130 discarded uncertainty columns into a handful of
**fractional-uncertainty** features (a poorly-measured signal is a suspicious signal), **log-scaled** the
quantities that span many orders of magnitude (period, depth, insolation, radius, SNR), and built
**physics-consistency ratios** — most notably `depth_consistency`, which compares the observed dimming to the
dimming implied by the fitted planet-to-star radius ratio and flags the tell-tale mismatch of an eclipsing binary.
SHAP later confirmed these engineered features earned their place: `depth_consistency`, `mes_ses_ratio`,
`duty_cycle` and several fractional-error features all rank among the top contributors.

**Modelling and results.** We compared Logistic Regression, Random Forest, HistGradientBoosting and XGBoost under
an identical stratified 80/20 split, then tuned XGBoost with a 40-configuration randomized search (which confirmed
our configuration was already near-optimal) and validated it with 5-fold cross-validation. **The tuned XGBoost led
with a cross-validated macro-F1 of 0.928 ± 0.007** (accuracy ≈ 0.947). The tight spread across folds shows the
result is robust, not the product of one lucky split. Reframed as the binary "real signal vs false positive"
question, the same model reaches macro-F1 ≈ 0.993 — showing that impostor rejection is nearly solved and the
residual difficulty lives entirely in the confirmation step.

**What the model learned — and its honest limits.** The confusion matrix tells a clean story: false positives are
caught almost perfectly (~99% recall), and nearly *all* remaining error is CANDIDATE-versus-CONFIRMED confusion.
Our with/without-flags experiment explains why. The diagnostic flags make false-positive rejection almost trivial,
but they — and every other photometric feature — are nearly useless for the CONFIRMED-vs-CANDIDATE split. That is
because the only real difference between a candidate and a confirmed planet is whether astronomers have *finished
the follow-up work*, which the light curve simply does not encode. Rather than bury this behind a single accuracy
number, our model makes it explicit: false-positive rejection is nearly solved, while confirmation status is an
*irreducible* uncertainty from photometry alone.

**Explaining it to a non-expert.** Picture the model asking three questions of every dip in starlight: *Does it
even look like a planet passing by? Is its size and shape physically believable? And is the signal strong and
repeated enough to trust?* Confident "no"s become false positives; confident "yes"s become planets; the genuine
"maybe"s are the candidates — the very signals human astronomers are still working to confirm. In short, the model
learned real astrophysics, delivers a strong and stable score, and is honest about the one boundary that no amount
of starlight can settle on its own.

*(≈ 790 words)*

# Calibrated, and useless where it mattered

**Where:** prospect projection, Stage 1, P(reaches NHL)
**Shipped in:** the first calibrated version of the head
**Fixed in:** v11 (`AgeBandCalibrator`)
**Found by:** looking at the board, not at the metrics. The metrics were fine.

---

## Summary

I calibrated Stage 1 scores with a single global isotonic fit on a held-out carve.
Textbook, and by every aggregate measure it worked.

Training rows are **age snapshots**: one row per player per age, 17 through 25,
each describing career shape as of that age. Pooled across ages, the score
distribution is dominated by older snapshots, where the model has three to eight
years of evidence and separates cleanly. A 17-year-old snapshot has almost none of
that, so those rows sit in a narrow band of raw scores near the base rate. A
monotone fit over the pooled distribution flattens that whole band to a near
constant.

Result: **every draft-eligible player got the same probability, and it was low.**
Calibration had collapsed the exact cohort the product exists to rank onto the
population base rate.

## Impact

The board's main job is comparing players *inside* a draft class. That comparison
was flat. A generational 17-year-old and a marginal one came back with the same
number, both reading as longshots.

It also propagates. Stage 1 feeds an acceptance threshold near 0.33; a cohort
squashed under that threshold does not just rank badly, it fails the gate, and the
failure carries into the tier head and every product label built on it.

The population hit hardest is the one worth the most: players not yet drafted,
where a projection is most valuable because least is known.

## Detection

Nothing flagged it. That is the whole story of this one.

- **ROC-AUC was 0.977.** AUC is a rank statistic over the whole evaluation set, and
  ranking *across* ages is nearly free: a 22-year-old with a pro season obviously
  beats a 17-year-old, and the model gets that right every time. 0.977 overall is
  perfectly compatible with zero discrimination inside the 17-year-old band.
- **Calibration curves looked good.** Plotted over the pooled population, which is
  the population the calibrator was fitted to. Circular by construction.
- **Held-out loss looked good**, for the same reason.

I found it by opening the board for a draft class and seeing the same number
next to everybody.

## Root cause

A mismatch between the population the calibrator was **fitted on** and the
population a user **compares within**.

Isotonic regression fits a monotone step function to the pooled scores. Where
scores are dense and outcomes are mixed, which is exactly where low-information
snapshots live, the fit flattens to the local base rate. That is correct behaviour: those
scores genuinely say little about the *absolute* probability.

But the product almost never asks an absolute question. It asks a conditional one:
*of this year's draft-eligible players, who makes it?* The raw model's ordering
inside the 17-year-old band was real, if weakly separated. The calibrator threw it
away, because a monotone fit made globally cannot preserve an ordering it has
already compressed into one step.

**Global calibration guarantees nothing about any subgroup.** A model can be
perfectly calibrated overall and wrong on every partition of the data. If the
product compares within a subgroup, calibration has to be fitted at that
granularity or it is measuring a question nobody asked.

## Fix

I built `AgeBandCalibrator`, one isotonic fit per as-of-age band:

| Band | Why |
|---|---|
| 17 | Draft-eligible. Lowest information, highest product value. |
| 18–19 | Draft year and the year after |
| 20–21 | Development years, first pro exposure |
| 22–25 | Settled outcomes, high information |

A band with too few positives to support its own fit falls back to the global one,
so the fix cannot turn a sparse band into noise. Within-band ordering survives
because each fit spans that band's own score range, not a compressed slice of a
pooled one.

I built the Stage-1b durability head with the same banded calibration from the
start, so I did not have to rediscover this.

## What changed structurally

- **Calibration is fitted at the granularity the product compares at.** Here that
  is the age band, because every user-facing comparison is within a cohort.
- **Metrics are reported per class, never only in aggregate.** Same discipline I
  run on the retrieval eval. Its query classes exist for this reason, and they
  caught a fusion setting that improved the average while wrecking one class
  ([retrieval](../retrieval/README.md#why-hybrid-and-why-both-legs-stay-load-bearing)).
- **Subgroup calibration is measured routinely**, and not only by age: every fresh
  test fold reports whether the model over- or under-values each development path
  (CHL, NCAA, Euro pro, AHL). A systematic bias on one path is this same bug in
  different clothes, and I found one that way.
- **Selection runs on grouped CV keyed by player id**, each seed's test fold scored
  exactly once by an already-chosen model. Selection never touches a test fold.

What I missed: I reviewed the calibrator as a finishing step, not as part of the
model. It changed behaviour on the most important cohort more than any feature.

## What to do differently

**Decide what the model is for before picking a metric.** AUC answers "can it rank
any two players?" The product asks "can it rank two players of the same age?"
Different questions; only one was ever measured. One line, AUC within each age
band, would have made this obvious from the first calibrated version.

**A calibrator is part of the model, not post-processing.** It got reviewed as a
finishing step, and finishing steps get less scrutiny than architecture. It changed
behaviour on the most important cohort more than any feature ever has.

**Read the board for a real cohort, every version.** Both bugs written up here were
found by looking at plausible numbers for known players; neither was caught by a
metric. That is now part of release: a version is not accepted until I have read
its board for a real draft class.

---

## Related

- [Bug: a label in disguise](label-leakage.md): the other failure on this head,
  also found by eye
- [How it was measured](../benchmarks/README.md): the evaluation protocol as it stands
- [ADR 0004](../decisions/0004-age-snapshot-training.md): why the rows are age snapshots

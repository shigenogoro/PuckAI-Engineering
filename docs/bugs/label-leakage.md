# A missing column that meant two things

**Where:** prospect projection, Stage 1 — P(reaches NHL)
**Shipped in:** the version that wired the Central Scouting rank feature live
**Fixed in:** v19 (`v19_2026-07-15`)
**Found by:** looking at the retro view, one version after an unrelated change
stopped hiding it

---

## Summary

I shipped Stage 1 with NHL Central Scouting rank as a feature. CSS lists only exist
from 2008, so every player drafted before that has no rank. Across the training set a
missing rank was ~96% a non-NHLer, and XGBoost routes missing values down a
learned branch, so it learned the rule the data actually supported:
**`css_rank` NaN → longshot.**

The feature had become a proxy for the label. Every pre-2008 player was penalised
for the calendar.

## Impact

**Sidney Crosby — first overall, Hall of Fame — read 0.59 P(NHL)** off his junior
record. Not because his junior record is ambiguous. Because he was drafted in 2005
and the lists start in 2008.

Three ways it hurt, in order of how much:

- The retro view under-graded every historical player. That view is the one a user
  opens to decide whether to trust the model at all.
- The training signal was polluted for the whole model, not just the affected
  rows. A feature that resolves 96% of one class in a single branch takes weight
  away from the production features that actually generalise.
- Held-out metrics looked *better* than they deserved to, because the test folds
  inherit the same era structure.

That last one is the point. Leakage does not show up as a regression. It shows up
as a number that is too good, on a split that shares the leak.

## Timeline

| Version | What happened |
|---|---|
| v11 | Rank feature wired end to end, deliberately a no-op — always NaN, XGBoost-routed — pending the scrape |
| v12 | Scrape lands, rank merges into `prospect_bio`. Feature goes live. Bug ships here. |
| v17 | Matcher fix takes CSS coverage 3,046 → 6,150 players (the old matcher gated on `draft_year`, discarding overagers and all 3,252 ranked-but-never-drafted players). Changes the missingness distribution; does not fix its ambiguity. |
| v18 | Unrelated: `p_nhl` becomes honest for everyone instead of being forced to 1.0 for anyone who reached the NHL. **The mask comes off.** |
| v19 | Crosby at 0.59 gets noticed. Root cause found, fixed. |

## Detection

It stayed invisible for six versions for one reason: **`p_nhl` used to be hardcoded
to 1.0 for anyone who factually reached the NHL.** Every graduated player — every
player whose real outcome a human would recognise on sight — showed a perfect
score. The only population that could expose the bug was the one population whose
scores were being overwritten.

I removed the hardcode in v18 for a product reason, not a debugging one: the retro
view exists to answer "how would this model have graded Crosby from junior alone?",
and forcing the answer to 1.0 throws that away. One version later the leak was
visible to the naked eye.

No automated check caught this. I saw a number that could not be right.

## Root cause

One column, two incompatible meanings of missing:

1. **CSS published no list for this player's era** — a coverage artifact, says
   nothing about the player.
2. **CSS published a list and left him off it** — a real, genuinely weak signal.

XGBoost cannot tell them apart, so it learned the blend, and the blend is dominated
by meaning 1, which is era, which correlates with the label.

Underneath that: I wired the feature as a no-op and then switched it on by a *data*
change rather than a *code* change. Nothing in that path forced me to ask what
its missingness would mean once real values arrived.

## Fix

I split the two meanings so the model can learn them apart:

| Signal | How it is represented |
|---|---|
| Was this era covered by CSS at all? | `css_covered` flag |
| Covered, left off the list | Sentinel rank — "scouted and passed over" is real, weak, negative |
| Not covered | Genuine NaN, no information |

The model now sees coverage as its own fact and can no longer confuse *we have no
data* with *the scouts were unimpressed*.

## Three more from the same sweep

Auditing for the pattern turned up others. All the same shape: something about the
*observation process* leaking into features meant to describe the *player*.

**Senior tournaments counted as development seasons.** The World Cup and 4 Nations
were being folded in as development-league seasons. Only established NHL stars play
in them. A "development seasons" feature that includes an event restricted to the
positive class is a label wearing a hat. I added them to the excluded-tournament list.

**Birth month inside a monotone constraint.** Age-level features come from a
continuous, Sept-15-anchored season age; monotone constraints operate on the age
cohort. Birth month was shifting a monotonically constrained feature *within* one
draft cohort — an effect with no hockey meaning whatsoever. Age-level features now
floor to the cohort `age_int`.

**A filename deciding a player's features.** The best one. Extraction deleted every
league row after a positive's NHL debut season — 108,184 rows across 7,736 players,
including 16,259 AHL/ECHL seasons of ≥ 10 GP. A prospect called up once and now
playing his first full AHL year had that year erased. Another prospect in the
identical situation kept his, purely because he was scraped into the prospect file
rather than the NHL one. Same player type, different features, decided by a
filename.

It also biased the whole model, not just the truncated players: the peer-percentile
population lost 16k AHL seasons belonging to precisely the *strongest* AHLers — the
ones who had touched the NHL — so every prospect's AHL percentile came out
inflated.

I fixed it by keeping the rows and flagging them `post_debut`: they feed the percentile
population and the serving snapshot, and are held out of that player's own training
features by an explicit parameter. That flag, not an age cap, is what enforces the
training window — the old ceiling would have admitted the debut+1 season anyway for
anyone not born exactly on September 15.

## What changed structurally

- **An invariant, written into the pipeline docs:** a column whose missing values
  carry two meanings will be learned as a label proxy. Splitting them is
  mandatory, not a refinement.
- **Probabilities stay honest.** Outcome facts live in flags (`graduated`,
  `reached_nhl`, `has_nhl_rows`); a probability is never overwritten with a fact.
  Overwriting one is what destroyed the surface that would have caught this. The
  retro view is monitoring now, not a feature.
- **Training and serving windows are set by an explicit flag**, never by which
  scrape file a row arrived in.
- **A feature switched on by a data change is treated as a code change.** "What
  will missingness mean once this column is populated?" belongs in the switch-on,
  not the wiring.

What I missed: I did not ask what missingness would mean when the scrape landed, and
I had overwritten the one view that would have exposed the problem.

## What to do differently

**Grade the known outcomes first.** The most effective test in this whole system
turned out to be the cheapest one: score a handful of players everybody knows and
look at the numbers. That should have been an assertion from the first version
that produced a probability — a small fixture of unambiguous historical outcomes
with loose bounds. It would have failed the week the feature went live instead of
six versions later.

**Treat "too good" as a smell.** Held-out metrics do not see leakage when the split
shares it. Grouping folds by player id, which this pipeline does, stops row-level
leakage between folds and does nothing about a feature that encodes era. The check
that catches this class is distributional: for every feature, how well does its
missingness *alone* predict the label? At 96% that is not a subtle signal.

**Never hardcode a model output to a known fact.** It hid this for six versions.
Whatever the product needs from the fact, put it in a flag next to the probability.

---

## Related

- [Bug: calibrated, and useless](serving-time-calibration.md) — a different failure
  on the same head
- [ETL pipeline](../etl/README.md#invariants) — where the invariants live
- [ADR 0004](../decisions/0004-age-snapshot-training.md) — why training rows are age snapshots

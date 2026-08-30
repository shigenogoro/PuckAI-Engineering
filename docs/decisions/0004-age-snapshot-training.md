# 0004 — Age-snapshot training rows

**Status:** Accepted

## Context

My projection model has to grade a live 17-year-old and a finished career with the
same machinery. The obvious training row — "this player's whole pre-NHL career" —
cannot do that, because a training positive's career is cut at his NHL debut while a
live prospect's is cut at today. Those are different objects wearing the same shape.

Worse, the cut itself is informative. Elite prospects leave for the NHL mid-season,
so their final amateur season is short, and the age cells above their debut are
empty. A model handed that learns time-to-debut, which is a label.

## Decision

Every training row is a **snapshot as of age a**, for a ∈ 17…25: career shape up to
that age, and nothing after it. A training positive cut at his pre-debut window and
a live prospect cut at his current age then mean the same thing.

Features are NHLe-adjusted production and peer percentiles within
(league, season, position, age) cells — never raw points-per-game, which is not
comparable across leagues.

## Alternatives considered

**Career-to-date rows.** My original design. Leaked time-to-debut through
final-season length and age-cell missingness.

**One row per player, features at draft age only.** Throws away every later season,
which is most of the evidence for the late-breakout paths the product exists to
find.

**Raw production, no league adjustment.** A point in the OHL and a point in the SHL
are not the same point. Without translation the model mostly learns which leagues
feed the NHL, which it can already see more directly.

## Consequences

- A player contributes up to nine rows, so **every split must be grouped by player
  id**. Splitting by row is direct leakage, and this constraint propagates into
  every evaluation in the project ([how it was measured](../benchmarks/README.md)).
- Serving and training snapshots differ deliberately — post-debut minor-league
  seasons are kept for serving and percentiles, held out of a player's own training
  features by an explicit flag. Getting that boundary wrong caused a real bug
  ([label leakage](../bugs/label-leakage.md)).
- Snapshot rows are dominated by older ages, where evidence is richest. Any
  aggregate fitted over the pool inherits that skew — which is exactly what broke
  calibration ([calibrated, and useless](../bugs/serving-time-calibration.md)).

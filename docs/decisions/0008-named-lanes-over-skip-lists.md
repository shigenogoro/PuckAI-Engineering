# 0008: In-season lanes name their stages rather than skip others

**Status:** Accepted

## Context

The ETL has 14 stages. An in-season refresh needs four of them: new box scores in,
season and career shape recomputed, serving tables exported.

The original in-season path selected stages by **exclusion**: skip a set named
`_STABLE_STAGES`, run whatever remained. That set did not name the projection stage,
so every "quick refresh" retrained XGBoost from scratch: median 53 minutes, up to two
hours, to update last night's goals and assists.

Worse than the runtime, that stage re-draws its train/test split and model seeds on
every invocation. So an unattended nightly would move every published projection each
time it ran, with no change in inputs and nothing to point at as the cause.

## Decision

An in-season run selects stages by **enumeration**. `LANES` maps a surface name to
the exact ordered list of stages it runs, and `_TRAINING_STAGES` names the stages that
must never appear in one. A test asserts the two sets do not intersect.

```python
LANES = {"nhl": ["feature_engineering", "season_trajectory",
                 "unified_trajectory", "db_export"], ...}
_TRAINING_STAGES = frozenset({"provisional_tiers", "pre_nhl_projection"})
```

## Alternatives considered

**Extend the skip list.** The one-line fix: add the projection stage to
`_STABLE_STAGES`. It restores correct behaviour today and re-breaks the moment a
fifteenth stage is added, because a skip list is silent about anything it does not
mention. The failure is additive and invisible, exactly the shape that produced the
bug.

**Make retraining cheap enough to leave in.** Even at zero cost this is wrong while
the stage re-draws its seeds: the problem is not the minutes, it is that published
numbers move for no reason. Fixing the seeds instead would mean deciding that a
nightly is allowed to retrain a model, which I do not want to be true by default.

**Keep training in a separate workflow on a slower schedule.** This is what I do, and
it is complementary, not an alternative. It only became safe once the frequent
path provably contained no training stage.

## Consequences

- **Measured at 101 seconds** for the NHL lane. My estimate before measuring was 6–7
  minutes, taken from worst-case maxima in the logs.
- Retraining becomes a deliberate, separately triggered act, which in turn made it
  worth pinning the artifacts it writes by hash (see [automation](../ci-cd/README.md)).
- Adding a stage to the pipeline does not add it to any lane. That is the intended
  property, and it means a genuinely needed new stage must be added to each lane
  explicitly.
- A lane skips the cohort-assignment stage, so a player who debuts this season joins
  no cohort until the next full run. This was already true of the old path;
  enumeration made it explicit rather than incidental.
- Fixing this exposed an adjacent bug: a mistyped stage name computed "skip
  everything" and produced a successful-looking run that did no work. Stage names are
  now validated and a typo raises. Green and empty looked exactly like green and
  correct, which is why the guard tests exist.

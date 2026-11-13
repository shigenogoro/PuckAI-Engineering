# 0007: Rookies stay on the prospect board

**Status:** Accepted

## Context

An NHL tier is a claim about a career. Plenty of players in my database do not have
enough career for the claim to mean anything: a 20-year-old with one 12-game call-up
has an NHL sample that says nothing, and an amateur record that says a great deal.

Grading everyone on the same scale is how a board fills with confident nonsense.

## Decision

`filtering` splits players into four cohorts, carved in a fixed order, and each
cohort is graded a different way:

| Order | Cohort | Rule | Graded |
|---|---|---|---|
| 1 | Fringe | `total_gp < 40`, `max_gp_per_season < 25`, `seasons_over_6_gp ≤ 2`, `age > 26` | Assigned, its own class |
| 2 | Rookie | `seasons_over_6_gp < 2`, `max_gp_per_season ≤ 25`, `age ≤ 26` | **Not tiered**, stays on the prospect board |
| 3 | Provisional | `25 < total_gp < 164`, `age ≤ 26` | Predicted from an early-career window |
| 4 | Stable | Everything left | Assigned from realised career value |

The prospect engine owns a player until his NHL sample is worth more than his
amateur record. A provisional player who has left the NHL/AHL system is folded back
into stable: the outcome is known, so there is nothing left to predict.

## Alternatives considered

**One tier scale for every NHL player.** Small samples produce extreme rates, so a
rate-driven tier on 12 games is noise presented as a grade.

**Show nothing until a player is tiered.** Worse for the user, and false: the
projection is real information, and for a rookie it is the *best* information
available.

**Force a provisional player's tier to his projection.** Conflates two different
claims: what an amateur record predicted, and what an early NHL sample shows.
Stage 10 predicts from the NHL window instead, on the same features and the same
scale as the assigned tiers.

## Consequences

- Four badge states on a profile (`tiered`, `provisional`, `rookie`, `unknown`),
  and the API serves whichever the pipeline assigned rather than re-deriving it from
  games played. Two copies of that logic would drift.
- Provisional players under the 60-GP threshold stage 10 requires are deferred, not
  guessed at, and keep the `Provisional` badge until they cross it.
- Aspect scores for predicted and assigned tiers are min-max scaled against the same
  stable reference, which is the only reason it is safe to put them on one radar.
- Age gates come *after* production gates in the carve order. A 26-year-old with two
  games is a rookie; a 30-year-old with the same line is fringe. Same stat line,
  different meaning, and only the sequence separates them. So the order is part of
  the decision, not an implementation detail.

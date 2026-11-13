# 0005: A deterministic validator, not an LLM judge

**Status:** Accepted

## Context

I ship 7,643 generated reports next to real projections. Every factual claim in them
has to be traceable to a pipeline value. Something has to check that, at a scale no
human can review.

I tried prompt-level grounding rules first and they did not hold: across three
generation rounds on a 5-player sample the model still invented an age, a trophy
count, a national ranking, and tracking-data vocabulary the system has no source
for.

## Decision

I built a deterministic checker with no model in it. Six mechanical checks (numbers,
proper nouns, ordinal phrases, banned vocabulary, season-scoped age, award-event
co-occurrence), each tracing a claim back to the player's context JSON. Fail once,
regenerate; fail twice, quarantine.

## Alternatives considered

**Prompt rules alone.** Already measured. A prompt is a request, not a guarantee.

**An LLM judge.** I rejected this on four counts. It is non-deterministic, so the same
report can pass and fail; it costs a second model call on every report across 7,670
attempts; it cannot be debugged, so a false positive has no fix beyond rewording the
judge prompt; and it fails in exactly the same direction as the generator: a model
that would write "his second Norris" is not reliably the model that catches it.

**Human review.** Correct and impossible at this volume. It survives as a 2%
spot-audit for the semantic class the deterministic checks cannot reach.

## Consequences

- The checks are only as good as their rules, and the number check in particular is
  a presence check rather than a semantic one. Limits are documented in
  [validation](../validation/README.md#known-limits) rather than left implied.
- False positives were the real engineering cost. A checker that fires on correct
  reports teaches the generator to delete true statements, so I narrowed several rules
  (whole-word matching, decimal-aware age parsing, phrase-level ordinals), each
  recorded at the rule it fixed.
- The proper-noun allowlist is hand-maintained and will keep needing additions.
- Measured: 27 rejections in 7,670 attempts, 0 quarantined after retry.

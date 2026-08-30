# 0002 — Weighted reciprocal rank fusion rather than score blending

**Status:** Accepted

## Context

Two retrieval legs return two ranked lists that must become one. The dense leg
produces cosine distances; the lexical leg produces a text-rank score. The two
live on unrelated scales, and neither has a stable range across queries.

## Decision

I fuse on **rank**, using reciprocal rank fusion, with the dense leg weighted at 2.0
and the lexical leg at 1.0. I reserve a floor of slots in the rerank pool for the
top lexical hits.

## Alternatives considered

**Normalise scores and blend numerically.** Requires inventing a conversion
between two incomparable quantities, and any per-query normalisation is dominated
by the query's score spread rather than by result quality. Rank fusion sidesteps
the problem entirely rather than approximating a solution to it.

**Unweighted reciprocal rank fusion.** This was my first implementation and it was
worse than doing nothing: dense beats lexical by roughly two to three times on
archetypes, while both legs already saturate name recall, so equal weighting
dragged hybrid *below* dense-only on the class where dense was carrying the
system. Weighting is not a tuning refinement here; it is what makes fusion
correct.

**Weight learned per query.** I rejected this as unjustifiable at this corpus and query
volume. It adds a model, and a fitted weight on 77 labelled queries would be
fitting noise.

## Consequences

- The dense weight is flat across a 2.0 to 5.0 sweep, which is the reassuring
  shape: the result is not balanced on a knife edge.
- Weighting dense at 2.0 introduced a new failure — on a query naming two players,
  the second name could be pushed past the rerank pool boundary and never scored
  by the cross-encoder. A lexical floor in the pool fixes it, and moved multi-name
  recall@5 from 0.833 to 1.000.
- Two constants now need justification rather than one. Both are recorded with the
  measurement that set them.

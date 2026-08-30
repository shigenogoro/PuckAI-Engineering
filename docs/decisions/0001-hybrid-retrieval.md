# 0001 — Hybrid retrieval rather than dense-only

**Status:** Accepted

## Context

PuckAI Scout answers questions over a corpus of scouting reports I generated. Two
question shapes dominate real usage and they are nothing alike:

- **Archetype** — "physical two-way defenceman with a heavy shot". Shared meaning,
  almost no shared keywords with the documents that satisfy it.
- **Name** — "Macklin Celebrini". Exact match is the entire task.

A dense retriever handles the first and smears the second across similarly-named
prospects. A lexical retriever does the reverse, and on archetype queries it fails
close to completely.

## Decision

I run both legs on every query and fuse the results, then rerank with a
cross-encoder. I report evaluation metrics per query class so a single average can
never hide one leg's collapse.

## Alternatives considered

**Dense only.** Simpler, one index. I measured it at 0.954 NDCG@10 on names against
lexical's perfect 1.000 — and name questions are the most common thing users ask.
Losing exact-match reliability on the highest-frequency query class to simplify
the stack is a bad trade.

**Lexical only.** I measured it at 0.051 NDCG@10 on archetypes. Not viable.

**Route between legs with a classifier.** I rejected this: it converts a fusion problem
into a routing problem, and a routing mistake costs the whole answer rather than a
few ranks. It also adds a model on the request path for a decision that fusion
makes for free.

## Consequences

- Two indexes to maintain and keep in sync with the corpus.
- Both legs are load-bearing, so neither can be degraded casually.
- Class-separated evaluation becomes mandatory rather than a nicety — and it
  immediately earned its keep by catching a fusion configuration that improved
  the average while scoring below dense-only on archetypes ([ADR 0002](0002-weighted-rrf.md)).
- Measured: hybrid with rerank reaches 0.582 NDCG@10 against 0.508 for dense-only
  and 0.488 for lexical-only.

# 0003 — Two text-search vectors, not one

**Status:** Accepted

## Context

The lexical leg has to serve two query shapes out of the same documents: prose
("physical two-way defenceman"), which wants stemming, and player names, which
absolutely do not. English stemming turns `Louis` into `loui`, and a surname that
has been stemmed is a surname that no longer matches.

## Decision

I use two persisted tsvector columns. `report_tsv` on the `english` configuration for
prose, `name_tsv` on `simple` and unstemmed for names. Names are scored separately
with `FTS_NAME_BOOST=3.0`. I also rewrite `websearch_to_tsquery`'s `&` to `|`.

## Alternatives considered

**One `english` vector over everything.** Stems surnames. Fails the most common
query class on the site.

**One `simple` vector over everything.** No stemming at all, so archetype prose
stops matching on anything but exact word forms.

**One tsquery OR-ing both configurations.** I tried this. Node count explodes — ~952 ms
against ~134 ms for the split version. A 7× latency cost on the request path to
avoid a second column.

**Keep the AND semantics of `websearch_to_tsquery`.** Not really an alternative
once measured: ANDing every term in a natural-language question drove archetype
recall to approximately zero, because no document contains all of a long question's
words.

## Consequences

- Two indexes to maintain, both persisted columns rather than expression indexes,
  so they are computed at write time and the read path stays cheap.
- The name boost is a third constant needing justification.
- A long question can still drown the name signal by sheer term count, which is
  what the rerank pool's lexical floor exists to catch ([ADR 0002](0002-weighted-rrf.md)).
- Measured: multi_name Recall@5 0.833 → 1.000.

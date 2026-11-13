# 0006: Explicit alias maps, not fuzzy matching

**Status:** Accepted

## Context

I have to bridge club names from a dozen league sources into a canonical club
vocabulary. Every source writes its own short names, and they collide in ways that
look tractable until you check the output.

`Lev Pp` is Lev Poprad. `Lev Pr` is Lev Praha. `DEK Klagenfurt` fuzzy-matches
`EC Salzburg`, a different club in a different city.

## Decision

I use an explicit alias map per source. No fuzzy pass. Every run prints the names that did
**not** join. The fix for an unjoined name is to add an alias, never to loosen the
threshold. A club with no canonical row stays unmapped.

The same principle governs league logos, which I hand-curate rather than scrape:
every heuristic I tried returns the wrong image *somewhere without saying so*. One
league site serves the same club crest on its WHL, OHL, and QMJHL pages, and an
encyclopedia API answers "Ontario Hockey League" with a photograph of an All-Star
game. With a dozen leagues covering most of the board, verifying each URL by hand is
cheaper than a scraper that is confidently wrong.

## Alternatives considered

**Fuzzy string matching with a threshold.** The failure mode is not "no match", it
is a *confident wrong match* that never surfaces. A wrong club silently reassigns a
player's season.

**An LLM to resolve names.** Same failure mode, worse latency, and non-deterministic
across runs of the same pipeline.

**Fuzzy matching plus a manual review queue.** This is the alias map with extra
steps, and the queue is where the actual work lives.

## Consequences

- Adding a league is manual work, and that is priced in.
- Coverage gaps are visible, not papered over. The source catalogue records
  the kind of each source, its client code, and the ones that are *not* reachable
  with the reason (no usable key published; a CDN challenge in front of the page).
- Related, and the same class of bug: one feed dedupes by club **name** rather than
  team id, so a franchise that has relocated keeps its id across every move. Keying
  on the id silently drops every name but the first. That is why I walk every
  season, not only the current one, and how folded clubs keep their crests.

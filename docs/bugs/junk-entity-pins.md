# Players nobody asked about

**Where:** PuckAI Scout: entity pinning, context assembly, answer prompt
**Shipped in:** fuzzy bare-surname matching was older; several of the other paths
landed with the planner rewrite
**Fixed in:** the pin-hardening pass (`PROMPT_VERSION` → 5)
**Found by:** a reported chat answer that named players nobody asked about, and an
answer harness that already existed before the fix

---

## Summary

One reported symptom, an answer discussing players the question never named, turned
out to be seven separate root causes stacked on the same turn. Four of them only showed
up because the measurement harness was already in place before I fixed anything.

The loudest was a bare-surname fuzzy match that treated ordinary English as a
player: "someone" → Simeone, "knights" → Corban Knight, "roster" → Foster,
"drives" → Dries. Measured rate on that path alone: **5 junk pins per 1 good**.

---

## Impact

Context blocks arrived for players the user did not ask about. The model then
wrote about them, because every dossier in the prompt looked the same and nothing
told it which ones were padding.

A concrete failure shape: a question about Draisaitl, Celebrini, and Kucherov
could also pull Spencer Knight (planner-fused name containing "Knight"), Corban
Knight (fuzzy "knights"), and whoever rerank order padded in to fill *k*. The
answer mixed asked-about players with junk.

---

## Timeline

| Stage | What happened |
|---|---|
| Pre-planner | Bare-surname fuzzy match already live against the full surname index |
| Planner rewrite | Intent, rewrite, and name proposals land; fused planner names and first-turn rewrites start feeding the resolver |
| Answer harness | Checks for unasked mentions and ungrounded names ship before the fix |
| Report | A chat answer surfaces irrelevant players |
| Pin-hardening | Seven causes fixed; neighbour fill replaces corpus-wide padding; dossier labels + prompt bump |

---

## Detection

The harness flagged **unasked mentions** and **ungrounded names** on saved cases.
That is why causes 2 to 5 never needed a second round of user reports. They were
already failing checks once I looked. Without that measurement, I would have fixed the
fuzzy matcher, re-tested the one reported question, and shipped the rest.

---

## Root cause

Seven independent mechanisms, same symptom:

| # | Cause | What it did |
|---|---|---|
| 1 | Fuzzy bare-surname match | Any 6+ letter word fuzzy-matched against 7,643 surnames |
| 2 | Ordinary-word exact surnames | "strong" → Cam Strong, "carrier" → William Carrier, even after fuzzy was gone |
| 3 | Planner-fused names | `'Mason Primeau Celebrini'` resolved as a string and pulled Primeau out of the middle, the same shape as "Spencer Knight Celebrini" producing a Spencer Knight dossier |
| 4 | First-turn rewrite injection | Planner rewrote `standalone_query` to inject a prior-turn player (e.g. Connor Bedard) when there was no pronoun to resolve |
| 5 | Compare *k* floor | Floor of *k*=2 when one player resolved meant a second doc from rerank order, usually a namesake (Brad→Charlie Marchand, Matt→Will Cullen) |
| 6 | Open-question padding | Remaining slots filled to `K_MAX=8` from rerank order over the whole corpus |
| 7 | Undifferentiated dossiers | Every context block looked identical, so the model had no reason not to discuss padding |

---

## Fix

| # | Fix |
|---|---|
| 1 | A bare surname must match **exactly** |
| 2 | Drop a one-word pin when the planner read the sentence and named nobody |
| 3 | A planner name must identify **one** corpus player whole; a fused multi-token string is not resolved |
| 4 | A first turn has no pronoun to resolve, so retrieval and name resolution use the user's actual words |
| 5 | Floor *k* only when nothing resolved |
| 6 | Pad open questions with `neighbours_of()`: pgvector KNN on the named players' own report vectors, no API call |
| 7 | Label blocks `(asked about)` / `(similar profile)`; prompt forbids explaining away an unused dossier; `PROMPT_VERSION` → 5 |

Neighbour cosine cut is **0.25**, measured not guessed: real alternatives cluster
≤ 0.220; Corban Knight sits at 0.323. My original 0.45 placeholder would have let
him back in.

### Two corrections to my own plan

I proposed exempting pins from the adaptive trim. The data showed the opposite:
every pin that reached that trim was junk, and the trim was the only thing keeping
William Carrier out. Dropped.

`--dry-run` on the answer harness was already broken: its stub lacked the intent
kwarg, so every case died into a swallowed `TypeError`. Fixed as part of making
the before/after numbers trustworthy.

---

## What changed structurally

Pinning is no longer "any token that looks like a surname." Exact bare surnames,
planner-empty one-word drops, whole-player planner names, and first-turn rewrite
discipline sit in front of retrieval. Open-question padding is neighbourhood
search around resolved players, not a second pass over the whole corpus. The
prompt finally distinguishes asked-about dossiers from similar-profile padding.

---

## What to do differently

**Build the check before the fix.** Four of seven causes were invisible to the
original report and obvious to the harness. Shipping the matcher fix alone would
have left fused names, first-turn rewrites, and the compare floor intact.

**Measure similarity cuts on real distractors.** A placeholder of 0.45 looked
safe and would have re-admitted Corban Knight. The cut that separates real
alternatives (≤ 0.220) from him (0.323) is 0.25.

**Do not protect pins from the trim that catches them.** The trim was not
over-eager; the pins were wrong.

**Keep dry-run on the same signature as production.** A stub that silently
`TypeError`s is worse than no dry-run, because it produces an empty "passing" run.

---

## Frontend companion: "No language model" flash

A second reported bug on the same surface. The providers list was `[]` both while
loading and when none exist, so the picker flashed "No language model" on every
thread switch. It is now `ChatProvider[] | null`, with a module-level cache read
synchronously by `useState` so a switch renders the picker instantly, and the
chosen model persists in `localStorage`.

---

## Measured result

132 questions, before and after this pass. Both runs are scored with the current
checks, which added coverage of every named player. (An older version of the checks
read 0.750 to 0.879 on the same runs.)

| | Before | After |
|---|---:|---:|
| Overall clean | 0.667 | **0.795** |
| Answers about a player nobody asked about | 9 | **0** |
| Ungrounded names | 20 | 7 |

| Slice | Before | After |
|---|---:|---:|
| general | 0.46 | 0.85 |
| rank_team | 0.56 | 1.00 |
| compare | 0.83 | 1.00 |
| stat_comps | 0.58 | 0.75 |
| archetype | 0.45 | 0.55 |
| ambiguous_name | 0.64 | 0.55 |

One slice got worse: ambiguous names went from 0.64 to 0.55, and that is still open.
A blind pairwise judge on `general` returned 7 wins, 6 ties, and 0 losses. The
reported question now retrieves only Draisaitl, Celebrini, and Kucherov.

---

## Related

- [Retrieval and grounded answers](../retrieval/README.md): planner rewrite and
  both answer-eval tables
- [How it was measured](../benchmarks/README.md#grounded-answers)

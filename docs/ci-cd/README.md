# Automation

The offline half of PuckAI used to be something I ran by hand. This is how I moved
the parts that should be unattended onto a schedule, and how I made an unattended run
something I am willing to point at a production database. That second part was more
of the work than I expected.

Three things run on GitHub Actions, and they are not all "CI":

```
push / PR       → tests                                  gate
merge to main   → Vercel redeploys frontend + backend    delivery
02:07 New York  → scrape → 4 ETL stages → upsert         data refresh
```

The third is the interesting one. It ships no code. It keeps *data* fresh on a timer,
which is a different risk: nobody is watching it, and it writes to the database the
product reads.

---

## What I automated, and what I left alone

| Job | Trigger | Runtime | Writes to |
|---|---|---|---|
| Logic tests | push, PR | ~1m20s–2m | nothing |
| Frozen-artifact pins | PR touching data paths | ~1m10s | nothing |
| NHL nightly refresh | 02:07 New York time + manual | ~3m20s | Postgres (staging by default) |
| Frontend / API deploy | merge to `main` | — | Vercel |
| **Model retraining** | manual, my machine | ~53 min median | frozen CSVs |
| **Prospect refresh** | not live | ~5.5 min | — |
| **Bot-protected source scrape** | never automated | — | — |

The bottom three are decisions.

**Retraining never runs unattended.** The projection stage retrains XGBoost from
scratch and re-draws its split and model seeds each time. Put that on a cron and
every published projection moves nightly for no reason anyone could explain. It stays
a deliberate act.

**One prospect source cannot run on a hosted runner.** It sits behind a bot wall and
only works in a real desktop browser session. I decided not to engineer around that.
It is also why I started moving prospect data to sources a runner can reach, a
migration that is paused for now.

**The prospect lane is built but not switched on.** It cannot go live until its
negative class comes from a source the nightly can reach *and* has been deduplicated
through identity resolution. Turning it on earlier would put the same player on the board
twice.

---

## Naming the stages, instead of skipping some

The pipeline has 14 stages. The nightly needs four of them.

The original in-season path worked by exclusion: skip a set of "stable" stages, run
the rest. That set did not include the projection stage, so the "quick refresh" was
retraining a model. Median 53 minutes, up to two hours, to update last night's box
scores.

I replaced exclusion with enumeration:

```python
LANES = {
    "nhl": ["feature_engineering", "season_trajectory",
            "unified_trajectory", "db_export"],
    "prospect": [...],
}
_TRAINING_STAGES = frozenset({"provisional_tiers", "pre_nhl_projection"})
```

A lane names what it runs, so it cannot silently acquire a stage when one is added to
the pipeline. A test asserts no lane intersects `_TRAINING_STAGES`.

**The lane runs in 101 seconds.** My own estimate before measuring was 6–7 minutes,
taken from worst-case maxima in the logs.

Fixing this surfaced a second bug in the same code: a mistyped stage name computed
"skip everything" and produced a **successful-looking run that did no work**. A typo
now raises. That failure mode is why I now write guard tests at all. Green and empty
looks exactly like green and correct.

The honest cost: the lane skips the cohort-assignment stage, so a player who debuts
this season joins no cohort until the next full run. That was already true; naming the
stages made it explicit rather than incidental.

---

## Cloning 180 MB instead of 1.4 GB

The repo tracks 1.4 GB of hockey data across 11,892 files. A default checkout would
dominate every run.

The PR gate takes **no data at all** (a blobless partial clone plus a sparse checkout
of source directories), and the data-dependent tests skip themselves, which the test
suite already did by design. In CI that currently reads as 164 passed and 35
skipped, and the skips are the point.

The nightly does need data, so I measured which. I ran the lane with file IO
instrumented and recorded what it actually opened: **57 MB of the 1.4 GB**. The
checkout is ~180 MB with git overhead.

That produced two lists that have to agree: the paths the workflow checks out, and
the paths the lane reads. Lists that must agree eventually don't, so a test compares
them.

---

## Pinning the files a scheduled job reads but never writes

Four artifacts are read by the in-season lanes and written only by stages those lanes
skip: the learned league-equivalency factors, the served projections, and two cohort
lists.

Nothing stopped me running a full pipeline locally and silently replacing all four
with different numbers, because the projection stage re-draws its seeds, so the values move
even when the inputs don't. Every published projection would shift, in a diff nobody
would read closely.

So each has a recorded SHA-256. The check **warns on my machine**, where I am often
mid-experiment, and is **fatal in CI**. A legitimate change has to arrive with a
re-stamp, which puts the new hashes in the same diff as the new files.

One detail I got wrong first: checking all four in the nightly failed immediately,
because the prospect artifacts are deliberately outside that lane's sparse checkout.
The check is now scoped to the lane.

---

## Making an unattended write safe

This is where most of the design went. A run on my laptop has me watching it; a cron
job does not.

**Default to staging, everywhere, including the schedule.** Reaching production takes
a deliberate act (a manual dispatch choosing it, or a repository variable), and both
are visible in the run log and in settings. Nothing writes to production because a
default quietly pointed there.

**Only ever upsert.** The nightly writes with `INSERT ... ON CONFLICT DO UPDATE`. The
import scripts truncate before loading, which is fine when supervised and dangerous
unattended: a crash between truncate and load leaves the table empty. They are never
called here.

Worth calibrating rather than overselling: upsert protects against **lost data**, not
against **wrong values**. "Cannot destroy" is not "proven correct", and that is what the
staging target is for.

**Refuse to guess.** An empty secret stops the run instead of falling through to a
localhost default, which on a runner is a connection timeout dressed as a
configuration error. The URL scheme is checked before SQLAlchemy sees it, because a
pasted dashboard link otherwise fails with `Can't load plugin:
sqlalchemy.dialects:https`, which tells you nothing about what to fix. And the run
logs the destination host only, never the credential.

---

## Testing a scheduled workflow without shipping it first

`schedule` and `workflow_dispatch` only fire for workflows on the default branch. So a
nightly cannot be exercised before merging. The only way to find out whether it works
is to ship untested automation to `main`.

A `push` trigger runs the workflow from the branch that was pushed, so a
`ci-test/**` branch became the hatch. Runs from it are **forced** into dry mode: on a
push event the workflow's inputs are empty, so a `dry_run` input defaults to false and
the "test" would write to a real database.

A dry run still drives headless Chrome and performs the scrape (the parts that can
only fail on a runner) and stops before the write. Skipping the scrape would test
everything except the risk.

It also settled a question I could not answer from my desk: **nhl.com serves an Azure
runner the same page it serves my browser** (450,945 vs 449,348 bytes, table present).
A smoke-test step now checks that in seconds rather than after the scrape has burned
its retries.

---

## Two bugs only an unattended run could find

Both were invisible locally. Both are the reason I think running it for real, early,
was worth more than more careful review.

### A partial scrape deleted 740 players, silently

A flaky render returned **200 of 940 rows**. The merge treated the short result as
truth and rewrote the season: 52,189 → 51,449 rows, no error, no warning.

Three layers now. The extractor waits for table rows to appear and retries three
times; a short page raises rather than returning; and the merge refuses to shrink a
season below 95% of its previous size. Verified by reproducing it: the guard fires
and leaves the file untouched.

### The same code was 87 seconds locally and 42 minutes in CI

| Where | Time |
|---|---|
| Local Docker, ~0.1 ms round trip | 87 s |
| GitHub runner → Supabase | **42 min, cancelled while still running** |
| After the fix | **19 s** |

Same statement, same 38,899 rows. Only the network latency changed.

`conn.execute(statement, [dict, dict, ...])` looks like a bulk insert and is not.
SQLAlchemy's multi-row rewriting applies to Core `insert()` constructs; against a raw
textual statement it hands the list to the driver's `executemany`, which is a Python
loop: **one network round trip per row**. Invisible against a local database.
Crippling against a remote one.

My first fix was to chunk the rows into pages of 500. It changed nothing (86 seconds
against 87), because every chunk still went through `executemany`. The round trips
have to be collapsed on the wire, not in the caller. `execute_values` expands a
`VALUES %s` template into one statement carrying many rows; the `UPDATE` had to be
rewritten to join against a values list.

I keep this one because the wrong fix looked so reasonable and measured so precisely
as useless.

---

## Guard tests

Ordinary tests check that code is correct. These check that an invariant the
automation depends on has not quietly changed. That is the class of regression where nothing
errors and the damage appears later, in a diff or a bill or a published number.

| Guard | Catches |
|---|---|
| No lane contains a training stage | A nightly starting to retrain a model |
| Workflow checkout covers every path the lane reads | Two lists drifting apart |
| Frozen artifacts still match their hashes | A stray run rewriting published numbers |
| A mistyped stage raises | An empty run reporting success |
| CSV writes use LF terminators | Whole-file rewrites in every diff |
| No lane can call a truncating importer | An unattended run emptying a table |
| A short scrape cannot shrink a season | The 740-player deletion above |
| The age cutoff derives from the date | A constant someone edits once a year |

127 of them now. They need no data, no database and no network, which is why they are the
first thing the PR gate runs.

---

## What I would still change

- **The API tests do not declare their own environment.** They currently get two fake
  values from the workflow file, because some code builds a database engine at import
  time even for routes that never query. That requirement should live in the test
  suite, so it runs the same way everywhere, not in CI configuration.
- **There is no automated home for the scrapers that need a real browser.** They stay
  manual until there is a self-hosted runner worth maintaining.
- **The nightly is not fully reliable yet.** Of its last 60 scheduled runs, 43
  succeeded and 17 failed. A failed night does not lose data, because every run
  re-scrapes the whole season's totals and the next good run catches up, but it is
  still a gap. GitHub also starts scheduled runs late: in September they were created
  3.5 to 6 hours after the cron time, which is why the job now runs at 02:07 New York
  time and nothing in it depends on firing at an exact minute.

---

## Related

- [Architecture](../architecture/README.md): the offline/online split this sits inside
- [ETL pipeline](../etl/README.md): the stages a lane selects from
- [ADR 0008](../decisions/0008-named-lanes-over-skip-lists.md): enumeration over exclusion

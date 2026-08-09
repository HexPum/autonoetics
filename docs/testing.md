Where testing effort should go, and why the usual ordering is wrong here.

**This document is a proposal.** No code exists yet, so nothing below is a
description of a test suite — it is an argument about which invariants are
worth enforcing mechanically once there is something to enforce them
against. Written against the design in the root README,
`docs/protocol.md` and `docs/media-recollection.md`.

## The failure mode that matters

Most software fails loudly. This system fails silently, and the thing it
damages cannot be re-collected.

A crashed ingestion job is a good outcome: it is visible, and the fix is
to run it again. The bad outcome is a baseline window computed five
seconds off, or an external timestamp event-locked because a flag did not
propagate, or a wave z-scored globally instead of within-wave. None of
these throw. All of them produce a number that looks reasonable, enters a
model, and survives into a finding. By the time a result looks wrong
enough to audit, the wave that produced it is closed and the physiology
is a year old.

This inverts the usual priority. Coverage of the code that computes
results matters less than coverage of the rules that decide **which data
is allowed to reach a computation at all** — and those rules are already
written down in the design docs, in the imperative, which is unusual and
worth exploiting. "Never event-locked." "Never probed twice." "Always
precedes viewing." Each of those is a test waiting for a subject.

Ranked below by how silent and how irreversible the failure is, not by
test type.

## Tier 1 — irreversible and invisible at collection time

### Timestamp assignment

The load-bearing invariant of the architecture: every record carries a
canonical UTC timestamp assigned at acquisition, not at import. Devices
drift on the order of seconds per day, which `hardware.md` notes is
already enough to destroy event-locking at a five-minute baseline.

Worth enforcing per adapter, as a shared conformance suite rather than
per-device tests: the stamp is applied at acquisition; it survives the
round trip through Parquet unchanged; timezone is explicit at every
boundary; DST transitions and leap seconds do not reorder records. The
LSL clock-offset correction deserves its own tests against synthetic
streams with **known injected drift**, because that is the one place the
correctness of the correction can be checked against a ground truth you
control.

Note that `docs/media-recollection.md` already documents exports that
round to the minute in an undeclared timezone. Parsers for those need
tests that assert the ambiguity is preserved and marked, not guessed at.

### The external-clock interlock

Stated three times across two documents: sources marked
`clock_source = 'external'` are never event-locked, and Tier 1 export
data is exploratory permanently.

This is a safety interlock, and interlocks are tested negatively. Not
"is the flag set correctly" but: **for any dataset, no event-locked
result ever contains a row whose `clock_source` is `external`**, and no
confirmatory model ever reads a Tier 1 feature. Property-based tests fit
this exactly — generate mixed-provenance data, run the analysis, assert
the prohibited rows are absent from the output. A flag that is set
correctly but not *checked* anywhere passes every unit test and none of
these.

The failure is quiet in the worst way. A baseline window computed before
a timestamp of unknown sign may sit inside the exposure, which does not
produce an error, it produces an effect.

### Append-only storage

"Transformations produce new derived layers and never overwrite."

Test that raw paths are immutable after write; that derived layers
cannot be written to a raw path; that re-ingesting overlapping data is
additive or idempotent but never destructive; and that an interrupted
write leaves either a complete record or nothing, never a truncated
Parquet file that reads as valid but short. `docs/protocol.md` currently
lists "the procedure for a session interrupted mid-recording" as open —
whatever it resolves to is a test.

### Probe assignment

"Each day is randomly assigned to exactly one probe interval and is never
probed twice."

A doubly-probed day is not noisy, it is consumed — the second probe
measures the durability of the first probe. There is no recovery and no
way to detect it after the fact from the data alone.

The scheduler should be tested as a partition: every day in a wave
assigned exactly once, allocation ratios matching configuration, and
assignment **deterministic and replayable from a recorded seed** — which
the exploratory/confirmatory separation wants anyway, since an
assignment you cannot reproduce is an assignment you cannot
preregister. Then the guard on top of it: a probe attempt against an
already-probed day must be refused, including across restarts, and
including for a probe that was abandoned halfway.

### Recall-before-review ordering

"Recall measurement always precedes viewing the recorded material,
without exception. Violating the order does not degrade the measurement,
it consumes the thing being measured."

An ordering constraint stated that strongly should not live in the
operator's discipline. It should be an interlock in the probe tooling —
source material for a target window stays locked until that window's
probe is recorded — and the tests are the awkward paths, because the
straight path will work: app restart mid-probe, a probe abandoned
partway, a resumed session, an operator who opens the timelapse for a
neighbouring day and gets frames from the target window.

## Tier 2 — analysis correctness

These produce wrong numbers rather than destroyed data. Recoverable by
recomputation, which is the entire argument for keeping raw immutable,
but wrong findings are expensive in the meantime.

### Window construction

Two windows per sample — proximal 30 minutes, and inter-prompt since the
previous sample — both normalised by length. Every feature value depends
on these being right, and the edge cases are ordinary rather than exotic:

- the first prompt of a day, with no previous sample
- an unanswered prompt — does the inter-prompt window stretch across it,
  or is the sample dropped? (`docs/protocol.md` leaves the timeout and
  the admissibility of late answers open; both change this)
- windows straddling midnight, or a wave boundary
- the proximal and inter-prompt windows overlapping, which they will
  whenever prompts fall closer than 30 minutes apart
- a near-zero-length window, where normalising by length divides by
  something close to zero

### Feature computation

Each confirmatory feature has a degenerate input that must not silently
return a plausible number:

| Feature | Degenerate case | Wrong answer |
| --- | --- | --- |
| `switch_rate` | one item, no switches | 0 conflated with undefined |
| `mean_pairwise_distance` | fewer than two items | 0, which reads as *maximally similar* — the opposite of the truth |
| `short_form_fraction` | no media in window | 0 conflated with undefined |
| `media_fraction` | overlapping app logs | >1, from double-counted foreground time |

The `mean_pairwise_distance` case is the one to be most careful about:
an empty window silently scoring as maximally undifferentiated points
the distinctiveness hypothesis in exactly the direction it predicts.
That is a bug that confirms the thing under test.

`media_fraction` exceeding 1 is a realistic ActivityWatch failure —
concurrent foreground events across windows — and is worth an assertion
at computation time rather than a test alone.

Golden-file tests over a handful of hand-computed windows are cheap here
and catch most of this.

### Transformations

Z-score within wave, not globally. Trivial to get wrong, and the design
doc explains precisely why it matters: global normalisation smuggles
between-wave variance into within-wave contrasts. Test the grouping key
directly, and test a single-sample or zero-variance wave. Test that
`log1p` lands on the dose and dwell features, and exactly once —
double-application is invisible in a distribution plot.

Worth a structural test too: `switch_rate` and `median_dwell_s` are
near-reciprocal and must never enter one model together. That is a
checkable property of a model specification, not a matter of remembering.

### Scoring

d′ is the primary outcome and it goes infinite at a hit rate of 1.0 or a
false-alarm rate of 0. Whether a log-linear or 1/(2N) correction is
applied — and which — is exactly the kind of decision that should be
pinned by a test rather than discovered later in a distribution of
infinities.

Free-recall embedding similarity needs the calibration set the design
doc already calls for. That set is also a regression fixture: it is what
tells you an embedding model upgrade has shifted every score. Without
it, a dependency bump silently redefines the outcome variable.

### Event-locked metrics

Baseline and response windows, autonomic change, recovery time. Test
against a baseline that overlaps a preceding event, against partial
physiological coverage, and against the per-stream coverage minima once
`docs/protocol.md` sets them.

## Tier 3 — boundaries and gates

### Gaps must be gaps

`hardware.md` names the trap directly: dropped BLE packets "appear in the
data as gaps that read like physiological events." So absence has to be
represented explicitly and survive to analysis — a missing interval must
never be silently bridged by interpolation or closed by a resampling
step. This belongs in the adapter conformance suite, and it is a
property, not an example: no derived series contains a value spanning an
interval with no underlying samples.

### Ranking-model isolation

"No physiological or momentary sampling data flows into the ranking
model." An architectural boundary, testable cheaply as an
import/dependency assertion plus a check on which columns the ranking
track reads from the shared content store. The failure — a model tuned
on the data used to test hypotheses about the person tuning it — is
circular in a way that is very hard to spot downstream and trivial to
prevent upstream.

### Validity gates

Compliance below ~70% invalidates a wave; clock alignment is verified
before a wave is promoted to analysed. Both are computable predicates
that gate everything after them, so both want boundary tests — 69.9
against 70.1 — and a decision, currently open, about how unanswered and
late-answered prompts count toward the denominator.

### Audio retention

Raw audio is discarded on a fixed schedule and is not stored. That is a
privacy-critical deletion path, and it should be tested before a
microphone is ever switched on: the retention job deletes what it
claims, a crashed pipeline leaves no raw audio behind, and no raw audio
reaches a backup — the backups are encrypted, but they are also
retained, which is the point.

## What to build first

Two fixtures carry most of the weight, and both are worth having before
the first adapter:

**A synthetic wave with known ground truth.** Generated signals and
events with injected drift, gaps, missed prompts and a known correct
answer for every derived feature. This is what makes end-to-end
assertions possible at all — without it, pipeline tests can only check
that something ran.

**A hand-scored calibration set** for free-recall matching, which the
design already requires for threshold calibration and which doubles as
the regression fixture against embedding drift.

Beyond those: property-based testing suits this codebase unusually well,
because most of the invariants above are already written as universally
quantified statements. "Never event-locked" is a property, and testing it
by example will miss the case that matters.

## Ordering problem

Several tests above cannot be written yet, and not because the code is
missing. They are undecided.

Prompt timeout, late-answer admissibility, minimum session length, the
interrupted-session procedure, per-stream coverage minima, whether a
wave can be partially invalidated by stream, the probe interval ratios,
foil construction, the recall threshold, and the mobile scope decision
are all marked open in `docs/protocol.md` or
`docs/media-recollection.md`. An undecided rule cannot be tested, and
in several cases the code cannot be written either — the
interrupted-session procedure is a branch in the ingestion path, not a
policy applied afterwards.

`docs/protocol.md` already makes the argument that these "stop being
neutral choices once data exists." The same is true a step earlier: they
stop being neutral once code exists, because whatever the implementation
does becomes the de facto protocol, and a test written after the fact
will encode the accident rather than the decision.

## Status

Proposal. No code, therefore no tests. The dependency order in
`docs/media-recollection.md` applies here too — the momentary sampling
layer is the first thing that can be tested, and the Tier 1 items above
are the ones worth having in place before a wave opens rather than after.

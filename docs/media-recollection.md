Media consumption is an unusually good probe for the gap this
repository was built to measure. It leaves an objective record of what
was consumed, it occurs during windows in which momentary state is
already being sampled, and it is later recollected — or not. Three
independent channels on the same interval, only one of which is
currently instrumented.

This document specifies the media layer and the recollection measure
built on top of it. It is design, not implementation; nothing here is
buildable until the momentary sampling layer runs.

## The question

Does media consumption change the recollectability of the period in
which it occurred?

Not whether media affects mood or stress. Those are confounded beyond
rescue in observational n-of-1 data — low mood causes scrolling at
least as reliably as scrolling causes low mood, and no amount of
lagging fixes a bidirectional relationship. Recollection is asymmetric
in time: consumption precedes recall, always, and the direction of the
arrow is not in question.

## The mechanism under test

Dose is the obvious hypothesis and probably the wrong one. Two hours
of one absorbing film and two hours of feed-scrolling are the same
dose, and there is no reason to expect the same effect on memory.

The hypothesis worth testing is **loss of distinctiveness**: media
renders a period internally undifferentiated, leaving retrieval
nothing to hook onto. A window of forty near-identical short clips has
no landmarks. A window with one long, unusual, emotionally variable
item has several.

Dose is retained in the confirmatory model as the competing null. If
only dose survives, the interesting hypothesis dies cleanly, which is
the point of including it.

## Unit of analysis

**The momentary sample, not the day.**

Every sampling prompt is already a memory target with a known
timestamp, concurrent physiology, and a determinable media window
preceding it. The outcome is per-sample and binary — was this specific
moment later recollected — with per-sample predictors. Mixed-effects
logistic regression, samples nested within days.

At day level the analysis yields on the order of sixty observations a
year and absorbs every day-level confound as noise. At sample level it
yields roughly six a day, and day-level confounds become a random
effect.

## Acquisition: two tiers

Media data arrives on two clocks, and conflating them will produce
confident wrong findings.

### Tier 1 — retrospective platform exports

GDPR exports from the platforms. Timestamps are assigned by the vendor
at logging, not by this system at acquisition, in violation of the
one-clock principle. YouTube's watch history records a single time
that is plausibly the end of viewing rather than the start; Reddit's
saved list carries no timestamps at all; several exports round to the
minute in an undeclared timezone.

Consequences, which are not negotiable:

- Marked `clock_source = 'external'` at ingestion.
- Never event-locked. A baseline window computed before a timestamp of
  unknown sign may sit inside the exposure.
- Aggregate use only: hourly or daily features against coarse
  outcomes.
- **Exploratory only, permanently.** This data arrived after the fact
  and no hypothesis preceded it. Under the exploratory/confirmatory
  separation it can generate hypotheses and can never test one.

### Tier 2 — prospective local logging

ActivityWatch, or an equivalent local logger, running continuously.
Gives true start and end at second precision, app and window title,
browser URL. Carries an acquisition-time UTC stamp and can be marked
into LSL like any other stream.

This is the only tier admissible for event-locked analysis and the
only tier admissible for confirmatory tests.

It is also the only component that cannot be backfilled. Exports can
be requested at any time and will still contain the past; every day
without Tier 2 logging is permanently absent. This argues for starting
it before anything else is ready, and ignoring it until it is needed.

### Validation

Add one field to the momentary sampling prompt: media in the preceding
interval, yes/no, plus a free-text what. This is ground truth against
which Tier 1 timestamp accuracy can be estimated, and it is needed
before Tier 1 is trusted for anything at all.

Note the direction of the labels-before-sensors principle here. A raw
media stream is precisely the uninterpretable channel that principle
warns against. It becomes analysable because momentary reports sit
alongside it, not the reverse.

## Feature set

Two windows per sample, both normalised by length: a **proximal**
window of 30 minutes before the prompt, for encoding-state effects,
and the **inter-prompt** window since the previous sample, for
cumulative exposure.

### Confirmatory — declared before the wave

Four predictors, no more. At roughly six samples a day, effective
sample size collapses quickly with model complexity.

| Feature | Family | Tests |
| --- | --- | --- |
| `switch_rate` | fragmentation | switches per minute between items or apps |
| `mean_pairwise_distance` | distinctiveness | mean cosine distance between items in window |
| `short_form_fraction` | format | share of media time in items under ~60s |
| `media_fraction` | dose | media time / window length — the competing null |

### Exploratory — computed, not tested

*Dose:* `at_prompt` (media active at the prompt moment).

*Fragmentation:* `median_dwell_s`, `n_distinct_items`, `dwell_cv`.
Note that `switch_rate` and `median_dwell_s` are near-reciprocal;
never enter both into one model.

*Distinctiveness:* `topic_entropy` over cluster shares; `novelty` as
mean distance from the long-run interest centroid;
`window_typicality` as similarity of this window to all other windows.
A period that resembles every other period should not be recallable —
this is the most direct test of the mechanism and the cheapest to
compute.

*Affect:* `mean_valence`, `mean_arousal` from content;
`valence_variance` for emotional whiplash.

*Format:* `text_fraction`, `passive_fraction` (scroll and watch versus
search, read, write).

### Transformations

Z-score within wave rather than globally — the baseline drifts across
waves and global normalisation smuggles between-wave variance into
within-wave contrasts. Apply `log1p` to dose and dwell features, which
are heavily right-skewed.

### Known scope limit

In-app mobile feeds are opaque. Android UsageStats yields app-level
foreground time and switch rate but no item-level content; iOS yields
less. The distinctiveness and affect families are therefore computable
for desktop and browser media and effectively not for in-app TikTok or
Instagram.

Either the confirmatory claim is scoped to browser media, or mobile is
admitted as dose-and-fragmentation only. **This must be decided before
the wave**, not after the results are visible.

## Recollection measure

Two instruments, because they fail differently.

**Free recall.** At the probe interval, write everything recalled from
the target window with no access to any record. Score by embedding
similarity against the momentary entries from that window: entries
matched above threshold count as recalled. Intrusions — recalled
content with no matching entry — are counted separately and are
interesting in their own right.

**Cued recognition.** Present real momentary entries interleaved with
foils drawn from other days. Hit rate and false-alarm rate yield d′,
which is scorer-independent. This is the primary outcome; free recall
supplies texture and intrusion counts.

### Probing is rehearsal

A probed day is a re-encoded day. Probing the same day at 24 hours and
again at 7 days measures, on the second occasion, the durability of
the probe rather than of the experience.

**Each day is randomly assigned to exactly one probe interval — 24h,
7d, or 30d — and is never probed twice.** Multiple intervals are
necessary because media may affect the rate of decay rather than its
level, and a single interval cannot distinguish these.

## Mediation

Autonomic arousal at encoding predicts later recall. It is being
recorded continuously, which makes the mediation path testable rather
than assumed:

    media features → arousal at sample → recollection

with a direct path alongside. If the effect runs through arousal,
that is a mechanism. If media predicts recollection with arousal held
constant, that is a different and more interesting finding. Fit both.

## Covariates

Minimum set for any model of recollection:

- time since encoding (dominates everything else)
- time of day
- prior-night sleep
- ordinal position of the sample within the day
- whether the sampled moment was socially interactive
- distinctiveness of the entry's own content — distinctive moments are
  recalled better irrespective of anything preceding them

## Relationship to the interest-ranking track

A separate line of work builds a personal ranking model from platform
export history. The two share a content dimension store — items,
extracted text, embeddings, derived topics — and nothing else.

The dependency is unidirectional. Both tracks read the content store;
neither reads the other. In particular **no physiological or momentary
sampling data flows into the ranking model**, which keeps a model
tuned by hand out of the data used to test hypotheses about the person
tuning it.

## Dependencies

Ordered. Nothing later is buildable before what precedes it.

1. Momentary sampling layer producing timestamped entries. Everything
   here is downstream of this and there is no partial version.
2. Tier 2 logging running continuously. Independent of (1), cannot be
   backfilled, so it starts first in wall-clock time even though it is
   useless until (1) exists.
3. Content dimension store: item, content, embedding, topic.
4. Feature computation over windows.
5. Probe protocol and scoring.

## Open decisions

- Mobile scope: browser-only confirmatory claim, or mobile admitted at
  dose-and-fragmentation resolution.
- Probe intervals: the 24h/7d/30d set is a proposal, not a decision.
  Ratio of days assigned to each is unset.
- Foil construction for cued recognition — same-wave foils control for
  period-typicality but risk false familiarity.
- Recall threshold for the free-recall embedding match. Needs
  calibration against hand-scored entries before it can be trusted.
- Whether wave 1 includes randomised media conditions or stays purely
  observational.

## Status

Design only. No collection running. The measurement protocol at
`docs/protocol.md` is at present a stub; this document assumes it and
should be reconciled with it once written.

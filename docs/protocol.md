The measurement protocol: what is collected, when, under what conditions,
and what invalidates a wave. Referenced from the root README and assumed
by `docs/media-recollection.md`.

**This document is a stub.** What follows is the structure the protocol
needs, not the protocol. Where a decision has already been made and
recorded elsewhere it is stated here with its source; everything else is
marked open. An empty section means undecided, not unconstrained.

## Waves

Collection runs in bounded waves rather than continuously. Hypotheses are
written down between waves and tested on data collected afterwards — the
exploratory/confirmatory separation is the reason waves exist at all, not
a scheduling convenience.

*Open:* wave length, number of waves, interval between them, and what a
wave is declared against before it opens.

## Momentary sampling

The non-substitutable layer. Randomised push notification, voice memo
capture, local transcription to CSV, ingested as `events`.

- 6 prompts/day during waves, 3/day in continuous operation.
- One added field for the media layer: media in the preceding interval,
  yes/no, plus free-text what. See `docs/media-recollection.md`.

*Open:* randomisation window and quiet hours, the capture schema beyond
the media field, timeout before a prompt counts as unanswered, and
whether a late answer is admissible or discarded.

## Physiological sessions

Wear is not uniform across devices, and the protocol has to say which
condition each stream is valid under.

- Cardiac (Polar H10): defined sessions and known interaction windows,
  not continuous wear.
- Nocturnal EEG (Muse S Athena): nights only in the essential
  configuration.
- Context layer (Amazfit strap): continuous.
- Sync markers at the start and end of every session — a movement visible
  simultaneously to accelerometers on multiple devices.

*Open:* what constitutes a session, minimum session length, and the
procedure for a session interrupted mid-recording.

## Recall probes

Specified in `docs/media-recollection.md`: free recall scored by
embedding similarity, cued recognition scored as d′, each day assigned to
exactly one probe interval and never probed twice.

One constraint belongs here rather than there, because it binds every
retrieval measure and not only the media one:

**Recall measurement always precedes viewing the recorded material,
without exception.** Review of stills or entries reactivates memory
strongly enough to overwrite it. Violating the order does not degrade the
measurement, it consumes the thing being measured.

## Validity conditions

What invalidates a wave, as opposed to what merely adds noise.

- Compliance below ~70% answered prompts invalidates the wave.
- Clock alignment is verified before a wave is promoted from exploratory
  to analysed.

*Open:* per-stream coverage minima, artifact-rejection thresholds, and
whether a wave can be partially invalidated by stream rather than
discarded whole.

## Data handling

Governed by the principles in the root README and restated here only
where the protocol has to enforce them at collection time:

- Every record carries a canonical UTC timestamp assigned at acquisition,
  not at import.
- Ingestion is append-only. Derived layers never overwrite raw.
- Externally timestamped sources are marked `clock_source = 'external'`
  and are never event-locked.

## Status

Stub. No collection running. This document should be written before the
first wave opens, since several of the items marked open above stop being
neutral choices once data exists.

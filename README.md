Neurophenomenology pairs disciplined first-person reports with simultaneous
physiological recording, so that neither channel has to be trusted alone.
This repository builds that as personal instrumentation rather than
laboratory apparatus: a reproducible pipeline for a single person over
years. The question it was built to answer is a narrow one. The
architecture is not — your question will differ, the instrumentation
problem will not.

## What this is

- A momentary sampling layer (randomised prompts, structured capture)
  that produces timestamped subjective annotations
- Adapters for continuous physiological streams — cardiac, electrodermal,
  electroencephalographic — normalised to a single schema
- A passive context layer (location, activity, environment) collected
  without user action
- An append-only local store with an event-locked analysis layer on top

Output is a queryable record in which any physiological window can be
matched to what the subject reported experiencing at that moment.

## The problem it's built around

Retrospective self-report is a biased sample of experience. What you
believe your days consist of and what your days consist of are different
datasets, and the difference is systematic rather than random. Measuring
that difference requires both sides to be recorded independently — the
momentary report and the later memory of it — which is an instrumentation
problem before it is a psychological one.

## Design principles

- **Labels before sensors.** Physiological data without concurrent
  subjective annotation is close to uninterpretable. The momentary
  sampling layer is what makes every other stream analysable, not one
  source among several.
- **One clock.** Streams arrive at rates from 256 Hz to six events per
  day. Every record carries a canonical UTC timestamp assigned at
  acquisition, not at import.
- **Raw data is immutable.** Ingestion is append-only. Transformations
  produce new derived layers and never overwrite. Analysis methods will
  change; unrecorded signal will not come back.
- **Exploratory and confirmatory phases are separated.** Collection runs
  in bounded waves. Hypotheses are written down between waves and tested
  on data collected afterwards. Without this, a system with fifty channels
  generates significant findings from noise indefinitely.
- **Hardware is configuration, not architecture.** Devices are adapters
  behind a common interface. The device list in `hardware/` is one working
  instantiation, not a requirement.

  ## Architecture

Everything reduces to two tables:

- `signals` — (timestamp, source, channel, value): continuous streams
- `events`  — (start, end, type, annotations): interactions, sessions,
  sampling prompts, state transitions

Storage is append-only Parquet in a local lake, queried through DuckDB.
Acquisition uses Lab Streaming Layer where devices support it. Nothing
requires a server; nothing requires a vendor cloud.

The core analysis is event-locked: a baseline window before a marked
event, a response window during and after, and per-event metrics —
autonomic change, recovery time, concurrent subjective annotation —
compared across event types.

## Status

Early. See `docs/protocol.md` for the measurement protocol and
`hardware/` for the current instantiation.

# Hardware

One working instantiation. Devices sit behind adapters; nothing in the
analysis layer depends on this particular list. Substitutions are expected —
what matters is that a replacement exposes the channels below at comparable
rates and that its artifact modes are documented here.

Rates given are the acquisition rates actually used, not device maxima.

---

## Essential

The minimum set required to run the protocol in `docs/protocol.md`.

### Smartphone — momentary sampling layer

| | |
|---|---|
| Channels | subjective annotation, modality tag, context tag |
| Rate | 6 prompts/day (waves), 3/day (continuous) |
| Interface | randomised push notification, voice memo capture |
| Data path | local transcription → CSV → `events` |

The only genuinely non-substitutable component. Every other stream is
uninterpretable without concurrent annotation; see the "labels before
sensors" principle in the root README. Compliance below ~70% answered
prompts invalidates a wave.

### Polar H10 — cardiac

| | |
|---|---|
| Channels | ECG, RR intervals, accelerometer |
| Rate | ECG 130 Hz, RR event-based, ACC 200 Hz |
| Interface | BLE, official SDK; LSL connector available |
| Data path | raw ECG → own RR extraction → `signals` |

Raw ECG rather than vendor-computed HRV is the point: RR intervals derived
from the waveform can be artifact-rejected, vendor HRV cannot. Worn in
defined sessions and known interaction windows rather than continuously.

**Artifact modes:** electrode dry-out over long wear; strap slippage during
vigorous movement; baseline wander with deep breathing.

### Amazfit Helio Strap — context and activity layer

| | |
|---|---|
| Channels | heart rate, motion, sleep duration, temperature |
| Rate | aggregated (1 min and coarser) |
| Interface | Gadgetbridge (local, cloudless) |
| Data path | Gadgetbridge SQLite → `signals` |

Continuous wear. Fills the coverage gaps the chest strap leaves. Paired
through the vendor app once to extract the authentication key, then read
locally.

**Known limitations:** no electrodermal channel. HRV, SpO₂ and VO₂ reported
as not populating reliably via Gadgetbridge on this model. Vendor exports
contain visible heart-rate artifacts that survive into analysis — treat
processed values as context, not as physiology.

### Muse S Athena — nocturnal EEG

| | |
|---|---|
| Channels | EEG (AF7, AF8, TP9, TP10), fNIRS, accelerometer |
| Rate | 256 Hz |
| Interface | Mind Monitor → CSV/EDF |
| Data path | raw EEG → own staging (YASA) → `signals`, `events` |

Nights only in the essential configuration. Sleep is the only condition in
which forehead electrodes achieve usable signal-to-noise without heavy
rejection: no blinks, minimal facial EMG, muscle atonia in REM.

Own staging rather than vendor staging, so that scoring is reproducible and
versioned.

**Artifact modes:** position-dependent contact loss; sweat-driven impedance
drift over the night; band displacement on side sleeping.

### Storage

External SSD (2–4 TB) plus a second drive for backup. Full-disk encryption
(VeraCrypt or LUKS), encrypted backups (restic or Borg). No vendor cloud in
the raw data path.

---

## Optional

Extends capability. Each of these adds a layer; none is required for a valid
wave.

### IDUN Guardian 4 — daytime EEG

| | |
|---|---|
| Channels | in-ear EEG, IMU |
| Interface | Python SDK, public API; raw access without cloud |
| Data path | raw EEG → `signals`; classifier output → `events` |

The reason daytime EEG is viable at all here: in-ear placement moves the
electrodes off the frontalis muscle and away from the eyes, which removes
the two artifacts that dominate forehead recording during waking hours.
On-device jaw-clench and eye-movement classifiers give an artifact channel
for free.

Also solves the visibility problem — earbuds are socially invisible in a way
a headband is not, which matters because a visible rig confounds the
interaction analysis.

**Artifact modes:** chewing and speech EMG (not solved by ear placement —
this is what the jaw-clench classifier is for); fit variation with ear
anatomy; small amplitudes from short inter-electrode distances.

### Electrodermal activity

No single good option. Site selection matters more than device:

| Site | Agreement with finger reference | Wearable |
|---|---|---|
| Finger | reference | no |
| **Foot** (medial plantar / instep) | r ≈ .72 | yes, under a sock |
| Mid-chest | good among torso sites | yes, integrates with strap |
| Wrist | r ≈ .30–.42 | yes |
| Shoulder / upper arm | ≈ 0 | no signal |

**Foot is the chosen site.** It is the only placement that both preserves
signal and disappears in daily wear, and it was the most robust of the
tested sites against motion artifact. Wires route up the leg to the hip hub.

Mid-chest is the fallback and the more elegant integration, since the ECG
strap is already there — one band, two channels.

Implementation options:
- DIY: ESP32 + GSR front-end + Ag/AgCl adhesive electrodes. Sufficient for
  N=1 exploration; sessions rather than continuous.
- Empatica EmbracePlus: research-grade, purchasable by individuals through
  the Academic & Basic Research plan. Multi-year subscription bundle, cloud
  data path, raw signals in avro chunks requiring reassembly.

**Artifact modes:** wrist placement accumulates sweat during low-movement
periods, inflating apparent conductance — this looks like a deep-sleep
finding and is a strap artifact. Absolute conductance values are not
comparable across sites; use within-person relative change around events.

### Omi — audio layer

Open-source hardware and firmware, which is why it is the choice: a local
processing pipeline can be built into it. Feeds voice-activity detection and
diarisation, which is what makes interaction events detectable automatically
rather than by manual marking.

Audio is not stored. Pipeline is VAD → diarisation → local transcription →
derived metrics, raw audio discarded on a fixed schedule.

### Raspberry Pi Zero 2 W + Camera Module 3 — image layer

Periodic stills (1 frame per 20–60 s), not continuous video. Comparable
storage to audio, reviewable as a timelapse in minutes, and a substantially
stronger retrieval cue than text.

**Protocol constraint:** image review reactivates memory strongly enough to
overwrite it. The recall measurement must always precede viewing the
material, without exception, or the instrument consumes the measurand.

### BLE hub

A Pi with multiple BLE dongles, worn at the hip. Phones handle only a
handful of concurrent BLE connections reliably; beyond that, dropped packets
appear in the data as gaps that read like physiological events. Required
once four or more peripherals stream simultaneously.

Also the correct home for battery and switching electronics — kept as far
from the ECG and EEG electrodes as the harness allows.

### Bicep band — alternative optical site

Upper-arm PPG is more motion-stable than wrist, which matters because the
events of interest involve gesture and movement. Useful for pulse and skin
temperature only; the upper arm carries no usable electrodermal signal.

---

## Placement constraints

Applies to any integrated harness:

- No EL wire anywhere. High-frequency inverters radiate broadband noise
  directly into EEG and ECG.
- Addressable LEDs below the waist only, drivers shielded. PWM dimming
  injects switching noise.
- Nothing conductive or heavy on the head. Added head mass increases motion
  artifact, which is already the limiting factor for waking EEG.
- Do not reshell certified devices. Custom housings change electrode contact
  pressure and therefore signal quality, invisibly until analysis.
- Battery and regulators at the hip, never near chest or head electrodes.

---

## Time synchronisation

Every record carries a UTC timestamp assigned at acquisition. Devices drift
on the order of seconds per day, which is enough to destroy event-locked
analysis at a 5-minute baseline window.

- Lab Streaming Layer where the device supports it — clock-offset correction
  at acquisition is the only real solution.
- Manual sync markers otherwise: a movement visible simultaneously to
  accelerometers on multiple devices, recorded at the start and end of every
  session.
- NTP on the hub and phone.

Verify alignment before promoting a wave from exploratory to analysed.

---

## Rejected

Documented so the reasoning is not rediscovered:

- **Consumer "focus"-metric EEG headbands for daytime use.** Frontal
  electrodes sit over the frontalis muscle; the beta and gamma bands the
  metrics compute from are contaminated by facial EMG. Worse, the artifact
  correlates with the psychological state of interest, so it produces
  plausible findings that are muscle tone.
- **Frontal alpha asymmetry as a mood or approach marker.** Large
  preregistered work has failed to support its validity as either a state or
  trait measure; the depression meta-analytic effect is near zero.
- **Empatica E4.** Device and software suite sunset February 2025.
- **Continuous video.** Storage and review burden are disproportionate;
  periodic stills carry nearly all the retrieval value.
- **A second general wearable (ring or smartwatch).** Redundant with the
  strap, costs charging time and compliance.
- **Cloud health-data aggregation services.** Raw data does not leave local
  storage.

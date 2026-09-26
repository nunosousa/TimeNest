# Multi-Standard LF Time-Signal Generator — Architecture Sketch

Target: a near-field transmitter that emits DCF77, WWVB, MSF, JJY (40/60) and BPC
time signals to synchronise consumer radio-controlled watches and clocks, with
full fidelity including DCF77 phase modulation and WWVB BPSK.

---

## 1. Top-level block diagram

```
 ┌────────────┐  UART+PPS   ┌──────────────────────────────────────┐
 │ GNSS module├────────────►│              RP2350                  │
 └────────────┘             │                                      │
 ┌────────────┐    I2C      │  core0: timebase, GNSS, UI, config    │
 │ RTC+cell   ├────────────►│  core1: NCO sample synthesis          │
 └────────────┘             │  PIO0:  R-2R sample clock (DMA fed)   │
 ┌────────────┐             │  PIO1:  PPS capture / spare           │
 │ TCXO       ├──► XIN      └───────────────┬──────────────────────┘
 └────────────┘                             │ 10-bit parallel
                                            ▼
                              ┌──────────────────────────┐
                              │ R-2R ladder (10-bit)     │
                              └────────────┬─────────────┘
                                           ▼
                              ┌──────────────────────────┐
                              │ Reconstruction LPF ~120k │
                              └────────────┬─────────────┘
                                           ▼
                              ┌──────────────────────────┐
                              │ Linear driver (op-amp)   │
                              └────────────┬─────────────┘
                                           ▼
                              ┌──────────────────────────┐
                              │ Untuned / damped loop    │
                              └──────────────────────────┘
```

Everything downstream of the R-2R is **linear** — no switching stages — because
the signal now carries simultaneous amplitude *and* phase information.

---

## 2. Clocking and the sample rate

**System clock: 150 MHz** (RP2350 in-spec), sourced from a TCXO into XIN.

**NCO sample rate: Fs = 2.000000 MS/s**, via PIO divider 150/75.

Why this pairing matters:

- `2 000 000` samples per second is an **integer**, so UTC second boundaries land
  exactly on sample boundaries. No accumulating fractional-sample error in the
  symbol scheduler.
- 2 MS/s gives ~26 samples/cycle at 77.5 kHz. Images sit at Fs±fc ≈ 1.92/2.08 MHz,
  ~25× above the carrier — trivial to filter.
- CPU budget: 75 clock cycles per sample. The inner loop (accumulator add, LUT
  fetch, amplitude multiply, store) is ~12–15 cycles on Cortex-M33. Roughly 20%
  of one core. Comfortable.

Note that integer *division* of the system clock into the carrier is no longer
required at all — see §3. The 150/2 MHz choice is purely for clean second
boundaries.

---

## 3. The NCO (why this solves multi-standard)

32-bit phase accumulator, advanced once per sample:

```
phase     += phase_increment          // frequency
out_phase  = phase + phase_offset     // phase modulation
sample     = sine_lut[out_phase >> 20] * amplitude   // amplitude modulation
```

`phase_increment = round(f_carrier * 2^32 / Fs)`

| Carrier | f (Hz) | phase_increment | residual error |
|---|---|---|---|
| JJY40   | 40 000 | 85 899 345.9 → 85 899 346 | < 0.05 mHz |
| WWVB/MSF| 60 000 | 128 849 018.9 → 128 849 019 | < 0.05 mHz |
| BPC     | 68 500 | 147 089 592.3 → 147 089 592 | < 0.2 mHz |
| DCF77   | 77 500 | 166 429 982.7 → 166 429 983 | < 0.2 mHz |

All well under 5 ppb — three orders of magnitude better than the TCXO itself,
so the crystal dominates and the NCO contributes nothing.

**This is the key architectural decision.** A divide-the-system-clock design
cannot do this: 68.5 kHz contains the prime factor 137, which pushes the LCM of
all five carriers to 509.64 MHz — unreachable on this silicon. The NCO makes
carrier frequency a runtime register value.

It also makes both modulations trivial:
- **PM/BPSK** = add to `phase_offset` (±15.6° for DCF77, 180° for WWVB).
- **AM** = scale `amplitude` (15% DCF77, −17 dB ≈ 14.1% WWVB, 0% MSF, ~10% JJY),
  with free soft ramps to limit splatter.

### Deriving the DCF77 chip clock correctly

DCF77's PM chip rate is 645.83 Hz = fc/120, i.e. **one chip per 120 carrier
cycles**. At 2 MS/s a chip is 3096.774 samples — *not* an integer, so counting
samples will drift.

Instead, derive chip boundaries from the NCO itself: count accumulator
overflows (= carrier cycles) and advance the PRBS every 120th. This is exact by
construction and self-aligning, because the chip rate is *defined* as a division
of the carrier.

---

## 4. Software layering

Three layers with narrow interfaces. The whole point is that adding a sixth
standard should touch only layer 2.

### Layer 1 — Timebase service
Produces a sample-accurate UTC: `(utc_seconds, sample_within_second)`.

- GNSS PPS captured against the free-running sample counter.
- Steer/align the second boundary; measure TCXO error over long intervals and
  store the learned ppm correction in flash for holdover.
- Leap-second state (current + pending) taken from the GNSS navigation message.
- RTC (battery-backed) provides time across power cycles.

### Layer 2 — Standard / protocol layer
`utc → local time for that standard → frame bits → symbol schedule`

- DST/offset rule engine (§5).
- One frame encoder per standard, each a pure function of civil time → bits.
- Emits a **symbol schedule**: a list of
  `{ sample_offset, amplitude_q15, phase_offset_q32, ramp_samples }` events for
  the coming second, plus an optional PRBS generator hook for DCF77 PM.

### Layer 3 — Modulator
Consumes the symbol schedule, runs the NCO, fills double-buffered DMA blocks
(e.g. 512 samples) on core1. Knows nothing about time or standards.

This split keeps the twice-a-year DST bugs and the real-time DSP in completely
separate, separately-testable code.

---

## 5. Time, date and DST — no network needed

GNSS supplies UTC, date and leap-second data (including *pending* leap seconds,
which is better than NTP's last-day-only leap indicator).

GNSS does not supply timezone or DST rules — but none of these standards need a
tzdata database, because each broadcasts exactly one zone under one fixed
algorithmic rule:

| Standard | Zone | DST rule |
|---|---|---|
| DCF77 | CET/CEST | EU: last Sun Mar 01:00 UTC → summer; last Sun Oct 01:00 UTC → winter |
| MSF | UTC/BST | Same instants as the EU rule |
| WWVB | UTC + DST status bits | US: 2nd Sun Mar, 1st Sun Nov |
| JJY | JST (UTC+9) | None |
| BPC | UTC+8 | None |

~30 lines of "Nth/last Sunday of month" arithmetic. **Store the rule parameters
in flash, not hardcoded**, so a future EU decision to abolish DST is a config
change rather than a firmware respin.

Announce bits are computed from the same rules: DCF77 A1 (bit 16) during the
hour before a change, A2 (bit 19) for leap seconds, and WWVB's DST status bits.
*Pin the exact announce timing against the PTB and NIST specs — these are the
classic "works all year, fails twice a year" defects.*

---

## 6. Output stage

### Recommendation: untuned (or lightly damped) loop

A resonant tank is the obvious choice and the wrong one here:

- DCF77 PM at a 645.83 Hz chip rate needs roughly ±1.5–2 kHz of passband.
  Q=100 at 77.5 kHz gives only 775 Hz — it smears the chips.
- MSF is true OOK (carrier fully off); a high-Q tank rings instead of stopping.
- Five carriers spanning 40–77.5 kHz would need a switched capacitor bank, and
  each band would have a different phase response to calibrate out.

Since range is deliberately only a few centimetres, radiation efficiency is
irrelevant — so drive a small **untuned** loop as a current source. Flat from 40
to 77.5 kHz, zero phase distortion, no band-switching hardware, and it helps
rather than hurts regulatory compliance. Compensate the per-band impedance
difference (loop Z rises with frequency) with a per-standard amplitude constant
in firmware.

All spectral cleanup then falls to the reconstruction LPF (3rd-order, ~120 kHz).

If more range is ever needed, add an optional switched-C bank targeting a
**deliberately low Q of 10–25** — never higher.

Worked example if you do resonate: L = 1 mH at 77.5 kHz → ωL = 487 Ω;
C = 4.22 nF; for Q = 20, total series R = 24 Ω.

### DAC

10-bit R-2R on GPIO, PIO+DMA driven. Prototype with 0.1% resistors (1% parts
give only ~7 effective bits, and the −17 dB WWVB level wants ~1% amplitude
accuracy). Note that **phase resolution lives in the NCO, not the DAC** — DAC
bits only bound amplitude accuracy and spurious floor. For production, evaluate
a monolithic parallel DAC for repeatability.

---

## 7. Pin budget (RP2350)

| Function | Pins |
|---|---|
| R-2R ladder | 10 |
| GNSS UART (TX/RX) | 2 |
| GNSS PPS | 1 |
| GNSS enable | 1 |
| I2C (RTC + OLED) | 2 |
| Amplifier mute/enable | 1 |
| UI (encoder + button) | 3 |
| Status LED | 1 |
| Optional band-select switches | 3 |
| **Total** | **~21–24** |

Fits the 30-GPIO QFN-60 package with margin.

---

## 8. Standard selection and UX

- Small OLED + rotary encoder, or USB-C serial config, or both.
- Config in flash: standard, TX power trim, DST rule parameters, learned TCXO ppm.
- Consider a "transmit window" scheduler: most radio-controlled watches only
  attempt sync overnight (typically 02:00–04:00), so the device can idle most of
  the day. This cuts duty cycle, power, and regulatory exposure — and is a
  genuine product feature ("only radiates 20 minutes a night").
- Provide a manual "sync now" button for watches that support forced receive;
  without it, development iterations take hours.

---

## 9. Test and validation strategy

- **A 192 kHz USB audio interface is your best analysis tool.** Nyquist is
  96 kHz, so all five carriers (40, 60, 68.5, 77.5 kHz) fit directly in the
  audio band. No SDR or upconverter needed — capture the loop with a pickup coil
  and decode/analyse in software.
- Commercial DCF77/WWVB/MSF receiver modules as golden-reference decoders.
- A second RP2350 running a decoder for loopback regression tests.
- **Golden-vector unit tests**: known timestamp → expected bit pattern, per
  standard, checked bit-for-bit against published examples.
- **Sweep the DST engine across ~20 years** of transitions in unit tests,
  including the leap-second and announce-bit edges.

---

## 10. Risks

| Risk | Severity | Mitigation |
|---|---|---|
| Regulatory compliance for sale | **High** | Near-field by design; engage a compliance consultant early; consider positioning as test equipment; pre-scan EMC before layout freeze |
| GNSS never gets a fix indoors | High | RTC holdover + learned TCXO correction; only needs one fix at setup; USB time-set fallback |
| DCF77 PRBS spec details (sequence, start offset within the second) | Medium | Obtain PTB specification before committing to layer-2 design |
| WWVB BPSK +0.1 s offset implemented wrong | Medium | Breaks BPSK receivers while AM ones still work — looks intermittent. Cover with golden vectors |
| R-2R amplitude accuracy | Low | 0.1% resistors, or monolithic DAC in production |
| GPS week rollover | Low | Current module firmware; handle epoch explicitly |
| BPC spec availability (Chinese-language sources) | Low | Defer to last; lowest commercial value |

---

## 11. Phased roadmap

| Phase | Goal | Proves |
|---|---|---|
| P0 | DCF77 AM only, square wave into a coil, time hardcoded | A watch actually syncs — de-risks the whole product in a weekend |
| P1 | NCO + R-2R + LPF + linear driver, DCF77 AM | Signal chain and CPU budget |
| P2 | DCF77 phase modulation | Hardest DSP; validates the low-Q/untuned decision |
| P3 | WWVB (incl. BPSK), MSF, JJY | Layer-2 abstraction actually holds |
| P4 | GNSS + PPS discipline + RTC holdover + DST engine | Standalone operation |
| P5 | Enclosure, UI, config, compliance | Sellable |
| P6 | BPC | Completeness |

Do P0 first and separately. It is a few hours of work and it answers the only
question that really matters: do your watches sync to a locally generated
signal at all?

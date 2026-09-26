# LF Time-Signal Generator — Architecture Sketch
## (single-standard per build, five variants from one design)

Target: a near-field transmitter that emits **one** LF time signal — DCF77,
WWVB, MSF, JJY (40/60) or BPC — to synchronise consumer radio-controlled
watches and clocks, with full fidelity including DCF77 phase modulation and
WWVB BPSK.

**The standard is fixed at build time**, not runtime-selectable: one PCB design
with a per-variant BOM (tank capacitor, PLL constants) and firmware build flag.
This constraint is exploited throughout — in §2, §3 and §6 it removes problems
rather than adding them.

**Companion documents:**
- `lf-timesignal-requirements.md` — formal design requirements
- `lf-timesignal-hardware.md` — schematic blocks, part selection, PCB design

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

## 2. Clocking and the sample rate — **coherent sampling, per variant**

Because each unit is built for **one** standard (see §2a), the system clock and
sample rate are chosen per variant so that:

```
Fs = N × f_carrier     (N = integer samples per carrier cycle)
sysclk = M × Fs        (M = integer PIO divider)
Fs = integer samples per second
```

This is *coherent sampling*, and it is only possible in a single-standard build.
It is a large simplification — see §3 for why.

| Variant | sysclk | PLL (refdiv/fbdiv/VCO/p1/p2) | Fs | N (samp/cycle) | PIO div | cycles/sample |
|---|---|---|---|---|---|---|
| DCF77 77.5k | 116.25 MHz | 2 / 155 / 930 MHz / 2 / 4 | 1.550 MS/s | 20 | 75 | 75 |
| WWVB 60k | 102.00 MHz | 1 / 68 / 816 MHz / 2 / 4 | 1.200 MS/s | 20 | 85 | 85 |
| MSF 60k | 102.00 MHz | 1 / 68 / 816 MHz / 2 / 4 | 1.200 MS/s | 20 | 85 | 85 |
| JJY 40k | 100.00 MHz | 1 / 75 / 900 MHz / 3 / 3 | 0.800 MS/s | 20 | 125 | 125 |
| BPC 68.5k | 102.75 MHz | 2 / 137 / 822 MHz / 2 / 4 | 1.370 MS/s | 20 | 75 | 75 |

All verified against RP2350 PLL constraints (VCO 750–1600 MHz, fbdiv 16–320,
postdiv 1–7). **N = 20 for every variant**, so the sine LUT, the DDS inner loop
and the DMA block sizing are identical across the whole family — only three
constants change.

Note that **BPC's awkward prime 137 is no longer a problem**: it is absorbed
directly into the PLL feedback divider (refdiv 2, fbdiv 137 → 822 MHz VCO). The
factor that made a runtime-switchable design impossible is free in a
per-variant one.

Sysclk also drops from 150 MHz to ~100–116 MHz, which reduces power and
lengthens the per-sample CPU budget.

---

## 2a. Product variants

One PCB, one firmware tree, per-variant BOM and build flags. Two complexity
tiers fall out of *which standards carry phase modulation*:

| Tier | Standards | Modulation | Signal chain |
|---|---|---|---|
| **Full** | DCF77, WWVB | AM **+ phase** (PRBS / BPSK) | NCO → R-2R → LPF → linear driver |
| **Simple** | MSF, JJY, BPC | AM / OOK only | PIO square wave → keying → tank |

The Simple tier needs no DAC, no linear amplifier and no reconstruction filter —
a PIO square wave into a tuned tank with switched drive is sufficient, because
there is no phase information to preserve. That is a materially cheaper kit.

**Recommendation: lay out one PCB that supports both**, with the R-2R ladder,
filter and linear driver as do-not-populate options on Simple-tier builds. Two
separate PCB designs would double layout, inventory and test-fixture work to
save a few euro of parts.

Per-variant differences are then: crystal/PLL constants, tank capacitor,
firmware build flag, silkscreen, and test limits.

---

## 3. The NCO — now *coherent*, which removes two problems

With Fs locked to an integer multiple of the carrier (N = 20), the NCO stops
being a general-purpose DDS and becomes something much simpler and more exact.

**The phase accumulator no longer accumulates error.** With N = 20 samples per
cycle, the carrier phase advances by exactly 1/20 turn per sample. A 20-entry
sine table *is* the carrier. There is no phase truncation, no fractional
residual, and therefore **no DDS phase-truncation spurs** — which were the main
spectral risk of the original design.

Keep a 32-bit accumulator anyway (it costs nothing and preserves the ability to
trim frequency for calibration), but the steady-state increment is now the exact
value `2^32 / 20`, and the arithmetic closes perfectly every 20 samples.

Modulation is unchanged and still trivial:
- **PM/BPSK** = add to `phase_offset` (±15.6° DCF77, 180° WWVB)
- **AM** = scale `amplitude` (15% DCF77, −17 dB ≈ 14.1% WWVB, 0% MSF, ~10% JJY)

Because a full carrier cycle is exactly 20 samples, a 180° BPSK reversal is
exactly a 10-sample index offset, and DCF77's ±15.6° needs interpolation or a
slightly larger LUT (a 256-entry table indexed by the accumulator's high bits
handles both cleanly).

### The DCF77 chip-alignment problem disappears

Previously flagged as a trap: at 2 MS/s a DCF77 PM chip was 3096.774 samples —
not an integer — so a sample counter would drift against the chip clock.

With coherent sampling this vanishes:

```
chip = 120 carrier cycles × 20 samples/cycle = 2400 samples, exactly
```

No overflow-counting workaround, no drift, no self-aligning trickery. A plain
sample counter is now exact. Similarly one second = 1 550 000 samples exactly,
so second boundaries and chip boundaries stay locked forever.

This is the single strongest argument for the per-variant approach.

### DCF77 PM timing, in samples (verified self-consistent)

| Event | Time | Samples @ 1.55 MS/s |
|---|---|---|
| 1 carrier cycle | 12.9 µs | 20 |
| 1 second | 1 s | 1 550 000 |
| 1 PM chip (120 cycles) | 1.5484 ms | 2 400 |
| PM start | +200 ms | 310 000 |
| PM duration (512 chips) | 792.77 ms | 1 228 800 |
| PM end | +992.77 ms | 1 538 800 |
| Guard before next second | 7.23 ms | 11 200 |
| AM drop, bit 0 | 100 ms | 155 000 |
| AM drop, bit 1 | 200 ms | 310 000 |
| AM ramp | 1.0 ms | 1 550 |

512 chips at 645.833 Hz = 792.77 ms, starting at +200 ms, ending at +992.77 ms —
inside the second with 7.23 ms to spare. The +200 ms start is exactly what keeps
PM clear of the longest AM drop, satisfying the "no phase modulation during the
amplitude drop" constraint structurally rather than by special-casing.

Every quantity above is an exact integer number of samples.

---

## 3a. Epoch discipline without touching the carrier

A subtle but important consequence of coherent sampling.

The carrier NCO **free-runs and is never steered**. Because f_c = F_s/20 is
structural, a reference-oscillator error scales the carrier, the chip rate and
the second length *together* — so a DCF77 chip stays exactly 120 carrier cycles
no matter how far off the crystal is.

Timing is therefore disciplined in the **event scheduler**, not the oscillator:
maintain a fractional samples-per-second accumulator (nominally 1 550 000) and
steer it against GNSS PPS. Because the carrier accumulator is untouched, this
introduces **no carrier phase glitch** — which would otherwise corrupt PM and
BPSK, and would be very hard to diagnose.

Consequences:
- Carrier accuracy requirement is loose (±10 ppm), so the reference is chosen
  for **holdover stability**, not absolute accuracy.
- Steering resolution is unlimited (fixed-point remainder), not quantised to
  whole samples.
- Never implement discipline by inserting/dropping samples: at 20 samples per
  cycle, one sample is an 18° phase step.

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

**Holdover is the binding constraint, not acquisition.** Measured budget:

| Reference | Residual | Drift | Reaches 100 ms in |
|---|---|---|---|
| Plain XTAL, uncorrected | 30 ppm | 2.6 s/day | 1 hour |
| Plain XTAL, ppm learned | ~3 ppm | 259 ms/day | 9 hours |
| TCXO 2 ppm | 2 ppm | 173 ms/day | 14 hours |
| **TCXO 0.5 ppm** | 0.5 ppm | 43 ms/day | **2.3 days** |

Even a good TCXO holds ±100 ms for only about two days, so **"get one fix at
setup and run forever" is not viable**. The design answer is duty-cycled GNSS:
re-acquire hourly (cheap in power, ~seconds of tracking), and treat true
holdover as a degraded mode specified at ±500 ms over 7 days — which 0.5 ppm
does meet.

This also means the GNSS antenna should be external and user-placeable, not a
patch buried in the enclosure.

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

## 6. Output stage — now tuned, but *not* high-Q

Per-variant tuning removes the objections that forced an untuned loop in the
runtime-switchable design: a single fixed resonant frequency means **no switched
capacitor bank and no per-band phase calibration**. So yes — resonate.

The question becomes *how high a Q*, and there are two independent ceilings.

### Ceiling 1 — modulation bandwidth

`BW = f0 / Q`, ring-down `τ = Q / (π·f0)`.

| Variant | Constraint | Required BW | Q ceiling |
|---|---|---|---|
| DCF77 | PM chips at 645.83 Hz | ≳ 3 kHz | **~25** |
| WWVB | BPSK reversal, 1 baud | transient ≪ 1 s | ~100 |
| MSF | OOK, 100 ms off | ring-down ≪ 100 ms | ~100 |
| JJY / BPC | AM, ≥ 200 ms | loose | ~100 |

Only DCF77 is genuinely tight. At Q = 25 its ring-down is 103 µs — negligible
against a 100 ms AM step, while still passing the phase chips.

### Ceiling 2 — manufacturability (this is the binding one for a kit)

At Q = 50 and 77.5 kHz the passband is 1550 Hz. A ±5% capacitor shifts
resonance by ~2.5%, i.e. ~1900 Hz — **further than the bandwidth is wide**. The
unit would ship detuned off its own peak.

High Q therefore forces either 1% C0G capacitors *and* a per-unit tuning step,
or a trimmer the customer must adjust with test gear they don't own. For a kit,
that is a support disaster.

**Q ≈ 20–25 is the sweet spot, and it is the same answer for every variant.**
It satisfies DCF77's phase modulation, tolerates ±2–3% component spread with no
per-unit tuning, and keeps one Q target across the whole product family.

Use **C0G/NP0 capacitors** regardless — X7R's temperature and DC-bias drift
would detune the tank across the operating range.

### Don't over-value the range gain

Field strength in a series-resonant loop scales with Q, but near-field coupling
falls as 1/r³. So going Q = 25 → 100 (4× current) buys only ∛4 ≈ **1.6× range**.
Paying for per-unit tuning to gain 60% more distance on a device meant to work
at a few centimetres is a bad trade.

### Worked example (DCF77)

L = 1 mH at 77.5 kHz → ωL = 487 Ω. C = 4.22 nF (C0G).
For Q = 25, total series R = 19.5 Ω. Ring-down τ = 103 µs. BW = 3.1 kHz.

### Drive requirement — no power amplifier needed

Computed field strength settles this. A 100-turn, 5 cm-radius loop at **10 mA
peak** sits roughly **65 dB under FCC §15.209** and **70 dB under EN-300-330**-
style limits, while still delivering a very strong near-field at watch distance.

| Coil current | Drive voltage | Power | Voltage across L |
|---|---|---|---|
| 10 mA pk | 0.19 V pk | 1.0 mW | 4.9 V pk |
| 30 mA pk | 0.58 V pk | 8.8 mW | 14.6 V pk |
| 50 mA pk | 0.97 V pk | 24.3 mW | 24.3 V pk |

The "linear amplifier" in the original block diagram is therefore just an
op-amp: 1 mW, 0.19 V peak, and a slew-rate requirement of 0.49 V/µs. Select for
distortion, not for power.

⚠️ **The voltage across the inductor is Q× the drive** — 24 V peak at 50 mA on a
5 V system. Rate the tank capacitor ≥50 V. This is the standard resonant-circuit
trap and the one place in this design where a 5 V assumption will destroy parts.

### DAC and filter (Full tier only)

10-bit R-2R, PIO+DMA driven, 0.1% resistors (1% parts give ~7 effective bits;
WWVB's −17 dB level wants ~1% amplitude accuracy). Reconstruction LPF at
~120 kHz. Phase resolution lives in the NCO, not the DAC — DAC bits bound only
amplitude accuracy and the spurious floor.

Simple-tier variants (MSF/JJY/BPC) omit all of this: PIO square wave → keyed
drive → tank. The tank's own selectivity suppresses the square wave's harmonics.

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

Fits the 30-GPIO QFN-60 package with margin. Simple-tier variants free the 10
R-2R pins, leaving room for a cheaper/smaller package if a cost-reduced SKU is
ever wanted.

---

## 8. Configuration and UX

The standard is **fixed at build time** (BOM + firmware flag), so there is no
runtime standard selector. What remains configurable in flash:

- TX power trim, DST rule parameters, learned TCXO ppm, transmit window.
- No OLED/encoder needed for standard selection — a status LED plus USB-C serial
  config is sufficient, which cuts BOM and enclosure cost.
- **Transmit window scheduler**: most radio-controlled watches only attempt sync
  overnight (typically 02:00–04:00), so the device can idle most of the day.
  Cuts duty cycle, power and regulatory exposure — and is a genuine product
  feature ("only radiates 20 minutes a night").
- Provide a manual "sync now" button for watches that support forced receive;
  without it, development iterations take hours.
- **Have the firmware refuse to run if the build flag and a hardware strap
  disagree.** A DCF77 board flashed with WWVB firmware would transmit off-tune
  into a Q=25 tank and mostly just not work — an expensive support ticket.
  A single resistor strap per variant makes this self-detecting.

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
- **Per-variant production test**: sweep the tank and confirm resonance lands
  within ±1% of target. Cheap to automate with the same audio interface, and it
  is the one build defect (wrong/out-of-tolerance capacitor) most likely to
  escape a kit builder.

---

## 10. Risks

| Risk | Severity | Mitigation |
|---|---|---|
| Regulatory compliance **per variant per region** | **High** | Each variant is a separate frequency in a separate jurisdiction — FCC (WWVB), CE/UKCA (DCF77/MSF), MIC (JJY, notoriously strict). Cost multiplies per variant, so launch **one** variant first and treat others as gated business decisions, not engineering tasks |
| GNSS never gets a fix indoors | High | RTC holdover + learned TCXO correction; only needs one fix at setup; USB time-set fallback |
| Kit builder mis-tunes or mis-stuffs the tank | Medium | Q≈25 tolerates ±2–3% spread; ship pre-matched L/C pair; hardware strap + firmware self-check |
| DCF77 PRBS spec details (sequence, start offset within the second) | Medium | Obtain PTB specification before committing to layer-2 design |
| WWVB BPSK +0.1 s offset implemented wrong | Medium | Breaks BPSK receivers while AM ones still work — looks intermittent. Cover with golden vectors |
| R-2R amplitude accuracy (Full tier only) | Low | 0.1% resistors, or monolithic DAC in production |
| GPS week rollover | Low | Current module firmware; handle epoch explicitly |
| BPC spec availability (Chinese-language sources) | Low | Defer to last; lowest commercial value |

Note that selling a **kit** rather than a finished product does not reliably
reduce regulatory obligation — do not assume it does without advice.

---

## 11. Phased roadmap

Build **one variant end-to-end** before generalising. DCF77 first: it is your
own use case and it is the hardest (phase modulation + tightest Q ceiling), so
it validates every constraint the other variants will face.

| Phase | Goal | Proves |
|---|---|---|
| P0 | DCF77 AM only, square wave into a coil, time hardcoded | A watch actually syncs — de-risks the whole product in a weekend |
| P1 | NCO + R-2R + LPF + linear driver + Q≈25 tank, DCF77 AM | Coherent-sampling signal chain and CPU budget |
| P2 | DCF77 phase modulation | Hardest DSP; validates the Q≈25 ceiling |
| P3 | GNSS + PPS discipline + RTC holdover + DST engine | Standalone operation — **first sellable product** |
| P4 | Second variant (WWVB or MSF) | Layer-2 abstraction and the per-variant BOM actually hold |
| P5 | Enclosure, config, per-region compliance | Sellable abroad |
| P6 | JJY, BPC | Completeness |

Compared with the runtime-switchable plan, GNSS/DST moves **earlier** (it is
needed for a shippable single-standard unit) and multi-standard work moves
**later** (it is now a business decision gated on compliance cost, not a
prerequisite).

Do P0 first and separately. It is a few hours of work and it answers the only
question that really matters: do your watches sync to a locally generated
signal at all?

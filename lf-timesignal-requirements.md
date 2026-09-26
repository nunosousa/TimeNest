# LF Time-Signal Generator — Design Requirements

Document status: **draft for review**. Requirement IDs are stable; values marked
🔶 are provisional and need confirmation against the cited primary spec or a
compliance authority before design freeze.

Companion documents:
- `lf-timesignal-architecture.md` — system architecture and rationale
- `lf-timesignal-hardware.md` — schematic/PCB design

---

## 0. Product definition

A mains- or USB-powered near-field transmitter that emits **one** LF time-code
signal so that consumer radio-controlled watches and clocks can synchronise
indoors, where the real broadcast is unusable.

**Primary variant: DCF77.** Others are per-build variants of the same PCB.

Two tiers, determined by whether the standard carries phase modulation:

| Tier | Variants | Signal chain |
|---|---|---|
| **Full** | DCF77, WWVB | NCO → DAC → LPF → linear driver → tank |
| **Simple** | MSF, JJY40, JJY60, BPC | PIO square wave → keyed drive → tank |

---

## 1. Functional requirements

| ID | Requirement | Verification |
|---|---|---|
| F-01 | Transmit a complete, standard-conformant time code for the variant's standard, continuously, one frame per minute. | Decode with reference receiver |
| F-02 | Synchronise at least one real production radio-controlled watch to the correct time and date. | Physical test, ≥3 watch models |
| F-03 | Derive UTC from GNSS without any network connection. | Functional test, airgapped |
| F-04 | Derive the correct broadcast-local time (incl. DST) algorithmically, with no tzdata and no network. | Unit test sweep, §5 |
| F-05 | Maintain correct output through a total loss of GNSS for the holdover period (P-05). | Bench test with antenna disconnected |
| F-06 | Retain time across a power cycle via battery-backed RTC. | Power-cycle test |
| F-07 | Emit DCF77 phase modulation (Full/DCF77 variant). | PM correlation decode |
| F-08 | Emit WWVB BPSK (Full/WWVB variant). | BPSK decode |
| F-09 | Provide a user-initiated "transmit now" action. | Functional |
| F-10 | Support a scheduled transmit window (default: enabled, 01:00–05:00 local). | Functional |
| F-11 | Refuse to transmit if the firmware variant and hardware strap disagree; indicate the fault. | Fault-injection test |
| F-12 | Expose a USB serial console for configuration, status and diagnostics. | Functional |
| F-13 | Accept an external UTC source over USB as a GNSS fallback. | Functional |
| F-14 | Indicate operating state (no time / holdover / locked / transmitting / fault) on an indicator. | Visual inspection |

### Explicit non-goals

| ID | Non-goal |
|---|---|
| NG-01 | Runtime switching between standards. Fixed per build. |
| NG-02 | Useful range beyond ~0.5 m. Range is deliberately bounded (see R-03). |
| NG-03 | Network/Wi-Fi connectivity. |
| NG-04 | Battery operation. USB-powered. |
| NG-05 | Substituting for a traceable laboratory time standard. |

---

## 2. Performance requirements

### Timing

| ID | Requirement | Value | Notes |
|---|---|---|---|
| P-01 | Second-marker error vs UTC, GNSS locked | **≤ 10 ms** | Dominated by firmware scheduling, not GNSS |
| P-02 | Second-marker error, ≤24 h since last fix | ≤ 50 ms | |
| P-03 | Internal epoch resolution | 1 sample (≤ 0.65 µs) | Sample-counted scheduler |
| P-04 | GNSS PPS capture jitter | ≤ 1 µs | |
| P-05 | **Holdover: error after 7 days with no GNSS** | **≤ 500 ms** | Requires ≤ 0.83 ppm — see note |
| P-06 | Time-to-first-transmit from cold start with GNSS | ≤ 5 min | |
| P-07 | Time-to-first-transmit from RTC only | ≤ 5 s | Degraded-accuracy mode |

> **Holdover drives the reference choice.** 7 days at ±500 ms permits only
> 0.83 ppm. A 0.5 ppm TCXO gives ~300 ms over 7 days (pass); a 2 ppm part gives
> ~1.2 s (fail). Note this is *residual* stability after GNSS has taught the
> firmware the part's offset — absolute initial accuracy is irrelevant.
> **Mitigation:** duty-cycled GNSS re-acquisition (default hourly) keeps normal
> operation far inside P-02, so P-05 only applies to a genuinely blind unit.

### Signal quality

| ID | Requirement | Value | Notes |
|---|---|---|---|
| S-01 | Carrier frequency error | ≤ ±10 ppm | Receiver front-ends are far looser; easily met |
| S-02 | Carrier frequency, coherence | f_c = F_s / 20 exactly | Structural, not a tolerance |
| S-03 | AM low-level accuracy (DCF77 15%) | ±1.5% absolute 🔶 | Confirm against PTB |
| S-04 | AM low-level accuracy (WWVB −17 dB) | ±1 dB | |
| S-05 | AM transition ramp time | 0.5–2 ms, monotonic 🔶 | Limits splatter; confirm shape vs PTB |
| S-06 | AM timing accuracy (drop start/length) | ≤ ±1 ms | |
| S-07 | PM phase deviation (DCF77) | ±15.6° ± 0.5° | |
| S-08 | PM chip rate | f_c/120 = 645.833 Hz exactly | Structural: 2400 samples/chip |
| S-09 | PM sequence | 512 chips, start +200 ms, end +992.77 ms | Verified consistent; obtain PRBS polynomial from PTB |
| S-10 | PM suppressed during AM drop | Yes | Guaranteed by the +200 ms start |
| S-11 | BPSK phase offset vs AM step (WWVB) | +100 ms ± 1 ms | Deliberate in the NIST spec |
| S-12 | Spurious/harmonic at antenna | ≤ −50 dBc 🔶 | Set by DAC + LPF + tank |
| S-13 | Sampling image rejection | ≥ 55 dB | 3rd-order LPF at 2×f_c gives ~59 dB |
| S-14 | Amplitude stability over temperature | ≤ ±5% over 0–40 °C | |

### Field strength and range

| ID | Requirement | Value | Notes |
|---|---|---|---|
| R-01 | Reliable watch sync distance | ≥ 5 cm, aligned | With ferrite-rod antenna |
| R-02 | Typical usable distance | 10–30 cm | |
| R-03 | **Design-bounded maximum** | ≤ 1 m | Deliberate: compliance and interference safety |
| R-04 | Output level adjustable | ≥ 6 dB software trim | Plus per-build analog gain resistor |

---

## 3. Regulatory requirements

> ⚠️ **These are design targets, not a compliance assessment.** The values below
> are engineering estimates for internal use. Formal applicability, measurement
> distance, extrapolation factors and test method **must** be confirmed with a
> qualified compliance body before sale. Selling a kit does not reliably reduce
> obligation.

| ID | Requirement | Notes |
|---|---|---|
| G-01 | Design margin ≥ 20 dB below the applicable radiated-emission limit | Calculated margin is ~65–70 dB at 10 mA drive |
| G-02 | Comply with FCC 47 CFR §15.209 (WWVB variant, US) 🔶 | Limit ≈ 31 µV/m at 300 m at 77.5 kHz |
| G-03 | Comply with EN 300 330 + EMC/LVD for CE/UKCA (DCF77, MSF variants) 🔶 | |
| G-04 | Japan MIC conformity (JJY variants) 🔶 | Known to be strict; treat as a separate business decision |
| G-05 | Conducted USB emissions within limits | |
| G-06 | RoHS/REACH compliant BOM | |
| G-07 | Not marketed or capable of interfering beyond the premises | Supported by R-03 |

**Estimated margin** (100-turn, 5 cm radius loop, 10 mA peak):

| Distance | Field | Reference |
|---|---|---|
| 10 cm | 119 dBµA/m | operating point |
| 10 m | 1.9 dBµA/m | vs EN-style 72 dBµA/m → **~70 dB margin** |
| 300 m | 0.017 µV/m equiv | vs FCC 31 µV/m → **~65 dB margin** |

Near-field 1/r³ rolloff dominates. Using E = 377·H is **conservative** here: a
magnetic dipole's near-field wave impedance is below 377 Ω, so true E is lower
than the figures above.

---

## 4. Interface requirements

| ID | Requirement |
|---|---|
| I-01 | USB-C receptacle, USB 2.0 full-speed, 5 V input, ≤ 500 mA |
| I-02 | USB CDC serial console, 115200 8N1, line-based text protocol |
| I-03 | USB Mass Storage bootloader (BOOTSEL) for firmware update — no programmer needed |
| I-04 | SWD header (2.54 mm or Tag-Connect) for development |
| I-05 | GNSS active antenna via u.FL or SMA, with switchable antenna bias |
| I-06 | Antenna coil on a 2-pin polarised connector, keyed, per-variant matched assembly |
| I-07 | CR2032 holder for RTC backup |
| I-08 | Variant strap: 3 GPIO, resistor-configured, 8 possible variants |
| I-09 | Test points: sample clock, DAC out, LPF out, driver out, coil current, PPS in, PPS out |

---

## 5. Time and DST rule requirements

| ID | Requirement |
|---|---|
| T-01 | DST/offset rules stored as **parameters in flash**, not hardcoded |
| T-02 | DCF77: CET/CEST, EU rule (last Sun Mar / last Sun Oct, 01:00 UTC) |
| T-03 | MSF: UTC/BST, same instants as the EU rule |
| T-04 | WWVB: UTC + DST status bits, US rule (2nd Sun Mar / 1st Sun Nov) |
| T-05 | JJY: JST = UTC+9, no DST, ever |
| T-06 | BPC: UTC+8, no DST, ever |
| T-07 | DCF77 A1 (bit 16) set during the hour preceding a DST change |
| T-08 | DCF77 A2 (bit 19) set for an announced leap second |
| T-09 | Leap-second state taken from the GNSS navigation message (current + pending) |
| T-10 | Correct behaviour across a leap-second insertion |
| T-11 | Unit tests sweep ≥20 years of transitions, both directions, both hemispheres of the rule |
| T-12 | Frame encodes the **upcoming** minute, transmitted from second 0 |

---

## 6. Environmental and mechanical

| ID | Requirement | Value |
|---|---|---|
| E-01 | Operating temperature | 0 to +40 °C |
| E-02 | Storage temperature | −20 to +60 °C |
| E-03 | Humidity | 10–90% RH non-condensing |
| E-04 | Enclosure | Desktop, non-conductive (plastic) — must not shield the coil |
| E-05 | Max dimensions (main PCB) | 80 × 60 mm |
| E-06 | Antenna | External ferrite rod on short lead, user-orientable |
| E-07 | ESD | ±4 kV contact on exposed connectors 🔶 |

> **E-04 and E-06 are coupled and easy to get wrong.** A watch's receiver uses a
> ferrite rod antenna; coupling is strongly orientation-dependent. A flat coil
> under a watch laid on top produces a field largely orthogonal to the watch's
> rod — poor coupling. A ferrite rod the user can lay *parallel* to the watch is
> markedly better and is the recommended form factor.

---

## 7. Power

| ID | Requirement | Value |
|---|---|---|
| W-01 | Input | 5 V ±5% from USB-C |
| W-02 | Total power, transmitting | ≤ 1.5 W |
| W-03 | Total power, idle (outside transmit window) | ≤ 0.5 W |
| W-04 | RF power into antenna | ~1–25 mW (see R-04) |
| W-05 | RTC backup current | ≤ 3 µA (≥ 5 years on CR2032) |
| W-06 | Analog supply noise at DAC reference | ≤ 100 µVrms, 10 Hz–1 MHz |
| W-07 | Inrush compliant with USB spec | |

---

## 8. Manufacturing and kit requirements

| ID | Requirement |
|---|---|
| M-01 | No per-unit tuning or calibration requiring test equipment |
| M-02 | Tank tolerates ±3% L and C spread without retuning (satisfied at Q ≈ 25) |
| M-03 | Antenna supplied as a pre-wound, pre-matched assembly with its capacitor |
| M-04 | Fine-pitch/QFN parts factory-assembled; kit content limited to through-hole and connectors |
| M-05 | Production test: verify resonance within ±1% of target, verify AM depth, verify a decoded frame |
| M-06 | Firmware self-test at boot: strap match, RTC present, GNSS comms, DAC continuity |
| M-07 | Per-variant differences limited to: PLL constants, tank capacitor, filter RC, gain resistor, build flag, silkscreen |

---

## 9. Software requirements (summary)

| ID | Requirement |
|---|---|
| SW-01 | Three-layer separation: timebase / standard / modulator (see architecture §4) |
| SW-02 | Frame encoders are pure functions of civil time → bits; no I/O, fully unit-testable on host |
| SW-03 | Golden-vector tests: known timestamp → expected bit pattern, per standard |
| SW-04 | Modulator runs from DMA double-buffering; must never underrun |
| SW-05 | Sample-generation CPU load ≤ 50% of one core, measured |
| SW-06 | Learned TCXO ppm correction persisted to flash, with sanity bounds |
| SW-07 | Fractional samples-per-second accumulator for epoch discipline (see architecture §3a) |
| SW-08 | Watchdog; safe state = transmitter off |
| SW-09 | Firmware update over USB MSC without special tools |

---

## 10. Open items

| # | Item | Blocks | Owner |
|---|---|---|---|
| 1 | Obtain PTB DCF77 PM specification: PRBS polynomial, seed, exact chip alignment | F-07, S-09 | — |
| 2 | Confirm DCF77 AM envelope shape/ramp spec | S-05 | — |
| 3 | Compliance pre-assessment for the launch variant and region | G-01…G-04 | — |
| 4 | Select GNSS module; confirm PPS spec and duty-cycle behaviour | P-04, P-05 | — |
| 5 | Confirm RP2350 XIN external-clock drive levels for TCXO | hardware | — |
| 6 | Decide launch region and therefore launch variant | all | — |
| 7 | Measure real watch sensitivity to set R-01 drive level empirically | R-01, R-04 | — |

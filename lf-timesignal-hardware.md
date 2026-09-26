# LF Time-Signal Generator — Hardware Design

Companion to `lf-timesignal-requirements.md` and `lf-timesignal-architecture.md`.
Requirement IDs (F-xx, P-xx, S-xx…) refer to the requirements document.

Values marked 🔶 need confirmation against a datasheet or a prototype
measurement before design freeze.

---

## 1. Block diagram (Full tier)

```
 USB-C 5V ──┬─ ESD ─┬─ LDO 3.3V DIGITAL ──── RP2350A, GNSS, RTC
            │       └─ LDO 3.0V ANALOG  ──┬─ R-2R buffers (reference!)
            │                              └─ op-amp rails
            └─ CC resistors

 TCXO 12 MHz 0.5ppm ──► XIN

 GNSS module ──UART──► RP2350      u.FL ──► active antenna
             ──PPS───►

 DS3231 + CR2032 ──I2C──► RP2350

 RP2350 GPIO0..9 ──► 74LVC244 ×2 ──► R-2R ladder ──► 3rd-order LPF
                       (clean 3.0V)                        │
                                                            ▼
                                            op-amp driver ──► Rsense ──► tank
                                                  │                      │
                                            ADC ◄─┴─ current sense       │
                                                                          ▼
                                                        external ferrite rod
```

Simple tier (MSF/JJY/BPC) depopulates the buffers, ladder, LPF and op-amp;
a PIO square wave drives the tank through a keyed gate stage.

---

## 2. Part selection

| Block | Part | Rationale / notes |
|---|---|---|
| MCU | **RP2350A**, QFN-60, 30 GPIO | Confirmed 30 GPIO — enough (§4). Dual M33, FPU, 12 PIO SM. 1.1 V core, 3.3 V IOVDD |
| Flash | W25Q32JV QSPI 4 MB | Ample; stores config + learned ppm |
| Reference | **12.000 MHz TCXO, ±0.5 ppm**, CMOS out | Driven by P-05 holdover. See §3 for XIN caveat 🔶 |
| GNSS | u-blox **MAX-M10S** (alt: NEO-M9N; budget: ATGM336H) | Low power, good PPS, duty-cycle friendly |
| GNSS antenna | Active patch, u.FL/SMA, 3.3 V bias | External so the user can place it near a window (P-05 mitigation) |
| RTC | **DS3231SN** + CR2032 | Integrated TCXO, ±2 ppm; backup only, not the transmit reference |
| DAC buffers | 2 × **74LVC244A** (or 1 × 74LVC16244) | Low, matched output impedance; isolates ladder from noisy IOVDD |
| Ladder | R = 10 kΩ / 2R = 20 kΩ, **0.1 % thin film** | See §5 for the tolerance analysis |
| Analog LDO | Low-noise 3.0 V (e.g. TPS7A20 / ADP7142 class) | This rail **is** the DAC reference — W-06 applies |
| Op-amp | Rail-to-rail, ≥ 50 mA out, ≥ 2 MHz GBW, low distortion | Requirement is tiny (§6) — many parts qualify 🔶 |
| Tank C | C0G/NP0, 2 %, ≥ 50 V | X7R drift would detune the tank |
| Antenna | Ferrite rod, ~1 mH, external lead | Orientation matters (E-06) |
| Connector | USB-C receptacle, 5 V only | |

---

## 3. Clock and timebase

**TCXO 12 MHz → RP2350 XIN.** PLL constants per variant (verified reachable:
VCO 750–1600 MHz, fbdiv 16–320, postdiv 1–7):

| Variant | refdiv | fbdiv | VCO | p1 | p2 | sysclk | PIO div | Fs |
|---|---|---|---|---|---|---|---|---|
| DCF77 | 2 | 155 | 930 MHz | 2 | 4 | 116.25 MHz | 75 | 1.550 MS/s |
| WWVB / MSF | 1 | 68 | 816 MHz | 2 | 4 | 102.00 MHz | 85 | 1.200 MS/s |
| BPC | 2 | 137 | 822 MHz | 2 | 4 | 102.75 MHz | 75 | 1.370 MS/s |
| JJY40 | 1 | 75 | 900 MHz | 3 | 3 | 100.00 MHz | 125 | 0.800 MS/s |

🔶 **Open item 5:** RP2350's XIN is a crystal-oscillator input; external-clock
drive level is constrained. Provide **both** a series-resistor/AC-coupled XIN
path *and* an alternate route to **GPIO20 (clk_gpin0)**, with the unused one
DNP. Decide after measuring on the first prototype. GPIO20 is reserved for this
reason.

### Why frequency accuracy is *not* the hard part

Carrier accuracy needs only ±10 ppm (S-01) — receiver front-ends are far
looser. What actually matters is **epoch** accuracy, and that is disciplined in
software: the scheduler keeps a fractional samples-per-second accumulator
(nominally 1 550 000) and steers it against GNSS PPS.

Crucially, the carrier NCO **free-runs** and is never steered. Because
f_c = F_s/20 structurally, a reference error scales carrier, chip rate and
second length together, so DCF77 chips remain exactly 120 carrier cycles
regardless of reference error. Steering the *scheduler* therefore introduces
**no carrier phase glitch** — which would otherwise corrupt PM/BPSK. This is
why the TCXO is a holdover part, not an accuracy part.

---

## 4. RP2350A pin allocation

| GPIO | Function | Notes |
|---|---|---|
| 0–9 | R-2R D0–D9 | **Must be contiguous** for PIO parallel OUT |
| 10 | Sample-clock test point | PIO side-set |
| 11 | Amplifier enable / mute | Safe state = muted |
| 12 | GNSS UART TX | |
| 13 | GNSS UART RX | |
| 14 | GNSS PPS in | PIO capture, ≤1 µs (P-04) |
| 15 | GNSS reset / enable | Duty-cycling |
| 16 | I2C0 SDA | DS3231 |
| 17 | I2C0 SCL | |
| 18 | RTC SQW / INT | 1 Hz cross-check |
| 19 | Variant strap 0 | |
| **20** | **Reserved: clk_gpin0** | Alternate TCXO route, DNP |
| 21 | Variant strap 1 | |
| 22 | Variant strap 2 | |
| 23 | Status LED | |
| 24 | User button ("sync now", F-09) | |
| 25 | PPS out test point | Bring-up and production test |
| 26 | ADC0 — coil current sense | Amplitude self-check, M-05 |
| 27 | ADC1 — 5 V rail monitor | |
| 28, 29 | ADC2/3 spare | |

30 GPIO, fully allocated with two spares. Simple-tier builds free GPIO0–9.

**Variant strap (I-08):** 3 GPIO, pull-up/down per build → 8 codes. Firmware
compares against its build flag and refuses to transmit on mismatch (F-11). A
DCF77 board running WWVB firmware would drive an off-tune Q≈25 tank and mostly
just not work — an expensive and confusing support case.

---

## 5. DAC: 10-bit R-2R

### Why buffered, not direct GPIO

Driving the ladder straight from GPIO makes **IOVDD the amplitude reference** —
and IOVDD carries the MCU's switching noise, which would appear as amplitude
modulation on the carrier. GPIO output impedance also varies with drive
strength and temperature.

So: 10 lines → 74LVC244 buffers → ladder, buffers powered from the clean 3.0 V
analog LDO. The buffers provide low, *matched* impedance and make the reference
a deliberate choice rather than an accident.

### Ladder value — computed trade-off

MSB-branch error from buffer output impedance R_o is R_o/(2R):

| R | R_o = 5 Ω | R_o = 10 Ω | R_o = 50 Ω | Thermal noise (200 kHz BW) |
|---|---|---|---|---|
| 2 kΩ | 1.3 LSB | 2.6 LSB | 12.8 LSB | 2.6 µVrms |
| 5 kΩ | 0.5 LSB | 1.0 LSB | 5.1 LSB | 4.1 µVrms |
| **10 kΩ** | **0.3 LSB** | **0.5 LSB** | 2.6 LSB | 5.8 µVrms |
| 20 kΩ | 0.1 LSB | 0.3 LSB | 1.3 LSB | 8.1 µVrms |

**R = 10 kΩ / 2R = 20 kΩ.** Sub-LSB impedance error with realistic buffers,
noise still negligible, and settling stays fast enough at 1.55 MS/s.

### Resistor tolerance

| Tolerance | Worst-case INL | Effective bits |
|---|---|---|
| 1 % | ±1 % FS = 10 LSB | ~7 |
| **0.1 %** | ±0.1 % FS = 1 LSB | ~10 |

0.1 % is required. Use a **thin-film network array** rather than 20 discretes —
better matching and tracking, fewer placements, lower cost. Prototype with
discretes.

### Why 10 bits

WWVB's −17 dB level is 14.13 % FS and DCF77's is 15 % FS. At 10-bit the LSB is
0.098 % FS, so the low level resolves to 0.65 % relative — comfortably inside
S-03/S-04. 8-bit would give 2.8 % relative: marginal. 12-bit buys nothing here
and costs two GPIO.

**Phase resolution lives in the NCO, not the DAC.** A 10-bit phase LUT
represents 15.6° to within 0.13°, well inside S-07.

---

## 6. Analog chain

### Reconstruction filter

3rd-order Butterworth, f_co = 2 × f_c (Sallen-Key 2nd order + passive RC):

| Variant | f_co | Image freq | Image rejection | Droop at f_c |
|---|---|---|---|---|
| DCF77 | 155 kHz | 1472.5 kHz | ~59 dB | 0.07 dB |
| WWVB/MSF | 120 kHz | 1140 kHz | ~59 dB | 0.07 dB |
| JJY40 | 80 kHz | 760 kHz | ~59 dB | 0.07 dB |

Meets S-13 with margin. For DCF77, C = 1 nF → R ≈ 1.02 kΩ. The filter's phase
shift at f_c (~80°) is **constant** and therefore harmless — it delays the whole
signal equally and does not distort the PM.

### Driver — much smaller than expected

Series-resonant, Q = 25, L = 1 mH at 77.5 kHz → R_total = 19.5 Ω:

| Coil current | Drive voltage | Power | Voltage across L |
|---|---|---|---|
| 10 mA pk | 0.19 V pk | 1.0 mW | 4.9 V pk |
| 30 mA pk | 0.58 V pk | 8.8 mW | 14.6 V pk |
| 50 mA pk | 0.97 V pk | 24.3 mW | 24.3 V pk |

**No power amplifier is needed.** At the 10 mA operating point (≈65 dB under
FCC, §3 of requirements) the driver delivers 1 mW into 19.5 Ω. Slew rate needed
is only 0.49 V/µs. Almost any decent op-amp with ≥50 mA output capability works;
select for low distortion rather than power.

⚠️ **Watch the voltage across the inductor: it is Q× the drive.** At 50 mA that
is 24 V peak across L and the tank capacitor, on a 5 V system. Hence the ≥50 V
C0G rating. This is the classic resonant-circuit trap.

### Output level

Coarse gain is a per-build resistor; fine trim is ≥6 dB in software (R-04).
Do *not* implement the full adjustment range in software — scaling the DAC
codes costs resolution directly.

### Current sense

Small series resistor into ADC0. Enables amplitude self-check (M-06),
production test (M-05), and detecting a disconnected or shorted antenna.

---

## 7. Tank and antenna

Per-variant, Q ≈ 25, L = 1 mH:

| Variant | f_0 | X_L | C (C0G) | R_total | BW | Ring-down τ |
|---|---|---|---|---|---|---|
| DCF77 | 77.5 kHz | 486.9 Ω | 4.217 nF | 19.5 Ω | 3100 Hz | 103 µs |
| WWVB/MSF | 60 kHz | 377.0 Ω | 7.036 nF | 15.1 Ω | 2400 Hz | 133 µs |
| BPC | 68.5 kHz | 430.4 Ω | 5.398 nF | 17.2 Ω | 2740 Hz | 116 µs |
| JJY40 | 40 kHz | 251.3 Ω | 15.831 nF | 10.1 Ω | 1600 Hz | 199 µs |

Q ≈ 25 satisfies DCF77's ~3 kHz PM bandwidth need *and* tolerates ±3 % component
spread without per-unit tuning (M-01, M-02). Ring-down is ≤200 µs — negligible
against a 100 ms AM step.

**Ship the antenna as a matched L+C assembly** (M-03). Coil inductance varies
with winding and ferrite grade; pairing the capacitor to the measured coil at
the supplier removes the one build variable that would otherwise need customer
test equipment.

Form factor: external ferrite rod on a short lead, so the user can lay it
parallel to the watch (E-06 — coupling is strongly orientation-dependent).

---

## 8. Power tree

```
USB-C 5V ─ ESD ─ ferrite ─┬─ LDO1 3.3V ─── RP2350 IOVDD, GNSS, RTC, flash
                          │
                          └─ LDO2 3.0V ─┬─ 74LVC244 buffers  ← DAC REFERENCE
                             low-noise    └─ op-amp
```

**LDO2 is the amplitude reference and needs W-06 treatment** (≤100 µVrms,
10 Hz–1 MHz): low-noise part, generous output capacitance, separate from all
digital loads. Any noise here appears directly as AM on the carrier.

Budget: RP2350 ~100 mW, GNSS ~90 mW tracking (far less duty-cycled), analog
~50 mW, RF ~1–25 mW. Comfortably inside W-02's 1.5 W.

GNSS duty-cycling (default hourly re-acquisition) serves both W-03 and the P-05
holdover mitigation.

---

## 9. PCB

**4-layer, 1.6 mm:** Signal+GND / **solid GND** / 3V3+3V0 planes / Signal.
A continuous reference plane under the analog chain is worth more than the
2-layer cost saving on a mixed-signal board.

### Floorplan

```
 ┌──────────────────────────────────────────────┐
 │ USB-C   power   │ RP2350 + flash │ GNSS  u.FL│
 │  ESD    LDOs    │  TCXO   SWD    │ module    │
 │─────────────────┼────────────────┼───────────│
 │ RTC + CR2032    │ buffers │ R-2R │ LPF       │
 │                 │         │      │ op-amp    │
 │ LED  button     │         │      │ ant conn ─┼──► ferrite rod
 └──────────────────────────────────────────────┘
```

Signal flows left→right, digital→analog, with the antenna connector at the far
edge from USB and the switching supplies.

### Critical layout rules

1. **TCXO adjacent to XIN**, short trace, guarded by ground. Keep the ladder and
   the driver away from it.
2. **R-2R ladder compact and symmetric.** Trace resistance adds directly to the
   ladder — keep branch lengths matched. Ground-flood around it.
3. **Star-ground the analog section** to the main plane at a single point near
   the analog LDO. Do not let return current from the 10 switching DAC lines
   share a path with the driver's return.
4. **Keep the 10 DAC lines short and equal-length**, ≤25 mm, with ground between
   them and the analog nodes.
5. **Antenna leads twisted/differential**, connector away from GNSS. The tank
   carries up to 24 V peak (§6) — respect clearance.
6. **GNSS u.FL and module away from the tank and the DAC.** A 77.5 kHz near
   field is not a direct threat to 1.5 GHz, but the module's own supply and
   harmonics are.
7. Decouple every rail at every pin; 100 nF + bulk per device.

---

## 10. Test and bring-up

**Test points (I-09):** sample clock, DAC out, LPF out, driver out, coil
current, PPS in, PPS out.

**Bring-up order** — each step independently verifiable:

| # | Step | Pass criterion |
|---|---|---|
| 1 | Power, USB enumeration, SWD | Rails within tolerance |
| 2 | TCXO → PLL lock at target sysclk | Measured sysclk correct |
| 3 | PIO sample clock | F_s within 1 ppm of table |
| 4 | DAC ramp → check monotonic | No missing codes, INL ≤1 LSB |
| 5 | Sine at f_c → spectrum | Images ≥55 dB down (S-13) |
| 6 | Filter + driver | THD, level, no clipping |
| 7 | Tank resonance sweep | f_0 within ±1 % (M-05) |
| 8 | AM only, hardcoded time | Reference receiver decodes |
| 9 | PM enabled | Correlation receiver decodes |
| 10 | GNSS + PPS discipline | Epoch error ≤10 ms (P-01) |
| 11 | Real watch | Syncs (F-02) |

**Primary instrument: a 192 kHz USB audio interface.** Nyquist is 96 kHz, so
77.5, 68.5, 60 and 40 kHz all land directly in the audio band — capture with a
pickup coil and analyse in software. No SDR or upconverter. This covers steps
5–9 and the production resonance check.

---

## 11. Cost estimate (indicative, 100 units) 🔶

| Item | Est. |
|---|---|
| RP2350A + flash | €2.50 |
| TCXO 0.5 ppm | €3.00 |
| GNSS module | €8.00 |
| GNSS active antenna | €4.00 |
| DS3231 + CR2032 + holder | €2.50 |
| Buffers, ladder network, op-amp | €2.00 |
| LDOs, passives, ESD | €2.50 |
| Connectors (USB-C, u.FL, headers) | €2.00 |
| Ferrite rod antenna assembly | €3.00 |
| PCB 4-layer | €2.00 |
| Enclosure | €5.00 |
| **Total BOM** | **~€36** |

GNSS + antenna is a third of the cost. If indoor fix reliability proves
acceptable on a cheaper module, or if a USB-time-source SKU (F-13) is offered,
there is meaningful room to cost-reduce.

---

## 12. Next steps

1. Resolve open items 1 (PTB PM spec) and 5 (XIN drive) — both block design freeze.
2. Breadboard **P0**: RP2350 dev board, square wave, hardcoded DCF77 AM, coil.
   Confirms F-02 before any PCB is ordered.
3. Prototype the analog chain on the same dev board (buffers, ladder, filter,
   driver, tank) and run bring-up steps 4–9.
4. Compliance pre-scan of the prototype before layout freeze.
5. Schematic capture → review → PCB rev A.

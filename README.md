# Dynamic-comparator-180nm-3.3V
Sub-block for 5-bit Full-Custom Flash ADC | Chips to Startup Project, NEHU Shillong
## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Design Specifications](#2-design-specifications)
3. [Circuit Topology](#3-circuit-topology)
4. [Device Sizing](#4-device-sizing)
5. [Bias Point Analysis](#5-bias-point-analysis)
6. [Simulation Setup](#6-simulation-setup)
7. [Simulation Results](#7-simulation-results)
8. [Performance Summary](#8-performance-summary)
9. [Mismatch & Offset Analysis](#9-mismatch--offset-analysis)
10. [Known Issues & Recommendations](#10-known-issues--recommendations)
11. [Next Steps Before ADC Integration](#11-next-steps-before-adc-integration)
12. [Tool & PDK Information](#12-tool--pdk-information)

---

## 1. Project Overview
This repository documents the design and simulation of a **differential-pair open-loop comparator** implemented on the **SCL 180nm process** using **3.3V thick-oxide MOSFETs**. This comparator is a fundamental building block of a **5-bit full-custom Flash ADC** being developed under the **MeitY "Chips to Startup" (C2S)** funded project at the **North-Eastern Hill University (NEHU), Shillong**.

A Flash ADC requires `2^N - 1` comparators operating in parallel, each with a unique reference voltage tap. For this 5-bit design, **31 comparator instances** will be arrayed with a resistor ladder reference network. The comparator design shown here is the unit cell for that array.

**Key goals for this comparator:**
- Correct binary output (HIGH/LOW) relative to a reference voltage
- Full rail-to-rail output swing (0V to VDD = 3.3V) to interface cleanly with the thermometer-to-binary encoder
- Low input-referred offset voltage relative to ADC LSB (~103 mV for 5-bit, 3.3V full scale)
- Sufficient speed for the target ADC sampling rate

---

## 2. Design Specifications

| Parameter | Value |
|-----------|-------|
| Technology | SCL 180nm CMOS |
| Supply Voltage (VDD) | 3.3 V |
| MOSFET Type | 3.3V Thick-Oxide |
| Input Signal (vin_1) | Sinusoidal, ±1.5V amplitude, DC offset = 0V |
| Reference Voltage (V_ref) | 1.0 V (DC) |
| Tail Bias Voltage (Vbias) | 700 mV |
| Output Swing (measured) | ~0V to ~3.28V |
| Target ADC Resolution | 5 bits |
| ADC Full-Scale Range | 0 to 3.3V |
| ADC LSB | 3.3V / 32 ≈ **103 mV** |
| Number of Comparators in ADC | 31 |
| Simulation Type | Transient |

---

## 3. Circuit Topology

The circuit is a **two-stage differential amplifier used as an open-loop comparator**, consisting of:

```
                        VDD (3.3V)
                    ┌──────┴──────┐
                  [M3/PMOS]    [M5/PMOS]       ← Cross-coupled / Active Load
                    │              │
         net_left ──┤              ├── net_right (Vout)
                    │              │
                  [M4/NMOS]    [M6/NMOS]       ← Differential Input Pair
                    │              │
         vin_1 ─────┤              ├───── V_ref
                    └──────┬───────┘
                         [M2/NMOS]             ← Tail Current Source
                            │
                         Vbias (700mV)
                            │
                           GND
```

### Stage Descriptions

**Differential Input Pair (M4, M6 — NMOS):**
The two NMOS transistors form the differential sensing core. M4 is driven by `vin_1` (the unknown input) and M6 is driven by `V_ref` (the reference). When `vin_1 > V_ref`, M4 conducts more current than M6, pulling the left node down and the right node (Vout) high through the PMOS load.

**Cross-Coupled PMOS Load (M3, M5):**
The PMOS pair acts as an active load. In this topology, the PMOS devices help provide gain and establish the operating point for the differential stage. The cross-coupling arrangement aids in regeneration, improving the output swing and speed of transition.

**Tail Current Source (M2 — NMOS):**
A single NMOS biased at `Vbias = 700 mV` sets the total drain current for the differential pair. The oversized width (W = 84 µm) ensures a large enough tail current to give adequate transconductance and transition speed.

**Output Stage:**
The right side of the schematic contains a mirrored structure that buffers and drives `Vout` to the output pin, completing the rail-to-rail output swing.

---

## 4. Device Sizing

| Device | Type | Width (µm) | Length (µm) | Role |
|--------|------|-----------|------------|------|
| M3 | PMOS (3.3V) | 4.2 | 0.5 | Active load / cross-coupled |
| M5 | PMOS (3.3V) | 4.2 | 0.5 | Active load / cross-coupled |
| M4 | NMOS (3.3V) | 8.4 | 0.5 | Differential input (vin_1) |
| M6 | NMOS (3.3V) | 8.4 | 0.5 | Differential input (V_ref) |
| M2 | NMOS (3.3V) | 84 | 0.5 | Tail current source |

> **Note:** L = 0.5 µm is the recommended minimum channel length for 3.3V thick-oxide devices in SCL 180nm to maintain adequate Vds reliability and reduce short-channel effects.

### Sizing Rationale

**NMOS-to-PMOS width ratio = 8.4 / 4.2 = 2 : 1**

This is a standard design choice to compensate for the mobility difference between electrons and holes in silicon. In SCL 180nm:

```
µn ≈ 2× µp  →  (W/L)_NMOS / (W/L)_PMOS = µp / µn ≈ 0.5
→ W_NMOS = 2 × W_PMOS  (for equal gm and symmetric trip point)
```

This ensures:
- Balanced transconductance across both arms of the differential pair
- A symmetric switching threshold (trip point ≈ VDD/2 at the output stage)
- Matched slew rates on rising and falling transitions

**Tail width = 84 µm = 10× input pair width**

The tail transistor is deliberately oversized to:
- Carry sufficient DC bias current (~160 µA estimated)
- Maintain strong saturation even at low common-mode input voltages
- Minimize tail current variation with supply and temperature

---

## 5. Bias Point Analysis

### Tail Current Estimation

Using the square-law MOSFET model for SCL 180nm NMOS 3.3V device:

- **Vth_n ≈ 0.55 – 0.65 V** (typical)
- **µn·Cox ≈ 170 – 200 µA/V²** (3.3V NMOS)
- **Vbias = 700 mV**, so **Vov = Vgs − Vth ≈ 100 mV**

```
I_tail = (1/2) × µn·Cox × (W/L) × Vov²
       = (1/2) × 190µ × (84/0.5) × (0.1)²
       = (1/2) × 190µ × 168 × 0.01
       ≈ 160 µA
```

Each input branch therefore carries approximately **80 µA**.

### Input Transconductance

```
gm = √(2 × µn·Cox × (W/L) × Id)
   = √(2 × 190µ × 16.8 × 80µ)
   ≈ 714 µA/V
```

This gives moderate gain, appropriate for an open-loop comparator (gain is traded off for speed by not adding compensation).

### Input Common-Mode Range (ICMR)

The tail device M2 must remain in saturation for proper differential operation. The minimum input common-mode voltage is:

```
V_CM(min) = Vgs_input + Vds_tail(sat)
           ≈ 0.8V + 0.2V = 1.0V (approximate)
```

Below this value, M2 enters the linear region and the tail current drops, reducing gain. In the test simulation, `vin_1` swings negative (below 0V), which pushes M2 toward its edge of saturation. In actual ADC deployment with a bounded input range (0 to 3.3V), this is not a concern.

---

## 6. Simulation Setup

### Testbench Configuration

| Source | Type | Value |
|--------|------|-------|
| VDD (V6) | DC | 3.3 V |
| vin_1 (V4) | Sine | Amplitude = 1.5V, DC offset = 0V, Freq = 1 kHz |
| V_ref (V1) | DC | 1.0 V |
| Vbias (V5) | DC | 700 mV |

### Simulation Parameters

| Setting | Value |
|---------|-------|
| Analysis | Transient |
| Stop Time | 4.7 ms |
| Time Step | Auto (Spectre default) |
| Temperature | 27°C (nominal) |
| Process Corner | TT (typical-typical) |

### Observed Signals

| Signal | Color | Description |
|--------|-------|-------------|
| `/vout` | Red | Comparator output |
| `/vin_1` | Green | Differential input (unknown) |
| `/V_ref` | Orange | Reference voltage (fixed at 1.0V) |

---

## 7. Simulation Results

### Transient Waveform

![Comparator Schematic](comparator_ckt.png)
*Figure 1: Comparator schematic in Cadence Virtuoso (SCL 180nm, 3.3V thick-oxide MOSFETs)*

![Transient Simulation Output](comparator_op.png)
*Figure 2: Transient response — Green: vin_1 (sine, ±1.5V), Orange: V_ref (1V DC), Red: Vout (digital output)*

### Observations

**Correct comparison logic confirmed:**
- `Vout` transitions to HIGH (~3.28V) whenever `vin_1 > V_ref (1.0V)`
- `Vout` transitions to LOW (~0V) whenever `vin_1 < V_ref (1.0V)`
- Output crossings align accurately with the sinusoidal input's 1V zero-crossings against the reference

**Output voltage levels:**
- Logic HIGH: ~3.28V (≈ 0.02V below VDD — near ideal)
- Logic LOW: ~0V (≈ 0V — ideal)
- Full output swing: ~3.28V

**Transition quality:**
- No glitching or metastability observed at 1 kHz test frequency
- Transitions are clean with no mid-rail lingering
- Slight asymmetry in rise vs. fall time is observed (expected — see Section 10)
## 8. Performance Summary

| Metric | Value | Notes |
|--------|-------|-------|
| Output HIGH | ~3.28 V | Near full VDD |
| Output LOW | ~0 V | Ideal GND |
| Output Swing | ~3.28 V | Rail-to-rail behaviour |
| Correct comparison | ✅ Yes | Verified across 4 full cycles |
| Glitch-free | ✅ Yes | At 1 kHz test frequency |
| Estimated I_tail | ~160 µA | Calculated from device parameters |
| Estimated gm (input pair) | ~714 µA/V | Per input transistor |
| ICMR lower bound | ~1.0 V | Tail saturation limit |
| Process corner simulated | TT only | FF/SS/FS/SF corners pending |
| Temperature simulated | 27°C only | −40°C / 125°C corners pending |

---

## 9. Mismatch & Offset Analysis

For a Flash ADC, the input-referred offset voltage of each comparator must be well below **LSB / 2** to avoid missing codes or non-monotonicity.

### ADC LSB

```
LSB = VFS / 2^N = 3.3V / 2^5 = 3.3V / 32 ≈ 103 mV
LSB / 2 ≈ 51.5 mV  (maximum acceptable 3σ offset)
```

### Pelgrom Mismatch Estimate (SCL 180nm)

The random threshold mismatch for an NMOS transistor is estimated using Pelgrom's law:

```
σ(Vth) = AVT / √(W × L)
```

For SCL 180nm NMOS 3.3V devices, **AVT ≈ 5 mV·µm** (typical).

```
σ(Vth) = 5 mV·µm / √(8.4 µm × 0.5 µm)
        = 5 / √4.2
        = 5 / 2.05
        ≈ 2.44 mV

3σ(Vth) ≈ 7.3 mV
```

**Conclusion:** The 3σ input-referred offset of ~7.3 mV is well below the allowable limit of 51.5 mV (LSB/2). The current sizing provides **adequate mismatch margin** for a 5-bit Flash ADC without requiring auto-zeroing or calibration.

> ⚠️ This is an analytical estimate only. **Monte Carlo simulation (minimum 100 runs, preferably 200–500) in Cadence Spectre with the SCL 180nm mismatch models must be performed before tapeout** to verify this margin statistically.

---

## 10. Known Issues & Recommendations

### Rise/Fall Time Asymmetry

**Observation:** The rising edge of `Vout` is slightly slower than the falling edge.

**Root cause:** When `vin_1` crosses above `V_ref` and `Vout` must rise, the PMOS load devices (W = 4.2 µm) pull the output up. PMOS drive strength is inherently lower than NMOS for the same overdrive, creating a slower pull-up.

**Fix (if needed):** Increase PMOS width from 4.2 µm to 5.0–6.0 µm while keeping NMOS at 8.4 µm. Re-simulate to verify trip point is not shifted.

### Negative Input Swing Below ICMR

**Observation:** In the test bench, `vin_1` swings to −1.5V, which is below the tail transistor's saturation boundary.

**Impact in test:** Tail current momentarily drops, causing slightly rounded transitions near vin_1 ≈ 0V. **This does not affect the correctness of the comparison output** since the threshold is at 1.0V.

**Impact in ADC:** Not an issue — in the Flash ADC, the input will be bounded to 0V to 3.3V. No negative input swings expected.

### Fixed Vbias Voltage Source

**Issue:** The tail current source (M2) is biased with a fixed `Vdc = 700 mV`. In real silicon, this voltage will shift with temperature, process corner, and supply variation, causing the tail current — and thus the comparator speed and gain — to vary unpredictably.

**Recommendation:** Replace the fixed voltage source with a **proper PTAT or bandgap-referenced current mirror** bias generator for the full ADC design. The 700 mV DC source is acceptable for single-cell characterisation only.

### Input Kickback Noise (Flash ADC Array)

**Issue:** In a Flash ADC, all 31 comparator instances share the same `vin` line. When each comparator switches, charge stored on the gate capacitances of M4 couples back onto the `vin` bus (kickback noise), disturbing the sampling accuracy of all other comparators.

**Recommendation:**
- Add a small source-follower input buffer before the comparator differential pair, OR
- Insert a series resistor (500Ω – 2kΩ) at each comparator input with a shunt capacitor to form an RC low-pass filter that absorbs kickback
- Alternatively, use a **bootstrapped sample-and-hold** (already in development) at the ADC input to isolate the sampling node

### No PVT Corner Simulation

**Issue:** All results shown are at TT corner, 27°C only.

**Recommendation:** Run full PVT sweep before tapeout:

| Corner | VDD | Temperature |
|--------|-----|-------------|
| TT | 3.3V | 27°C |
| FF | 3.6V | −40°C |
| FF | 3.6V | 125°C |
| SS | 3.0V | −40°C |
| SS | 3.0V | 125°C |
| FS | 3.3V | 27°C |
| SF | 3.3V | 27°C |

Verify that output swing, switching threshold, and propagation delay remain within specification across all corners.

---

## 11. Next Steps Before ADC Integration

- [ ] **Step-response testbench** — Drive a fast step input (0→2V, rise time < 1 ns) and measure propagation delay (tp_rise, tp_fall at 50%)
- [ ] **Monte Carlo simulation** — 200+ runs with SCL 180nm mismatch models; measure σ(Vos) and verify 3σ < LSB/2
- [ ] **PVT corner sweep** — All 7 corners listed above; check output swing and threshold stability
- [ ] **Replace Vbias source** — Implement a current mirror bias generator from the project's reference block
- [ ] **31-instance array simulation** — Instantiate all comparators with the resistor ladder; simulate kickback coupling on the shared Vin node
- [ ] **Layout** — Draw full custom layout in Virtuoso with careful matching (common-centroid placement for M4/M6, M3/M5)
- [ ] **DRC / LVS** — Run Pegasus DRC and LVS on the comparator cell
- [ ] **Post-PEX simulation** — Extract parasitics with Quantus and re-simulate; pay attention to layout-induced mismatch and capacitive loading on Vout
## 12. Tool & PDK Information

| Item | Details |
|------|---------|
| Schematic Entry | Cadence Virtuoso Schematic Editor (IC 6.1.8) |
| Simulation | Cadence Spectre (ADE L / ADE Explorer) |
| Process Design Kit | SCL (Semiconductor Laboratory) 180nm CMOS PDK |
| MOSFET Models | SCL 180nm BSIM3v3 (3.3V thick-oxide, TT corner) |
| DRC / LVS | Cadence Pegasus (pending) |
| Parasitic Extraction | Cadence Quantus (pending) |
| Institution | North-Eastern Hill University (NEHU), Shillong, Meghalaya |
| Project Funding | MeitY Chips to Startup (C2S) Programme |
## Author

**Pronita Bora**
Project Assistant, Chips to Startup Project
Department of Electronics & Communication Engineering
North-Eastern Hill University, Shillong — 793022, Meghalaya, India

---

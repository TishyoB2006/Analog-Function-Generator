# Multi-Waveform Function Generator (LTSpice Design and Hardware Verification)

An analog function generator built entirely from discrete op-amp (LT1056) stages, capable of producing six distinct waveforms — **Square, Triangle, Sine, PWM, Sawtooth, and Cosine**r. Designed and simulated in LTSpice, and cross-verified on physical hardware.

All waveforms currently run at a fixed frequency of **1 kHz**.

---

## 1. Overview

The generator is built around a classic **comparator + integrator relaxation oscillator** as the timing core, which produces the Square and Triangle waves. These two "master" waveforms are then reshaped and combined by downstream stages to derive the remaining waveforms:

```
                ┌────────────┐      ┌────────────┐
                │  Comparator │ SQUARE │ Integrator │ TRIANGLE
   Core ───────▶│   (U1)      │──────▶│    (U2)    │───────┬────────────┐
   Oscillator    └─────┬──────┘       └────────────┘        │            │
                       │ (hysteresis feedback: R2, R3)       │            │
                       └──────────────────────────────────┘            │
                                                                        │
                                        ┌───────────────────────────────┘
                                        │
                         ┌──────────────▼───────────────┐
                         │  RC Shaping Network (R9,R10,C4) │ ──▶ Buffer (U5, C5) ──▶ SINE
                         └──────────────────────────────┘
                                        │
                         ┌──────────────▼───────────────┐
                         │  Comparator vs. DC ref (U6)    │ ──▶ PWM
                         └────────────────────────────────┘

   Independent Oscillator (U3, R4, R5) ──▶ Asymmetric charge/discharge network
   (D1, D2, R6, R7) ──▶ Integrator (U4, C2) ──▶ SAWTOOTH
```

---

## 2. Stage-by-Stage Description

### 2.1 Square + Triangle Core (U1, U2)
- **U1 (LT1056)** is configured as a comparator with positive feedback through **R2 (10k)** and **R3 (40k)**, forming a Schmitt-trigger threshold. Its output swings between +Vsat and −Vsat, producing the **Square** wave.
- The Square wave drives **R1 (1k)** into **U2**, which is wired as an integrator with feedback capacitor **C1 (1µF)**. Integrating a square wave produces a linear ramp, giving the **Triangle** wave.
- The Triangle output feeds back into U1's threshold-setting node, closing the loop and sustaining oscillation. Frequency is set jointly by the R2/R3 hysteresis ratio and the R1–C1 integrator time constant, tuned in simulation to give a 1 kHz output.

### 2.2 Sine Shaper (R9, R10, C4, U5, C5)
- The Triangle wave is passed through an **RC shaping network** (**R9 = 9.1k**, **R10 = 9.1k**, **C4 = 10n**) that progressively rounds off the sharp corners of the triangle wave.
- **U5** buffers this shaped signal, with **C5 (22n)** in the feedback path acting as a low-pass filter to further smooth the waveform, producing an approximate **Sine** wave. (This is the standard triangle-to-sine "corner rounding" technique rather than a lookup-table or diode-shaping approach.)
- **Cosine** can be obtained from the same sine-generation path, simply taken as a 90°-phase-shifted tap (i.e., referenced from the Triangle/Square core rather than the Sine output directly), giving a quadrature version of the same waveform.

### 2.3 PWM Generator (U6, V3)
- **U6** is wired as a comparator, with the **Triangle** wave on one input and a **DC reference / control voltage** on the other (represented in simulation by source **V3**, a slow ramp/pulse: `PULSE(-13.5 13.5 0 49u 1u 0 50u)`).
- Comparing a triangle wave against an adjustable DC level is the standard analog technique for generating a **PWM** wave — as the reference level moves up/down relative to the triangle's peak-to-peak swing, the comparator's output duty cycle increases/decreases accordingly.
- In the simulation, V3 sweeps the reference to demonstrate duty-cycle variation; in hardware this would be replaced by a manual potentiometer-derived DC level for duty-cycle control.

### 2.4 Sawtooth Generator (U3, U4, D1, D2, R4–R7, C2)
- A **second, independent oscillator** is built around **U3**, with **R4 (40k)** and **R5 (10k)** forming its own hysteresis feedback — functionally identical in principle to the U1 core, generating its own square-wave drive signal.
- This square wave is routed through **two diodes (D1, D2)** in parallel, each in series with a different resistor — **R6 (500k)** and **R7 (1k)**. Because the diodes conduct only on opposite half-cycles, the charge path and discharge path into the downstream integrator use **different resistances**.
- This asymmetric R6/R7 charge-discharge network feeds **U4**, an integrator with feedback capacitor **C2 (3n)**. Since the charge and discharge time constants are unequal (one path is ~500x more resistive than the other), the integrator output is no longer a symmetric triangle but a **Sawtooth** wave — fast ramp on one edge, slow ramp on the other.

---

## 3. Component Summary

| Stage | Reference Designators | Key Values | Function |
|---|---|---|---|
| Square/Triangle core | U1, U2, R1, R2, R3, C1 | R1=1k, R2=10k, R3=40k, C1=1µF | Base oscillator (comparator + integrator) |
| Sine shaper | R9, R10, C4, U5, C5 | R9=R10=9.1k, C4=10n, C5=22n | Rounds triangle into sine |
| PWM generator | U6, V3 | — | Compares triangle vs. DC ref for variable duty cycle |
| Sawtooth generator | U3, U4, D1, D2, R4–R7, C2 | R4=40k, R5=10k, R6=500k, R7=1k, C2=3n | Independent oscillator + asymmetric integrator |

All active stages use the **LT1056** op-amp, operating from ±Vsat rails.

---

## 4. Design Specifications

| Parameter | Value |
|---|---|
| Output frequency (all waveforms) | 1 kHz |
| Op-amp | LT1056 |
| Supply | ±Vsat (dual rail) |
| Waveforms generated | Sine, Square, Triangle, PWM, Sawtooth, Cosine |
| Simulation tool | LTSpice |

---

## 5. Verification

- **Simulation:** All six waveforms were verified via LTSpice transient analysis, confirming correct shape, frequency, and phase relationships between the core Square/Triangle signals and their derived waveforms.
- **Hardware:** The circuit was built and tested physically, with waveforms captured on an oscilloscope to confirm agreement with simulation results.

---

## 6. Possible Future Improvements

- Add potentiometer-based frequency control (replacing fixed R/C values with variable resistors) for a tunable output range instead of a fixed 1 kHz.
- Add amplitude control / attenuator stage on the output.
- Replace the RC "corner-rounding" sine shaper with a diode-shaping network for improved THD.
- Convert the manual DC-reference PWM control into a proper potentiometer-based duty-cycle knob for hardware use.

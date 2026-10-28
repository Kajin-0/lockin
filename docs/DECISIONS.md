# Architecture decision record (ADR)

Entries document intent, alternatives, limits and rationale. They are not proof of performance. Do not rewrite previous decisions without dated superseding entry.

## ADR-001 — Optimize Rev A for first-spin success (2026-10-28)

**Decision:** Build a precision-first single voltage-channel digital lock-in; defer MFLI-class 500 kHz/5 MHz hardware and FPGA. Choose manufacturer-proven analog/ADC topology and robust testability.

**Reasons:** Lower ADC/FPGA routing, clocking, firmware and power risk; cheaper/reworkable PCB; trustworthy amplitude/phase/noise and simple USB automation add more value than an unverified 500 kHz headline.

**Consequences:** Rev A cannot claim MFLI-equivalent high-speed performance, TIA current input, 60-MSPS raw acquisition or full MFLI dynamic reserve. These are later independent engineering programs.

## ADR-002 — ADS8900B SAR + MCU, no FPGA (2026-10-28)

**Decision:** Candidate 20-bit, 1 MSa/s ADS8900B and STM32H723 using hardware timer + SPI DMA and instrument DSP in Rust; benchmark hardware first.

**Open:** Verify enhanced SPI data framing, 22 MHz SCLK/1-MSPS timing, MCU peripheral compatibility and caches, 1–4 demodulator CPU budget. Use external dev/eval boards or representative prototype to prove before full board.

## ADR-003 — AD8421 primary differential front-end (2026-10-28)

**Decision:** Candidate AD8421BRZ instrument amp with relays selecting G=1/10/100. Candidate Rg values: G1 open; G10 1.10 kohm; G100 100 ohm from G=1+9.9 kohm/Rg. **These are provisional calculated resistor values, not verified resistor choices.**

**Reasons:** Low input voltage noise, integrated differential measurement, strong CMRR and convenient SOIC package vs dual precision front ends.

**Open:** Differential common-mode headroom, valid output swing into anti-alias/driver, low leakage protection, resistor noise, switching glitches, high-source-impedance performance. Manufacturer noise 3.2 nV/sqrt(Hz) maximum at 1 kHz; whole instrument <5 target still unverified. Separate high-Z JFET input and TIA deferred.

## ADR-004 — Frequency band is gated by alias test (2026-10-28)

**Decision:** Primary qualification **1 Hz–50 kHz**; 100 kHz as conditional extension. Do not describe the prior "four-pole 150 kHz" as a frozen anti-alias design.

**Reason:** At fs=1 MSa/s, 900 kHz interferer aliases into 100 kHz. A 4-pole 150 kHz Butterworth low-pass has only about 62 dB ideal attenuation at 900 kHz, inadequate to guarantee a high dynamic-range 20-bit front end under large interferers. Filter requires a formal noise/alias/overload budget and suitable topology.

## ADR-005 — Rust yes; critical timing still in hardware (2026-10-28)

**Decision:** Favor Rust no_std on MCU and Rust on host; Python golden reference. Require hardware timer conversion, DMA, bounded buffers, validated numerical behavior. Prototype MCU SPI/DMA with vendor drivers if needed before porting to Rust.

**Reason:** Memory safety and typed state machines reduce error classes, but they do not prove ADC timing, analog accuracy, clock stability or safety. Avoid putting time-critical sample timing in async task scheduling.

## ADR-006 — Fail-safe excitation and measurement invalidation (2026-10-28)

**Decision:** Hard output-disable circuitry independently driven by voltage supervisor/watchdog/arm logic; output defaults OFF on reset/power-off/fault. Measurements include validity bitmask and cannot silently ignore missing samples, ADC clipping, reference loss, wrong gain or invalid calibration.

**Open:** Exact interlock components, failure-mode proof, relay topology and status register interface.

## ADR-007 — Data isolation is not measurement isolation (2026-10-28)

**Decision:** Candidate full-speed ISOUSB111 data isolation, powered correctly on both barrier sides; external certified SELV supply. Enclosure/chassis and BNC shield scheme require formal design. No hazardous voltage or safety-rating claims.

## ADR-008 — Range and performance aspirations remain conditional (2026-10-28)

**Decision:** Proposed ±10 V, ±1 V, ±100 mV, ±10 mV nominal ranges, selectable 50 Ω / 1 MΩ termination, and ≤5 nV/√Hz noise target at 1 kHz. These will be revised if headroom, protection, input noise or thermal budget fails.

**Reason:** Precision and safety matter more than preserving arbitrary marketing targets.

## Open decision register (next to resolve)

| ID | Question | Information required |
|---|---|---|
| OPEN-01 | Exact analog rail voltages? | AD8421 output/CM headroom, filter op-amp and driver swing, noise, dissipation |
| OPEN-02 | Exact protection/clamp/attenuation? | power-off fault path, source Z, current/thermal rating, capacitance and leakage |
| OPEN-03 | Anti-alias filter order/cutoff/topology? | strong interferer spec and response simulations |
| OPEN-04 | Precise REF/ADC driver and 5 V power network? | TI reference topology and rail/common-mode budget |
| OPEN-05 | DDS clock selection and output amplitude/calibration? | low-frequency tuning resolution, PLL coherence and waveform response |
| OPEN-06 | STM32H723 part/package/pin mapping? | actual SPI/timer/DMA, USB, SWD and relay allocation |
| OPEN-07 | USB/chassis/isolated rail plan? | whole instrument grounding and power diagram |
| OPEN-08 | Relay part and unpowered default? | gain+range fault states and switching artifacts |
| OPEN-09 | Analog overrange detection? | independent comparator need vs cost/noise, ADC saturation thresholds |
| OPEN-10 | Production cost and procurement? | qualified priced BOM with current distributor availability |

Update this register with actual decisions and source evidence as engineering progresses.

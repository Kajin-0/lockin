# Lockin — Precision Digital Lock-In Amplifier

**Status:** Architecture planning / pre-schematic. **No PCB, schematic, firmware, or measurement results have been completed.**

This repository is the source of truth for a low-cost, robust, general-purpose **precision digital lock-in amplifier**, with an emphasis on **a reliable first PCB revision** rather than an unverified specification race against commercial instruments.

## Mission

Build an affordable, serviceable scientific instrument capable of measuring small coherent AC signals with trustworthy amplitude/phase readouts, differential voltage sensing, internal/external references, calibration, clear overload detection, and fail-safe excitation. Use sound analog design and measurement evidence before making performance claims.

## Current Rev A architecture (candidate, NOT fabrication-approved)

| Subsystem | Starting candidate | Status |
| --- | --- | --- |
| Voltage-input amplifier | AD8421BRZ instrumentation amplifier | Candidate: input protection, full range, common-mode budget need validation |
| ADC | TI ADS8900B, 20-bit, up to 1 MSa/s | Candidate: timing/throughput and analog drive need proof |
| ADC driver | TI THS4551 | Candidate: use manufacturer reference circuits |
| Reference | TI REF5045, 4.5 V | Candidate: verify total noise, drift, drive, decoupling |
| Processor | STM32H723 | Candidate: acquisition DMA/latency feasibility gate |
| Excitation | AD9833 DDS, with external output buffer and hard disable | Candidate: clock, amplitude, distortion need proof |
| External reference | Protected comparator + timer capture | Architecture requirement |
| Gain | Relays selecting nominal G = 1 / 10 / 100 | Architecture requirement; safe default G=1 |
| Input modes | Differential voltage; nominal 50 Ω / 1 MΩ termination | Architecture requirement; verify actual loading |
| Power | External certified SELV adapter; filtered, supervised rails | Architecture requirement |
| USB | Full-speed USB isolation (ISOUSB111 candidate) | Candidate; isolated power and chassis plan required |
| Processing | Rust on MCU, bit-exact host verification | Design direction |
| PCB | Four-layer with continuous reference ground, designed for rework | Provisional |

### Performance goals — not measured

- **Primary qualification band:** 1 Hz–50 kHz, subject to actual filter and noise validation.
- **Extended Rev A target:** 100 kHz **only if** anti-aliasing, gain/phase, dynamic reserve, and timing tests pass. Previous 100 kHz proposal is **not** an unconditional promise.
- **Noise target:** ≤ 5 nV/√Hz at 1 kHz, low-source-impedance, high-gain input mode, with clearly specified measurement conditions.
- Nominal voltage spans: ±10 V, ±1 V, ±100 mV, ±10 mV. These are **proposed calibrated ranges**, not verified safe/linear inputs.
- One voltage acquisition channel; internal and external references; initial goal of 1–4 virtual I/Q demodulators subject to throughput profiling.
- Synchronous internal reference, PC control/SCPI-like protocol, selectable LPF, sweep, calibration, and invalid-measurement flags.

**Safety scope:** low-voltage laboratory signals only. Never connect mains, hazardous voltages, or unknown common-mode sources. Isolation of USB data does **not** make a measurement input safely isolated or CAT-rated.

## Read these files in order

1. [AGENTS.md](AGENTS.md): mandatory handoff rules, current state and immediate work.
2. [docs/DESIGN_PLAN.md](docs/DESIGN_PLAN.md): architecture, requirements, PCB sheets, technical assumptions.
3. [docs/ENGINEERING_GATES.md](docs/ENGINEERING_GATES.md): rigorous go/no-go criteria and qualification.
4. [docs/VIRTUAL_VERIFICATION.md](docs/VIRTUAL_VERIFICATION.md): SPICE, DSP, firmware, PCB, and manufacturing simulation-and-test plan.
4. [docs/SAFETY_AND_RISKS.md](docs/SAFETY_AND_RISKS.md): faults, safe states, hazards, high-risk circuits.
5. [docs/DECISIONS.md](docs/DECISIONS.md): architecture decisions, alternatives and unresolved items.
6. [docs/REFERENCES.md](docs/REFERENCES.md): manufacturer primary sources.

## Workflow

**Requirements → electrical budgets → datasheet/pin review → simulations → ADC/DMA proof → KiCad hierarchical schematic → independent review → PCB layout → ERC/DRC → fabrication → bring-up → calibration → qualification.**

No agent should mark a specification as achieved without test evidence. No PCB fabrication should proceed until the pre-fab gates in [docs/ENGINEERING_GATES.md](docs/ENGINEERING_GATES.md) pass.

## Next step

Start with the electrical interface budgets and pin-level feasibility of **AD8421 → anti-aliasing/driver → ADS8900B → STM32H723**, especially DC headroom, differential common-mode, relay fault states, full-rate SPI/DMA, source-impedance noise, and alias rejection. Document the evidence before drawing irreversible PCB geometry.

## Cost discipline

Prior electronics BOM allowance: roughly **$150–$285**, not a vendor quote or assembled-unit price. Optimize after validation, not at the expense of reliable first hardware.

## Longer-term scope

MFLI-like 500 kHz/5 MHz operation, FPGA processing, high-speed ADC/DAC, ultralow-noise current inputs, and multi-channel instruments are **future projects**, not Rev A requirements.

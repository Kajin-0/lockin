# Agent instructions — Lockin

**Read this entire file before working.** Canonical repository: https://github.com/Kajin-0/lockin

## Mission and non-negotiable priorities

Create an affordable, **general-purpose, low-noise, safe, robust and scientifically trustworthy** digital lock-in amplifier. The first fabricated PCB should have a very high probability of working. Optimize for low engineering and integration risk, not maximum advertised bandwidth or lowest possible component count.

Priorities, in order:
1. Electrical safety, controlled failure modes and testability.
2. Correct, traceable and calibrated measurement behavior.
3. Reliable first-pass manufacture/assembly/bring-up and repair.
4. Good input noise and usable dynamic range.
5. Reasonable cost.
6. Additional features and extended frequency coverage.

**Do not imply that a candidate part, schematic, simulated waveform, nominal ADC resolution, or software test proves instrument-level performance.**

## Snapshot as of 2026-10-28

- Repository began empty and was initialized with planning documents only.
- No KiCad project, netlist, schematic, PCB layout, firmware, GUI, physical prototype, pricing-validated BOM, or measurement exists yet.
- Rev A candidate: AD8421BRZ differential instrumentation amplifier, selectable attenuation / input loading, relayed gain (nominal 1/10/100), analog anti-alias and TI THS4551 driver, TI ADS8900B 20-bit 1 MSa/s ADC, STM32H723 MCU, AD9833 waveform DDS, protected external reference, optional ISOUSB111 isolated USB, external certified SELV supply and monitored rails.
- Rust is preferred for application firmware (after bring-up of peripheral timing) and host; Python golden models are welcome. NO FPGA for Rev A.
- Baseline qualification band: 1 Hz–50 kHz. 100 kHz is a conditional extension, NOT guaranteed; 500 kHz/5 MHz FPGA design deferred.
- Design-level noise aspiration: <=5 nV/sqrt(Hz) at 1 kHz, specified low-source-Z/high-gain/termination conditions. Not a measurement.
- Initial provisional electronics BOM allowance $150–$285; NOT a quote.
- Formal project status: PRE-SCHEMATIC, NO DESIGN FREEZE, NO FAB APPROVAL.

## Mandatory reading order

1. README.md
2. docs/DESIGN_PLAN.md
3. docs/ENGINEERING_GATES.md
4. docs/SAFETY_AND_RISKS.md
5. docs/DECISIONS.md
6. docs/REFERENCES.md
7. docs/HANDOFF.md

If a document disagrees with another, check its date, the decision log, and the latest verified datasheet. Record and resolve discrepancies; do not silently choose the convenient value.

## Next agent: resume exactly here

1. Inspect branch, repository files, commits, open PRs, and docs/HANDOFF.md. Never assume previous work is complete from a discussion summary.
2. Create a pin- and rail-level **electrical feasibility ledger** for each proposed IC, including manufacturer datasheet revision, package, supply/current, absolute maxima, I/O thresholds, common-mode limits, allowed ADC input swing, timing, reference load, and any errata.
3. Derive **signal/headroom budget** for each nominal ±10 V / ±1 V / ±100 mV / ±10 mV input range, through attenuation → gain → filter → driver → ±VREF differential ADC input. Evaluate worst-case common-mode and power-off behavior; revise ranges if needed.
4. Derive **noise and alias budgets**: input-referred voltage and current noise, resistors, amplifier gain effects, driver and ADC noise, jitter, 1/f noise, alias paths around fs ± f, and interferer overload. A simple four-pole 150 kHz analog filter at 1 MSa/s is NOT adequate for an unconstrained 20-bit anti-alias claim; choose filter architecture using quantified rejection requirements.
5. Prove **ADS8900B acquisition timing** with STM32H723 timer triggering / enhanced SPI / DMA / parity / DMA-safe memory. Distinguish ideal data sheet bandwidth from sustained observed throughput.
6. Simulate AD8421 gain switching (including contacts, transient recovery, leakage, fail-safe defaults) and the THS4551/ADS8900B drive/filter topology. Prefer source-proven TI circuits with measured deviations clearly justified.
7. Only after those gates: create hierarchical KiCad schematic, BOM with supplier SKUs and alternates, ERC/simulation outputs, and review checklist. Do not order a PCB prematurely.

## Hard engineering rules

- Faults must not silently produce credible-looking numeric results. Every measurement has validity, overload, reference-lock, acquisition integrity, and calibration-status flags.
- Excitation output OFF on unpowered/boot/reset/fault/MCU hang; independent hardware inhibit required, not just a firmware GPIO.
- Gain and range switching should default to a safe attenuated/unity mode, be break-before-make, and invalidate data until settled; evaluate relay failure cases and ESD/protection current paths.
- Include explicit power sequencing, brown-out behavior, external supervisor/watchdog, current-limited rails, test points, SWD, USB/ADC diagnostics, and calibration storage strategy.
- Input is **low-voltage laboratory only**. No claims of mains safety, CAT rating, medically safe isolation or hazardous-voltage differential measurement.
- USB data isolation does not automatically isolate analog BNC shields or solve enclosure/chassis grounding. Check powered and unpowered fault paths.
- Maintain a continuous ground reference plane unless a quantitative return-current/isolation analysis justifies a change. Respect actual isolator PCB creepage, clearance and supply architecture.
- Use conservative packages and allow rework. Layout does not begin without a signed schematic and model review.
- Never silently overwrite someone else's active work, calibrated constants, pin assignments, or prior accepted test evidence.
- Every performance claim must include test setup, source impedance, gain, bandwidth, temperature, reference mode, filter settings, instrument calibration and raw logs.
- Do not use raw 20-bit ADC code width as an effective-number-of-bits claim. Do not claim dynamic reserve without a defined strong-interferer test.
- Do not assume 4 simultaneous demodulators at 1 MSa/s will fit STM32 CPU budget; benchmark memory/cache/DMA and worst-case throughput.
- No uncontrolled network access or automatic firmware updates in safety-critical measurement path.

## How to work

- Work in small, auditable commits; explain what is evidence vs assumption.
- Append architecture decisions, with rationale and rejected alternatives, to docs/DECISIONS.md.
- Add simulations and test fixtures reproducibly (scripts, seeds, tool versions, graphs/logs). Do not only store screenshots.
- Keep manufacturer reference links and document revision in docs/REFERENCES.md.
- Mark files and schematic symbols as provisional until reviewed. Avoid inventing pinouts, packages, values or manufacturer performance.
- Prefer tests that can fail. A simulated sine matching expected lock-in output is not sufficient; test phase reversal, overload, missing samples, noisy interferers, and phase loss.
- Update docs/HANDOFF.md at every major checkpoint with: commit SHA, what changed, exact commands and outputs, pass/fail, blockers, assumptions, next three actions.
- Stop and record a **NO-GO** if a critical gate fails; propose alternatives rather than weakening the acceptance criterion without explicit review.

## Software direction

- Rust no_std for MCU instrument state machine, data validation and acquisition supervision; keep unsafe boundary small and audited.
- Use hardware timer for conversion; DMA for SPI transfers; never tie sample timing to USB or GUI scheduling.
- Keep a platform-independent Rust DSP library testable on host with golden Python/NumPy reference vectors.
- Version protocol; CRC, sequencing, bounds checks and explicit error states; host may use Rust and expose Python automation API.
- Implement watchdog fault actions and safe output disable in hardware, then verify with fault injection.

## Design is not finished

The previously discussed AD8421 + THS4551 + ADS8900B signal chain is a sensible **candidate**. It is NOT a validated drop-in series circuit; especially review output bias, analog supply, differential ADC common mode, anti-alias topology, input clamp leakage, relayed gain behavior, and reference drive.

See docs/HANDOFF.md for latest verified repository work.

# Rev A virtual verification and JLCPCB manufacture-readiness plan

**Created:** 2026-10-28
**Status:** verification PLAN ONLY. No device-level SPICE, system-level simulation, KiCad ERC/DRC, MCU hardware test, or JLCPCB upload has been performed. This is not manufacturing sign-off.

## Objective and philosophy

Maximize the probability that the **first fabricated and assembled PCB** is electrically safe, functionally correct, recoverable, and capable of meeting its properly qualified metrology targets. Simulate both ordinary operation and adversarial failure conditions, then require documented sign-off for independent electrical / digital / production checks.

Separate:
- **Electrical functional correctness** (headroom, conversion, reference, gain ranges, output shutdown).
- **Metrological performance** (noise, SNR, accuracy, dynamic reserve, temperature drift, harmonic contamination).
- **Physical manufacturing readiness** (correct footprints and orientations, stock, panel/stackup tolerances, assembly selection and programming).

Never assign an invented probability of success. A passing simulation is evidence only within the modeled assumptions. A fabricated first article and bench characterization remain mandatory.

## Formal simulation architecture

Use five complementary executable models, with a shared machine-readable configuration of fs, gains, physical rails, VREF, filters, amplifier/noise assumptions, DAC clock and reference behavior.

### 1. SPICE analog circuits and models

**Preferred free tools:** LTspice for ADI-focused AD8421 circuits; TI PSpice-for-TI / TINA-TI where manufacturer model compatibility warrants it; KiCad's ngspice for portable subcircuits. Do **not** assume PSpice vendor models are automatically supported by ngspice. Document model-version and deviations, compare against manufacturer examples.

Primary models / sources:
- AD8421 manufacturer model at https://www.analog.com/en/products/ad8421.html
- THS4551 PSpice/TINA-TI models at https://www.ti.com/product/THS4551
- ADS8900B PSpice/TINA-TI transient and IBIS models at https://www.ti.com/product/ADS8900B
- KiCad simulation documentation https://www.kicad.org/discover/spice/

Circuit decks to build when schematic exists:
1. Input protection and termination/attenuation; powered/unpowered; nominal/worst-case source impedance; electrostatic clamp currents.
2. AD8421 gain, DC and differential/common-mode transfer; gain changes; output bias; supply limitations.
3. Analog anti-alias/reconstruction filters, frequency and phase; time-domain settling; stability and tolerance.
4. THS4551 + ADC sampling input/reference interface, charge kickback, common-mode, full-scale recovery.
5. AD9833 output reconstruction/buffer and default OFF hardware inhibit, and any low-voltage output limiting.
6. Supplies, supervisors, startup/brownout and rail interaction where suitable switching-regulator models exist.

Required analysis:
- DC operating point / sweep / output headroom by nominal range and common-mode.
- AC gain/phase, cutoff, gain peaking, loop response and group delay over 0.1 Hz–10 MHz as applicable.
- Transient pulse, step, overload recovery, relay switching and startup / brownout.
- Input-referred **noise** across source Z (50, 200, 1k, 10k, 1M ohms) and gain, including current noise and resistance.
- Harmonic distortion/spurious response to test tones.
- Monte Carlo/parameter sweep: resistor 0.1% and capacitor 2–5% (actual selected part tolerances override), op-amp offset/drift, supply, temperature, part model corners. Count fails; do not claim real manufacturing yield from a simplistic distribution.
- Fault injection: relay stuck / coil open, shorted protection device, rail missing, reference missing, differential input common mode out of range, output disconnect short/open.

**Model-limits log:** SPICE macromodels may omit 1/f noise, package parasitics, distortion beyond fitting range, long-term drift, ESD destruction and switch charge injection. Confirm exact coverage from model documentation.

### 2. System-level sampled signal-chain simulator

**Tools:** Python/NumPy/SciPy for independent golden reference and Rust host library test harness. Must be deterministic with seeded noise and archived test data.

Model:
physical source / Johnson noise / 1/f / burst noise / large interferers → time-domain analog transfer function → sampling jitter, ADC quantization, modelled ADC INL/DNL/noise → missing/corrupt samples → reference NCO/PLL with drift/jitter → lock-in I/Q → low-pass/decimation → calibrated X,Y,R,theta and error flags.

Always compute physical aliasing before ADC quantization: 1 MSa/s with an out-of-band 900 kHz tone aliases into the 100 kHz bin. A fourth-order, 150 kHz Butterworth approximation alone attenuates ~62 dB at 900 kHz and is **not** an accepted system filter. Define acceptable interfering-signal envelope and reject/flag unsafe measurements.

Required adversarial vectors:
- 1 Hz, 10 Hz, 1 kHz, 10 kHz, 50 kHz and conditional 100 kHz.
- Amplitude from system noise floor through each range's linear maximum; DC offsets and common-mode variations.
- Phase grid 0°, 45°, 90°, 180°, 270°; amplitude and phase rapid jumps; reversals.
- Noncoherent frequency offset, drifting reference, missing/extra cycles, phase jump, reference frequency harmonic ambiguity.
- Strong near-band and aliased tones (including at fs−fin and fs+fin), high harmonics, nearby spurs and multi-tone superposition.
- Gaussian, 1/f, narrowband switching power and impulsive noise.
- Multiple digital LPF orders and time constants; verify numerical ENBW and settling.
- Saturation, gain switching, dropped samples, corrupted ADC frame/parity, stale calibration, output disable and re-arm.

Golden model tests MUST assert both measured values and validity/error behavior; never permit silently plausible data on an invalid acquisition.

### 3. Rust software / interface verification

- Host-compile the Rust DSP core; compare output arrays bitwise where possible or with fixed tolerances to reference Python vectors.
- Fuzz / property-test the USB packet parser and controls: invalid lengths, CRC failures, duplicate/out-of-order sequences, bad enums, overflow, disconnect and reconnect.
- State-machine model checking/property testing for BOOT, IDLE, SELFTEST, ARMED, MEASURING, FAULT. Required invariant: **excitation never enabled without hardware-ready plus explicit arm and valid rails**.
- CPU and memory budget: worst-case simultaneous demodulators, fixed maximum DMA buffer latency, bound memory allocation, no overflow, no sample loss. Host benchmark alone cannot establish STM32 performance.
- MCU peripheral proof: actual STM32H723 eval board or electrically representative test bench for timer + SPI + DMA + parity + cache coherence at selected sample rate. Emulator (Renode/QEMU) is optional and not evidence of electrical peripheral timing unless target and exact devices are supported/validated.
- Use no_std on target; hardware sampling clock independent from scheduler/USB tasks.

### 4. PCB schematic, layout, signal and power integrity

- **KiCad**: ERC zero unexplained errors, all pin electrical types, net labels, footprint/pinout datasheet audit (double-check IC rotation, pin 1 and connector BNC shell), BOM-to-footprint cross-check.
- Ground path/USB isolation/chassis **block diagram and actual net connectivity audit**. No split return path crossing ADC loop; no USB shell accidentally bypassing isolation barrier.
- Select JLCPCB's **actual supported four-layer stackup before routing**; verify minimum clearance/hole, soldermask, assembly/courtyard, trace/copper thickness and via sizes.
- Review fast nets (SPI, ADC conversion, clock, USB) for return path, parasitic loading, ringing and threshold; apply IBIS simulation to critical interfaces where models exist, e.g. TI ADS890XB IBIS. Simple MHz frequency spec doesn't guarantee layout correctness.
- Analyze analog power/reference decoupling with PCB parasitics and worst-case transient noise. Do not assume schematic-only ideal decoupling.
- DRC zero unexplained violations; visually inspect high-resolution 3D rendering, 2D Gerber overlay, paste/solder mask, pin-1 and silkscreen, optical inspection marks, test pads and SWD header.
- Independently compare schematic netlist against PCB-export netlist; review all components and pads.
- Deliberately place test points for rail, reference, ADC driver, conversion trigger and output interlock, but avoid sensitive-node stubs.

### 5. JLCPCB or equivalent DFM / DFA validation

**JLCPCB latest official references checked 2026-10-28**:
- https://jlcpcb.com/help/article/advice-for-bom-and-cpl-files-preparation
- https://jlcpcb.com/help/article/component-matching-guidelines-for-pcba-orders
- https://jlcpcb.com/help/article/terms-and-conditions-of-jlcpcb-assembly-service
- https://jlcpcb.com/help/article/common-pcba-after-sales-issues-and-faq
- https://jlcpcb.com/help/article/multi-layer-pcb-standard-laminated-structures

Checks:
- CAD output: all Gerber copper/mask/silk/paste layers, outline, Excellon drills, fabrication/assembly drawings, pick-and-place/CPL and BOM; generated from exact same Git commit.
- Match all populated designators in schematic ↔ PCB ↔ BOM ↔ CPL, matching case; no duplicate or unselected components; explicit DNP list.
- Validate **every** MPN/package/pin-1/polarity/orientation with manufacturer data + JLC assembly preview. JLC automated matching can be wrong.
- Use latest live JLC parts library; identify exact JLC part code/stock, substitute candidates, minimum quantities, assembly type and lead times. Do not infer availability from schematic part numbers.
- Review assembly capability: IPC-7351B medium/low footprint density, body-to-board-edge limits, assembly side, mounting, keepouts and potential manual soldering of relays/connectors.
- Upload files to vendor pre-order checker / DFM preview and manually inspect all warnings and component selections. Do not submit paid order without release approval.
- Save vendor quote/preview/BOM matching artifacts with date (some are account/session specific; redact private fields before publication).

## Coverage matrix and suggested automated regression

| Area | Unit / model gates | Interface / system gates | Physical requirement |
|---|---|---|---|
| Amplifier | gain/DC/CM/temperature/noise | filter + driver + ADC range | scope, gain/noise test |
| ADC | behavioral quantization/INL | actual sampled signal/DMA errors | real eval board at 1 MSa/s |
| Analog filters | AC/Monte Carlo | alias/interferer tests | VNA/function generator measurements |
| Reference | NCO/PLL synthetic jitter | actual DDS/clock synchrony | phase/noise measurement |
| Rust DSP | golden vectors/property tests | input fault simulation | measured MCU runtime |
| Protection | fault truth table | brownout and output default OFF | independent watchdog and power injection |
| PCB | ERC, DRC, pin/footprint audit | Gerber-vs-netlist/BOM/CPL | assembly inspection and bring-up |
| Metrology | end-to-end noise model | calibration and source Z | calibrated bench characterization |

## Non-negotiable release gates

**Gate V0** — Model provenance, selected exact MPN/package, requirements and falsifiable acceptance metrics documented.

**Gate V1** — 100% of designated analog circuit test scenarios produce pass evidence, or resolved NO-GO/redesign. All unknown model limitations documented. Every input range and CM combination meets margins without hitting absolute maximum ratings.

**Gate V2** — Deterministic digital system regression passes including 900 kHz alias injection and explicit invalidation on dropped samples, clipping and unlocked reference; timing/phase conventions specified.

**Gate V3** — Physical or manufacturer-evaluation-board proof of exact ADC–MCU SPI/DMA acquisition at maximum approved sample rate. **Simulation alone cannot close this gate**.

**Gate V4** — KiCad ERC/DRC clean; manufacturer pinout/footprint cross-check; tested safe-off fault logic; signal/ground/isolation/thermal layout review.

**Gate V5** — Gerber/BO​M/CPL/assembly preview fully reconciled, in-stock components selected, fabrication stackup and assembly rules documented, independently reviewed.

**Gate V6** — Explicit final sign-off with limitations and known risks. Order **small first-article batch**, not production quantity.

Physical **Gate P** after fabrication: current-limited bring-up, thermal and rail inspection, reference stability, analog noise, amplitude/phase/alias test, physical watchdog brownout, 72 h soak. These are not covered by V0–V6.

## Reproducible repository structure to create progressively

\`\`\`text
hardware/
  kicad/                # actual schematics, symbols, PCB, library tables
  spice/                # source decks, vendor models where redistribution permitted
  fabrication/          # versioned Gerbers/BOM/CPL/assembly artifacts (release only)
sim/
  analog/               # AC, DC, noise, transient, Monte Carlo drivers/reports
  system/               # Python SciPy reference sampled-signal simulation
  vectors/              # seeded input stimuli, expected outputs
firmware/
  dsp-core/             # portable Rust reference DSP and tests
  stm32/                # no_std hardware supervisor and data acquisition
verification/
  reports/              # CI logs, requirements cross-reference, design waivers
.github/workflows/
  verify.yml            # pytest/Rust tests, KiCad ERC/DRC when design exists
\`\`\`

No empty placeholder directory needs to be created until files exist. Respect vendor redistributable SPICE model terms; download restricted models during authorized builds rather than commit them.

## Immediate action order

1. Freeze a machine-readable Rev A requirement/acceptance matrix, identify exact MPNs and get manufacturer SPICE model source files legitimately.
2. Simulate **AD8421 and THS4551/ADS8900B analog sections independently**, especially headroom and passband gain/phase/noise, while deriving worst-case alias rejection.
3. Implement a Python end-to-end behavioral lock-in simulator with seeded noise, nonlinear clipping, ADC quantization/jitter, reference drift, and measured filter transfer functions; compare Rust DSP to golden vectors.
4. Validate STM32 timer/SPI/DMA using inexpensive dev hardware before final pin routing.
5. Create KiCad, automated ERC/DRC, enclosure/chassis layout and full JLCPCB BOM/CPL/DFM checks.

**Status on creation: ALL GATES OPEN. None of these simulations has yet been executed.**

# Engineering design plan — Rev A (pre-schematic)

**State:** provisional architecture, not an approved manufacturing specification. Date: 2026-10-28.

## Design objective and scope

Create a low-cost, dependable scientific **digital lock-in amplifier** with one high-quality differential **voltage** measurement channel; external/internal reference; internal excitation; programmable digital filtering; robust error detection; reproducible calibration; and desktop/automation interface.

The instrument is for **low-voltage bench measurements only**. No assumption of safe connection to mains, floating high potentials, or arbitrary electrical systems.

Avoid feature inflation: no FPGA, TFT, high-current source, arbitrary waveform generator, gigabit network, or TIA current input in Rev A. Leave well-defined expansion opportunities.

## Revision-A target matrix (nothing yet verified)

| Metric | Proposed target | Qualification caveat |
|---|---|---|
| Primary measurement band | 1 Hz to 50 kHz | qualification across gain/ranges, reference lock and filters |
| Extended measurement band | Up to 100 kHz | conditional on validated anti-alias rejection and amplitude/phase behavior |
| ADC | 20-bit, maximum 1 MSa/s | ADC nominal bits != effective resolution |
| Input noise | <= 5 nV/√Hz at 1 kHz | defined low-Z source / high gain / temperature / termination |
| Nominal ranges | ±10 V, ±1 V, ±100 mV, ±10 mV | subject to headroom/protection and calibration |
| Analog gain | 1 / 10 / 100 | break-before-make and fail-safe relay logic |
| Voltage input | differential IN+ and IN- | practical CMRR, CM voltage, connectors and shield grounding TBD |
| Nominal termination | 50 Ω or 1 MΩ | verify signal loading and resistor noise; exact topology TBD |
| Synchronous outputs | X, Y, R, phase | define peak vs RMS conventions |
| Digital filters | 1–4 pole / time constants | exact filter design and numerical limits TBD |
| Simultaneous demodulators | target 1–4 | performance/CPU gate before commitment |
| Reference | internal DDS or protected external | external lock quality, phase tracking TBD |
| Continuous mode | timer-driven ADC + DMA | no unreported samples lost |
| USB | isolated full-speed transport | aggregate streaming after decimation, not raw 1-MSPS |
| Prototype power | certified external SELV adapter | rails chosen from headroom and loss budget |
| PCB | 4-layer, planned ~120 x 100 mm | sizing may change with review |

## Candidate hardware chain

IN+ and IN- BNC centers → well-defined protection and fail-safe selectable termination / attenuation → AD8421 instrumentation amp, relay-selected gain → high-order **anti-alias network to be designed from an alias budget** → THS4551 differential drive / charge-kickback network → ADS8900B → clocked SPI/DMA → STM32H723 Rust DSP → isolated USB → PC.

Parallel channels:
- DDS: shared reference clock → AD9833 → reconstruction filter → calibrated output conditioning → **hardware output disconnect** → EXC OUT BNC.
- REF IN: protected input / level adaptation / hysteresis → MCU timer-capture reference tracking. Optional edge-based/analog phase recovery must be validated.
- Reliability: voltage/temperature monitors, separate watchdog, valid-data flags, immutable factory calibration + verified versioned user records, self-test/loopback using safe attenuated signal.
- Power: certified external isolated SELV adapter, appropriately separated switching converters and quiet post-regulation, defined analog chassis/digital return currents and USB-side isolated power.

### Why these parts?

- AD8421BRZ: instrumentation amplifier provides differential input and resistor-selected gain; manufacturer states 3.2 nV/√Hz **maximum at 1 kHz**, 200 fA/√Hz input current noise, G=1 BW 10 MHz and G=100 BW 2 MHz. **Does not establish full system noise.**
- ADS8900B: 20-bit 1 MSa/s SAR, 104.5 dB typical SNR, configurable parity, enhanced SPI 22 MHz at 1 MSa/s; verify whether MCU peripheral supports the selected transaction framing and DMA correctly.
- THS4551: suitable precision differential driver; actual 1 MSa/s settling, amplifier noise, loop gain, supplies and target REF/common mode must be simulated.
- REF5045: 4.5 V reference candidate within ADC VREF range; total reference noise/drive requirements, filtering and layout TBD.
- STM32H723: candidate CPU; no demonstrated DMA plus demodulation profile exists.
- AD9833: 28-bit DDS with external MCLK; **clock/reference phase coherence and tuning granularity must be established**, including deterministic reset and phase offsets. Do not assume output amplitude is a user-settable precision value without an output gain/attenuation circuit.
- ISOUSB111: full-speed USB galvanic isolation with independent barrier-side power and appropriate board clearances; not a license for hazardous voltage input.

## High-risk electrical budget work, required first

### Input voltage and CM budgets

For every range and gain, work out:
1. Differential peak input, allowable DC CM and CM transients, unpowered/powered fault envelope.
2. Attenuator ratios and resistor voltage, precision, thermal noise, capacitance, impedance/loading and dissipation.
3. AD8421 input headroom vs output headroom for selected analog rails; gain resistor values and noise; output reference pin offset and slew/settle.
4. Anti-alias active-stage supply, output swing, noise, THD and overload margin.
5. THS4551 input common mode, output common mode and swing relative to supplies; full-scale 2-input ADC voltage around VREF.
6. ADC input common-mode, dynamic kickback, reference requirements and absolute limits.
7. Relay control during fault/power-off: default attenuation enabled, G=1, output off. Verify actual relay topology does this.
8. AC/DC coupling: **DC-coupled baseline**. AC-coupling is a future/optional mode requiring measured corner-frequency and recovery. Do not silently add a leaky or noisy capacitor in the sensitive path.

### Input noise budget

Include spectral densities from sensor source Johnson noise (sqrt(4 k T R)), AD8421 voltage and current noise, resistance/gain network, later amplifier stages input-referred by gain, sampling uncertainty, reference and power contamination. State assumptions and uncertainties.

Example: at 300 K, a 200-ohm source has ~1.82 nV/√Hz thermal voltage noise. With 3.2 nV/√Hz amplifier voltage noise, ideal voltage-noise RSS alone ~3.68 nV/√Hz **before** current noise, resistor noise and later stages. This is not measured nor a guarantee of <=5 nV/√Hz.

Measurement bandwidth must be explicit: white voltage noise RMS = spectral density × sqrt(equivalent noise bandwidth). For lock-in low-pass filter, derive ENBW based on actual numerical transfer function, not a generic 1/tau approximation.

### Aliasing is a formal design gate

A 20-bit / 1-MSa/s ADC does not imply usable 20-bit rejection of signals above Nyquist. For a 100 kHz wanted signal, an interferer at **900 kHz aliases to 100 kHz** when sampling at 1 MSa/s. A nominal fourth-order 150 kHz Butterworth LPF attenuates ~62 dB at 900 kHz, insufficient to promise robust 20-bit interferer suppression. That earlier filter suggestion is **rejected as an unqualified final design**.

Derive acceptable worst-case out-of-band interfering amplitude, overload levels, target alias floor (in ADC counts/input-referred volts), needed order/topology, filter group delay and calibration. Evaluate 6th-/8th-order alternatives, passive/active partition and narrow-band switchable filter options. Choose a simpler qualified 50 kHz band if needed.

### Timing and data path

At 1 MSa/s × 20 bits ≈20 Mbit/s minimum ADC measurement payload before parity/overhead. USB full-speed 12 Mbit/s **cannot stream continuous uncompressed full-rate samples**. Stream decimated demodulation products, bursts, bounded diagnostics, and status. Hardware-driven conversion timing + measured DMA throughput + cache coherent buffers are mandatory.

Reference measurement:
- The internal DDS and MCU sampling clock may derive from the same oscillator; verify PLL relationships, phase resets, actual sample frequencies and drift.
- External reference is a separate synchronization problem; provide lock/unlock indication, jitter/phase error budget, harmonic ambiguity resolution and timeout.

## KiCad sheet hierarchy (to be created after gates)

1. 01_INPUT_BNC_PROTECTION.sch — connectors, shielding, input clamps, mode/range relays.
2. 02_AD8421_GAIN.sch — low-noise amplifier, gain selection, power and control.
3. 03_ANTIALIAS_DRIVER.sch — full filter and THS4551 network.
4. 04_ADS8900B_REFERENCE.sch — converter, REF5045, 5 V/logic and precise CONV/SPI.
5. 05_CLOCK_REFERENCE.sch — TCXO, input reference comparator, timer.
6. 06_DDS_EXCITATION.sch — AD9833, filter, buffered excitation and hardware inhibit.
7. 07_MCU_USB.sch — STM32H723, USB isolator, SWD, hardware control.
8. 08_POWER_SUPERVISION.sch — DC input, protection, rails, watchdog, monitoring.
9. 09_SELFTEST_CALIBRATION.sch — controlled loopback, internal stimulus and storage.

Use KiCad native projects, manufacturer-verified symbols/footprints, ERC; BOM keyed by refdes, MPN, package, tolerance, stock status and second source. Preserve actual component reference topology whenever possible.

## Layout rules (after design review)

- Four-layer preferred: signals / unbroken ground / power / signals. Split by **placement and return current**, not by cutting the only ground reference.
- Physically isolate analog input from DDS, MCU clock and switching supplies. Keep analog filter, ADC driver, ADC and reference physically close.
- Provide RF/ESD ingress strategy at connectors, leakage-controlled sensitive input layout, Kelvin reference returns, appropriate decoupling, proper BNC shell-to-chassis strategy.
- Review USB isolation clearance/creepage and isolated DC/DC. Ground loops must be assessed on the actual enclosure.
- Include reworkable packages where possible, SWD, test headers and safe probing.
- Conduct independent schematic + layout peer reviews and ERC/DRC before fabrication.

## Firmware and application

- STM32 Rust no_std application, bounded deterministic acquisition, DMA-safe statics, explicit unsafe review and panic behavior.
- Instrument state machine: OFF → SELF_TEST → IDLE → SETTLING → ARMED → MEASURING / FAULT; no excitation unless independently valid hardware enable.
- Sample metadata: monotonically increasing index, timestamp/frequency configuration, gain, calibration ID, reference status, overload/ADC parity/overrun status.
- I/Q convention: I = LPF{2 v cos θ}, Q = LPF{-2 v sin θ}; R=sqrt(I²+Q²) = **peak sinusoidal amplitude** under this convention; RMS=R/sqrt(2). State DC and harmonics separately.
- Golden vectors for phase reversal, noise, close-spaced tones, dropouts, reference jumps, gain changes, numerical overflow and filter transitions.
- Versioned CRC-protected host protocol with bounded packets and explicit errors; SCPI-compatible command vocabulary optional.
- Desktop Rust GUI, Python automation binding and straightforward data export. GUI is not part of measurement timing or safety interlock.

## Phase plan

P0: formal electrical budgets, reference design audit, risk register and priced BOM candidates.
P1: SPICE and Python reference simulations; ADC-SPI-DMA prototype; end-to-end DSP feasibility.
P2: hierarchical KiCad and independent review, then fabrication gate.
P3: careful low-risk bring-up with current-limited supplies, rail checks, safe unarmed state, clock/ADC and self-test before external signals.
P4: full characterization, calibration, 72 h soak, thermal drift, interferer and fault injection.
P5: only after P4, iterate price/production/performance/optional 100 kHz extension.

See ENGINEERING_GATES.md for mandatory hold points.

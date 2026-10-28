# Engineering gates and acceptance matrix

**Status:** proposed test plan, **zero tests executed**. No manufacturing approval. 2026-10-28.

Use explicit GO/NO-GO with evidence, not descriptive confidence levels. No gate passes because datasheet nominal specifications look favorable.

## P0 — requirements and integration feasibility (required before schematic design freeze)

| Gate | Evidence needed | Pass/NO-GO |
|---|---|---|
| P0-1: pin/rails | All manufacturer-revision-linked pinout, power, logic, footprint and absolute-max tables; check 1.8/3.3/5 V interfaces | All net interfaces and legal voltages shown |
| P0-2: signal headroom | Per-range differential and common-mode math for full-scale and transients, powered/unpowered | No stage exceeds operating/absolute limits across intended conditions |
| P0-3: noise | Full 1 Hz–100 kHz (where applicable) input-ref noise spectrum, every gain and source Z, 1/f, thermal/current, simulated ADC/filter contributions | Meeting noise target plausible with traceable margins; specify limits |
| P0-4: alias/interferer | System alias budget for fs ± fin, 2fs ± fin, large interfering tones and filter passband | Demonstrate quantified rejection and no ADC/analog clipping |
| P0-5: sample acquisition | ADS8900B enhanced SPI framing/parity, CONV timing, STM32H723 pin/clock/DMA feasibility | Timing diagrams, memory topology, observed bench throughput if available |
| P0-6: generator | DDS output signal/clock/phase behavior, analog amplitude range, filtering, output-disable circuit | Predictable safe off; waveform fidelity and calibration strategy |
| P0-7: USB/chassis | ISOUSB111 full-speed, both-side supply, chassis, shield, adapter and BNC return diagram | No hidden ground-bypass path or unsupported isolation claim |
| P0-8: cost and availability | Per-MPN manufacturer/distributor part numbers and live stock plus alternatives | Reasonable BOM; no unobtainable single point failure |
| P0-9: hazardous input | Low-voltage use envelope, surge/ESD protection leakage, power-off fault analysis | Limits visibly documented; no hazardous-voltage rating claims |

**STOP** if any P0 item lacks evidence. Reduce promised bandwidth/range or change part rather than hand-wave.

## P1 — simulations and firmware feasibility (before PCB layout)

- SPICE tests: AD8421 each gain; +CM and −CM; max/mid/small amplitude; power-up/down; relay bounce/open/short; overload recovery; input clamp injection; noise and GBW; THS4551 + ADC differential swing and transient response.
- Full analog anti-alias response with worst-case R/C tolerance and finite op-amp GBW. Evaluate response **after** ADC driver/charge kickback network; exact poles, output impedance and stability.
- Full-rate acquisition: timer-triggered 1 MSa/s with SPI/DMA and parity on real MCU/dev hardware, verified with pseudorandom or ADC known pattern, no missed frame over 24 h. Record clocks, DMA ring design, cache, IRQ latency and CPU load.
- Multi-demodulator worst-case CPU profile at 1 MSa/s under USB traffic, sweeps and fault/status handling. Commit reproducible benchmark. If >70% sustained CPU or missed deadlines, lower concurrency / decimate earlier.
- Numerics against Python golden vectors: <0.1% amplitude / <0.1° phase error on nonpathological coherent ideal signals after settling; document ENBW vs time constant; include underflow, overflow, cancellation and acquisition gaps. These are **DSP software** tests, not full-instrument accuracy.

## P2 — schematic/layout manufacture release

Required artifacts:
- Exact KiCad project and complete ERC-clean schematic.
- Footprints cross-checked against package drawings and orderable MPN, polarity and pin 1 marks.
- Manufacturing BOM with MPN, manufacturer, acceptable alternates, tolerance/temperature coefficients, supplier links, unit pricing and stock checked on a stated date.
- Documented power sequencing and every fault output/control state, including MCU not populated, boot loader, SWD programming, watchdog timeout, broken USB, failed relay and VIN brownout.
- Layout DRC clean; sign-off for ground, return currents, sensitive loops, anti-alias filter, reference pin, ADC input, USB isolation clearance/creepage, chassis connector strategy and test points.
- Independent human/agent design-review checklist and issue dispositions.
- Reworkability and DFM/DFA review; assembly drawings, pick-and-place, fabrication outputs after sign-off.

**No boards ordered before P2 complete.**

## P3 — first-power bring-up with no external sensor

1. Visual inspection, resistance-to-ground and short checks, polarized component checks.
2. Current-limited supply, limited-current lab procedure, check rails in sequence, ripple/heat/supervisor.
3. Verify MCU boot/flash/SWD, failsafe reset and watchdog; **excitation must remain electrically OFF**.
4. Verify TCXO, MCU timing, ADC reference stability, conversion sampling, SPI/DMA/parity and data integrity.
5. Apply safe internal test signal, verify all gains/ranges, hardware interlock, loopback. Compare to predicted DC/midband results.
6. Test input overrange and reference loss only within qualified electrical input envelope.

Document voltages, currents, scope captures, raw data, anomalies, part substitutions and firmware version.

## P4 — measured instrument acceptance targets

These are proposed goals. **Failing target does not mean hardware is unsafe; it means claims cannot be made.**

| Test | Target | Essential conditions |
|---|---|---|
| Input noise density | <=5 nV/sqrt(Hz) at 1 kHz | low-Z shorted/terminated input, high gain, fixed temp, reference conditions, PSD estimator and calibration; subtract **nothing** without explanation |
| Primary frequency band | 1 Hz–50 kHz | calibrate each range and gain; specify AC/DC behavior, time constant, phase reference |
| Conditional extended band | up to 100 kHz | ONLY after alias, phase, linearity, interferer and timing criteria demonstrated |
| Amplitude accuracy | <=0.5% through 50 kHz (stretch) | calibrated relative to traceable source, linear input, low-source Z, rated amplitude range |
| Phase accuracy | <=1° through 50 kHz (stretch) | calibrated with defined reference point and stable clock |
| Range/gain switching | no hidden invalid samples | settling window flagged; compare before/after baseline, monitor transients |
| Acquisition integrity | 0 **unreported** drops/parity errors | >=24 h full sample-rate stress; arbitrary acquisition errors explicitly flagged |
| Soak | >=72 h without unreported measurement errors | repeated sweeps, USB activity, power and temperature monitored |
| Reference loss | invalid flag + defined safe behavior | missing, wrong frequency, jittered and disconnected reference |
| Power failures | excitation OFF; restart in safe IDLE | abrupt removal, brownout, adapter bounce, MCU reset/halt |
| Alias rejection | meets specific quantified interferer budget | swept unwanted tones near fs ± f, including overload |
| Harmonic/spur response | measured and reported | DDS leakage, input nonlinearity, EMI from MCU and power |
| Calibration persistence | version/hash/check integrity | interrupted writes, corrupted user record, restore factory |
| Thermal response | report 15–35°C characterization | at least two gains, source impedances and input ranges |

## Measurement validity bits (minimum)

- CONFIG_VALID
- CALIBRATION_VALID / CALIBRATION_EXPIRED
- POWER_OK
- ADC_PARITY_OK
- NO_SAMPLE_LOSS
- NO_ANALOG_OVERLOAD
- REFERENCE_LOCKED
- GAIN_SETTLED
- EXCITATION_ARMED
- SELF_TEST_PASSED

Every reported result includes a validity mask and settings ID. Do not treat calibrated values as valid after changing gain, clock, analog filter, reference mode or relevant temperature before correction.

## Documentation and reproducibility

For each gate save: dated test plan; equipment model and calibration status; wiring diagram; board revision and serial; firmware commit; data; numerical processing scripts; pass/fail; remaining uncertainties; reviewer. Preserve failed tests as first-class artifacts.

If a target is relaxed, open a dated change entry in DECISIONS.md with rationale and impact on product claims; do not quietly modify the result after testing.

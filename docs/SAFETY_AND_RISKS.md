# Safety, fault tolerance and risk register — Rev A

**Safety classification:** LOW-VOLTAGE LABORATORY PROTOTYPE ONLY. Not a CAT-rated multimeter, safety monitor, isolated differential voltage probe, medical device, hazardous-voltage instrument or certified laboratory appliance. No safety standard compliance has been established.

## Fundamental safe state

On power loss, power ramp, supply brownout, MCU reset/hang, watchdog timeout, uninitialized IO, relay failure detection or firmware panic:
- **Excitation output electrically OFF by independent hardware circuit** (do not rely solely on a GPIO).
- Default measurement front-end **maximum attenuation + unity gain**, to the extent the selected relay topology actually enforces this when unpowered; validate.
- Acquisition disarmed and all results marked INVALID until rail/reference/ADC checks, calibration read, settling and explicit operator arm complete.
- Preserve log/cause if possible, without depending on logging for safe state.
- Software cannot override analog hardware inhibit on undervoltage or independent watchdog fault.

Use normally-open excitation disconnect with a properly biased gate/default state, current-limiting and output protection; a dedicated interlock state machine with hardware supervisory inputs must be designed and fault-injected.

## Isolation and physical topology

A USB data isolator (ISOUSB111 candidate) only isolates the USB data barrier if power/return/chassis paths are separately designed. **It does not create safe isolation between the input BNC centers, shield/chassis, power adapter and the user.** Provide connector/chassis relationship diagram and accessible metalwork protection for SELV use. No hazardous voltages allowed on signal or reference terminals.

Use a certified, isolated, low-voltage SELV DC power adapter with input protection, reverse-polarity protection, current limiting and monitored rails. No internal mains supply in Rev A. Respect manufacturer's requirements for creepage/clearance for the USB isolation circuit; no user claim of high-voltage system isolation.

## Preliminary FMEA (design-stage, not validated)

| Failure / threat | Consequence | Proposed mitigation | Validation |
|---|---|---|---|
| MCU hangs, GPIO floats | stale measurement; unintended excitation | independent watchdog + hardware gated output; invalid status | halt CPU, disconnect debug, inject reset |
| Brownout/undervoltage | corrupt ADC readings, unstable drive, relay chatter | power supervisors, interlock, hysteresis, power-on delay | sweep/cycle VIN |
| Gain relay stuck closed / cross-connect | ADC saturation, input stress | break-before-make state sequencing, contact-state diagnostics if feasible, conservative drive | forced relay failures |
| Attenuator bypass on boot | overload damage | hardware default attenuated path, test when unpowered | power-off input analysis |
| External overvoltage or ESD | input component damage, false readings | protected input with low leakage/low noise compromise, absolute-max budget | safe bounded fault pulses, ESD standard later |
| Common-mode violation | in-amp saturation, apparent signal | explicit common-mode envelope, differential CM protection, invalidation | CM ramp at each gain |
| Broken/shorted BNC shield | ground-loop, exposure to equipment potential | proper low-voltage physical grounding and user labeling | chassis continuity review |
| DDS glitch / stuck enabled | energizes DUT unexpectedly | analog switch/relay + hardware inhibit, output voltage/current limiting | forced MCU hang/boot and fault |
| DDS spur | false lock-in harmonics | reconstruction filtering, source isolation, spectral verification | spur/FFT test |
| ADC sample drop or parity fail | biased amplitude/phase, silent false result | parity, DMA ring overruns, sequence counters, invalidation | injected errors |
| Reference lost/unlocked | meaningless phase/value | separate lock state, timeouts, no hold-last masquerading as measurement | jitter/outage sweeps |
| Calibration corrupted | systematic measurement bias | two versioned records with CRC/hash, immutable factory image, readback | interrupted writes and bad CRC |
| USB link drop | stale host data or runaway sweep | explicit timeouts, run state policy, freeze invalid display | unplug/replug repeatedly |
| Clock drift/jitter | incorrect frequency/phase | characterized TCXO, clock-domain plan and calibration | phase drift and temp test |
| Aliased interfering signal | credible but wrong coherent amplitude | quantitative anti-alias rejection and overload flags | swept interferer near fs ± fin |
| Thermal damage / supply short | board destruction or fire | qualified adapter, eFuse/current limiting, fusing, thermal behavior | controlled fault-current tests |
| Firmware update interruption | unusable instrument or unsafe startup | safe bootloader, verified image, no activation until good image | interrupted update simulation |

## Redundancy strategy: targeted, not gratuitous

Useful independent protection layers:
- Firmware watchdog **and** external hardware supervisor (separate failure modes).
- Software output arm **and** analog excitation disable circuit.
- Input overload measured by ADC **plus** optional independent analog comparator for gross front-end clipping.
- Factory calibration immutable + two integrity-checked user records.
- Sample parity + overrun detection + application sequence/health counters.
- Actual protected external reference + internal DDS self-test path.

Avoid an uncalibrated second ADC channel marketed as redundant: it adds complexity, noise and calibration risk without automatically making readings safer.

## Analogue-specific checks

- AD8421 internal ESD/overvoltage protection does not eliminate need to assess off-state input current, power rail injection, protection leakage, resistor noise, differential overload and long-term damage.
- 50 Ω termination introduces large DUT load and heat under accidental input voltage. Provide explicit connector markings/range warnings; verify resistor pulse power and switching.
- High-Z input with an instrumentation amplifier has bias-current-related voltage error and source-current noise; never imply ideal input insulation or unchanged sensitivity for megaohm sources.
- Avoid placing noisy switching-supply traces under BNC/input, AD8421, precision reference or charge-kickback loop.
- Emphasize fast recoverability/repair for V1: accessible test points, normal packages where practical, per-rail isolation and connector/service documentation.

## Status and risk severity

**Open critical pre-fab risks:** input protection vs noise, ±10 V range and CM budget, high-order anti-alias response, full-rate enhanced-SPI/DMA viability, clock coherent DDS/reference, power/chassis return strategy, excitation OFF interlock.

Every one of these needs a named test case and an explicit PASS before board fabrication. See ENGINEERING_GATES.md.

# Handoff checkpoint

## Checkpoint — 2026-10-28: Repository initialized; documentation only

**Repository:** https://github.com/Kajin-0/lockin  
**Default branch:** main  
**Phase:** P0 architectural feasibility; **no PCB order authorized**.

### What was done

- Created README.md to establish project scope and proposed Rev A component set.
- Created AGENTS.md for mandatory recovery rules.
- Created docs/DESIGN_PLAN.md architecture and stage plan.
- Created docs/ENGINEERING_GATES.md verification gates/test matrix.
- Added docs/VIRTUAL_VERIFICATION.md with vendor-supported SPICE/model plan, seeded system simulation, Rust DSP comparisons, KiCad/ERC/DRC, JLCPCB BOM/CPL/DFM, release gating and limits of simulation. **Plan only; no tests executed.**
- Added docs/MULTIPHYSICS_VERIFICATION.md for coupled electrothermal drift, PCB EM/crosstalk, precision reference/power distribution, optional mechanical/enclosure analysis, uncertainty/mesh convergence, and mandatory physical first-article evidence. **Plan only; MP0–MP5 all OPEN.**
- Created docs/SAFETY_AND_RISKS.md safe-state rules and FMEA.
- Created docs/DECISIONS.md ADR record and unresolved choices.
- Created docs/REFERENCES.md manufacturer source links.
- Created this handoff record; agent.md points to standard AGENTS.md.

### What is **not** done

- No schematic, KiCad project, netlist, PCB, GERBER, purchased BOM or physical circuit.
- No measured performance, no numerical simulations, no hardware firmware or acquisition benchmark.
- No component pricing/stock audit, full pin/rail/headroom check, source-impedance noise simulation, anti-alias filter design or lab safety qualification.
- No accepted output stage, gain relay part, fault comparator, supply selection or verified USB/chassis plan.

### Important corrections and lessons captured

1. User chose **most feasible high-quality first PCB**, not maximum MFLI-like bandwidth.
2. Candidate architecture stays 20-bit 1-MSa/s MCU based, with precision analog input and calibration.
3. **50 kHz primary measured-qualification goal; 100 kHz conditional.** An earlier suggested four-pole 150 kHz LPF cannot guarantee strong alias rejection of 900 kHz→100 kHz unwanted energy at 1 MSa/s.
4. AD8421 manufacturer 3.2 nV/sqrt(Hz) maximum voltage-noise spec does not alone prove the whole instrument <=5 nV/sqrt(Hz. Current noise, source Z, resistor and ADC contributions must be modelled.
5. Software in Rust improves memory/state safety; it cannot replace measured DMA throughput and independent hardware output inhibit.
6. USB isolation does not make inputs safe for hazardous voltages.

### Next three concrete actions

1. Create docs/ELECTRICAL_BUDGET.md with per-range CM, voltage headroom, stage gain, rail, clipping and fault path tables, grounded in exact latest datasheets and pin tables. Keep numerical unknowns explicit.
2. Create reproducible analog noise + anti-alias simulations (e.g. Python scientific scripts and/or SPICE models) with adversarial 900 kHz alias test, target source impedances, frequency response, gain and THD estimates. Add objective pass/fail output.
3. Determine exact STM32H723 variant, inspect SPI timing/enhanced SPI modes, DMA SRAM/cache map and worst-case demodulation budget; ideally execute on an STM32 development board before a full custom PCB.

### Definition of a good next checkpoint

Commit testable budget/simulation evidence, update DECISIONS.md open-item register and ENGINEERING_GATES.md pass/fail, state tool versions/commands, record current head commit SHA and give the next three actions. Be precise about untested assumptions.

### Status log

- **2026-10-28:** Initialized. Planning and reference collection only. All P0/P1/P2 acceptance gates OPEN. No tests run. No known hardware to connect or energize.

# Multiphysics, co-simulation and physical-test minimization plan

**Date:** 2026-10-28
**Status:** PLAN ONLY. No FEM, FDTD, parasitic extraction, coupled analysis or real-world qualification has run. No PCB geometry exists.
**Primary goal:** maximize first-spin success and minimize the number of physical prototype redesign cycles by making simulation falsifiable, reproducible, conservative and connected to instrument-level specifications.

## The non-negotiable boundary

**Do not claim that simulation replaces physical qualification.** A validated model is conditional on input material properties, boundary conditions, component models and mesh/numerical correctness. The fabricated board will still need a short but indispensable set of physical first-article tests before meaningful scientific metrology or safety claims.

Distinguish:
1. *Eliminable physical experiments*: expansive parameter sweeps, ideal and adversarial DSP cases, many alternative filter layouts, thermal placement studies, geometry variants, and firmware fault permutations.
2. *Reducible experiments*: frequency/noise curves and environmental points after model calibration and sensitivity analysis; full-range qualification can be minimized but not omitted for any promised condition.
3. *Irreducible evidence*: actual PCB assembly and electrical continuity; safe first power-on; actual ADC/SPI/DMA; real input noise/EMI/phase and amplitude calibration; watchdog/disable fault response; thermal drift; shielding/cable effects; long-term operation.

No unsubstantiated numeric "percentage confidence" in the project. Report *test coverage*, solver convergence, sensitivity bands, measured model residuals and actual gate pass/fail.

## What physical mechanisms matter for a 1-MSPS / <=5 nV/sqrt(Hz) lock-in

Rank by **metrology impact × model predictive power × computation cost**; avoid running huge solvers because they sound sophisticated.

| Domain | Core risk / coupled output | Suggested model | Priority |
| --- | --- | --- | --- |
| ADC driver + reference | charge kickback, settling, reference impedance, dynamic voltage error | SPICE, vendor macromodel, PDN extraction | P0 critical |
| PCB supply / return currents | regulator ripple, transient droop, reference noise, ground coupling | SPICE + compact parasitic RLC and PDN network | P0 critical |
| Thermoelectric drift | power hotspots → thermal gradients → ref/gain/offset/frequency drift | steady/transient 3D thermal FEM back-annotated into SPICE | P1 critical |
| Parasitic EMI/crosstalk | SPI/DDS/MCU currents capacitively/magnetically couple to input | analytical near-field model → selected 3D EM/FDTD | P1 critical |
| Anti-alias and shielding | out-of-band tone alias, coupling at fs±f, harmonics | extracted parasitic SPICE + sampled-system model | P0 critical |
| Mechanical deformation | package and board strain changes precision components and connector loads | structural / thermo-mechanical FEM | P2 optional |
| Enclosure airflow | hot spots vs convection, natural convection / conduction and shielding | compact thermal RC → enclosure FEM/CFD when justified | P1 medium |
| Vibration/microphonics | capacitor/connector piezoelectric pickup, cable motion | mechanical modal FEM + empirical coupling model | P3 low until measured concern |
| ESD, radiated compliance | manufacturing, cable dependence and stress-induced failure | targeted EM and protection-current SPICE *only as supporting evidence* | cannot be signed off without test |

## Free / open tools (choose on coverage, not tool count)

- **KiCad**: schematic, board, ERC/DRC, stackup and STEP / XAO / BREP geometry exports; https://docs.kicad.org/master/en/pcbnew/pcbnew.html
- **ngspice / LTspice / PSpice for TI**: device-level analog, rail transient, noise, frequency, transfer function and Monte Carlo. Vendor models may be proprietary and incompatible across engines; note each assumption. https://www.kicad.org/discover/spice/
- **Elmer FEM**: conduction, static current, electrostatics, heat and selective coupled FEM; https://github.com/ElmerCSC/elmerfem and https://docs.csc.fi/apps/elmer/
- **FreeCAD FEM + CalculiX**: structural, thermomechanical and enclosure geometry; integrate with Elmer where practical; https://freecad.github.io/Website/news/getting-started-with-fem/
- **Gmsh**: geometry preparation, mesh generation and mesh convergence; https://gmsh.info/
- **ParaView**: thermal/field/mesh postprocessing; https://www.paraview.org/
- **openEMS**: full-wave FDTD for *selected* high-speed current-return, coupling, enclosure and connector cases, not mandatory for every low-frequency trace; https://docs.openems.de/en/latest/intro.html
- **Python/SciPy + Rust**: compact thermal networks, uncertainty propagation, response/backannotation, DSP and calibration. Rust host tests and golden vectors in Python.

Commercial COMSOL/Ansys/CST equivalents may help, but are **not required**; do not assume licenses or cloud costs.

## Coupled physics model definitions

### A. Electrothermal digital twin

Geometry: real KiCad PCB outline, copper polygons/planes/traces, via arrays, FR4 stackup, component footprint/case/thermal pad, BNC and relay bodies, mounting screws, metal/plastic enclosure.

Inputs: component worst/idle/active dissipation including regulator loss, conductivity k(T) for copper/FR4/packages, anisotropic board properties, thermal contact resistances and convection/radiation coefficients (with wide uncertainty bounds).

Solve steady and transient:
  rho c_p ∂T/∂t = ∇·(k ∇T) + q'''
with q''' from measured/estimated component powers and, when justified, Joule self-heating from ∇·(σ(T)∇V)=0.

Backannotate temperature at:
- AD8421, THS4551, REF5045, ADS8900B, TCXO, gain resistors, relay, regulators and input protection.
- Input-to-reference *gradient* and heat cycling, not merely max chip temperature.

Map into analog/system model:
- ref: ΔV_REF/V_REF ≈ TC_REF ΔT + nonlinear/hysteresis bounds.
- gain: resistor ratio vs temperature, thermal tracking and resistor self-heating.
- offsets: op amp and thermoelectric junction temperature gradients plus device drift specifications.
- source/reference thermal noise: e_R=sqrt(4 k_B T R) (correct physical thermal temperature).
- TCXO frequency shift: manufacturer temperature-vs-frequency model + uncertain aging.
- ADC effective gain/offset and numerical phasing from changing reference.

**Example sanity check (not design prediction):** a regulator dissipating 0.5 W with local effective thermal resistance 30 K/W causes ~15 K rise relative to a chosen boundary/reference node, *only if* that thermal resistance is valid. 5 ppm/K reference TC × 15 K = 75 ppm (0.0075%) voltage ratio drift; this is merely a sensitivity calculation. Spatial gradients and dynamic changes often matter more than absolute temperature.

Outputs: thermal maps, power-to-temperature sensitivity matrix (K/W), time constants, predicted ppm gain drift, offset µV, phase/frequency drift; confidence interval from uncertain convection and contact.

### B. Electromagnetic coupling / SI and return-path model

Signals of concern: 1 MSPS conversion trigger, high-speed SPI, MCU clock, AD9833 MCLK/output, USB, relay coils and switching regulator fundamental/harmonics.

Start **analytically / circuit-level**, including:
- parasitic C coupling i_C ≈ C_c dV/dt
- mutual inductive coupling v_M ≈ M dI/dt
- shared return impedance ΔV=I Z_return
- ADC driver/ref input susceptibility from SPICE transfer functions

Extract/layout-estimate R, L and C for the 4-layer stack, ground vias/return loops, differential/CM imbalance and shields. Use IBIS where supported. Avoid using one perfect ground node to hide coupling. Add deliberate adversarial injections at aggressor edge rates, clocks, fs±fin and aliases to system model.

Only run openEMS full-wave 3D when:
- fast edge harmonics, connector geometry or enclosure resonances make lumped RLC inadequate;
- a well-defined source field/port, material, geometry and boundary condition exist;
- sensitivity ranking predicts the result can change placement/stackup/return decisions.

Compute transfer functions from each aggressor to a victim: e.g. V_input / V_spi and phase, integrated disturbance through analog transfer into I/Q results. Do NOT equate a visually pleasing E/H field map with a validated dB noise budget.

### C. Power and precision reference co-simulation

Build a worst-case impedance and transient network including DC/DC ripple, LDO PSRR(f), decoupling ESL/ESR, ground impedance, ADC current spikes, reference buffer output impedance and MCU activity.

Determine analogue input-referred error from each rail ripple at user-chosen modulation frequencies and harmonic families; include both amplitude path and phase path. Use temperature-dependent rail behavior from section A.

Test low/high VIN, no/min/max load, USB active/inactive, relay switching, startup/brownout and shorts limited by external supply protection.

### D. Structural and mechanical

Only when there is an enclosure and actual BNC loads, estimate board deflection, mount constraints, package strain and thermal expansion mismatch. Include shock/vibration if application warrants. This is lower value than power/thermal/alias checks on Rev A.

## Uncertainty and model validation discipline

- Record manufacturer source and revision for every electrical, thermal and material parameter. Missing data is a range, NOT invented precision.
- Convergence checks: refine mesh and time step until critical outputs (temperature at REF/AD8421, coupling transfer and input-referred error) change by a stated tolerance (initial goal <2% for defined output); report mesh element count, discretization, solver residual and runtime.
- Interval / sampled sensitivity: conservative worst-case plus uncertainty ranges for k_FR4, copper thickness, contact thermal resistance, convection coefficients, component dissipation, resistor TCs, capacitance, placement, signal edges and clock jitter.
- Separate unit tests, cross-solver verification and **physical validation**. Agreement of two solvers sharing same bad boundary assumptions is not proof of correctness.
- Verify conservation of energy and suitable field/source boundary conditions; do not trust solver output without a sanity check against an analytical or compact equivalent network.
- Thermal backannotation should use board component coordinates and package-level temperature; do not treat air ambient as junction temperature.

## Automation / run economics

Run testing in tiers, tied to one Git commit and content-addressed model inputs:

**Tier 0, every change (seconds–minutes):** analytical budgets, SPICE quick DC/AC, Rust/Python DSP regressions, schema check, PCB ERC/DRC when present. Fail on violations.

**Tier 1, nightly or before design review (minutes–hours depending on hardware):** full Monte Carlo corners, time-domain interference, bounded stress/random campaigns, compact thermal network, rail/PDN transient.

**Tier 2, after placement and before design freeze (hours or longer):** 3D thermal FEM, local EM parasitic/full-wave selected cases, enclosure conduction/convection, tolerance sensitivity, mesh convergence. Benchmarks and compute time determined by actual model complexity; no promises of durations/cost.

**Tier 3, before JLC release:** qualified full evidence bundle: KiCad project/ERC/DRC, frozen stackup and component model, coupled performance budgets, fault-state table, BOM/CPL/Gerber DFM check, reviewer sign-off, unresolved waivers, and proposed low-cost first-article test fixture.

A solver failure, missing macromodel, unverified boundary condition, CPU timeout or absent mesh-convergence report means **NOT VERIFIED**, never PASS.

## Specific experiments worth running

- Analog input ±10 V/±1 V/±100 mV/±10 mV ranges and common-mode under temperature and rails.
- Input source resistance 50 Ω, 200 Ω, 1 kΩ, 10 kΩ, 1 MΩ at all gains (Johnson + amplifier current noise; flag high-Z limitation).
- ADC capture at 1 MSa/s, intermodulation and alias tone at 900 kHz for 100 kHz wanted signal, plus fs−f for all bands.
- Power heater active at worst-case regulator dissipation, evaluate reference-vs-input gradient under no convection / nominal / enhanced convection.
- Aggressor switching synchronized to excitation frequency or harmonic, so crosstalk is **coherent**, cannot average to zero.
- Reference crystal warm-up/frequency drift under nearby regulator heat.
- Brownout and intermittent DC supply with analog output hard-disable, sample validity and recovery.
- Missing input BNC ground, floating differential common-mode, cable capacitance and source loading.
- PCB trace/via RLC extraction for critical reference, clock, conversion and high-impedance nodes; ensure phase perturbation below allocated budget.
- Thermal cycle + mechanical constraints of connectors and enclosure once those are designed.

## How to minimize required real-world testing

Physical first-article tests **cannot be skipped**:
1. Optical and continuity inspection, orientation, shorts, power-off safe-state.
2. Current-limited power-up and independently verify all supply rails / supervisor response / temperatures.
3. Verify ADC conversion timing and sustained DMA/parity/data integrity on **real** MCU/ADC silicon.
4. Calibrated input gain and phase measurements over every *claimed* range/frequency band.
5. Shorted-input low-noise spectrum/PSD and high-Z validation; external noise pickup with real cables and typical bench environment.
6. Direct real-hardware test of hardware excitation disable during reset, watchdog trip, brownout and reference loss (where required).
7. Real input overload and safe-bounded fault recovery, then thermal drift and extended soak.
8. Calibration repeatability and traceability, plus assembly defects/repair inspection.

To reduce cost, build a cheap fixture with test pads and controlled internal self-test injection, pre-qualified ADC/MCU on eval boards, and a scripted test harness that sweeps known stimuli/reads status. A factory test (if offered, contracted and scoped for this exact order) is supplementary; do not presume JLCPCB runs our instrument's metrological test automatically.

JLCPCB background references:
- https://jlcpcb.com/help/article/smt-inspection-and-testing-capabilities (inspections, customizable flying probe / functional testing described)
- https://jlcpcb.com/blog/pcb-functional-testing-guide (distinguishes customer-run FCT from PCB/assembly inspection; confirm service-specific availability with vendor)

## Proposed repository outputs (create when physical design is available)

\`\`\`text
sim/
  multiphysics/
    thermal/          # board geometry export, mesh, solver configs, heat source table
    em/               # aggressor-victim extraction; openEMS only when justified
    pdn/              # rail and reference transient SPICE decks
    mechanical/       # optional enclosure/mount study
    coupling/         # transfer temperature/EM/PDN results into sampled DSP model
    parameter_sets/   # all published material properties, tolerance/uncertainty ranges
    reports/          # mesh convergence, sensitivity, plots and machine-readable PASS/FAIL
verification/
  first_article/      # automatic lab fixture protocol and minimum compulsory tests
\`\`\`

Do not commit restricted vendor models. Preserve CAD version, solver version, mesh algorithm, solver settings, hashes, random seeds, temperature and assumptions in each report.

## Mandatory gates

- **MP0**: independent analytical sanity checks, geometry and physical parameters listed with sources and uncertainty.
- **MP1**: electrothermal case converged, thermal gradient and combined drift budget demonstrated across operating corners.
- **MP2**: PDN/reference transient co-simulation meets noise/settling budget under worst power and digital activity.
- **MP3**: layout-based coupling extraction meets spurious and alias budgets; targeted 3D EM when the model is inadequate.
- **MP4**: independent PCB/DFM/FMEA review and explicit unresolved margin record.
- **MP5**: minimal physical first-article test plan and instrumentation prepared; no simulation claim can waive this.

**At document creation: MP0–MP5 all OPEN. No tests performed.**

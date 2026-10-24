---
name: process-simulation
description: Solve the flowsheet in DWSIM (heat and material balance) with cheme process simulate - write process/simulation.yaml (property package, step specs), check the deviations from the given and assumed values, and fill the missing stream values. Use when outlet temperatures, pressures, flows or compositions of compressors, exchangers, flash drums, columns or recycle loops are needed, or when assumed stream values should be checked against a rigorous calculation.
---

# Process simulation

## Purpose and when to use

`cheme process simulate` builds a DWSIM flowsheet from `process/flowsheet.yaml` and
`process/simulation.yaml`, solves it headless and writes the results back as `calculated` values.
Use it once the flowsheet's steps, streams and feeds stand (`process-design`) and stream values
downstream of the feeds are needed or assumed: outlet states of compressors, pumps, expanders,
heaters and coolers, valves, mixers and splitters, flash drums, two-stream heat exchangers,
shortcut and rigorous columns, and recycle loops. `cheme process propagate` covers only
pass-through, mixer and splitter balances; simulate covers the energy balance and separations.

## Required inputs and missing inputs

- Every feed stream with temperature, absolute pressure, flow and composition (given or
  assumed). A missing one is `sim-feed-incomplete`: ask for it or mark an assumption.
- Components in `process/materials.yaml` with CAS numbers; a component DWSIM does not find by
  CAS needs its DWSIM name under `components:`.
- The property package (`property_package:`), from `thermodynamic-model-selection`, never
  chosen silently. Supported: Peng-Robinson (PR), Peng-Robinson 1978 (PR78),
  Soave-Redlich-Kwong (SRK), NRTL, UNIQUAC, Steam Tables (IAPWS-IF97), CoolProp
  (`cheme process methods`; another name is `sim-property-package`). Record the selection's
  facts as `method_selection: {facts: {...}}`: the translation re-runs the selection and flags
  a package it does not recommend. A model the worker does not run has at most an explicit
  substitute (`methods[...].simulation.substitution`); use it only with approval and record
  why as `method_selection.substitution`. Electrolytes, solids and petroleum
  pseudocomponents are not supported by this route. Where
  DWSIM's binary interaction parameter of PR, PR78 or SRK is not fitted to the data that
  matter (argon/oxygen near pure O2, for example), set the pair under
  `interaction_parameters: [{components: [AR, O2], kij: {value: ..., provenance: ...,
  source: ...}}]`, with the data it comes from.
- The specs of every step (`cheme schema process-simulation` lists them per unit):
  outlet pressure and efficiency of machines, outlet temperature or duty of heaters and
  coolers, the flash temperature or pressure (none: adiabatic), the exchanger's hot or cold
  outlet temperature, the column's keys or stages and end specs. Each spec is a quantity with
  provenance: an efficiency taken from experience is `assumed` with its source. Ask for what
  the design basis does not state; do not invent specs.
- DWSIM: the Docker image (`docker build --platform linux/amd64 -t cheme-dwsim:9.0.5 -f
  docker/dwsim/Dockerfile .` in the cheme-as-code repository) or `CHEME_DWSIM_PATH`. Without
  it, `sim-unavailable`: report it, do not estimate the values instead.

## Workflow and tools

1. Write `process/simulation.yaml`:

   ```yaml
   schema: cheme/simulation/v1
   kind: simulation
   property_package: Peng-Robinson (PR)
   specs:
     K1: {outlet_pressure: {value: 8, provenance: given}, efficiency: {value: 0.78, provenance: assumed, source: vendor typical}}
     E1: {outlet_temperature: {value: 35, provenance: given}, pressure_drop: {value: 0.2, unit: bar, provenance: assumed}}
     C1:
       light_key: C3
       heavy_key: NC4
       light_key_in_bottoms: 0.01
       heavy_key_in_distillate: 0.02
       reflux_ratio: 3.5
       condenser_pressure: {value: 5.2, provenance: assumed}
       reboiler_pressure: {value: 5.5, provenance: assumed}
   ```

   Steps map to unit operations by their DEXPI class; give `unit:` where the class does not
   say. Wet gas cooled below its dew point needs a drain: give the cooler (or the intercooled
   compressor) a second outlet port `condensate` to a product, else the water stays in the gas.
   Give machines a `drive_efficiency` (motor and gear) when their energy stream stands
   for electric power (utility balances, LCA). `exclude: [step or block ids]` simulates the rest: the streams from the excluded part
   need their values (given or assumed). Use it to simulate what the units cover while a part
   (a double column with side draws) is still assumed. Adsorbers and membranes are
   component separators (`recoveries`), a main heat exchanger a multi-stream exchanger
   (`outlet_temperatures` for all streams but one). Flash and column outlets are recognised by port ids (`vapour`, `liquid`, `top`,
   `bottom`), exchanger sides by matching ids (`hot_in`/`hot_out`); else give `outlets:` or
   `pairs:`.
2. `cheme validate`: `simulation.yaml` against its schema, specs for unknown steps.
3. `cheme process simulate --check`: solves and reports, writes nothing. Read every
   `sim-deviation` (a given or assumed value the simulation contradicts) and the summary lines
   (duties, power, minimum reflux, stages).
4. Start a column with the shortcut method (keys, reflux ratio); the minimum reflux ratio and
   stages it prints give the rigorous column (`stages`, `feed_stage`, `condenser_spec`,
   `reboiler_spec`) its first numbers. A reflux ratio below the minimum does not converge.
5. `cheme process simulate --fill [--save process/simulation.dwxmz]`, then
   `cheme process props --fill` for phase, density and enthalpy, `cheme process build`
   and `cheme validate`. Record the run in `PROGRESS.md`. `--fill` also writes
   `process/simulation-run.json` (generated, never edited: convergence, unit results, stream
   values, input fingerprints), which the viewer shows; commit it with the flowsheet.

## Validation and failure handling

- `sim-unmapped`: a step's class maps to no unit, a unit has the wrong number of streams, or
  energy streams do not fit; give `unit:` or split the step.
- `sim-spec`: a spec is missing, contradictory or out of range; the message names it.
- `sim-property-package` (error): the package is not one the worker runs; choose from the
  hint. `sim-method-not-recommended` (warning): the recorded `method_selection` does not
  recommend it; switch to a recommended package or record the approved reason as
  `method_selection.substitution` (then `sim-method-substituted`, info, stays visible).
  `sim-method-selection`: the recorded facts are incomplete or invalid; ask for the missing
  ones.
- `sim-not-converged` (exit 3): no stream value is written (the run is recorded). Check the specs for feasibility (reflux
  above the minimum, outlet temperatures that the duty can reach, a column spec its stages can
  meet), save the flowsheet with `--save` and open it in DWSIM to look.
- `sim-recycle` (info): where a loop was torn; give `tear_streams:` to tear elsewhere, and
  `guesses: {stream: {temperature, pressure, flow, composition}}` in `simulation.yaml` if it
  converges slowly. A guess is no stream value: never put one into the flowsheet as `assumed`,
  where it would stay and contradict the result.
- `sim-deviation`: never overwrite the given or assumed value to silence it. Either the
  assumption is wrong (change it, with a decision in `DESIGN.md`) or a spec is (change the
  spec); an engineer decides which.
- `sim-stale`: feeds, specs, package or DWSIM changed since the values were written; fill again.
- `sim-run-stale` (`--check`): `process/simulation-run.json` records a run from other inputs;
  fill again (`sim-run-missing`, info: no run recorded for the case yet).

Double columns (air separation): the HP column with `condenser: partial` (GAN as
`vapor_flow`, liquid nitrogen as distillate) and `reboiler: false`, LIN as a liquid side draw
from stage 2 (not the condenser stage); the LP column with `condenser: none`, the liquid
nitrogen fed to stage 1, GOX as a vapour side draw from the last stage, LOX by
`reboiler_spec`, and `heat_integration: {condenser_of: <HP column>}`. Read `sim-reboiler-unbalanced`
as the liquid the columns can (or cannot) make, `sim-approach` as a pressure problem. DWSIM's
column solver is sensitive: start with few stages (40 HP, about 60 LP) and the feeds placed as
in the example, then add stages; a column that does not converge is reported, never estimated.

Reactors are simulated from the flowsheet (skill `reactions`): a step with `reactions:` is a
conversion reactor, `reactor: equilibrium` and `reactor: gibbs` DWSIM's equilibrium and Gibbs
reactors (each with its own reaction set). Their spec is `mode` (`isothermal`, `adiabatic`, or
`outlet_temperature` with `outlet_temperature`) and `pressure_drop`. An equilibrium takes ln K
from the reaction's `equilibrium_constant`, else from DWSIM's own formation data, so it may
differ from `cheme process propagate` (ideal gas, the components' `ideal_gas` data): that
deviation is the check. A reactor with one outlet must stay all vapour (the run says when
liquid forms); give it a `liquid` outlet otherwise.

A fixed bed (`reactor: pfr` with `bed:` and `catalyst:`, rate laws in `reactions.yaml`) is a
`kinetic_pfr` unit, **coupled as an external unit, not a native DWSIM reactor**. DWSIM 9.0.5's
kinetic PFR cannot take cheme's rate laws without loss (no `(1 - Q/K)` term, no `(T/T_ref)^n`,
its own property package for partial pressures and enthalpies), so cheme-as-code solves the bed
(the same model as `cheme process propagate`: 1D pseudo-homogeneous plug flow, ideal gas,
effective particle rates, adiabatic, isothermal or cooled, Ergun pressure drop only with the
bed's diameter, viscosity and particle data) at the inlet DWSIM gives, and DWSIM sees a
conversion reactor with the bed's conversions, outlet temperature and pressure drop. An outer
loop repeats the DWSIM solve until the bed at the solved inlet reproduces DWSIM's outlet:
`specs.<id>.coupling` sets `tolerance` (relative, component flows and pressure; default 1e-4),
`temperature_tolerance` (K, 0.05) and `max_iterations` (DWSIM solves, 30). Not converged means
nothing is written. No `mode`, `outlet_temperature` or `pressure_drop` for it (the bed sets
them); one outlet, vapour-phase reactions only; a reaction whose reactants are all formed in the
bed (series reactions) cannot be written as an inlet conversion and is refused. The run record
marks every unit `coupling: native` or `external-unit`, the latter with its `outer_loop`
(iterations, residuals, the bed at the last inlet). `sim-coupling` (info) says how it converged;
`sim-coupling-energy` warns when DWSIM's property package needs a duty beyond 5 % of the reaction
heat to reach an adiabatic bed's outlet temperature (the ideal-gas bed and the package disagree
on the enthalpies, typical at high pressure).

Not simulated (keep these parts assumed and `exclude:` them): CSTRs and other kinetic reactors
than a fixed bed, solids, a liquid side draw from the condenser stage. Evaporators and dryers need `unit:` (their heat duty matters; a component
separator would drop it). A unit simulates one duty: a machine's cooling water or a cooler's pump
power stays a given or assumed value.

## Outputs and provenance

Simulated streams get `temperature`, `pressure`, `flow` and `composition` (energy streams
their `duty`) with `provenance: calculated`, source `cheme process simulate: DWSIM
<version>, <property package>` and a fingerprint of the simulator model. Given and assumed values
stay; `--save` writes the DWSIM flowsheet (`.dwxmz`, a generated file: commit it only if the
project wants it). The specs and their provenance in `simulation.yaml` are the record of what
was assumed.

## Required human judgment

The property package and its validity for the mixture and conditions; every assumed spec
(efficiencies, pressure drops, approach temperatures, reflux ratio, column stages and key
specs); whether a deviation means the assumption or the spec is wrong; whether a converged
recycle or column is physically sensible. A simulated value is no more certain than its specs.

## References

- `process-design`, `thermodynamic-model-selection`, `separation-method-selection`
- `cheme schema process-simulation`; examples `examples/process-simulation` and
  `examples/process-separation` in the cheme-as-code repository
- DWSIM: https://dwsim.org (GPL v3); the Fenske-Underwood-Gilliland shortcut and the rigorous
  column are DWSIM's

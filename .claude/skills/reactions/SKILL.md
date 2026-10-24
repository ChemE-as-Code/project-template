---
name: reactions
description: Write the chemical reactions of a process in process/reactions.yaml (stoichiometry over the components of materials.yaml, conversion of a key component, heat of reaction, equilibrium constant, catalyst, sources), give components ideal-gas formation data, and name the reactions and the reactor model (conversion, equilibrium, gibbs) on the reactor steps of process/flowsheet.yaml, so that cheme validate balances elements and extents, cheme process propagate solves the reactors and cheme process reactors reports duties and the distance from equilibrium. Use when a process has a reactor, or a reaction, conversion or catalyst is added or changed.
---

# Reactions

## Purpose and when to use

Reactions are YAML, like the materials: `process/reactions.yaml` (schema
`cheme/reactions/v1`) holds each reaction's stoichiometry, and every reactor step or block
of `process/flowsheet.yaml` names the reactions that take place in it. With reactions named, a
reactor no longer fails the component balance: `cheme validate` checks that mass and every
element balance and that the reactions' extents explain each component's change, and
`cheme process propagate --fill` computes the outlet of a conversion reactor. Use this
skill whenever a process has a reactor, or a reaction, conversion or catalyst changes.

## Required inputs and missing inputs

- The components with a `formula` each (`materials`, `property-retrieval`): the element balance
  needs them, and `molar_mass` for the extents.
- Per reaction: the stoichiometry, which component the conversion refers to (`key`) and the
  conversion per pass where it is a design input, with its source.
- Optional: the heat of reaction (kJ per kmol of extent as written, negative when exothermic),
  the catalyst, literature references.
- For equilibrium and Gibbs reactors and for duties: each component's `ideal_gas` data
  (`cheme chemistry thermo NAME ...`, extra props: ATcT formation enthalpy, entropy and
  Poling's heat capacity, valid to 1000 K; `--fit 298.15,1500` fits the heat capacity to a
  wider range), or a reaction's own `equilibrium_constant` (`ln K = a + b/T + c ln T + d T`).

If the reaction set itself is open (side reactions, by-products, an equilibrium-limited step),
ask: that is the chemist's decision. Never invent a conversion; when none is known yet, leave
it out and give the reactor's outlet as `assumed`.

Where the route or its conditions still have to be found in the literature, gather the evidence
first (skill `reaction-evidence-search`, `process/reaction-evidence.yaml`). Choose the model
from it with skill `route-selection`: the model basis names the reactions written here, and
each value taken from the evidence cites its observation in `source`.

## Workflow and tools

0. For a known industrial process, start from the reaction library: `cheme chemistry library`
   lists 23 processes (ammonia, methanol, steam reforming, shift, methanation, ethylene oxide,
   sulfuric and nitric acid, Claus, Fischer-Tropsch, hydrotreating, styrene, acrylonitrile,
   propylene oxide, ethane cracking, vinyl chloride, formaldehyde, maleic anhydride, cumene,
   ethylbenzene, MTBE, HCN); `cheme chemistry library methanol` shows one, `--add` adds its
   components (with ideal-gas data and critical constants), reactions and typical catalysts to
   the plant's files without touching what is there (an id the plant has with other data is a
   conflict). The catalyst windows are typical ranges (assumed): confirm them for your licensor
   or vendor, and add the kinetics you have.

1. Write `process/reactions.yaml` (schema: `cheme schema process-reactions`):

   ```yaml
   schema: cheme/reactions/v1
   kind: reactions
   reactions:
     - id: NH3_SYN
       equation: N2 + 3 H2 <=> 2 NH3          # for readers; not parsed
       stoichiometry: {N2: -1, H2: -3, NH3: 2} # reactants negative, products positive
       reversible: true
       conversion: {key: N2, value: 0.25, provenance: assumed, source: typical per pass}
       heat_of_reaction: {value: -91800, provenance: given, source: NIST WebBook}
       catalyst: promoted iron
       sources: ["doi:10.1002/14356007.a02_143.pub3"]
   ```

2. On the reactor step (or a block without steps), name the reactions, and give a conversion
   where this reactor's differs from the reaction's:

   ```yaml
   - {id: R1, block: LOOP, class: ReactingChemicals, reactions: [NH3_SYN],
      conversion: {NH3_SYN: 0.3}, ports: [...]}
   ```

   A block is reactive when anything inside it names reactions; a block with
   `balance_scope: reactive` and no reactions is checked on elements only.
3. Choose the reactor model on the step (`reactor:`):
   - `conversion` (the default when every reaction has a conversion): every reaction converts
     its fraction of the key component's *inlet* flow, all at once (as Aspen's RStoic).
     Series reactions on the converted stream need two reactor steps.
   - `equilibrium`: the named reactions reach equilibrium at the outlet stream's temperature
     (give it, `assumed`) and pressure, ideal gas; `approach: 20` takes K 20 K hotter (less
     conversion of an exothermic reaction; negative for an endothermic one). Mark the
     reactions `reversible: true`.
   - `gibbs` (no `reactions:`): the minimum Gibbs energy over every component of the streams'
     templates, ideal gas; inerts take no part.
   - `pfr`: a fixed catalyst bed, integrated along its catalyst mass from the reactions'
     `rate:` laws (1D plug flow, ideal gas): give `catalyst:` and `bed:` (`catalyst_mass`, or
     `volume`, or `tubes` x `diameter` x `length`; `heat`: `adiabatic`, `isothermal` or
     `cooled` with `coolant_temperature` and `heat_transfer_coefficient` in W/(m2 K); the gas
     `viscosity` in Pa s and a `diameter` for the Ergun pressure drop). Propagate writes the
     outlet's composition, flow, temperature and pressure; the inlet needs its temperature and
     pressure. `examples/process-shift` is an adiabatic high-temperature shift bed.
   Ideal gas is a limit: at high pressure real equilibria lie further to the fewer moles
   (ammonia at 450 C and 300 atm: 31 % ideal, 35.5 % measured). Give the components their
   `critical:` constants (`cheme chemistry thermo NAME` prints them) and the reactor
   `fugacity: peng_robinson` (Peng-Robinson, no interaction parameters: 34.4 % there); `approach`
   remains for the kinetic distance from equilibrium.
4. Give the reactor its catalyst: declare it in `process/catalysts.yaml` (schema
   `cheme schema process-catalysts`: active phase, promoters, support, `physical` (form,
   particle diameter in m, particle and bulk density, bed voidage, surface area), `window`
   (temperature K, pressure kPa(a)), `space_velocity`, `poisons` by component id,
   `deactivation`, `price`, each block with provenance and source) and name it on the step
   (`catalyst: FE_NH3`). Kinetics go on the reaction as `rate:` (power law or LHHW, per kg of
   catalyst or per m3, on partial pressures in bar or concentrations in kmol/m3, `reverse:
   equilibrium` for the (1 - Q/K) factor, `catalyst:` it was measured on, and its validity
   `temperature_range`). For an elementary gas-phase reaction, `cheme chemistry kinetics -r
   CH4=methane -r OH=[OH]` (network; RMG with `--rmg-endpoint`) prints a `rate:` from NIST and
   RMG with its confidence.
5. Where the catalytic mechanism is known, give the reaction a `mechanism:` (mean field): the
   `site_density` (mol/m2), the `adsorbates` (ids ending in `*`, e.g. `CO*`; formula = id
   without the `*`) and the elementary `steps` (stoichiometry over gas components, adsorbates and
   the free site `*`; `forward` and `reverse` rate constants per site in 1/s with pressures in
   bar, or `sticking` for an adsorption), with provenance and source. `cheme chemistry
   microkinetics ID -T 500 -p CO=0.01 -p O2=0.1` solves its steady state: coverages, TOF, the
   degree of rate control of every step (which step to improve), the apparent activation energy
   and the orders; `--cantera` checks it with Cantera, `--export mech.yaml` writes Cantera's
   input. Use it to judge a lumped `rate:` (its orders and Ea should match). A step may give
   `energies:` instead of rate constants (DFT reaction energy and barrier in eV, e.g. from
   Catalysis-Hub or a paper, with the DOI in `source`): transition state theory
   (k_B T/h prefactors); an adsorption keeps its `sticking` and desorbs over its binding energy.
   Treat DFT energies as assumed: errors of 0.1-0.2 eV change rates by orders of magnitude.
6. To get a lumped rate for a fixed bed, fit one: from measurements, write
   `experiments/kinetics/<name>.yaml` (schema `cheme schema kinetic-data`: the reaction, the
   basis, per point temperature, partial pressures in bar and rate, with provenance and source),
   or sample the mechanism (`cheme experiments sample-mechanism ID -T 480 -T 520 -p
   CO=0.008,0.0125 -p O2=0.08,0.125 --sites-per-kg 1e-4 --out experiments/kinetics/x.yaml`; the
   sites per kg from the dispersion), then `cheme experiments fit-rate FILE [--form lhhw
   --adsorb CO --power 2] [--reverse]`: the `rate:` to paste, each parameter with its 95 %
   interval. A fit holds inside its data's range only (`temperature_range` is set to it).
7. Run `cheme process propagate --fill`, then `cheme validate`, then `cheme process reactors`:
   per reactor the extents, the heat of each reaction and ln K at the outlet, `ln(Q/K)`
   (0 at equilibrium, negative short of it), the rate at the outlet where there is a rate law,
   and the duty H(out) - H(in) on the elements basis (give every stream's temperature).

## Validation and failure handling

- `cheme.reaction-element-balance`: the atoms differ; correct the coefficients or a formula.
- `cheme.reaction-element-balance-not-evaluable` (info): a component has no `formula`.
- `cheme.reaction-component`, `cheme.reaction-conversion`: an undeclared component, or a
  conversion key that is no reactant.
- `cheme.process-reaction`: a step names an undeclared reaction, or a conversion of a
  reaction it does not name.
- `cheme.process-balance` on a reactive node: `element X ... in, ... out` means the streams
  lose or create atoms; `... not explained by reactions ...` means the streams balance but the
  declared reactions cannot produce that composition change (a missing side reaction, or a
  wrong assumed composition).
- `cheme.process-propagate-reactor`: the conversions consume more of a component than is
  fed, an equilibrium has no outlet temperature, a component lacks ideal_gas data, or a
  temperature is outside a heat capacity's range.
- `cheme.process-reactor`: a model that cannot be solved as declared (a conversion reactor
  without conversions, an equilibrium without reactions, a Gibbs reactor naming reactions,
  `approach` without equilibrium; an irreversible reaction at equilibrium is a warning).
- `cheme.process-catalyst`: an undeclared catalyst (error); a reactor stream outside the
  catalyst's temperature or pressure window, a poison in a feed, or a rate measured on another
  catalyst (warnings). `cheme.catalyst-*`: the catalysts file itself (densities, voidage,
  poisons that are no component). `cheme.reaction-rate`: a rate naming an undeclared
  component.
- `cheme.reaction-mechanism`: an elementary step that does not balance its atoms or its sites,
  or names an undeclared component.
- `cheme.process-heat-of-reaction` (warning): a given heat of reaction more than 2 % from
  the formation data; check the coefficients and the sources.

## Expected outputs and provenance

The edited YAML, reviewed in the diff. Every conversion and heat of reaction has a provenance
and a source; reactor outlets propagate writes are `calculated` and become stale when the
reaction, the conversion or the feed changes.

## Required human judgments

- The reaction set: main and side reactions, which are neglected, and their key components.
- Conversions and selectivities as design inputs, and the reactor model: a conversion reactor
  says nothing about equilibrium or kinetics, so check with `cheme process reactors` that a
  conversion is reachable (`ln(Q/K)` below 0) at the reactor's temperature and pressure.
- The approach to equilibrium, and whether the ideal-gas limit is good enough.
- The catalyst choice, its window and lifetime, and whether published kinetics apply to it
  (another catalyst, another particle size, transport limits).

## References

- `materials` (components and formulas), `property-retrieval`, `process-design` (the flowsheet),
  `reaction-evidence-search` and `route-selection` (the evidence behind the reactions).
- `cheme schema process-reactions`, `cheme schema process-flowsheet`.

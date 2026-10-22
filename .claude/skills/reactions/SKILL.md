---
name: reactions
description: Write the chemical reactions of a process in process/reactions.yaml (stoichiometry over the components of materials.yaml, conversion of a key component, heat of reaction, catalyst, sources) and name them on the reactor steps of process/flowsheet.yaml, so that cheme validate balances elements and extents and cheme process propagate solves conversion reactors. Use when a process has a reactor, or a reaction, conversion or catalyst is added or changed.
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

If the reaction set itself is open (side reactions, by-products, an equilibrium-limited step),
ask: that is the chemist's decision. Never invent a conversion; when none is known yet, leave
it out and give the reactor's outlet as `assumed`.

## Workflow and tools

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
3. Run `cheme process propagate --fill`. A reactor with one material outlet and a conversion
   for every reaction is solved: every reaction converts its fraction of the key component's
   *inlet* flow, all at once (as Aspen's RStoic by default). Series reactions on the converted
   stream need two reactor steps.
4. Run `cheme validate`, then `cheme process streams`: each reactive balance line shows
   the fitted extents (kmol/h per reaction).

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
  fed; lower them or correct the feed.

## Expected outputs and provenance

The edited YAML, reviewed in the diff. Every conversion and heat of reaction has a provenance
and a source; reactor outlets propagate writes are `calculated` and become stale when the
reaction, the conversion or the feed changes.

## Required human judgments

- The reaction set: main and side reactions, which are neglected, and their key components.
- Conversions and selectivities as design inputs, and the reactor model: a conversion reactor
  says nothing about equilibrium or kinetics, so check that a conversion is reachable at the
  reactor's temperature and pressure.
- The catalyst choice.

## References

- `materials` (components and formulas), `property-retrieval`, `process-design` (the flowsheet).
- `cheme schema process-reactions`, `cheme schema process-flowsheet`.

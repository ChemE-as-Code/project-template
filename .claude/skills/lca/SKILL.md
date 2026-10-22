---
name: lca
description: Life cycle assessment of the plant per functional unit - plantcode assess lca (goal and scope in lca/lca.yaml, the flowsheet's feeds, products and utilities as the operation inventory, take-off and equipment weights as the construction, allocation between co-products, characterisation with the project's factors or a Brightway project with ecoinvent, prospective scenarios such as premise 2050 databases). Use when a carbon footprint, an LCA, environmental impacts, a comparison of design alternatives by impacts, or a prospective (future) assessment is asked for, or when the gate lca fails.
---

# Life cycle assessment

## Purpose and when to use

Give the plant's environmental impacts per functional unit (e.g. kg CO2-eq per t of product),
with every number traceable to a flow of the design and a background with a source. Use it for
a carbon footprint, a screening LCA at concept stage, the comparison of alternatives (each a
branch: same scope, different flowsheet), the contribution analysis that shows which utility or
material dominates, a prospective assessment (electricity mix in 2030 or 2050), and when the
gate `lca` fails.

## Required inputs and missing inputs

- A flowsheet with flows: `plantcode process streams` shows every feed and product with a flow
  (or duty) in the case to assess. Utilities must be **energy terminals**: a feed `POWER` or
  `STEAM`, a product `CW` (heat rejected), with streams `kind: energy` and a `duty` per
  consumer (skill `process-design`). Without them the inventory has no utilities.
- The goal and scope in `lca/lca.yaml` (`plantcode schema lca`): the functional unit (an amount
  of one product), the boundary, the co-products and the allocation, the background of every
  flow. Ask the user for the goal (what decision the LCA serves) and the functional unit when
  they are not given; they decide everything after.
- Backgrounds, with sources. **Never invent a factor** and never present a number from memory
  as data.
  - Without a database licence: `lca/factors.yaml` with published values (a national grid
    factor from the agency that publishes it, an EPD, a peer-reviewed dataset), each with
    status, source, year and licence. The file `plantcode init` writes is an **example**.
  - With a Brightway project (ecoinvent licence, or an open database such as USLCI): `backend:
    brightway`, with `activity:` and `biosphere:` mappings that `plantcode assess lca-search`
    finds. The project is the user's; licensed data never goes into the repository.
- Construction: `estimate/mto.csv` (skill `mto-estimate`) and `dimensions.weight` of the
  equipment, plus `operation: {hours_per_year, life_years}` from the design basis.

## Workflow and tools

1. **Scope.** Write `lca/lca.yaml`: `goal`, `functional_unit`, `boundary`, `case`. Decide the
   co-products. With several products (an ASU: GOX, GAN, LOX, LIN), allocation changes the
   answer more than most factors. ISO 14044 prefers avoiding allocation (system expansion:
   `method: substitution`, with the avoided production of each co-product); otherwise a
   physical basis (`exergy` when the products differ in pressure, temperature or purity; run
   `plantcode process props --fill` first, or give `properties.<product>.exergy`), then
   economic. Read the allocation comparison in `lca/results.md`: every method whose data is
   complete is computed. Choose one, record why in `allocation.source`, and report the
   spread.
2. **Map every flow.** Run `plantcode assess lca` and read `plantcode.lca-unmapped`. For each
   feed, product and emitted component give one entry:
   - `free: true` for resources without burden (ambient air) and for atmospheric components
     returned unchanged (N2, O2, Ar, water, CO2 taken in with the air);
   - `emission: true` for a product released to the environment, then each component free or
     `elementary: {compartment: air, origin: fossil}` (characterised by its CAS number);
   - a `factor:` or an `activity:` otherwise, or a `dataset:` (another process's unallocated LCI
     dataset, `kind: lci_dataset`) when the input comes from a process modelled in the
     repository or a process library.
   Choose market activities ("market for electricity, medium voltage") for the right location.
   For energy, mind the unit: the inventory counts kWh, a heat activity is per MJ, and the
   conversion is automatic when the kinds match.
3. **Construction.** Add `operation` and `construction:` entries: steel, pipe, concrete and
   insulation from the take-off, and equipment by material from the equipment list. Read
   `lca-construction-unmapped` and `lca-equipment-weight`. Missing weights count as zero: give
   them as ASSUMED estimates with a source, or state the gap.
4. **Run and read.** `plantcode assess lca` (Markdown): totals per scenario, the split by phase,
   the contributions. Check the dominant contributor against an independent estimate. Example:
   specific power x grid factor for an air separation, or the fuel's carbon for a reformer.
   Investigate any result that disagrees by more than the data's accuracy.
5. **Uncertainty.** Give `uncertainty:` with defaults by provenance (assumed values wider than
   given ones), and the known uncertainties of the main flows and factors (gsd2, or pedigree
   scores for factors from a database). Report the 95 % interval with the result, and read the
   sensitivity: refine first what ranks highest.
6. **Scenarios.** Add `scenarios:` for the questions the goal asks, e.g. a grid factor for 2030
   or 2050 (`factors:`), or premise databases (`databases:`, see docs/framework/lca.md). Name
   the model, pathway and year in `source`.
7. **Write and gate.** `plantcode assess lca --write`; commit `lca/*` with the inputs. The gate
   `kind: lca` requires `mapped`, then `lca-current`, and `factors` once the project's own
   replace the example.

## Validation and failure handling

- `plantcode validate` checks `lca/lca.yaml` and `lca/factors.yaml` against their schemas.
- Errors stop the LCA. Examples:
  - `lca-functional-unit`, `lca-coproduct`, `lca-allocation` (scope);
  - `lca-flow-invalid`, `lca-flow-mixed`, `lca-emission-components` (the flowsheet);
  - `lca-factor-unknown`, `lca-factor-unit`, `lca-activity-missing`, `lca-activity-unit`,
    `lca-brightway` (backgrounds).
  Fix the input they name; never work around them by mapping a flow `free`.
- `plantcode assess lca --check` exits 1 when the written files are stale: rewrite them, never
  edit them.
- A flow you cannot map yet stays unmapped (it carries no impact), and the results say so.
  Record the gap and its likely size in `DESIGN.md`.

## Outputs and provenance

- `lca/inventory.csv`: each flow with its phase, the plant's amount, the quantity per
  functional unit, its provenance (given, calculated, assumed: the weakest value used) and its
  mapping.
- `lca/results.csv` and `lca/results.md`: impacts per scenario, category, phase and
  contributor, with the factor's status, source and year, or the Brightway activity.
- `lca/dataset.yaml`: the plant's unallocated LCI dataset (foreground scope, per hour): what
  a process library keeps and other studies reference.
- `--json` adds the findings and the source revision (git commit, fingerprints of the files
  read).

## Required human judgment

The goal, the functional unit, the system boundary, the allocation method, the choice of
background datasets and scenarios, and the interpretation are the LCA practitioner's decisions
(ISO 14044). An agent may assemble the inventory, propose mappings with their sources, run the
calculation, check plausibility and draft the interpretation. It does not choose the
allocation or the datasets without saying so, never presents example factors as results, and
never signs off. A comparative assertion disclosed to the public needs a critical review (ISO
14044 section 6); say so when the goal implies one.

## References

- `plantcode schema lca`, `plantcode schema lca_factors`, `lca/README.md`,
  `docs/framework/lca.md` (calculation, Brightway, ecoinvent and premise set-up)
- ISO 14040:2006 and ISO 14044:2006 (principles, requirements, allocation hierarchy)
- skills `process-design` (energy streams, cases), `mto-estimate` (take-off), `review`

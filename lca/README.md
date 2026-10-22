# LCA

The life cycle assessment of the plant's operation (ISO 14040/44), per functional unit.

| file | what | written by |
|---|---|---|
| `lca.yaml` | goal and scope: the functional unit (a product of `process/flowsheet.yaml`), the case, the allocation between co-products, the impact categories, and the background of every feed, product and emitted component (`cheme schema lca`) | people; `cheme init` writes none, since the flowsheet has no products yet |
| `factors.yaml` | the project's characterised factors per kg, t, kWh, MJ, ... of a flow, one value per impact category, each with status, source, year and licence (`cheme schema lca_factors`) | people; the file `cheme init` writes is an **example** (`example: true`) and characterises nothing |
| `inventory.csv` | the foreground inventory per functional unit: feeds and products of the flowsheet (kg, energy in kWh), emissions by component, co-products | `cheme assess lca --write` |
| `results.csv`, `results.md` | impacts per category, with the contribution of each flow | `cheme assess lca --write` |
| `dataset.yaml` | the plant's unallocated LCI dataset (`kind: lci_dataset`, per hour of operation): what other studies and a process library reference (`dataset: {path, product, allocation}`) | `cheme assess lca --write` |
| `datasets/*.yaml` | LCI datasets of other processes, written by hand or copied from a library, used as backgrounds | people |

A minimal `lca.yaml`:

```yaml
schema: cheme/lca/v1
kind: lca
goal: Carbon footprint of the product for the concept comparison (DESIGN.md D-0xx)
functional_unit: {product: PROD, amount: 1, unit: t}
allocation: {method: exergy, coproducts: [PROD, BYPROD], source: "DESIGN.md D-0xx"}  # mass, economic, energy, exergy, shares, substitution, none
flows:
  - {key: feed:AIR, free: true, source: "ambient air, a resource without burden"}
  - {key: feed:POWER, factor: electricity, source: "grid supply at the site"}
  - {key: product:VENT, emission: true, source: "released to the atmosphere"}
  - {key: emission:CO2, factor: co2-fossil, source: "fossil CO2 from the reformer"}
```

The construction phase counts the take-off (`estimate/mto.csv`) and the equipment weights,
spread over the plant's life:

```yaml
operation: {hours_per_year: 8000, life_years: 20, source: "design basis"}
construction:
  - {id: steel, takeoff: mto, category: steel, factor: steel-sections, source: "rack and structure steel"}
  - {id: vessels, takeoff: equipment, equipment: {material: "*SS*"}, factor: stainless-steel, source: "empty weights"}
```

Supplies the flowsheet and the take-off do not hold: `consumables` (e.g. an adsorbent
replaced every 5 years), `maintenance` (a yearly fraction of the construction), `transport`
(t*km of matched exchanges) and `end_of_life` (treatment or recycling of the construction
mass).

With a Brightway project (ecoinvent or an open database, `pip install 'cheme-as-code[lca]'`), set
`backend: brightway` and give the flows an `activity:` (`cheme assess lca-search` finds
them). `scenarios:` compare the baseline with other factors or databases (e.g. premise 2050).
See the skill `lca`.

Never edit `inventory.csv` or `results.*` by hand: change the flowsheet, `lca.yaml` or
`factors.yaml`, then rewrite them. Energy streams (`kind: energy`, with `duty`) from an energy
feed (electric power, steam) or to an energy product (cooling water) are counted in kWh. The
gate `kind: lca` (`checks/checks.yaml`, require `mapped`, `factors`, `lca-current`) checks the
mapping is complete, the factors are the project's, and the outputs are current.

Licensed databases (ecoinvent and others) must not be copied into this repository. Record the
dataset a factor comes from in its `source` and its terms in `licence`.

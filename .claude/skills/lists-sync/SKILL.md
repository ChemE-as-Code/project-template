---
name: lists-sync
description: Keep the equipment, line and instrument lists (YAML) consistent with the P&ID sheets - add missing items, flag removed ones, align types and sheets. Use after any P&ID change that adds, removes or retags an item, or when plantcode validate reports equipment-not-listed / equipment-not-on-pid.
---

# Keeping the lists in step with the P&IDs

The P&IDs and the lists describe the same plant. `plantcode validate` compares equipment tags
and classes; lines and instruments are kept by this procedure.

1. Run `plantcode validate` and collect the `plantcode.equipment-*` warnings.
2. For each tag on a P&ID but not in `equipment/equipment-list.yaml`: add an entry with `tag`,
   `type` (the DEXPI class, e.g. `CentrifugalPump`), `sheet`, and what you can read from the
   P&ID (design pressure and temperature, material). Leave unknowns out rather than guessing.
3. For each tag in the list but on no P&ID: ask whether it was removed on purpose. If yes, delete
   the entry; if no, it still needs drawing.
4. Lines: every `PipingNetworkSystem` `LineNumber` should have an entry in
   `piping/line-list.yaml` (`number`, `from`, `to`, `nominal_diameter`, `fluid`).
   `grep -n 'LineNumber' -A1 pid/*.xml` lists them.
5. Instruments: every `ProcessInstrumentationFunction` should have an entry in
   `control/instrument-list.yaml` with its printed tag (category + letters + number).
6. `plantcode validate` again; the list schemas are printed by `plantcode schema <kind>`.

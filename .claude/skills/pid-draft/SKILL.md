---
name: pid-draft
description: Draft a new P&ID sheet from the design basis and equipment list - start from the empty sheet, add equipment, lines, valves and instruments, either by writing the XML directly (small sheets) or with a throwaway pyDEXPI generator script in scripts/ (large sheets). Use when a sheet does not exist yet or a section is redrawn from scratch.
---

# Drafting a new P&ID sheet

Inputs: `process/design-basis.yaml`, `equipment/equipment-list.yaml`, `piping/line-list.yaml`,
`control/instrument-list.yaml`, `DESIGN.md` for decisions already made.

1. **Plan on paper first.** Write the sheet's content as a short list in `DESIGN.md`: equipment
   (tag, class), lines (number, from, to, DN), valves per line, instruments. Decide a rough
   layout: process flow left to right, equipment in a row at mid height, the equipment-data
   band at the bottom.
2. **Start the file** from `.claude/skills/dexpi-xml/snippets/empty-sheet.xml` as
   `pid/<sheet>.xml`; set the title-block data under `MetaData` (drawing number, title).
   Add the sheet to `plant.yaml` under `sheets`.
3. **Fill it**, one of two ways:
   - **Small sheet (up to ~10 items)**: write the XML directly, one item at a time, following
     the `dexpi-xml` skill. Run `plantcode fmt` and `plantcode validate` after each item.
   - **Large sheet**: write `scripts/draft_<sheet>.py`, which builds the model with the pyDEXPI
     classes and writes the XML:
     ```python
     from pydexpi.loaders.public import load_xml_file
     from pydexpi.loaders.xml_serializer import DexpiXmlSerializer
     from pydexpi.pydexpi_classes import pydexpi_classes as dx

     model = load_xml_file("pid/P-02.xml")          # the empty sheet
     plant = model.conceptualModel
     pump = dx.CentrifugalPump(id="Pump_P201", tagName="P201", ...)
     plant.taggedPlantItems.append(pump)
     # ... nozzles, piping systems, segments, pipes, and their RepresentationGroups
     DexpiXmlSerializer().save(model, "pid", "P-02")
     ```
     Use the pyDEXPI model classes and serializers only. Keep the script in `scripts/` for
     provenance, and run `plantcode fmt` on its output. After that the XML is the source of
     truth; later changes are edits to the XML (`pid-edit`), not to the script.
4. **Validate, render, review** as in `pid-edit`, and check every listed item is on the sheet
   (`plantcode validate` warns `equipment-not-on-pid`).
5. **Commit** the sheet, the plant.yaml entry and the script together.

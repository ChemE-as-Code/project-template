---
name: dexpi-xml
description: How to read and write DEXPI 2.0.1 XML P&IDs by hand in this repository - document structure, ids and references, equipment with nozzles, piping systems/segments/pipes, valves, instruments, and how each is drawn (RepresentationGroup, ShapeUsage, node positions, ConnectorLine). Use before editing any pid/*.xml file.
---

# Writing DEXPI 2.0.1 XML

P&IDs in `pid/` are DEXPI 2.0.1 XML, edited directly as text. A sheet holds two parallel
trees: **what the plant is** (the conceptual model) and **how it is drawn** (the diagram). Every
plant item is in the first, and a `RepresentationGroup` in the second draws it.

```
Model
  Import Core / Plant (models/2.0.1)
  Object Core/EngineeringModel
    Components ConceptualModel -> Object Plant/PlantModel
        Components TaggedPlantItems           equipment (Tank, CentrifugalPump, ...)
        Components PipingNetworkSystems       lines -> segments -> items + pipes
        Components ProcessInstrumentationFunctions   instruments
        Components ActuatingSystems / InstrumentationLoopFunctions
        Components MetaData                   title block data
    Components Diagram -> Object Core/Diagram.Diagram
        Components Groups                     Border, title block Label, one RepresentationGroup per drawn object
    Components ShapeCatalogues                the symbols (Shape objects) ShapeUsages point at
```

## The five element kinds

| XML | meaning |
|---|---|
| `<Object id="X" type="Plant/Piping.GlobeValve">` | an object of a DEXPI class |
| `<Components property="Nodes">` children | objects this one **owns** (composition) |
| `<References objects="#A #B" property="SourceItem"/>` | objects it **points at**, by `#id`, space-separated |
| `<Data property="TagName"><String>T4750</String></Data>` | a value: `String`, `Double`, `Integer`, `Boolean`, `DateTime` |
| `<DataReference data="Core/Diagram.DashStyle.Solid"/>` inside `Data` | an enumeration value |
| `<AggregatedDataValue type="Core/Diagram.Point">` inside `Data` | a structured value (Point, Color, Stroke, PhysicalQuantity) with its own `Data` children |

A physical quantity:

```xml
<Data property="NominalCapacityVolume">
  <AggregatedDataValue type="Core/PhysicalQuantities.PhysicalQuantity">
    <Data property="Unit"><DataReference data="Core/PhysicalQuantities.VolumeUnit.MetreCubed"/></Data>
    <Data property="Value"><Double>26.0</Double></Data>
  </AggregatedDataValue>
</Data>
```

Pressures are gauge or absolute: `PressureGaugeUnit.Bar` (barg) vs `PressureAbsoluteUnit.Bar`.

## Rules that keep a sheet valid

1. **Every `<Object>` has an id, unique in the document**, matching `[A-Za-z_][A-Za-z0-9_]*`
   (no hyphens, no dots). Use readable ids: `Pump_P101B`, `Pipe_P101B_suction`.
   `plantcode fmt` names any object you leave without one.
2. **References point at ids that exist.** Renaming an id means renaming every `#id` that names
   it (`grep -n '#OldId' pid/*.xml`).
3. **A pipe connects two nodes.** Each `Plant/Piping.Pipe` has `SourceItem` + `SourceNode` and
   `TargetItem` + `TargetNode`. Nodes belong to items: a nozzle owns one `PipingNode`, an inline
   valve owns two. Convention: **node 1 (first) is the inlet, node 2 (second) the outlet**; flow
   runs source -> target.
4. **A segment is one run in flow order.** `PipingNetworkSegment` owns `Items` (valves, reducers,
   ...) and `Connections` (pipes), and references its own ends: `SourceItem`/`SourceNode` = where
   the first pipe starts, `TargetItem`/`TargetNode` = where the last one ends (usually nozzles).
   One node is connected by at most one pipe.
5. **Tags** are `Data property="TagName"` on equipment (with `TagNamePrefix`,
   `TagNameSequenceNumber`), `PipingComponentNumber` on valves, `LineNumber` on a piping system.
   Keep the equipment list (`equipment/equipment-list.yaml`) in step.
6. **Every plant item is drawn**: a `RepresentationGroup` under `Diagram/Groups` (or nested in
   its owner's group) with `<References objects="#TheItem" property="Represents"/>`.
7. Coordinates are millimetres on the sheet, **Y down** (like SVG). The reference sheet is
   420 x 297 (A3; `MinX`/`MaxX`/`MinY`/`MaxY` on the Diagram). Keep new symbols inside the border and off existing ones.

## Drawing an object

A drawing is a `Core/Diagram.RepresentationGroup` that `Represents` the object and holds:

- a `Core/Diagram.Static` group with one `Core/Diagram.ShapeUsage`: `Shape` reference to a symbol
  in the catalogue (see `shapes.md`), `Position`, `Rotation` (degrees), `ScaleX`/`ScaleY`
  (valves use 0.4), `IsMirrored`;
- for each piping node of the object, a nested `RepresentationGroup` representing the node with
  one `Plant/Diagram.PipingNodePosition` (`Node` reference + `Position`): where pipes attach;
- optionally a label group (`Plant/Diagram.*Label`) with `Core/Diagram.Text` elements whose
  `Template` fragments show attributes (`AttributeRepresentation` -> `Object` + `AttributeName`).

A pipe is drawn by a group representing the `Pipe` with a `Static` holding one
`Core/Diagram.ConnectorLine`: `Source` and `Target` reference the two **PipingNodePosition**
objects (not the nodes), `InnerPoints` lists the bends, `Stroke` the line style.

When you insert a valve into an existing pipe:

1. conceptual: add the valve with two `PipingNode`s to the segment's `Items`; split the pipe into
   two (old pipe now ends at valve node 1, a new pipe starts at valve node 2);
2. drawing: add the valve's group (ShapeUsage on the line, two PipingNodePositions 2 mm either
   side along the line), shorten the old ConnectorLine to end at the first node position, add a
   ConnectorLine for the new pipe from the second node position to the old end.

## Snippets (copy, then rename every id)

| file | what |
|---|---|
| `snippets/empty-sheet.xml` | a new sheet: metadata, border, title block, full shape catalogue |
| `snippets/equipment-tank.xml`, `...-drawing.xml` | Tank with chambers and nozzles, and its drawing |
| `snippets/equipment-pump.xml`, `...-drawing.xml` | centrifugal pump with two nozzles, and its drawing |
| `snippets/piping-system-with-valve.xml`, `...-drawing.xml` | a line: nozzle -> globe valve -> nozzle, and its drawing |
| `snippets/instrument-function.xml`, `...-drawing.xml` | a field instrument, and its bubble |
| `shapes.md` | every shape in the catalogue and which object uses it |

The snippets are long because drawings are verbose. Read them with `grep -n` and line ranges
rather than in full.

## Loop

After every change: `plantcode fmt pid/<sheet>.xml && plantcode validate && plantcode render`.
Fix `error`s before anything else; each finding gives `file:line`, the rule and a hint.

| finding | usual cause | fix |
|---|---|---|
| `xml.syntax` | unclosed tag, stray `&` | fix the XML at that line |
| `dexpi.load` "Reference 'X' ... not found" | `#X` names no object | fix the id or add the object |
| `dexpi.load` unknown class / property | misspelt `type` or `property` | copy the spelling from a snippet |
| `plantcode.duplicate-id` | copied snippet without renaming | rename, and its references |
| `plantcode.not-drawn` | conceptual object without a drawing | add a RepresentationGroup |
| `pydexpi_segment_connectivity` | segment ends disagree with its pipes | point SourceItem/TargetItem at the first/last pipe's ends |
| `pydexpi_piping_node_single_connection` | two pipes on one node | give the item another node, or reroute |

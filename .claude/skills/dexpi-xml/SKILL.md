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
   the first pipe starts, `TargetItem`/`TargetNode` = where the last one ends (see Conventions).
   One node is connected by at most one pipe.
5. **Tags** are `Data property="TagName"` on equipment (with `TagNamePrefix`,
   `TagNameSequenceNumber`), `PipingComponentNumber` on valves, `LineNumber` on a piping system,
   category + functions + number on an instrument function (printed `PI-4712.01`). Keep the
   equipment, line and instrument lists in step (`lists-sync`).
6. **Every plant item is drawn**: a `RepresentationGroup` under `Diagram/Groups` (or nested in
   its owner's group) with `<References objects="#TheItem" property="Represents"/>`.
7. Coordinates are millimetres on the sheet, **Y down** (like SVG). The reference sheet is
   420 x 297 (A3; `MinX`/`MaxX`/`MinY`/`MaxY` on the Diagram). Keep new symbols inside the
   border and off existing ones.

## Conventions

`plantcode validate` checks the ones that name a rule id. The snippets are unmodified excerpts
of the reference P&ID and follow them.

- **Coordinates**: millimetres, origin top left, **Y grows downwards**. `--crop X0,Y0,X1,Y1` of
  `plantcode render` takes the same numbers. Rotation is in degrees.
- **Segment ends** (`plantcode.segment-ends`): `SourceItem`/`SourceNode` of a segment are the
  first pipe's `SourceItem`/`SourceNode`; `TargetItem`/`TargetNode` are the last pipe's
  `TargetItem`/`TargetNode`. Prefer the node the last pipe ends at. Validate also accepts the
  reference convention: when the segment **owns** its end item (it is in the segment's `Items`,
  e.g. the tee or check valve it ends at), the node may be that item's other node -- the one the
  next segment starts at, so consecutive segments share a node (`snippets/tee-branch.xml`: S3
  ends at `PipeTee1`/`PipingNode30`, where S4 starts). Do not "fix" existing segments that use
  it. For a segment that ends in a `DirectPipingConnection` (valve bolted to a blind flange),
  that connection is the last one.
- **Tees and property breaks only at segment ends.** A `PipeTee` (or `PropertyBreak`) is where
  segments meet. It is an `Item` of exactly one segment: the one that **ends** at it. The
  segments leaving it start at its other nodes and do not own it. A tee owns three
  `PipingNode`s: run in, run out, branch.
- **Drawing a tee**: one `ShapeUsage` of `Shape12` at the tee point, `ScaleX`/`ScaleY` 0.0001
  (a dot), and the three `PipingNodePosition`s 0.001 mm from it towards their pipes: run in at
  (x - 0.001, y), run out at (x + 0.001, y), branch at (x, y - 0.001) for a branch going up
  (`snippets/tee-branch-drawing.xml`). The lines then appear to meet in one point.
- **Instrument bubble** (`Shape16` field, `Shape17` central/DCS with a middle bar; scale 1):
  `Position` is the centre (x, y). The bubble is a stadium **13.5 mm wide, 7.5 mm high**: straight
  top and bottom from x - 3 to x + 3, ends of radius 3.75 -- keep 6.75 mm clear left and right.
  Its group has four `InstrumentationNodePosition`s where lines attach: top (x, y - 3.75),
  bottom (x, y + 3.75), left (x - 6.75, y), right (x + 6.75, y). In the reference measuring lines
  end at the bottom one; signal lines leave from the top one (going up) or a side one (going
  sideways). The label (`ProcessInstrumentationFunctionLabel`)
  has two `Text`s, size 3, `CenterCenter`: the letters (category + functions, e.g. `PI`) at
  (x, y - 2.25), the number at (x, y + 1.2). The measuring line is a `ConnectorLine` (green,
  0.2 mm) from an `InstrumentationNodePosition` on the sensing location to the bubble.
- **Nominal diameter is stated three times** on every `PipingNode`, segment and system:
  `NominalDiameterRepresentation` (`DN 50`), `NominalDiameterNumericalValueRepresentation`
  (`50`) and `NominalDiameterTypeRepresentation` (`DN`). Change all three together. Two nodes
  joined by a pipe have the same DN unless one item is a reducer (`plantcode.dn-mismatch`).
- **Labels cache their text.** A `Core/Diagram.Text` holds a `TextTemplate` (fragments that name
  attributes) and the rendered `Text` string. When you change an attribute a label shows, update
  the cached `Text` too (`plantcode.label-stale` finds the ones you missed).

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

## Recipes

Start every recipe with `plantcode show <id or tag>`: it prints the `file:start-end` of the
object's `<Object>` block and of its drawing(s), its owner chain, and the ids it references and
that reference it, each with a line number. Read those line ranges instead of the whole file.
After every recipe: fmt, validate, `plantcode render --png --crop ...` around the change, and
`plantcode diff HEAD --list` to check the change list is what you meant (see Loop).

### Insert a valve into an existing pipe

1. conceptual: add the valve with two `PipingNode`s to the segment's `Items`; split the pipe into
   two (old pipe now ends at valve node 1, a new pipe starts at valve node 2). If the old pipe was
   the segment's last one, the segment's `TargetItem`/`TargetNode` stay as they were (the new
   pipe ends there).
2. drawing: add the valve's group (ShapeUsage on the line, two PipingNodePositions 2 mm either
   side along the line), shorten the old ConnectorLine to end at the first node position, add a
   ConnectorLine for the new pipe from the second node position to the old end.

### Insert a tee and a branch (split a segment)

Copy from `snippets/tee-branch.xml` and `tee-branch-drawing.xml`.

1. `plantcode show <segment id>` and pick the pipe to split and the point on its line.
2. Split the segment at that pipe into **A** (from the old source to the tee) and **B** (from the
   tee to the old target):
   - A keeps the pipes before the split; the split pipe now ends at tee node 1 (run in). The tee
     with its three `PipingNode`s goes into **A's `Items`** (A ends at it). A's
     `TargetItem`/`TargetNode` = the tee / node 1.
   - B is a new `PipingNetworkSegment` in the same system, holding a new pipe from tee node 2
     (run out) to where the split pipe used to end, the remaining pipes and the items after the
     split (move their `<Object>` blocks from A to B). B's `SourceItem`/`SourceNode` = the tee /
     node 2; B's `TargetItem`/`TargetNode` = A's old target.
   - Give B the segment data A has (fluid, piping class, the three DN properties).
3. The branch: a new segment (in this system or in the branch's own `PipingNetworkSystem`, with
   its own `LineNumber`) whose first pipe starts at tee node 3.
4. Drawing: the tee group (Shape12, three node positions, see Conventions) goes in A's drawing
   group; a group for B with its pipes' ConnectorLines (move the moved items' groups into it);
   a group for the branch segment. The split ConnectorLine ends at the run-in node position; the
   new one starts at the run-out position.
5. `plantcode validate`: `plantcode.segment-ends` catches ends that disagree with their pipes.

### Remove an item cleanly

1. `plantcode show <id or tag>`: note its block, its drawing(s) and **every incoming reference**.
2. Conceptual: delete the item's `<Object>` block (with its nodes). Reconnect or delete the pipes
   that referenced its nodes: for an inline valve, delete one of its two pipes and point the
   other at the far node (`SourceItem`/`SourceNode` or `TargetItem`/`TargetNode`); for
   equipment, delete the pipes and segments (or whole systems) that only served it. Fix the ends
   of every segment whose first or last pipe changed.
3. Drawing: delete the item's `RepresentationGroup` (symbol, node positions, labels -- valve,
   nozzle and equipment labels, an `EquipmentBarLabel` data table on the equipment band), the
   groups of every deleted pipe and segment, and stretch the remaining `ConnectorLine` to the
   node position of the new end (`Source`/`Target`, and `InnerPoints` if it bent).
4. Instrumentation: `ActuatingSystem`s whose `OperatedValveReference` names the valve (with
   their actuator drawing and `ActuatingSystemNumberLabel`/`FailActionLabel`),
   `ProcessSignalGeneratingFunction`s whose `SensingLocation` is the item, and the measuring and
   signal lines drawn to it. Delete them or re-point them, as the change asks.
5. `grep -n '#<id>\b' pid/<sheet>.xml` for each deleted id (item, nodes, pipes, node positions)
   must print nothing. Then remove the tag from the lists (`lists-sync`).

### Add an instrument with an instrument nozzle

The measured point is an item on the line (here a blind flange on a short branch with an
isolation valve) that carries both a `PipingNodePosition` (the pipe attaches) and an
`InstrumentationNodePosition` (the measuring line attaches). Copy from
`snippets/instrument-nozzle.xml`, `instrument-nozzle-drawing.xml`, `instrument-function.xml`
and `instrument-function-drawing.xml`.

1. Conceptual: a tee on the line (recipe above), a branch segment: pipe from tee node 3 to the
   isolation valve, `DirectPipingConnection` from the valve's node 2 to the `BlindFlange`'s one
   node; segment ends = tee node 3 and the blind flange node.
2. A `ProcessInstrumentationFunction` under `ProcessInstrumentationFunctions` with
   `ProcessInstrumentationFunctionCategory` (`P`), `ProcessInstrumentationFunctions` (`I`),
   `ProcessInstrumentationFunctionNumber`; inside it a `ProcessSignalGeneratingFunction` whose
   `SensingLocation` references the blind flange, and a `MeasuringLineFunction` from the signal
   generating function to the instrument function.
3. Drawing: the blind flange group holds its symbol (`Shape13`, scale 0.4, rotated to face the
   valve), a `PipingNodePosition` for its node, and an `InstrumentationNodePosition` in its
   `NodePositions` 0.8 mm off the node position, at the flange face. The bubble group (see
   Conventions) holds the label texts, the four bubble `InstrumentationNodePosition`s and the
   `MeasuringLineFunction` group with a `ConnectorLine` from the flange's
   `InstrumentationNodePosition` to the bubble's lower one.
4. Add the printed tag (e.g. `PI-4712.01`) to `control/instrument-list.yaml` (`lists-sync`).

### Change a line's DN everywhere

1. `plantcode show <LineNumber>` lists the system, its segments, items, pipes and nodes with
   their DN and line numbers.
2. Decide which segments change size, by `SegmentNumber`. A line often has branches of its own,
   smaller DN (line 47126 is DN 50 with DN 25 segments S2 drain, S5 and S9 instrument
   connections). **Leave those branch segments, their nodes and the tee's branch node as they
   are** unless the branch is resized too; a tee whose branch keeps its size is then a reducing
   tee (run nodes new DN, branch node old DN), as in the reference.
3. Set the three DN properties (Conventions) on the `PipingNetworkSystem` (if its main run
   changes), on each segment that changes, and on each `PipingNode` those segments' pipes connect
   -- the run in/run out nodes of the tees and the nodes of the nozzles at the run's ends. If the
   equipment nozzle keeps its size, put a reducer in the segment instead of changing that node.
   `plantcode.dn-mismatch` does not notice a branch resized together with the run: check the
   `plantcode show <LineNumber>` output again afterwards, segment by segment.
4. Update the cached `Text` of every label that shows the DN: the system label
   (`PipingNetworkSystemLabel`, e.g. `MNc 47126 75HB13 50`), segment labels, nozzle labels.
   `grep -n '<String>[^<]*\b50\b' pid/<sheet>.xml` inside the line's drawing groups finds them.
5. Change `nominal_diameter` in `piping/line-list.yaml`, and list every segment whose DN differs
   from it under `segments` (all of them, so `plantcode.line-dn-mismatch` compares each).
6. `plantcode validate` must not report `plantcode.dn-mismatch`, `plantcode.label-stale` or
   `plantcode.line-dn-mismatch`.

## Snippets (copy, then rename every id)

| file | what |
|---|---|
| `snippets/empty-sheet.xml` | a new sheet: metadata, border, title block, full shape catalogue |
| `snippets/equipment-tank.xml`, `...-drawing.xml` | Tank with chambers and nozzles, and its drawing |
| `snippets/equipment-pump.xml`, `...-drawing.xml` | centrifugal pump with two nozzles, and its drawing |
| `snippets/piping-system-with-valve.xml`, `...-drawing.xml` | a line: nozzle -> globe valve -> nozzle, and its drawing |
| `snippets/tee-branch.xml`, `...-drawing.xml` | a run segment ending at a tee, the run continuing, and a branch segment |
| `snippets/instrument-function.xml`, `...-drawing.xml` | a field instrument, and its bubble and measuring line |
| `snippets/instrument-nozzle.xml`, `...-drawing.xml` | the measured point: branch, isolation valve, blind flange with the `InstrumentationNodePosition` the measuring line starts at |
| `shapes.md` | every shape in the catalogue and which object uses it |

The snippets are long because drawings are verbose. Read them with `grep -n` and line ranges
rather than in full. Snippets with several fragments say above each where it goes.

## Loop

After every change:

```sh
plantcode fmt pid/<sheet>.xml && plantcode validate
plantcode render pid/<sheet>.xml --png --crop 240,120,320,180   # build/<sheet>.png of the changed region
plantcode diff HEAD --list                                       # one line per change
```

Fix `error`s before anything else; each finding gives `file:line`, the rule and a hint. Look at
the PNG (crop with a margin of 20 mm around what you changed; `--scale` sets the pixels per mm).
`plantcode diff --list` prints a `<sheet>:` line, then per change `<kind> <class> <id> [tag] (<domain>) <attrs> -> drawn as <owner>`:
every line should be a change you meant, and every changed plant item should be drawn.
`plantcode diff --json` gives the same as data.

| finding | usual cause | fix |
|---|---|---|
| `xml.syntax` | unclosed tag, stray `&` | fix the XML at that line |
| `dexpi.load` "Reference 'X' ... not found" | `#X` names no object | fix the id or add the object |
| `dexpi.load` unknown class / property | misspelt `type` or `property` | copy the spelling from a snippet |
| `plantcode.duplicate-id` | copied snippet without renaming | rename, and its references |
| `plantcode.not-drawn` | conceptual object without a drawing | add a RepresentationGroup |
| `plantcode.segment-ends` | segment Source/Target Item/Node is not where its pipes start/end (pyDEXPI's `pydexpi_segment_connectivity`, `..._reference_nodes_and_items_match`) | point the segment's SourceItem/SourceNode at where its pipes start, TargetItem/TargetNode at where they end (or, for an end item the segment owns, its other node; see Conventions) |
| `plantcode.dn-mismatch` | a pipe joins nodes of different DN (also at a reducer port) | give both nodes the DN of the pipe; a reducer's own two nodes carry the two DNs |
| `plantcode.label-stale` | label text no longer what its template shows, or AttributeName misspelt | update the cached `Text`, or fix the AttributeName |
| `pydexpi_piping_node_single_connection` | two pipes on one node | give the item another node, or reroute |
| `plantcode.dn-mismatch` | DN changed on one side of a pipe only | change all three DN properties on both nodes, or add a reducer |
| `plantcode.label-stale` | data changed, cached label `Text` not | set the `Text` to what the template renders |
| `plantcode.line-*`, `plantcode.instrument-*` | sheet and list disagree | `lists-sync` skill |

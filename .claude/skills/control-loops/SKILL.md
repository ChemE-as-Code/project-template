---
name: control-loops
description: Define the regulatory control loops in control/loops.yaml - measurement, controller, final elements with fail positions, action, setpoints with units and provenance, modes, cascade/ratio/feedforward/override links, alarms with operator responses - and check them with cheme validate and cheme control report. Use when a P&ID or flowsheet gains control loops, before a HAZOP, or when cheme.control-* findings name control/loops.yaml.
---

# Control loops

## Purpose and when to use

Keep one register of the plant's regulatory loops: what each controls, with which measurement
and which final element, to which target, in which modes, and how it is linked to other loops.
It is the source the narrative, the control philosophy, the sequences and later tuning or DCS
work refer to. Use it while the P&IDs get their instrumentation, and before a HAZOP. A loop is
regulatory control, never a safeguard: trips belong in `control/cause-effect.yaml` (`cheme cause-effect`).

## Required inputs and missing inputs

- The instrument list and the P&IDs: the measurement and the valve or drive tags.
- `process/flowsheet.yaml`: the operating cases and the stream values a setpoint comes from.
- The equipment datasheets for limits; the vendor for a package's own controls.
- What only people decide: the control strategy (which variable is paired with which valve),
  the fail positions (a safety question for each valve), the modes. A fail position or action
  nobody gave is left out, never guessed: it stays a `cheme.control-incomplete` finding.

## Workflow and tools

1. **One entry per loop** in `control/loops.yaml` (`cheme schema loops`): `id` (the loop tag),
   `title`, `purpose`, `section`, `cases`, `variable`, `measurement: [{tag}]`, `controller`.
2. **Action and final element.** `action: direct` when the output rises with the measurement,
   `reverse` when it falls. Each final element: `tag`, `kind`, `actuator`, `fail` (open,
   closed, last; a drive stop, a heater off: the position on loss of signal, power or air).
   The two are independent; do not infer one from the other. A split range gives each
   element its `range` of the output; `travel: [100, 0]` an element that closes as the output
   rises.
3. **Targets** with `value`, `unit`, `provenance` and `source`; `modes` and `cases` they apply
   to; a setpoint that is a case value names it (`stream: S1, property: flow`) and validate
   keeps the two in step. A value not known yet: `unresolved: <why>`. Pressures with (a) or
   (g), temperatures in K or °C.
4. **Modes** (`manual`, `auto`, `cascade`), `normal_mode`, and `transitions` with who makes them
   and when (`condition`).
5. **Links** on the loop they act on: `cascade` (the master's output is this loop's setpoint;
   the slave needs the `cascade` mode, the master no final element), `ratio`, `feedforward`,
   `override` (`select: low | high`; the loops share the final element). A ring of cascade links
   is an error; ratio, override and feedforward links may form rings.
6. **Alarms and trips.** List process alarms with what the operator does (`response`). A
   high-high or low-low condition is a trip: put it in the C&E and list its id under
   `interlocks`.
7. **Rationale and judgement.** `rationale`: why this pairing and these modes; `operator`: what
   the operator has to judge; `unresolved`: each open point.
8. `cheme validate`, then `cheme control report` (the loop summary in `build/`).

## Validation and failure handling

- Errors: `cheme.control-duplicate`, `cheme.control-cycle` (setpoints that depend on each
  other).
- Warnings: `cheme.control-unknown-tag` (a tag renamed or removed on the P&ID: fix the
  register, or the list), `cheme.control-ref`, `cheme.control-conflict` (two loops on one
  valve without an override, a setpoint outside its limits, a mode the loop does not have),
  `cheme.control-unit`, `cheme.control-case` (the setpoint left its operating case: update one
  of them).
- Info: `cheme.control-incomplete` (draft gaps), `cheme.control-unresolved`,
  `cheme.control-shared-element` (the loop shares a sensor or valve with an interlock: not an
  error, but the HAZOP must judge independence), `cheme.control-alarm-trip`.
- The stage gates decide when the info findings must be gone: the template's gate
  `control-complete` (detail, `fail_on: info`) lists the rules it requires; the ones to judge
  rather than remove (`-shared-element`, `-alarm-trip`, `-logic-text`) and `-stale` are left
  out.

## Outputs and provenance

`control/loops.yaml`, committed; `build/control-loops.html` and `.csv` from `cheme control
report` (not committed). Each target carries its provenance; record the inputs the register
rests on under `basis:` with their revision.

## Required human judgment

Pairings, fail positions, the modes and the operator's role are design decisions with safety
consequences. The register states intent: it assigns no SIL, claims no independence of
protection layers and sets no safe operating limit. A reviewer (skill `review`) accepts it.

## References

- `docs/framework/control.md` of cheme-as-code: conventions and rules.
- Skills `control-narrative`, `control-sequences`, `review`.
- Worked example: `examples/process-separation/control/loops.yaml` of cheme-as-code.

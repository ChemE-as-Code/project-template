# Working in this plant repository

This repository is a process plant design. The files are the design; there is no other database.

## Ground rules

- P&IDs are DEXPI 2.0.1 XML in `pid/`. Edit them **directly as text** (skills `dexpi-xml`,
  `pid-edit`, `pid-draft`). For large drafts a throwaway generator script in `scripts/` using
  the pyDEXPI model classes is fine; its output is then the source of truth.
- After every change: `plantcode fmt <sheet> && plantcode validate`. Fix every `error`. Do not
  commit with errors; the pre-commit hook and CI refuse it anyway.
- One logical change per commit. Say what changed on which sheet and why. Append the change to
  `PROGRESS.md`.
- Keep the lists in step with the P&IDs (skill `lists-sync`).
- Record decisions and assumptions in `DESIGN.md`, with the reason. Mark assumed values
  `ASSUMED` in the YAML or the decision text, never present a guess as data.
- Work on a branch for anything larger than one fix; review with `plantcode diff main --list`.
- Do not edit `checks/status.json` or `build/` by hand; they are generated. `build/` is ignored.
- `checks/status.json` records a stage sign-off: `plantcode check` only prints the gates;
  `plantcode check --stage <stage> --write` writes the file, and you commit it only when signing
  off that stage, together with its review record (skill `review`). Do not commit a
  `status.json` from an ordinary check run.

## Commands

| | |
|---|---|
| `plantcode validate [files] [--json]` | all P&IDs and lists, and lists vs P&IDs (equipment, lines and DN, instruments); the last line says what was compared |
| `plantcode fmt [files] [--check]` | canonical XML, readable id on every object |
| `plantcode show <id or tag> [--sheet F]` | `file:start-end` of an object and its drawing, owner chain, references in and out; for a line number its segments, items and nodes with DN |
| `plantcode render [files] [--png] [--crop X0,Y0,X1,Y1] [--scale S] [--rev R]` | `build/<sheet>.svg` (`.png` with `--png`, `<sheet>@<rev>` with `--rev`); `--crop` in sheet mm, Y down |
| `plantcode diff <old> [new] [--list \| --json]` | HTML report in `build/diff.html`; `--list` one line per change with the drawing that shows it; `--json` the data |
| `plantcode check [--stage S] [--write]` | engineering-step gates in `checks/checks.yaml`; `--write` only at sign-off |
| `plantcode schema [kind]` | JSON Schema of a YAML file kind; no kind lists them |
| `plantcode git-setup` | once per clone: `git diff` shows P&IDs as review text with their XML ids, pre-commit hook |

## YAML files and their schema kind

| file | kind |
|---|---|
| `plant.yaml` | `plant` |
| `process/design-basis.yaml` | `design_basis` |
| `equipment/equipment-list.yaml` | `equipment` (tag, type, service, sheet, design data, notes) |
| `piping/line-list.yaml` | `lines` (number, from, to, nominal_diameter, ..., notes, `segments` with a DN each) |
| `control/instrument-list.yaml` | `instruments` (tag as printed, service, sheet, notes) |
| `checks/checks.yaml` | `checks` |

`plantcode schema <kind>` prints the full JSON Schema.

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
- Work on a branch for anything larger than one fix; review with `plantcode diff main`.
- Do not edit `checks/status.json` or `build/` by hand; they are generated.

## Commands

| | |
|---|---|
| `plantcode validate [files] [--json]` | all P&IDs and lists, and lists vs P&IDs |
| `plantcode fmt [files] [--check]` | canonical XML, readable id on every object |
| `plantcode render [files]` | SVG in `build/` |
| `plantcode diff <old> [new]` | HTML report in `build/diff.html`, changed items highlighted |
| `plantcode check [--stage S]` | engineering-step gates in `checks/checks.yaml` |
| `plantcode schema <kind>` | JSON Schema of a YAML file kind |

---
name: pid-edit
description: Change an existing P&ID sheet in pid/ - add, remove or modify equipment, lines, valves or instruments - by editing the DEXPI XML directly, then formatting, validating and rendering until clean. Use for any change to an existing sheet.
---

# Editing a P&ID sheet

Read the `dexpi-xml` skill first if you have not in this session.

1. **Find the place.** `grep -n` for the tag or id (`grep -n 'T4750\|Tank1' pid/P-01.xml`),
   then read that line range. Find both the conceptual object and its drawing
   (`grep -n '#Tank1" property="Represents"' pid/P-01.xml`).
2. **Plan the change in both trees**: what objects are added, removed or changed in the
   conceptual model, and which groups, symbols, node positions and lines in the drawing follow.
   Look at nearby coordinates so new symbols sit on the line and off other symbols.
3. **Edit the file in place.** Copy structure from `dexpi-xml/snippets/` and rename every id.
   Use readable ids that say what the object is (`GlobeValve_FV101_bypass`).
4. **Format, validate, render**:
   ```sh
   plantcode fmt pid/P-01.xml && plantcode validate && plantcode render pid/P-01.xml
   ```
   Fix every `error`, then every new `warning`. Repeat until clean. Look at `build/P-01.svg`
   (read the SVG or open it) to check the drawing makes sense.
5. **Review the change** before committing: `plantcode diff HEAD` writes `build/diff.html`
   (added green, removed red, modified orange) and prints counts. The counts should match what
   you meant to change; anything else is an accident.
6. **Keep the lists in step** (`lists-sync` skill): a new tagged item goes into the equipment,
   line or instrument list.
7. **Commit** one logical change per commit, message saying what changed on which sheet and why.
   Log the change in `PROGRESS.md` (date, sheet, what, why).

Large restructures (many items, a whole new section) are often easier with a throwaway
generator script: see `pid-draft`.

Never delete a validation finding by deleting the content it is about unless that is the change
you were asked for.

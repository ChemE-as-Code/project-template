---
name: review
description: Review the plant repository or a change to it - run validation and the engineering-step checks, look at the rendered sheets and the diff, and write findings to reviews/. Use when asked to review, check, or sign off a stage or a branch.
---

# Reviewing

1. `plantcode validate` and `plantcode check` (writes `checks/status.json`). Every failing check
   and every error is a finding.
2. For a branch: `plantcode diff main` and read `build/diff.html`. Does every change have a
   reason in the commit messages and `PROGRESS.md`?
3. Engineering review of each changed sheet (`plantcode render`, read the SVG and the XML):
   isolation valves around equipment that is maintained, check valves on pump discharges,
   relief for anything that can be blocked in, drains and vents at low and high points,
   instruments for what the design basis says is controlled. Note what you checked and what you
   could not.
4. Write `reviews/<YYYY-MM-DD>-<topic>.md`:
   ```markdown
   # Review: <topic>
   Scope: <branch / sheets / stage>   Reviewer: <name or agent>   Date: <date>
   ## Findings
   | # | sheet | item | finding | severity | proposed action |
   ## Checked and fine
   ## Not checked
   ```
5. Do not fix findings in the same commit as the review; each fix is its own change.

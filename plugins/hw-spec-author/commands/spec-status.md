---
description: Inspect an existing hardware spec, report current D-stage (D0/D1/D2/D3), and list what's missing for the next gate.
argument-hint: "[path]"
allowed-tools: Read, Glob, Grep
---

You are activating the spec-status check of the **hw-spec-author** skill.

## Steps

1. **Read the stage gates reference**: open `${CLAUDE_PLUGIN_ROOT}/skills/hw-spec-author/references/process/stage_gates.md` to load the full checklist for each stage.

2. **Locate the spec**. Parse the user's invocation `/spec-status [path]`:
   - If a path is given, use it as the spec root.
   - If no path, look for a likely spec root in CWD: a directory containing `README.md` plus a `doc/` subfolder, or `./spec/`, or `./<repo_root>/spec/`. If multiple candidates, ask the user which one.
   - If nothing found, tell the user and suggest `/spec-init <ip_name>`.

3. **Read all 6 files** that exist:
   - `README.md`
   - `doc/theory_of_operation.md`
   - `doc/programmers_guide.md`
   - `doc/interfaces.md`
   - `doc/registers.md`
   - `dv/plan.md`

   Note which are missing (= work not yet started for that section).

4. **Score against each gate**. Walk the checklist for D0, D1, D2, D3 in order. For each item:
   - Mark **✓** if the spec satisfies it.
   - Mark **✗** if it doesn't.
   - Mark **?** if the answer is genuinely unclear from inspection (e.g., "FSM transitions are present but completeness vs. RTL can't be checked without RTL").

5. **Determine current stage**:
   - The current stage is the **highest stage where every item is ✓**.
   - If D0 has any ✗, the spec is pre-D0 (skeleton not yet done).
   - Don't round up. A spec at "D1 minus one missing FSM" is still D0.

6. **Output the report** in this structure:

```
Spec: <path>
Current stage: <D0 | D1 | D2 | D3 | pre-D0>

D0 — Concept
  ✓ <item>
  ✓ <item>
  ✗ <item>           ← what's needed: <one-line action>

D1 — Spec ~90% complete
  (only show this section if D0 is fully ✓)
  ...

Recommended next actions (top 3, ordered by leverage):
  1. <concrete next step that closes the most ✗ items>
  2. ...
  3. ...

TODO markers found: <count> across <N> files
  - README.md: <count>
  - doc/theory_of_operation.md: <count>
  ...
```

7. **Don't auto-advance**. `/spec-status` is observational. If the user wants to push to the next stage, they invoke `/spec-gate <stage>` after the gaps are filled, or ask in natural conversation to fill specific gaps.

## Constraints

- Read every section before scoring. Do **not** skim.
- For "?" items, briefly explain what would be needed to convert them to ✓ or ✗.
- Be honest. If the spec looks superficially complete but key audience-separation rules are violated, say so. The whole point of stage gates is they're not gameable.
- Limit the recommended next actions to 3. More than 3 is paralysis-inducing.

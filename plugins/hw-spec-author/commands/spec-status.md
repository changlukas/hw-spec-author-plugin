---
description: Inspect an existing hardware spec (mode-aware), report current D-stage (D0/D1/D2/D3), and list what's missing for the next gate.
argument-hint: "[path]"
allowed-tools: Read, Glob, Grep
---

You are activating the spec-status check of the **hw-spec-author** skill.

## Steps

1. **Read the stage gates reference**: open `${CLAUDE_PLUGIN_ROOT}/skills/hw-spec-author/references/process/stage_gates.md` to load the full checklist for each stage, including mode-conditional `**Applies when:**` annotations.

2. **Locate the spec**. Parse the user's invocation `/spec-status [path]`:
   - If a path is given, use it as the spec root.
   - If no path, look for a likely spec root in CWD: a directory containing `README.md` plus a `doc/` subfolder, or `./spec/`, or `./<repo_root>/spec/`. If multiple candidates, ask the user which one.
   - If nothing found, tell the user and suggest `/spec-init <ip_name>`.

3. **Detect mode**. Read `<spec_root>/MODE.md` and parse the `mode:` line. If `MODE.md` is absent, default to `behavioral-block` (legacy spec). Record the mode for use in steps 4–5.

4. **Read all spec files** that exist, based on mode:

   **For `behavioral-block`:**
   - `README.md`
   - `doc/theory_of_operation.md`
   - `doc/programmers_guide.md`
   - `doc/interfaces.md`
   - `doc/registers.md`
   - `dv/plan.md`

   **For `protocol-bfm`:**
   - `README.md`
   - `doc/theory_of_operation.md`
   - `doc/signal_interface.md`
   - `doc/pin_level_reset.md`
   - `doc/protocol_rules.md`
   - `doc/channel_handshake.md`
   - `doc/transaction_api.md`
   - `doc/channel_api.md`
   - `doc/active_passive_mode.md`
   - `doc/registers.md` (optional — only if the BFM has a CSR)
   - `dv/plan.md`

   Note which expected files are missing (= work not yet started for that section).

5. **Score against each gate** (mode-aware). Walk the checklist for D0, D1, D2, D3 in order. For each item:
   - Check the item's `**Applies when:**` clause (at the section heading or per-item). If the clause does not match the spec's mode, mark the item **N/A** (not ✗) and skip it from the score.
   - For applicable items:
     - Mark **✓** if the spec satisfies it.
     - Mark **✗** if it doesn't.
     - Mark **?** if the answer is genuinely unclear from inspection.

6. **Determine current stage**:
   - The current stage is the **highest stage where every applicable item is ✓**. N/A items do not count toward or against the score.
   - If D0 has any ✗, the spec is pre-D0 (skeleton not yet done).
   - Don't round up. A spec at "D1 minus one missing item" is still D0.

7. **Output the report** in this structure:

```
Spec: <path>
Mode: <behavioral-block | protocol-bfm | (default behavioral-block — no MODE.md)>
Current stage: <D0 | D1 | D2 | D3 | pre-D0>

D0 — Concept
  ✓ <item>
  ✓ <item>
  N/A <item>         ← skipped: applies only to <other mode>
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
  - doc/<file>.md: <count>
  ...
```

8. **Don't auto-advance**. `/spec-status` is observational. If the user wants to push to the next stage, they invoke `/spec-gate <stage>` after the gaps are filled, or ask in natural conversation to fill specific gaps.

## Constraints

- Read every applicable section before scoring. Do **not** skim.
- For "?" items, briefly explain what would be needed to convert them to ✓ or ✗.
- Be honest. If the spec looks superficially complete but key audience-separation rules are violated, say so. The whole point of stage gates is they're not gameable.
- Mode-conditional items get **N/A** (skipped from score), not **✗**. Treating an inapplicable item as a failure makes the gate ungameable in the wrong direction — the spec can never reach D1 if it's penalised for files that don't apply to its mode.
- Limit the recommended next actions to 3. More than 3 is paralysis-inducing.

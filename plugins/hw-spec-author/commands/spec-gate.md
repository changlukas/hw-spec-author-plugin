---
description: Check whether a hardware spec meets the criteria for a target stage gate (D1, D2, or D3). Lists every checklist item and which are still open. Reads and writes WAIVERS.md.
argument-hint: <D1|D2|D3> [path]
allowed-tools: Read, Write, Edit, Glob, Grep
---

You are running a **stage-gate ceremony** for the hw-spec-author skill.

## Steps

1. **Read the gate definitions**: open `${CLAUDE_PLUGIN_ROOT}/skills/hw-spec-author/references/process/stage_gates.md` to load the full criteria for D0, D1, D2, D3.

2. **Parse the invocation** `/spec-gate <stage> [path]`:
   - Required argument: `<stage>` ∈ {D1, D2, D3}. (D0 is the starting state; there's no gate to "achieve" D0.)
   - Optional `[path]` to spec root. If absent, find it like `/spec-status` does.
   - If `<stage>` is invalid, tell the user and abort.

3. **Verify prerequisites**. Before checking the requested gate, confirm the previous gate is met:
   - For `/spec-gate D2`, all D1 items must be ✓.
   - For `/spec-gate D3`, all D2 items must be ✓.
   - If a prerequisite gate is not met, do **not** check the requested gate. Tell the user which earlier gate to close first, and run that check instead.

4. **Run the checklist**. For the requested gate, walk **every item** from `stage_gates.md`. Each item carries an explicit anchor in the form `(id: <stage>.<scope>.<short_name>)` — extract this anchor verbatim; do not derive or paraphrase it. For each item:
   - Mark **✓** if satisfied.
   - Mark **✗** with a one-line reason if not.
   - Mark **⚠ waived** if the item's anchor is recorded as waived in `<spec_root>/WAIVERS.md` (see step 4a).

### 4a. Waiver persistence

Waivers must persist across sessions. Read `<spec_root>/WAIVERS.md` if it exists. The waiver file uses each item's anchor (from `stage_gates.md`) as an `## H2` heading. Format:

```markdown
# Spec Waivers

Each entry waives one stage-gate checklist item with a documented rationale.
The H2 heading must match a checklist anchor from stage_gates.md exactly
(case-sensitive). Waivers do not auto-expire; remove them when the underlying
issue is resolved.

---

## D2.sec_cm_list

- **Item text** (copy from stage_gates.md): "Sec_cm list (if security-critical) is exhaustive and reflected in dv/plan.md."
- **Waived by**: <user identifier or name>
- **Date**: 2026-04-15
- **Rationale**: Block has no security role. Asset list confirmed empty by security WG review (link or quote).

---

## D1.too.performance

- **Item text**: "Performance commitments stated, or 'no performance commitment' explicitly stated."
- **Waived by**: <user>
- **Date**: 2026-04-20
- **Rationale**: Performance specification deferred to architecture review meeting on 2026-05-01. Tracked in issue #87.
```

Matching rule: an item is **⚠ waived** iff its anchor (the `(id: ...)` token in `stage_gates.md`) appears verbatim as an `## H2` heading in `WAIVERS.md`. If the anchor is missing or misspelled, the item is **not** waived — surface this as a separate report line so the user can fix the waiver.

### 4b. Adding a waiver

If during the gate check the user explicitly waives an open ✗ item, do **not** mark it waived in the report immediately. Instead:
- Confirm the user wants to record the waiver permanently.
- Ask for: who is waiving (their name or identifier) and a one-sentence rationale.
- Append the waiver to `<spec_root>/WAIVERS.md` (creating the file if needed). Use the item's anchor from `stage_gates.md` verbatim as the `## H2` heading. Copy the item text into the **Item text** bullet so the waiver is self-explanatory if the source checklist evolves.
- Re-run step 4 so the waiver shows as ⚠ in the report.

A bare "skip that" without rationale is rejected; ask for the rationale before recording.

5. **Special handling for security and performance items**:
   - If the spec has no security countermeasure section but the IP is in a security-critical role, that's a ✗ for D2/D3.
   - If performance is unspecified, that's only ✗ if the IP claims a performance feature. A spec that explicitly states "no performance commitment" satisfies the item.

6. **Output the report** in this structure:

```
Stage Gate Check
Spec: <path>
Target: <D1 | D2 | D3>
Prerequisite gate (<previous>): <met | not met>

(if previous gate not met:
  → run '/spec-gate <previous>' first. Stopping here.
)

(if previous gate met:
Checklist for <target>:
  ✓ <item>
  ✓ <item>
  ✗ <item>          ← <reason>
  ⚠ <item>          ← waived: <rationale from earlier>
  ...

Score: <N>/<total>  (<N>/<total - waived> excluding waivers)

Verdict:
  - PASS  → all items ✓ or ⚠. Spec is at <target>.
  - FAIL  → <count> open items. Spec remains at <previous gate or pre-D0>.
)
```

7. **If FAIL**: list the open items in priority order. Suggest concrete next actions for the top 3.

8. **If PASS**: explicitly state the spec has reached the target stage. Suggest the next ceremony:
   - At D1: run `/spec-review` for reader test (if not already done), then start RTL implementation.
   - At D2: pull RTL into the conversation; sync any changes back to the spec.
   - At D3: this is sign-off; recommend the user obtain HW-lead, DV-lead, SW-lead approval and freeze the spec.

## Constraints

- **No partial passes**. A spec is either at a stage or not. There is no "D1 minus."
- **Waivers must have rationale**. If the user wants to waive an item, they must state why in the conversation. A bare "skip that" is rejected; ask for a one-sentence reason and record it.
- **Do not auto-edit the spec**. This command is a check, not a fix. Editing flows from the user's response to the report.
- **Be uncomfortable when warranted**. If the spec has many ✗ items dressed up as "minor," say so plainly. Stage gates exist because spec debt accumulates silently otherwise.

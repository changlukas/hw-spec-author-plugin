---
description: Check whether a hardware spec meets the criteria for a target stage gate (D1, D2, or D3). Mode-aware (reads MODE.md and respects **Applies when:** annotations). Lists every applicable checklist item and which are still open. Reads and writes WAIVERS.md.
argument-hint: <D1|D2|D3> [path]
allowed-tools: Read, Write, Edit, Glob, Grep
---

You are running a **stage-gate ceremony** for the hw-spec-author skill.

## Steps

1. **Read the gate definitions**: open `${CLAUDE_PLUGIN_ROOT}/skills/hw-spec-author/references/process/stage_gates.md` to load the full criteria for D0, D1, D2, D3, including mode-conditional `**Applies when:**` annotations.

2. **Parse the invocation** `/spec-gate <stage> [path]`:
   - Required argument: `<stage>` ∈ {D1, D2, D3}. (D0 is the starting state; there's no gate to "achieve" D0.)
   - Optional `[path]` to spec root. If absent, find it like `/spec-status` does.
   - If `<stage>` is invalid, tell the user and abort.

3. **Detect mode**. Read `<spec_root>/MODE.md` and parse the `mode:` line. If `MODE.md` is absent, default to `behavioral-block` (legacy spec). The mode determines which `**Applies when:**` items count.

4. **Verify prerequisites**. Before checking the requested gate, confirm the previous gate is met:
   - For `/spec-gate D2`, all applicable D1 items must be ✓ (mode-conditional N/A items do not block).
   - For `/spec-gate D3`, all applicable D2 items must be ✓.
   - If a prerequisite gate is not met, do **not** check the requested gate. Tell the user which earlier gate to close first, and run that check instead.

5. **Run the checklist**. For the requested gate, walk **every item** from `stage_gates.md`. Each item carries an explicit anchor in the form `(id: <stage>.<scope>.<short_name>)` — extract this anchor verbatim; do not derive or paraphrase it. For each item:
   - Check the item's `**Applies when:**` clause (at the section heading or per-item). If the clause does not match the spec's mode, mark the item **N/A** and skip from scoring.
   - For applicable items:
     - Mark **✓** if satisfied.
     - Mark **✗** with a one-line reason if not.
     - Mark **⚠ waived** if the item's anchor is recorded as waived in `<spec_root>/WAIVERS.md` (see step 5a).

### 5a. Waiver persistence

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

### 5b. Adding a waiver

If during the gate check the user explicitly waives an open ✗ item, do **not** mark it waived in the report immediately. Instead:
- Confirm the user wants to record the waiver permanently.
- Ask for: who is waiving (their name or identifier) and a one-sentence rationale.
- Append the waiver to `<spec_root>/WAIVERS.md` (creating the file if needed). Use the item's anchor from `stage_gates.md` verbatim as the `## H2` heading. Copy the item text into the **Item text** bullet so the waiver is self-explanatory if the source checklist evolves.
- Re-run step 5 so the waiver shows as ⚠ in the report.

A bare "skip that" without rationale is rejected; ask for the rationale before recording.

6. **Special handling for security and performance items**:
   - If the spec has no security countermeasure section but the IP is in a security-critical role, that's a ✗ for D2/D3.
   - If performance is unspecified, that's only ✗ if the IP claims a performance feature. A spec that explicitly states "no performance commitment" satisfies the item.

7. **Output the report** in this structure:

```
Stage Gate Check
Spec: <path>
Mode: <behavioral-block | protocol-bfm>
Target: <D1 | D2 | D3>
Prerequisite gate (<previous>): <met | not met>

(if previous gate not met:
  → run '/spec-gate <previous>' first. Stopping here.
)

(if previous gate met:
Checklist for <target>:
  ✓ <item>
  ✓ <item>
  N/A <item>        ← skipped: applies only to <other mode>
  ✗ <item>          ← <reason>
  ⚠ <item>          ← waived: <rationale from earlier>
  ...

Score: <N>/<applicable_total>  (<N>/<applicable_total - waived> excluding waivers; <K> N/A items skipped)

Verdict:
  - PASS  → all applicable items ✓ or ⚠. Spec is at <target>.
  - FAIL  → <count> open items. Spec remains at <previous gate or pre-D0>.
)
```

8. **If FAIL**: list the open items in priority order. Suggest concrete next actions for the top 3.

9. **If PASS**: explicitly state the spec has reached the target stage. Suggest the next ceremony:
   - At D1: run `/spec-review` for reader test (if not already done), then start implementation (RTL for behavioral-block mode; BFM source for protocol-bfm mode).
   - At D2 (behavioral-block): pull RTL into the conversation; sync any changes back to the spec.
   - At D2 (protocol-bfm): pull BFM implementation into the conversation; verify protocol_rules.md self-checks pass.
   - At D3: this is sign-off; recommend the user obtain HW-lead, DV-lead, SW-lead approval and freeze the spec.

## Constraints

- **No partial passes**. A spec is either at a stage or not. There is no "D1 minus."
- **Waivers must have rationale**. If the user wants to waive an item, they must state why in the conversation. A bare "skip that" is rejected; ask for a one-sentence reason and record it.
- **Do not auto-edit the spec**. This command is a check, not a fix. Editing flows from the user's response to the report.
- **Be uncomfortable when warranted**. If the spec has many ✗ items dressed up as "minor," say so plainly. Stage gates exist because spec debt accumulates silently otherwise.
- **Mode-conditional N/A is not a waiver**. An item skipped because its `**Applies when:**` clause does not match the spec's mode is N/A — it does not need a WAIVERS.md entry. Only ✗ items that the user wants to skip become ⚠ waived.

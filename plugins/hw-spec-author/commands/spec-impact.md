---
description: Analyse the cross-file ripple of a proposed change to a hardware spec. Two phases — a mechanical keyword grep (always) and an opt-in LLM section-impact pass (--analyse). Read-only; never edits the spec.
argument-hint: "<change-description> [--analyse]"
allowed-tools: Read, Grep, Glob
---

You are running the **impact analysis** of the hw-spec-author skill. A spec change rarely touches one file — renaming a register, dropping a mode, or changing a parameter ripples across registers / theory_of_operation / dv-plan / protocol_rules. This command surfaces the likely blast radius before the author starts editing, so stale references are found up front rather than one wave at a time.

It is **read-only**. It never edits the spec and never propagates a change automatically — it reports where a change is likely to land, and the author decides what to act on.

## Steps

1. **Locate the spec**. Same logic as `/spec-status`. If nothing found, stop.

2. **Detect mode**. Read `<spec_root>/MODE.md`; default `behavioral-block` if absent. Inventory the mode-appropriate file set (same list as `/spec-lint`).

3. **Phase 1 — mechanical keyword grep (always runs, no LLM judgement)**:
   - Extract candidate keywords from `<change-description>`: register names, signal names, parameter names, mode names, rule IDs, and other distinctive identifiers (UPPER_SNAKE tokens, `*_i`/`*_o` signals, `0x` offsets, quoted phrases).
   - **Echo the extracted keyword list back to the user first**: `extracted keywords: [...]`. If the list looks wrong or empty, tell the user to rephrase with sharper nouns. This makes a vague change description visible instead of silently returning "no impact."
   - grep each keyword across all spec files. Produce a `file:line:context` hit list grouped by file.

4. **Phase 2 — LLM section-impact (only when `--analyse` is passed)**:
   - For each file with hits, judge whether the section is *materially* affected by the change (not just a textual match).
   - Output a list of `(file, section, expected-action)` tuples — e.g. `(registers.md, §CTRL field table, "rename field row")`.
   - This is **advisory**. Mark it clearly as a judgement, not a fact. The author decides.

5. **Output**:

```
Impact analysis: "<change-description>"
Spec: <path>   Mode: <...>

Extracted keywords: [<k1>, <k2>, ...]
  (if this list is wrong, rephrase the change with sharper identifiers)

Phase 1 — textual hits (mechanical):
  registers.md
    :42  <context line>
    :88  <context line>
  theory_of_operation.md
    :211 <context line>
  ...
  Total: <N> hits across <M> files.

Phase 2 — section impact (advisory, --analyse):    [omitted unless --analyse]
  registers.md §CTRL       → likely: rename field row
  dv/plan.md §Testpoints   → likely: TP-07 description references old name
  ...
```

## Constraints

- **Read-only.** Never edit. Never apply the change. Output is a map, not an action.
- **Vague input must be visible.** A change description with no extractable identifier (e.g. "tighten the timing") yields a near-empty keyword list — say so explicitly rather than reporting "no impact found," which a caller would misread as safe.
- **Phase 2 is opt-in.** Without `--analyse`, stop after the deterministic Phase 1. The grep output is auditable; the LLM judgement is not, so it is gated behind an explicit flag.
- **Complements, does not replace.** After acting on the impact list, run `/spec-lint` to confirm no broken cross-references remain. For renaming a term across files, `/spec-rename` does it interactively.

---
description: Run mechanical consistency checks on a hardware spec. Detects broken cross-references, untracked TODO markers, and Feature-to-Testpoint mapping gaps.
argument-hint: "[path]"
allowed-tools: Read, Grep, Glob
---

You are running **mechanical lint** over a hardware spec. This complements `/spec-status` (gate checklist) and `/spec-review` (ambiguity detection) by catching the silent rot that erodes a spec over time.

## Steps

1. **Locate the spec**. Same logic as `/spec-status`. If nothing found, stop.

2. **Inventory** the six canonical files using Glob.

3. **Run each lint check** in order. For each violation found, record:
   - File and line number
   - Lint rule violated
   - Suggested fix

4. **Output a structured report** at the end.

---

## Lint rules

### LINT-001: Broken cross-references

Scan all spec files for cross-reference patterns:
- "see <file> §<N>"
- "see Theory of Operation §<N>"
- "described in <section name>"
- "(see <X>)"
- Markdown links of the form `[text](./path/to/file.md#anchor)`

For each reference, verify the target exists:
- File path: must resolve to an existing file under spec root.
- Section anchor: the named heading must exist in the target file (case-insensitive match on heading text).

**Violation example:**
```
LINT-001 in registers.md:42
  Reference: "see Theory of Operation §5.2"
  Problem: theory_of_operation.md has no §5 with a subsection 5.2.
  Suggested fix: review the cross-reference or rename the target heading.
```

### LINT-002: Untracked TODO markers

Per `writing_principles.md` §6, every TODO must have an owner and a tracking link or explicit "no issue yet" rationale.

Scan for these patterns:
- `TODO(designer):` — must be followed by issue link, e.g. `(see #42)` or `(no issue yet — <reason>)`
- Bare `TBD` — always a violation; should be `TODO(designer): <description> (...)`
- `IMPLEMENTATION-DEFINED:` — must include a brief justification

**Violation example:**
```
LINT-002 in theory_of_operation.md:103
  Pattern: bare "TBD"
  Problem: untracked. Cannot be triaged or resolved.
  Suggested fix: replace with "TODO(designer): <what's missing> (see issue #N)" or similar.
```

### LINT-003: Feature ↔ Testpoint mapping

Per the writing principles and `dv_plan.md` template, every Feature in `README.md` must map to at least one testpoint in `dv/plan.md`.

Procedure:
- Extract Feature bullets from `README.md` `## Features` section.
- Extract testpoints from `dv/plan.md` `## Testpoints` table.
- For each Feature: search testpoints for any reference to a keyword distinctive of that feature.
- Report Features with no apparent testpoint coverage.
- Optionally, also report testpoints with no apparent matching Feature (these are not strict violations — DV can have testpoints not tied to a Feature — but worth flagging).

**Violation example:**
```
LINT-003 in README.md:Features bullet 4
  Feature: "Optional prescaler (1, 2, 4, ..., 32768) selectable at runtime."
  Problem: no testpoint in dv/plan.md mentions prescaler or PRESCALE.
  Suggested fix: add a testpoint exercising prescaler at multiple divisors, OR remove the Feature if not intended for verification.
```

### LINT-004: Register name consistency

Scan for register names. A register has a canonical name in `registers.md` (e.g., `INTR_STATE`, `CTRL`). All references in other files must use the same case.

Procedure:
- Extract canonical register names from `registers.md` register map table (first column).
- For each spec file, find candidate register references using regex like `\b[A-Z][A-Z0-9_]{2,}\b` (or use already-quoted register names in backticks).
- Flag mismatches: e.g., `Intr_State` or `intr_state` when the canonical is `INTR_STATE`.

**Violation example:**
```
LINT-004 in programmers_guide.md:67
  Reference: "intr_state"
  Canonical: "INTR_STATE"
  Suggested fix: use canonical case for register names.
```

### LINT-005: Port / signal name consistency

Same as LINT-004 but for ports listed in `interfaces.md`.

### LINT-006: Reset value format

In `registers.md`, every register and field must have a reset value. Allowed formats:
- `0x<hex>` for full registers (e.g., `0x00000000`)
- `0x<hex>` or `0b<bin>` for fields, with width matching the field
- `—` (em dash) only for write-only registers
- `strap-dependent (see <ref>)` for strap-dependent resets

Flag any other patterns (e.g., bare `0`, "all zero", "default", missing column entry).

**Violation example:**
```
LINT-006 in registers.md:CTRL field "mode"
  Reset value: "0"
  Problem: ambiguous. mode is 4 bits — should be "0x0".
  Suggested fix: use width-matched hex literal.
```

### LINT-007: Stale block diagram references

If a file references an SVG (e.g., `![block diagram](block_diagram.svg)`), verify the SVG exists in the same directory.

Mermaid blocks are inline and self-contained, so do not need this check.

---

## Output format

```
Spec Lint Report
Spec: <path>
Files inspected: <count>

Violations found: <total>
  LINT-001 (broken xref):           <count>
  LINT-002 (untracked TODO):        <count>
  LINT-003 (Feature/Testpoint map): <count>
  LINT-004 (register casing):       <count>
  LINT-005 (port casing):           <count>
  LINT-006 (reset value format):    <count>
  LINT-007 (missing SVG):           <count>

---

Details:

[LINT-001] registers.md:42
  Reference: "see Theory of Operation §5.2"
  Problem: target section not found.
  Suggested fix: ...

[LINT-002] theory_of_operation.md:103
  ...

(continue for each violation)

---

Summary:
  - <N> blocking violations (LINT-001, LINT-006 typically): fix before proceeding.
  - <N> hygiene violations (LINT-002, LINT-004, LINT-005): fix before D1 sign-off.
  - <N> coverage gaps (LINT-003): triage with DV lead.
```

## Constraints

- **Read-only.** This command does not edit the spec. If the user wants edits, they invoke them in conversation after reviewing the report.
- **Be precise about line numbers**. False positives are worse than missed positives — they erode trust in the lint.
- **Don't conflate with `/spec-status`**. Lint catches mechanical inconsistency; `/spec-status` checks gate completeness. They are complementary.
- If the spec is small and all checks pass, output `No violations.` and stop. A clean lint is a real result and shouldn't be padded with prose.

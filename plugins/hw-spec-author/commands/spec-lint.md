---
description: Run mechanical consistency checks on a hardware spec. Mode-aware (LINT-001..007 always; LINT-BFM-001..005 additionally in protocol-bfm mode). Detects broken cross-references, untracked TODO markers, Feature-to-Testpoint mapping gaps, and BFM-mode signal/rule inconsistencies.
argument-hint: "[path]"
allowed-tools: Read, Grep, Glob
---

You are running **mechanical lint** over a hardware spec. This complements `/spec-status` (gate checklist) and `/spec-review` (ambiguity detection) by catching the silent rot that erodes a spec over time.

## Steps

1. **Locate the spec**. Same logic as `/spec-status`. If nothing found, stop.

2. **Detect mode**. Read `<spec_root>/MODE.md` and parse the `mode:` line. If `MODE.md` is absent, default to `behavioral-block` (legacy spec).

3. **Inventory** the spec files using Glob, based on mode:
   - `behavioral-block`: README.md, doc/theory_of_operation.md, doc/programmers_guide.md, doc/interfaces.md, doc/registers.md, dv/plan.md.
   - `protocol-bfm`: README.md, doc/theory_of_operation.md, doc/signal_interface.md, doc/pin_level_reset.md, doc/protocol_rules.md, doc/channel_handshake.md, doc/transaction_api.md, doc/channel_api.md, doc/active_passive_mode.md, dv/plan.md, plus doc/registers.md if present.

4. **Run each lint check** in order. LINT-001..007 apply in both modes. LINT-BFM-001..005 apply only when `mode == protocol-bfm`. For each violation found, record:
   - File and line number
   - Lint rule violated
   - Suggested fix

5. **Output a structured report** at the end (header includes Mode line).

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

A register has a canonical name in `registers.md` (e.g., `INTR_STATE`, `CTRL`). All references in other files must use that exact spelling.

The check is **canonical-driven**, not pattern-driven — do not flag arbitrary uppercase tokens. Acronyms like `APB`, `PSLVERR`, `RTL`, `FSM`, `IRQ`, `FPV`, `RW1C` are not register names and must not be flagged.

Procedure:
1. Extract the canonical register name set `R` from the first column of the `## Register map` table in `registers.md`. These are the only names eligible for this check.
2. For each canonical name `r ∈ R`, build the set of *plausible case variants* you would expect to see if the author drifted:
   - lowercase: `intr_state`
   - Title_Snake: `Intr_State`
   - PascalCase (drop underscores): `IntrState`
   - all-lowercase joined: `intrstate`
3. For each spec file other than `registers.md` itself, search for occurrences of any variant from step 2 (case-sensitive, whole-word, **excluding** any occurrence equal to the canonical `r`).
4. Each hit is a violation. Report file, line, the variant found, and the canonical spelling.

**Do not** scan with a generic uppercase regex; that produces 50–75% false-positive noise (acronyms, parameter names, RTL tokens, FSM state names).

**Violation example:**
```
LINT-004 in programmers_guide.md:67
  Found: "intr_state"
  Canonical (per registers.md): "INTR_STATE"
  Suggested fix: use canonical spelling "INTR_STATE".
```

### LINT-005: Port / signal name consistency

Apply the same canonical-driven approach as LINT-004, but with the canonical set drawn from the signal columns of the tables in `interfaces.md` (Clocks, Resets, Bus interface signal tables, Sideband, Interrupts, Alerts, Inter-IP). Ports often use mixed-case suffixes (`_i`, `_o`, `_ni`); apply variant generation accordingly:
- canonical `clk_i`: variants include `CLK_I`, `Clk_I`, `clk_in`, `clki`
- canonical `intr_cmp0_o`: variants include `INTR_CMP0_O`, `intr_cmp0`, `IntrCmp0_o`

Same exclusion rule: do not flag generic identifiers that happen to resemble a port name; only flag explicit case-variants of canonical entries from the interfaces tables.

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

## BFM-mode lint rules

The following rules apply only when `MODE.md` declares `mode: protocol-bfm`. Skip all of them in `behavioral-block` mode.

### LINT-BFM-001: signal_interface ↔ pin_level_reset wire-set parity

Every wire listed in `doc/signal_interface.md` §Wire table must appear in **both** tables of `doc/pin_level_reset.md` (the during-reset table and the after-reset table). Missing wires in either reset table are violations; extra wires in pin_level_reset that don't exist in signal_interface are also violations.

**Scope**: this check applies only to the §Wire table of signal_interface.md. The protocol clock and reset signals (e.g. ACLK / ARESETn / PCLK / PRESETn) live in §Protocol clock and reset and are **excluded** from this check — they are not expected to appear in pin_level_reset.md tables.

Procedure:
1. Extract the canonical wire-name set `W` from the first column of `signal_interface.md` §Wire table (NOT §Protocol clock and reset).
2. Extract the wire-name set `R_during` from the during-reset table in `pin_level_reset.md`.
3. Extract the wire-name set `R_after` from the after-reset table.
4. Report `W \ R_during` (wires missing from during-reset).
5. Report `W \ R_after` (wires missing from after-reset).
6. Report `R_during \ W` and `R_after \ W` (extra wires not in signal_interface §Wire table).

**Violation example:**
```
LINT-BFM-001 in pin_level_reset.md
  Wire WSTRB present in signal_interface.md §Wire table but missing from after-reset table.
  Suggested fix: add WSTRB row to after-reset table with appropriate value.
```

### LINT-BFM-002: protocol-rule ID format

Every rule ID in `doc/protocol_rules.md` must match the format `<PROTO>_<ROLE>_<CHANNEL>_<NAME>` (or `<PROTO>_<ROLE>_<NAME>` for protocols without channels — declared by absence of channel grouping in signal_interface.md).

Allowed component values:
- `<PROTO>`: uppercase identifier matching the **Protocol** declaration at the top of protocol_rules.md (e.g., `AXI4LITE`, `AXI4`, `APB`, `TLUL`).
- `<ROLE>`: `MST` (master), `SLV` (slave), or `MON` (monitor-only/passive).
- `<CHANNEL>`: matches a channel from `signal_interface.md` §Channel grouping, or the special tokens `RST` (reset rules), `XCH` (cross-channel rules), `CFG` (configuration-knob rules).
- `<NAME>`: snake_case, non-empty.

Procedure:
1. Extract every rule ID from the first column of every rule table in `protocol_rules.md`.
2. For each ID, verify it matches the format with components from the allowed sets above.
3. Verify ID uniqueness across all tables (no two rules share an ID).

**Violation example:**
```
LINT-BFM-002 in protocol_rules.md:42
  Rule ID: "axi4lite_slv_aw_awvalid_stable" (lowercase)
  Problem: PROTO and ROLE must be uppercase.
  Suggested fix: rename to "AXI4LITE_SLV_AW_AWVALID_STABLE".
```

### LINT-BFM-003: protocol-rule completeness

Every row in every rule table in `doc/protocol_rules.md` must have a non-empty Condition column AND a non-empty Required behavior column AND a non-empty Severity column AND a non-empty ARM SVA equivalent column.

Severity must be one of: `FAIL`, `RECOMMEND`. Any other value (e.g., `WARN`, `ERROR`) is a violation.

ARM SVA equivalent column must be either: a verified ARM ID, an `(unverified)`-suffixed ARM-pattern ID, or the literal `(none)`. Empty cells are violations.

**Violation example:**
```
LINT-BFM-003 in protocol_rules.md:55
  Rule ID: AXI4LITE_SLV_AW_AWREADY_NO_LATCH
  Problem: ARM SVA equivalent column is empty.
  Suggested fix: list the ARM ID, mark "(unverified)", or write "(none)".
```

### LINT-BFM-004: protocol_rules channel ↔ signal_interface channel grouping

Every channel referenced in `protocol_rules.md` (either as a `<CHANNEL>` field in a rule ID or as a `## <Channel> rules` sub-section heading) must exist in `signal_interface.md` §Channel grouping. Inventing a channel name in protocol_rules that doesn't appear in signal_interface is a violation.

The special tokens `RST`, `XCH`, `CFG` are exempt from this check.

Procedure:
1. Extract the canonical channel set `C` from `signal_interface.md` §Channel grouping (first column of the table).
2. Extract every `<CHANNEL>` field from rule IDs in `protocol_rules.md` (third underscore-delimited component).
3. Extract every `## <name> rules` sub-section heading.
4. Report any channel referenced in protocol_rules that is not in `C ∪ {RST, XCH, CFG}`.

**Violation example:**
```
LINT-BFM-004 in protocol_rules.md:78
  Channel referenced: "AWA"
  Problem: signal_interface.md §Channel grouping declares channels {AW, W, B, AR, R}; AWA is not among them.
  Suggested fix: rename to "AW" (likely typo) or add AWA to signal_interface.md §Channel grouping.
```

### LINT-BFM-005: Transaction API ↔ Channel API decomposition cross-reference

Every method sub-section in `doc/transaction_api.md` must contain an "Equivalent channel API decomposition" entry. The entry's value must be either:
- A non-empty step-by-step mapping to channel_api.md calls (e.g., naming `begin_phase_AW`, `assert_valid_AW`, `wait_for_ready_AW`, `end_phase_AW`).
- The literal `(none — this is Transaction-API-only)` or equivalent explicit "no channel-API decomposition" annotation.

Empty or missing decomposition entries are violations.

Additionally, every channel-API call name referenced in a decomposition must exist as a method in `doc/channel_api.md`.

**Violation example:**
```
LINT-BFM-005 in transaction_api.md
  Method: apply_burst_write
  Problem: "Equivalent channel API decomposition" sub-section is missing.
  Suggested fix: add the decomposition listing the channel_api.md method calls in order, or write "(none — this is Transaction-API-only)".
```

---

## Output format

```
Spec Lint Report
Spec: <path>
Mode: <behavioral-block | protocol-bfm>
Files inspected: <count>

Violations found: <total>
  LINT-001 (broken xref):                  <count>
  LINT-002 (untracked TODO):               <count>
  LINT-003 (Feature/Testpoint map):        <count>
  LINT-004 (register casing):              <count>
  LINT-005 (port casing):                  <count>
  LINT-006 (reset value format):           <count>
  LINT-007 (missing SVG):                  <count>
  LINT-BFM-001 (wire-set parity):          <count> (BFM mode only)
  LINT-BFM-002 (rule ID format):           <count> (BFM mode only)
  LINT-BFM-003 (rule completeness):        <count> (BFM mode only)
  LINT-BFM-004 (channel cross-ref):        <count> (BFM mode only)
  LINT-BFM-005 (txn↔chan API xref):        <count> (BFM mode only)

(LINT-BFM-* lines are omitted entirely in behavioral-block mode reports.)

---

Details:

[LINT-001] registers.md:42
  Reference: "see Theory of Operation §5.2"
  Problem: target section not found.
  Suggested fix: ...

[LINT-BFM-001] pin_level_reset.md
  ...

(continue for each violation)

---

Summary:
  - <N> blocking violations (LINT-001, LINT-006, LINT-BFM-001, LINT-BFM-004 typically): fix before proceeding.
  - <N> hygiene violations (LINT-002, LINT-004, LINT-005, LINT-BFM-002, LINT-BFM-003, LINT-BFM-005): fix before D1 sign-off.
  - <N> coverage gaps (LINT-003): triage with DV lead.
```

## Constraints

- **Read-only.** This command does not edit the spec. If the user wants edits, they invoke them in conversation after reviewing the report.
- **Be precise about line numbers**. False positives are worse than missed positives — they erode trust in the lint.
- **Don't conflate with `/spec-status`**. Lint catches mechanical inconsistency; `/spec-status` checks gate completeness. They are complementary.
- If the spec is small and all checks pass, output `No violations.` and stop. A clean lint is a real result and shouldn't be padded with prose.

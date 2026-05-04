# Process: Stage Gates

Hardware specs have a defined completion level. This document defines the gates and their entry criteria. Adapted from the OpenTitan project signoff checklist, with simplifications for general use.

The author and reviewer should agree on the current stage and not advance until every box is checked or explicitly waived (with a rationale and an issue link).

---

## D0 — Concept

**Definition:** The block has been proposed. The shape is rough.

**Exit criteria (transitioning to D1 work):**

- [ ] One-sentence description exists.
- [ ] At least three Features bullets are written, even if rough.
- [ ] Top-level bus interface is named (e.g., "AXI4-Lite slave at 32-bit").
- [ ] Clock and reset domains are roughly identified.
- [ ] A skeleton README.md exists (using the summary template).
- [ ] Skeleton files for `theory_of_operation.md`, `programmers_guide.md`, `interfaces.md`, `registers.md`, `dv/plan.md` exist with section headers and `TODO(designer):` markers.

**What can stay TBD at D0:** internal block diagram, FSM details, register contents beyond top-level CTRL/STATUS, performance targets, full feature list.

---

## D1 — Spec ~90% complete

**Definition:** The spec is sufficient for a verification engineer to start writing the testbench and for an implementer to start writing RTL stubs.

**Exit criteria (transitioning to D2 work):**

### README.md
- [ ] Overview, Features (final list), Description, Compatibility all written.
- [ ] No TODO markers remain in this file.

### theory_of_operation.md
- [ ] Block diagram exists (SVG or ASCII).
- [ ] Datapath section walks every transformation stage.
- [ ] Every FSM has a state list and transition table.
- [ ] Reset behavior is enumerated for every register and FSM.
- [ ] Clock domains and CDC paths are identified, even if the block is single-domain ("none" stated explicitly).
- [ ] Error and fault handling section names every error condition and the block's reaction to each.
- [ ] Performance commitments stated, or "no performance commitment" explicitly stated.

### programmers_guide.md
- [ ] Initialization sequence is concrete and references real registers.
- [ ] At least one Use case is written end-to-end with code fragment.
- [ ] Error handling section pairs every error from theory_of_operation.md with a software response.
- [ ] Interrupt handling section covers every IRQ source.
- [ ] Register-accesses-during-operation section is written.

### interfaces.md
- [ ] Parameters table complete with type, default, constraints.
- [ ] Clocks and resets tables complete.
- [ ] Each bus interface has a sub-section with protocol, role, widths, deviations.
- [ ] Every sideband, IRQ, alert, inter-IP signal listed in a table.
- [ ] Naming convention stated at top of file.

### registers.md
- [ ] Register map table lists every register.
- [ ] Each register has a sub-section with field-by-field decomposition.
- [ ] Reset value of every register and field is stated explicitly.
- [ ] Reserved-bit policy stated at top.
- [ ] Every register cross-references theory_of_operation.md for behavior.

### dv/plan.md
- [ ] Verification scope paragraph written.
- [ ] Testpoints exist; every README.md feature maps to at least one.
- [ ] Coverage model named (covergroup names listed; bins can still be roughly defined).
- [ ] ABV/FPV strategy stated, even if "no FPV planned."

### Cross-document
- [ ] Reader test (see `reader_test.md`) has been run with at least 8 questions; gaps fixed.
- [ ] No `TODO(designer):` markers without a tracking issue link.

**What can stay TBD at D1:** exact test counts, final coverage numbers, post-synthesis timing analysis, exhaustive security countermeasure (sec_cm) list (if security-critical), final IP version number.

---

## D2 — RTL functional, spec frozen

**Definition:** RTL exists, compiles, elaborates, and is wired up at the top level. The spec is frozen against further changes except for clarification.

**Exit criteria (transitioning to D3 work):**

- [ ] All D1 criteria still met.
- [ ] RTL `.sv` exists for the unit and matches the interfaces.md signal list exactly.
- [ ] RTL compiles and elaborates cleanly with no errors.
- [ ] Top-level instantiation does not propagate X through the configured bus interface.
- [ ] Connecting clocks and resets do not break top-level functionality.
- [ ] Sec_cm list (if security-critical) is exhaustive and reflected in dv/plan.md.
- [ ] DV testbench compiles and a smoke test passes.
- [ ] Any spec change since D1 is recorded in a changelog or commit history with reviewer sign-off.

**What can stay TBD at D2:** full coverage closure, sign-off-grade lint, formal property completion.

---

## D3 — Sign-off

**Definition:** The block is ready for integration into a tape-out candidate.

**Exit criteria:**

- [ ] All D2 criteria still met.
- [ ] DV plan reaches V3 (100% functional coverage, ≥99% code coverage, signoff review complete).
- [ ] Lint passes at sign-off level.
- [ ] All security countermeasure assertions proven in FPV (if applicable).
- [ ] Spec reviewed and signed off by HW lead, DV lead, and SW lead.
- [ ] No open issues at severity "spec-blocking."

---

## Using the gates

When the user asks "is this spec done?", the answer is always "done for what stage?" Walk the checklist for the current stage. Report which boxes are not yet checked. Do not silently mark a box checked — if the author has not produced the artifact, the box stays unchecked.

When the user is at D0 and asks for a review, the only useful review is "what's missing for D1." Don't review D1 prose for D2-grade quality if the spec hasn't reached D1 yet.

A spec at D1 with one missing FSM transition table is **not** at D1. There is no "D1 minus." The author either supplies the missing artifact or explicitly waives it with a rationale.

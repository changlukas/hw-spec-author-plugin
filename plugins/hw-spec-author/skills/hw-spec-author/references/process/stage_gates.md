# Process: Stage Gates

Hardware specs have a defined completion level. This document defines the gates and their entry criteria. Adapted from the OpenTitan project signoff checklist, with simplifications for general use.

The author and reviewer should agree on the current stage and not advance until every box is checked or explicitly waived (with a rationale and an issue link).

## Checklist anchor convention

Every checklist item below carries a stable anchor in the form `(id: <stage>.<scope>.<short_name>)`. These anchors are the *contract* between this document and `WAIVERS.md`: when an item is waived, the waiver file uses the exact anchor as its `## H2` heading. Anchors are stable across edits — when an item's wording is clarified, keep the anchor unchanged. When an item is split or replaced, retire the old anchor and choose a new one rather than reusing.

Tools (`/spec-gate`, `/spec-status`) parse anchors from the `(id: ...)` token literally; do not derive them.

## Mode-conditional items

Some sub-sections below are tagged `**Applies when:** <mode>` at the section heading level. Tools must:
- Read `<spec_root>/MODE.md` to determine the spec's mode (default `behavioral-block` if absent).
- Skip items in sub-sections whose `**Applies when:**` clause does not match.
- Score skipped items as N/A (not ✗); they do not count toward or against gate completion.
- Apply BFM-only items (anchors `D1.bfm.*`, `D2.bfm.*`, `D3.bfm.*`) only when `mode: protocol-bfm`.

---

## D0 — Concept

**Definition:** The block has been proposed. The shape is rough.

**Exit criteria (transitioning to D1 work):**

- [ ] (id: D0.mode_recorded) `MODE.md` exists at spec root and declares `mode: behavioral-block` or `mode: protocol-bfm`.
- [ ] (id: D0.one_sentence_desc) One-sentence description exists.
- [ ] (id: D0.three_features) At least three Features bullets are written, even if rough.
- [ ] (id: D0.bus_named) Top-level bus / protocol interface is named (e.g., "AXI4-Lite slave at 32-bit", or for BFM mode: "AXI4-Lite slave BFM").
- [ ] (id: D0.clock_reset_named) Clock and reset domains are roughly identified.
- [ ] (id: D0.skeleton_readme) A skeleton README.md exists (using the summary template).
- [ ] (id: D0.skeleton_others) **Applies when:** `mode == behavioral-block`. Skeleton files for `theory_of_operation.md`, `programmers_guide.md`, `interfaces.md`, `registers.md`, `dv/plan.md` exist with section headers and `TODO(designer):` markers.
- [ ] (id: D0.bfm.skeleton_others) **Applies when:** `mode == protocol-bfm`. Skeleton files for `theory_of_operation.md`, `signal_interface.md`, `pin_level_reset.md`, `protocol_rules.md`, `channel_handshake.md`, `transaction_api.md`, `channel_api.md`, `active_passive_mode.md`, `dv/plan.md` exist with section headers and `TODO(designer):` markers.

**What can stay TBD at D0:** internal block diagram, FSM details, register contents beyond top-level CTRL/STATUS, performance targets, full feature list. (BFM mode: protocol rule rows beyond reset, channel handshake diagrams, full transaction/channel API method list.)

---

## D1 — Spec ~90% complete

**Definition:** The spec is sufficient for a verification engineer to start writing the testbench and for an implementer to start writing RTL stubs.

**Exit criteria (transitioning to D2 work):**

### README.md
- [ ] (id: D1.readme.complete) Overview, Features (final list), Description, Compatibility all written.
- [ ] (id: D1.readme.no_todos) No TODO markers remain in this file.

### theory_of_operation.md
- [ ] (id: D1.too.block_diagram) Block diagram exists (SVG or ASCII).
- [ ] (id: D1.too.datapath) Datapath section walks every transformation stage.
- [ ] (id: D1.too.fsm) Every FSM has a state list and transition table.
- [ ] (id: D1.too.reset_behavior) Reset behavior is enumerated for every register and FSM.
- [ ] (id: D1.too.cdc) Clock domains and CDC paths are identified, even if the block is single-domain ("none" stated explicitly).
- [ ] (id: D1.too.errors) Error and fault handling section names every error condition and the block's reaction to each.
- [ ] (id: D1.too.performance) Performance commitments stated, or "no performance commitment" explicitly stated.

### programmers_guide.md
**Applies when:** `mode == behavioral-block`.
- [ ] (id: D1.pg.init) Initialization sequence is concrete and references real registers.
- [ ] (id: D1.pg.use_case) At least one Use case is written end-to-end with code fragment.
- [ ] (id: D1.pg.errors) Error handling section pairs every error from theory_of_operation.md with a software response.
- [ ] (id: D1.pg.irqs) Interrupt handling section covers every IRQ source.
- [ ] (id: D1.pg.access_during_op) Register-accesses-during-operation section is written.

### interfaces.md
**Applies when:** `mode == behavioral-block`. In `protocol-bfm` mode, the analogous file is `signal_interface.md`; see `D1.bfm.signal_interface` below.
- [ ] (id: D1.if.parameters) Parameters table complete with type, default, constraints.
- [ ] (id: D1.if.clocks_resets) Clocks and resets tables complete.
- [ ] (id: D1.if.bus_subsections) Each bus interface has a sub-section with protocol, role, widths, deviations.
- [ ] (id: D1.if.sideband_irq_alert) Every sideband, IRQ, alert, inter-IP signal listed in a table.
- [ ] (id: D1.if.naming_convention) Naming convention stated at top of file.

### registers.md
**Applies when:** `mode == behavioral-block`. In `protocol-bfm` mode, `registers.md` is optional; if present (BFM has a software-visible CSR), all D1.reg.* items apply. If absent, items are N/A.
- [ ] (id: D1.reg.map) Register map table lists every register.
- [ ] (id: D1.reg.field_decomposition) Each register has a sub-section with field-by-field decomposition.
- [ ] (id: D1.reg.reset_values) Reset value of every register and field is stated explicitly.
- [ ] (id: D1.reg.reserved_policy) Reserved-bit policy stated at top.
- [ ] (id: D1.reg.xref_too) Every register cross-references theory_of_operation.md for behavior.

### theory_of_operation.md (BFM mode)
**Applies when:** `mode == protocol-bfm`.
- [ ] (id: D1.bfm.too) `## BFM internal architecture` section populated (driver, monitor, sequencer, configuration store, implementation-specific algorithms, reset entry sequencing, performance commitments). If `MODE.md` declares `has-rtl-counterpart: yes`, the `## RTL internal architecture` section is also populated (RTL block structure, pipeline / timing, RTL reset behavior, RTL-vs-BFM behavioral equivalence sub-section). If `has-rtl-counterpart: no` (or absent), the RTL section is omitted entirely.

### signal_interface.md
**Applies when:** `mode == protocol-bfm`.
- [ ] (id: D1.bfm.signal_interface) Wire table complete (every wire: signal, direction-from-BFM, width, active level, sample edge, reset-value link, optional-in-protocol, BFM-supports). Channel grouping declared. Parameters table complete with constraints. Optional protocol features explicitly listed as supported / not-supported.

### pin_level_reset.md
**Applies when:** `mode == protocol-bfm`.
- [ ] (id: D1.bfm.pin_reset) During-reset table and after-reset table both list every wire from `signal_interface.md` (LINT-BFM-001 cross-checks). Reset assertion duration stated. Reset entry sequencing described.

### protocol_rules.md
**Applies when:** `mode == protocol-bfm`.
- [ ] (id: D1.bfm.protocol_rules) Reset rules sub-section present. Per-channel rules sub-section present for every channel listed in `signal_interface.md` §Channel grouping. Cross-channel rules sub-section present (or "(none)" for protocols without cross-channel constraints). Configuration-knob rules sub-section present for every knob in `transaction_api.md` and `active_passive_mode.md`. Every rule has unique ID matching `<PROTO>_<ROLE>_<CHANNEL>_<NAME>`, non-empty Condition + Required behavior, and a Severity (FAIL or RECOMMEND). ARM SVA equivalent column populated for every row (rule ID or `(none)`).

### channel_handshake.md
**Applies when:** `mode == protocol-bfm`.
- [ ] (id: D1.bfm.handshake) Per-transaction-type sub-section for every transaction type the protocol supports (typically write, read, plus protocol-specific types). Each sub-section has dependency diagram (Mermaid or equivalent), textual dependency list, deadlock-avoidance commentary (or explicit "(none)"). Cross-transaction-type dependencies stated (or "(none)"). Out-of-order completion policy stated.

### transaction_api.md
**Applies when:** `mode == protocol-bfm`.
- [ ] (id: D1.bfm.txn_api) API conventions stated (pseudocode style, naming, error reporting, blocking discipline). Method index covers every method. Each method has all five sub-sections (Signature, Preconditions, Side effects, Return value, Error modes) plus an Equivalent channel API decomposition (or explicit "(none)"). Behavior under reset described. Concurrency rules stated.

### channel_api.md
**Applies when:** `mode == protocol-bfm`.
- [ ] (id: D1.bfm.channel_api) "When to use this API" criteria stated. Channel state machine diagrammed. Per-channel API table complete for every channel in `signal_interface.md` (Method, Signature, Legal in state, Side effect, Returns). Ordering constraints with Transaction API enumerated (forbidden mixing, permitted mixing, detection mechanism). Behavior under reset described. Concurrency rules stated.

### active_passive_mode.md
**Applies when:** `mode == protocol-bfm`.
- [ ] (id: D1.bfm.active_passive) Capability table complete (every capability gets a yes/no per mode). Mode switch knob named with type, default, and effect-of-switching for both directions. Common testbench setups described. Reset interaction stated.

### dv/plan.md
- [ ] (id: D1.dv.scope) Verification scope paragraph written.
- [ ] (id: D1.dv.testpoints) Testpoints exist; every README.md feature maps to at least one.
- [ ] (id: D1.dv.coverage_model) Coverage model named (covergroup names listed; bins can still be roughly defined).
- [ ] (id: D1.dv.abv_fpv) ABV/FPV strategy stated, even if "no FPV planned."

### Cross-document
- [ ] (id: D1.cross.reader_test) **Applies when:** `mode == behavioral-block`. Reader test using `reader_test.md` bank has been run with at least 8 questions; gaps fixed.
- [ ] (id: D1.cross.bfm_reader_test) **Applies when:** `mode == protocol-bfm`. Reader test using `bfm_reader_test_bank.md` has been run with at least 8 questions covering all 8 universal-BFM categories applicable to the protocol; gaps fixed.
- [ ] (id: D1.cross.todos_tracked) No `TODO(designer):` markers without a tracking issue link.

**What can stay TBD at D1:** exact test counts, final coverage numbers, post-synthesis timing analysis, exhaustive security countermeasure (sec_cm) list (if security-critical), final IP version number.

---

## D2 — RTL functional, spec frozen

**Definition:** RTL exists, compiles, elaborates, and is wired up at the top level. The spec is frozen against further changes except for clarification.

**Exit criteria (transitioning to D3 work):**

- [ ] (id: D2.d1_still_met) All applicable D1 criteria still met (mode-conditional items respected per `MODE.md`).
- [ ] (id: D2.rtl_matches_iface) **Applies when:** `mode == behavioral-block`. RTL `.sv` exists for the unit and matches the interfaces.md signal list exactly.
- [ ] (id: D2.rtl_compiles) **Applies when:** `mode == behavioral-block`. RTL compiles and elaborates cleanly with no errors.
- [ ] (id: D2.no_x_propagation) **Applies when:** `mode == behavioral-block`. Top-level instantiation does not propagate X through the configured bus interface.
- [ ] (id: D2.toplevel_clk_rst) Connecting clocks and resets do not break top-level functionality.
- [ ] (id: D2.sec_cm_list) Sec_cm list (if security-critical) is exhaustive and reflected in dv/plan.md.
- [ ] (id: D2.tb_smoke) DV testbench compiles and a smoke test passes.
- [ ] (id: D2.changelog) Any spec change since D1 is recorded in a changelog or commit history with reviewer sign-off.

### BFM-specific D2 items
**Applies when:** `mode == protocol-bfm`.
- [ ] (id: D2.bfm.implementation) BFM implementation (in chosen language: SystemVerilog/UVM, SystemC, C++ with DPI-C, etc.) compiles and links against a representative testbench.
- [ ] (id: D2.bfm.self_check) BFM passes its own protocol_rules.md assertions when driven against a known-good DUT or against itself in loopback.
- [ ] (id: D2.bfm.coverage_plan) Coverage hooks defined in `dv/plan.md` for every protocol-rule ID; covergroup names enumerate the protocol-rule cross-points.

**What can stay TBD at D2:** full coverage closure, sign-off-grade lint, formal property completion.

---

## D3 — Sign-off

**Definition:** The block is ready for integration into a tape-out candidate.

**Exit criteria:**

- [ ] (id: D3.d2_still_met) All applicable D2 criteria still met.
- [ ] (id: D3.dv_v3) DV plan reaches V3 (100% functional coverage, ≥99% code coverage, signoff review complete).
- [ ] (id: D3.lint_signoff) **Applies when:** `mode == behavioral-block`. RTL lint passes at sign-off level.
- [ ] (id: D3.fpv_sec_cm) **Applies when:** `mode == behavioral-block`. All security countermeasure assertions proven in FPV (if applicable).
- [ ] (id: D3.signoff_leads) Spec reviewed and signed off by HW lead, DV lead, and SW lead.
- [ ] (id: D3.no_blocking_issues) No open issues at severity "spec-blocking."

### BFM-specific D3 items
**Applies when:** `mode == protocol-bfm`.
- [ ] (id: D3.bfm.coverage_closed) 100% protocol-rule coverage closed in regression — every rule ID in `protocol_rules.md` has a corresponding cover-property hit at least once across the full test suite.
- [ ] (id: D3.bfm.cross_check) BFM cross-checked against an independent reference (commercial VIP, golden RTL, or formal proof). Mismatches resolved or documented as known divergence.

---

## Using the gates

When the user asks "is this spec done?", the answer is always "done for what stage?" Walk the checklist for the current stage. Report which boxes are not yet checked. Do not silently mark a box checked — if the author has not produced the artifact, the box stays unchecked.

When the user is at D0 and asks for a review, the only useful review is "what's missing for D1." Don't review D1 prose for D2-grade quality if the spec hasn't reached D1 yet.

A spec at D1 with one missing FSM transition table is **not** at D1. There is no "D1 minus." The author either supplies the missing artifact or explicitly waives it with a rationale.

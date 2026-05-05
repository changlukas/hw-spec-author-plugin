# BFM Mode — Design Draft

**Status:** Design draft. No implementation work yet.
**Target plugin:** `hw-spec-author`
**Proposed addition:** Optional "protocol-bfm" mode alongside the existing default mode.

---

## Table of Contents

1. [Motivation](#1-motivation)
2. [Background — what we already have](#2-background--what-we-already-have)
3. [Problem statement](#3-problem-statement)
4. [Why a new mode, not a new plugin](#4-why-a-new-mode-not-a-new-plugin)
5. [Scope decisions and non-goals](#5-scope-decisions-and-non-goals)
6. [Reference sources](#6-reference-sources)
7. [Design — what gets added to the plugin](#7-design--what-gets-added-to-the-plugin)
8. [User-facing surface](#8-user-facing-surface)
9. [Worked example](#9-worked-example)
10. [Validation and acceptance criteria](#10-validation-and-acceptance-criteria)
11. [Open questions](#11-open-questions)
12. [Future extensions explicitly out of scope](#12-future-extensions-explicitly-out-of-scope)

---

## 1. Motivation

The plugin's user (hereafter "the designer") regularly writes two distinct kinds of artifact in their daily IC design work:

- **Behavioral IP blocks** (counters, peripherals, controllers, accelerators) — what the existing plugin handles well
- **Protocol-bound interface models** (BFMs / VIPs / transactors) — sitting at the cycle-accurate boundary between RTL and a higher-level test environment

The second category came up explicitly during workflow discussion. The designer's stated use cases are:

1. **Performance modeling** — exploring throughput / latency / arbitration / queue depth before RTL is finalized
2. **Acting as a VIP at the interface level** — example: an AXI slave C model verifying an AXI master RTL by driving the actual AXI pins

Use case 2 is fundamentally different from "functional reference model used as DV scoreboard" (which is what UT/LT/AT TLM would solve). It requires **cycle-accurate behavior at the pin boundary** — the C model must respect AXI handshake rules, drive `AWREADY` legally, track outstanding IDs, and so on.

The existing `hw-spec-author` plugin produces specs that are:
- Excellent for behavioral block design (six-file OpenTitan layout)
- Sufficient for UT-level reference models
- **Insufficient for writing BFMs** — missing protocol-cycle-level rules, channel handshake dependencies, pin-level reset behavior, and protocol-API surface design

This draft documents what to add, why, and where, so the work is ready to execute when needed.

---

## 2. Background — what we already have

### 2.1 The existing plugin (v0.1.0)

```
plugins/hw-spec-author/
├── skills/hw-spec-author/
│   ├── SKILL.md                          # 4-phase workflow
│   └── references/
│       ├── templates/
│       │   ├── 01_summary.md
│       │   ├── 02_theory_of_operation.md   # block diagram, datapath, FSM, reset, errors
│       │   ├── 03_programmers_guide.md
│       │   ├── 04_interfaces.md            # ports, parameters, bus interface
│       │   ├── 05_registers.md
│       │   └── 06_dv_plan.md
│       └── process/
│           ├── stage_gates.md              # D0 → D3
│           ├── reader_test.md              # ambiguity detection
│           └── writing_principles.md
├── agents/
│   └── spec-reader.md                    # isolated subagent for reader test
├── commands/                             # /spec-init, /spec-status, /spec-review,
│                                         # /spec-lint, /spec-gate, /spec-help
└── examples/wctmr/                       # 64-bit timer reference example
```

### 2.2 Where the existing structure already covers BFM needs

The OpenTitan-derived six-file layout is more general than it first appears. Mapping BFM concerns onto existing files:

| BFM concern                          | Existing file                                           |
|--------------------------------------|---------------------------------------------------------|
| Pin / wire list                      | `interfaces.md` (existing schema sufficient)            |
| Configuration parameters             | `interfaces.md` parameters table                        |
| Reset behavior                       | `theory_of_operation.md` resets section (partial)       |
| Test plan                            | `dv/plan.md`                                            |

### 2.3 Where the existing structure is silent on BFM needs

| BFM concern                                      | Existing coverage                          |
|--------------------------------------------------|--------------------------------------------|
| Protocol cycle-level rules (per-channel)         | None                                       |
| Channel handshake dependencies                   | None                                       |
| Pin-level reset behavior (every wire's value during reset) | Partial — only register-level reset |
| Outstanding transaction tracking / ID rules      | None                                       |
| Transaction API (high-level test-facing methods) | None                                       |
| Channel API (phase-level control methods)        | None                                       |
| Active vs Passive mode                           | None                                       |
| Coverage hooks at protocol-event level           | None                                       |

These are the gaps BFM mode fills.

---

## 3. Problem statement

A user who runs `/spec-init my_axi_slave_bfm` today, expecting to write a spec for an AXI slave BFM, would receive:

- A skeleton biased toward "internal block design" (datapath, FSM, registers as software-visible state)
- No prompts for protocol rules
- No prompts for channel API design
- A reader-test bank with no protocol-specific questions
- A `/spec-lint` that does not check protocol-rule formatting

The user could still write a BFM spec by hand by treating the existing files loosely, but **the plugin would not enforce or guide the BFM-specific structure** — defeating the plugin's value proposition (audience separation, single-source-of-truth, stage gates, reader-testing).

The work in this document fixes that gap.

---

## 4. Why a new mode, not a new plugin

Three options were considered:

| Option | Description | Decision |
|---|---|---|
| **A — New plugin `bfm-spec-author`** | Separate plugin, separate marketplace entry, separate command set | ❌ Rejected |
| **B — Mode flag in existing plugin** | `/spec-init` asks "IP class": `behavioral-block` (default) or `protocol-bfm` | ✅ Chosen |
| **C — Detect from user description** | Plugin auto-detects from interview answers | ❌ Rejected for v1 |

**Reasoning for B:**

- BFM specs and IP specs share ~70% of structure (six-file layout, audience separation, stage gates, reader-test methodology). Splitting into two plugins forces duplication.
- The designer's real-world IPs (NoC NI, DMA engine, bus bridge) are **simultaneously** behavioral blocks AND have BFM-style interfaces. One IP needs both kinds of spec — same plugin should handle both.
- Sharing `process/` files (stage_gates, writing_principles) means improvements propagate across both modes.
- The `spec-reader` subagent works identically for both modes.

**Reasoning against C:**

- Auto-detection is unreliable for v1 — a "DMA engine" could be either a behavioral block (with descriptor processing logic) or treated as a BFM (when used as a stimulus generator). The designer's intent is the disambiguator, not the IP name.
- Auto-detection also masks the structural difference from the user, which works against the plugin's transparency principle.

**v1 explicit, v2 may add suggested defaults** based on keywords if useful.

---

## 5. Scope decisions and non-goals

### In scope

- Add `protocol-bfm` mode to `/spec-init` interview
- Add BFM-specific template sections in ToO and interfaces
- Add new optional template files for protocol rules and APIs
- Extend reader-test bank with BFM-specific questions
- Extend `/spec-lint` with BFM-specific rules
- Add a `bfm-` prefix worked example (likely a small AXI-Lite slave BFM)
- Update SKILL.md and stage gates to handle the mode-conditional checks

### Out of scope (this draft)

- **CA reference models that replace RTL** (gem5-style cycle simulators, HLS-source C++) — different problem, different methodology
- **TLM AT model spec extension** — separate concern, separate design draft if/when needed
- **Mixed-mode specs** (BFM and behavioral block in one spec). v1 picks one mode per spec. Mixed concerns are handled by writing two specs that cross-reference each other.
- **Auto-detection from natural language** — explicit mode selection only in v1
- **Generation of executable BFM stubs** — plugin produces spec only, not code
- **Protocol-specific knowledge baked into the plugin** (i.e., the plugin does not know AXI rules; it provides the structure for the user to write them in)

The last point is important: **this plugin remains protocol-agnostic**. It teaches "how to write a BFM spec," not "what AXI rules are." The user provides the protocol knowledge.

---

## 6. Reference sources

The design draws on four established lineages, identified during research. Each contributes a specific structural concept:

### 6.1 ARM AMBA Protocol Checker User Guides (DUI 0534B for AXI4)

**Contribution:** Rule-by-rule table format. Each protocol channel gets a dedicated table where every row is one cycle-level rule with an ID, a condition, the required behavior, and a corresponding assertion name.

**Why it matters:** This is the most directly imitable structural pattern for the new "Protocol Rules" template section. Each rule maps 1:1 to an SVA assertion when implemented, and 1:1 to a reader-test question when reviewed.

**Example row pattern (reproduced structurally, not the actual rule text):**

| ID | Channel | Condition | Required behavior |
|---|---|---|---|
| AXI4_ERRM_AWVALID_STABLE | AW | AWVALID rises | Must remain HIGH until AWREADY observed HIGH on a rising clock |

### 6.2 ARM AMBA AXI Specification (IHI0022, §A2)

**Contribution:** Channel handshake dependency diagrams and cross-channel ordering rules. The `Valid-Ready transport`, `Channel handshake`, and `Dependencies between channel handshake signals` sections capture the inter-channel rules that single-channel rule tables miss (e.g., "BVALID can only assert after WLAST has been observed").

**Why it matters:** Drives the design of the `Channel Handshake & Dependencies` template section. These are the rules that catch deadlocks and out-of-spec masters.

### 6.3 UVM agent architecture (driver / monitor / sequencer triplet)

**Contribution:** Internal architectural decomposition consensus. Whether the BFM is implemented in SV/UVM, SystemC, or C++ with DPI-C, the internal three-way split into stimulus generation (sequencer), pin-level driving (driver), and pin-level observation (monitor) is universal. Active mode includes all three; passive mode is monitor-only.

**Why it matters:** Drives the `Active vs Passive Mode` template section and informs the API layering. Even though the plugin produces a *spec*, not code, the spec must commit to this internal architecture so reviewers and implementers can map the design to one of the standard frameworks.

### 6.4 OSVVM and Aldec/Cadence AXI BFM (three-layer API)

**Contribution:** Explicit separation of:
- **Signal interface** — the wire bundle between BFM and RTL
- **Channel API** — phase-level test-facing methods (per-channel control, allowing out-of-order / interleave use cases)
- **Function (Transaction) API** — high-level test-facing methods (`single_write`, `burst_read`)

OSVVM articulates this as `Transaction API` + `Transaction Interface` separation. Aldec's writeup of the Cadence-for-Xilinx AXI BFM splits it into the three layers above.

**Why it matters:** Gives the structure for the new `Transaction API` and `Channel API` template sections, and clarifies why both are needed (not redundant): function API is what ~95% of tests use, channel API is what the remaining 5% (out-of-order, interleave, fault-injection) need.

---

## 7. Design — what gets added to the plugin

This is the core deliverable list. Each item is independent; they can be implemented in any order.

### 7.1 SKILL.md changes

Add a new section near the top:

```markdown
## IP class (mode)

Two modes are supported. The mode is set during /spec-init and recorded in
the spec root's MODE.md file.

- **behavioral-block** (default) — internal IP designs (peripherals, accelerators,
  controllers, blocks with software-visible registers and on-chip behavior)
- **protocol-bfm** — bus functional models, verification IPs, transactors,
  protocol-bound interface models

Both modes share the same six-file layout, stage gates, and reader-test
methodology. They differ in:
- Which template sections are required vs. optional
- Which reader-test bank applies
- Which lint rules apply
- The worked example referenced

If unsure which mode to pick: BFM mode is for anything whose primary purpose
is to talk to RTL via a defined wire-level protocol. Everything else is
behavioral-block.
```

Update the workflow section to note that Phase 2 templates branch by mode.

### 7.2 New: `MODE.md` file in the spec root

Generated by `/spec-init`. Contents:

```markdown
# Spec Mode

mode: <behavioral-block | protocol-bfm>
created: <ISO date>
spec-author-plugin-version: <version>
```

This is the single source of truth for mode. All plugin commands read it. If absent, default is `behavioral-block` (back-compat with existing specs).

### 7.3 New template files for `protocol-bfm` mode

Added under `references/templates/bfm/`:

```
references/templates/bfm/
├── 02b_protocol_rules.md             # NEW — rule tables per channel
├── 02c_channel_handshake.md          # NEW — cross-channel dependencies
├── 02d_pin_level_reset.md            # NEW — every wire's reset behavior
├── 03b_transaction_api.md            # NEW — function-level API surface
├── 03c_channel_api.md                # NEW — phase-level API surface
├── 03d_active_passive_mode.md        # NEW — agent role configuration
└── 04b_signal_interface.md           # NEW — strengthened wire table
```

Each is a standalone template file analogous to `02_theory_of_operation.md` etc., with required structure, writing rules, anti-patterns, and a mini-example.

#### 7.3.1 `02b_protocol_rules.md` (sketch)

```markdown
# Template: protocol_rules.md

**Primary audience:** BFM implementer, RTL designer (DUT side), DV engineer.
**Goal:** Every protocol-cycle-level rule the BFM must enforce or respect is
listed exactly once, with a stable ID, mapped to its corresponding assertion.

## Required structure

One subsection per channel or rule class. For each subsection, a table with
columns: ID | Condition | Required behavior | Severity (FAIL / WARN / RECOMMEND).

ID format: <protocol>_<role>_<channel>_<short_name>
Example:    AXI4_ERRM_AW_VALID_STABLE

Channels typical for AXI: AW, W, B, AR, R.
Other rule classes: reset, low-power, exclusive-access, X-checks,
internal-logic, ID-rules.

## Writing rules

- One rule per row. No "and / or" splits.
- "Required behavior" is testable. If you can't write an assertion for it,
  rewrite the rule until you can.
- Severity: FAIL = protocol violation (BFM must reject or signal error);
  WARN = recommended-but-not-mandatory; RECOMMEND = quality-of-implementation.

## Anti-patterns

- Conditional rules that depend on configuration knobs without naming the
  knob ("If outstanding count is large enough, ..." — name the knob).
- Rule descriptions that say "see the spec" instead of stating the rule.
- Combining two unrelated channels' rules in the same row.
```

#### 7.3.2 `02c_channel_handshake.md` (sketch)

```markdown
# Template: channel_handshake.md

**Primary audience:** BFM implementer.
**Goal:** Cross-channel dependencies and ordering rules are explicit, with
diagrams.

## Required structure

For each transaction type (e.g., write, read), provide:
1. A dependency diagram (Mermaid `flowchart LR` showing signal-to-signal
   dependency arrows)
2. A textual list of dependencies, one per line, format:
   "<signal A> assertion may/must precede/follow <signal B> assertion"
3. Deadlock-avoidance commentary if the channel pair has known deadlock modes

## Writing rules

- Use AXI's "single-headed = may, double-headed = must" arrow convention.
- Every dependency must be testable: "BVALID rises before WLAST observed"
  is a runtime checkable predicate.
```

#### 7.3.3 `02d_pin_level_reset.md` (sketch)

```markdown
# Template: pin_level_reset.md

**Primary audience:** BFM implementer, RTL designer.
**Goal:** Every wire's value during and immediately after reset is specified.
No "default" or "implementation-defined" allowed at this level.

## Required structure

Two tables:
1. **During reset (rst_n LOW)**: every wire, direction, value
2. **After reset (first cycle rst_n HIGH)**: every wire, direction, value or
   "may take any value"

## Writing rules

- "X" or "Z" is acceptable as an explicit value.
- "May take any value" is acceptable but must be deliberate (commented).
- The set of wires must match interfaces.md's signal interface table exactly.
  /spec-lint flags mismatches.
```

#### 7.3.4 `03b_transaction_api.md` (sketch)

```markdown
# Template: transaction_api.md

**Primary audience:** Test author (driver code / sequence writer).
**Goal:** The high-level test-facing API. ~95% of tests use only this.

## Required structure

One subsection per API method. For each method:
- Signature (pseudocode, language-agnostic)
- Preconditions
- Side effects (which wires get driven, in what order)
- Return values
- Error modes
- Example call

Methods to consider for a typical bus BFM:
- single_write(addr, data) → resp
- single_read(addr) → (data, resp)
- burst_write(addr, len, data[]) → resp
- burst_read(addr, len) → (data[], resp)
- set_response_delay(min_cycles, max_cycles)
- set_outstanding_limit(n)
- inject_fault(<fault type>)
```

#### 7.3.5 `03c_channel_api.md` (sketch)

```markdown
# Template: channel_api.md

**Primary audience:** Advanced test author (out-of-order, interleave,
fault injection, protocol corner cases).
**Goal:** Phase-level fine-grained control. ~5% of tests use this directly,
but they need it.

## Required structure

One subsection per channel. For each channel, list the per-phase tasks:
- begin_phase(<channel>, <transaction id>, <fields>)
- assert_valid(<channel>, <transaction id>)
- wait_for_ready(<channel>, <transaction id>)
- end_phase(<channel>, <transaction id>)

Specify ordering constraints relative to the Function API:
- Channel API calls and Function API calls cannot be mixed for the same
  transaction (or: can be mixed under <these> conditions).
```

#### 7.3.6 `03d_active_passive_mode.md` (sketch)

```markdown
# Template: active_passive_mode.md

**Primary audience:** Testbench integrator.
**Goal:** Same BFM, two roles — explicitly defined.

## Required structure

A table with columns: Capability | Active | Passive

Capabilities to address:
- Drives outputs to RTL (yes / no)
- Samples inputs from RTL (yes / yes)
- Reconstructs transactions from observed pin activity (no / yes)
- Generates stimulus from sequencer (yes / no)
- Reports protocol violations (yes / yes)
- Provides coverage hooks (yes / yes)

Plus: configuration knob name and value to switch modes.
```

#### 7.3.7 `04b_signal_interface.md` (strengthened wire table)

```markdown
# Template: signal_interface.md (BFM mode replacement for interfaces.md
bus interface subsection)

**Primary audience:** Anyone integrating the BFM at RTL boundary.
**Goal:** Every wire, fully specified.

## Required structure

A single table, every wire listed. Columns:
- Signal name (matching protocol spec exactly)
- Direction (from BFM perspective)
- Width (bits, parameterized if applicable)
- Active level (HIGH / LOW / N/A)
- Sample edge (rising clk, falling clk, async)
- Reset value (cross-references pin_level_reset.md)
- Optional in protocol (yes / no)
- BFM supports (yes / no / configurable)
- Notes
```

### 7.4 Mode-conditional template selection in SKILL.md

The recommended writing order changes by mode:

**behavioral-block (existing):**
1. README → interfaces → registers → ToO → programmer's guide → DV plan

**protocol-bfm (new):**
1. README → signal_interface → pin_level_reset → protocol_rules → channel_handshake → transaction_api → channel_api → active_passive_mode → DV plan

Note: `registers.md` is **omitted** for protocol-bfm mode — BFMs typically have no software-visible register file (configuration is via testbench API, not memory-mapped registers). If a specific BFM does have registers, they go under transaction_api or as a separate file the user adds.

### 7.5 New process file: `references/process/bfm_reader_test_bank.md`

Adds BFM-specific reader-test questions to the universal bank. Sketched questions:

**Protocol rules**
- For each channel listed in interfaces.md, is there a corresponding rule
  table in protocol_rules.md?
- Does every rule have a unique ID matching the prescribed format?
- Does every rule have a testable "required behavior" statement?

**Handshake dependencies**
- For each transaction type, is the dependency diagram present?
- Are deadlock-prone channel pairs explicitly called out?

**Pin-level reset**
- Does the reset table include every wire from signal_interface.md?
- Are post-reset constraints stated for both BFM-driven and BFM-observed wires?

**API completeness**
- Can every common test scenario (single read, single write, burst,
  back-pressure, error injection) be expressed using only Transaction API?
- Are Channel API methods sufficient for out-of-order / interleave?

**Mode coverage**
- Is the active-vs-passive switch a single configuration knob?
- Does passive mode definitively not drive any wire?

**Outstanding tracking**
- What is the maximum outstanding count the BFM tracks?
- What happens if the test exceeds this limit?

**ID rules (for ID-bearing protocols)**
- Does the BFM enforce same-ID ordering at the response side?
- Does the BFM allow different-IDs to complete out of order?

### 7.6 `/spec-lint` extensions for BFM mode

Add lint rules conditional on `MODE.md`:

| Lint rule | Applies to | Description |
|---|---|---|
| LINT-BFM-001 | protocol-bfm | Every wire in signal_interface.md must appear in pin_level_reset.md, both tables |
| LINT-BFM-002 | protocol-bfm | Every protocol-rule ID must match format `<proto>_<role>_<channel>_<name>` |
| LINT-BFM-003 | protocol-bfm | Every protocol-rule must have non-empty "Condition" and "Required behavior" cells |
| LINT-BFM-004 | protocol-bfm | Every channel referenced in protocol_rules.md must exist in signal_interface.md |
| LINT-BFM-005 | protocol-bfm | Every Transaction API method must reference at least one Channel API equivalent or note "no channel-level decomposition" |

### 7.7 Stage gate adjustments

D1 / D2 / D3 checklists already exist in `process/stage_gates.md`. Add mode-conditional rows.

For protocol-bfm at D1 exit:
- [ ] (id: D1.bfm.protocol_rules) Protocol rule tables exist for every channel in signal_interface.md
- [ ] (id: D1.bfm.handshake) Channel handshake dependencies documented for every transaction type
- [ ] (id: D1.bfm.pin_reset) Pin-level reset behavior table complete
- [ ] (id: D1.bfm.txn_api) Transaction API surface defined with at least one example per method

For protocol-bfm at D2 exit:
- [ ] (id: D2.bfm.implementation) BFM implementation passes self-checks against its own protocol rules
- [ ] (id: D2.bfm.coverage_plan) Coverage hooks defined for every protocol-rule ID

For protocol-bfm at D3 exit:
- [ ] (id: D3.bfm.coverage_closed) 100% protocol-rule coverage closed in regression
- [ ] (id: D3.bfm.cross_check) BFM cross-checked against an independent reference (commercial VIP, golden RTL, formal proof)

### 7.8 New worked example: `examples/axi_lite_slave_bfm/`

A complete D1-grade BFM spec for AXI-Lite slave. Why AXI-Lite:
- Smallest non-trivial protocol with all the structural ingredients (channels, handshakes, reset, ordering)
- Subset of AXI4, so the example transfers learning to bigger protocols
- Doesn't drag in burst / outstanding-ID / atomic complexity that distracts from BFM-specific structure

Files:
```
examples/axi_lite_slave_bfm/
├── README.md
├── MODE.md                              # mode: protocol-bfm
├── doc/
│   ├── theory_of_operation.md           # BFM-mode skinnier than block-mode
│   ├── signal_interface.md              # full AXI-Lite wires
│   ├── pin_level_reset.md               # every wire's reset value
│   ├── protocol_rules.md                # rule table per AXI-Lite channel
│   ├── channel_handshake.md             # AW/W/B and AR/R dependencies
│   ├── transaction_api.md               # apply_write, apply_read, expect_*
│   ├── channel_api.md                   # phase-level
│   └── active_passive_mode.md           # active = drive AWREADY etc; passive = monitor only
└── dv/
    └── plan.md                          # coverage hooks per protocol-rule ID
```

This example serves the same function as `wctmr/` does for behavioral-block mode: it grounds the templates in something readable and testable.

### 7.9 `/spec-init` interview update

Current first interview question is implicit IP-class. New version asks explicitly:

> Q1: What kind of IP is this?
>   A) Behavioral block — peripheral, accelerator, controller, bus master, or any
>      block whose main concern is internal logic with software-visible registers
>   B) Protocol BFM / VIP — a verification model that talks to RTL via a
>      defined wire-level protocol (AXI master, AXI slave, custom-protocol BFM, etc.)

Default: A. Both modes proceed with the same skeleton command after this branch.

### 7.10 `/spec-help` update

Workflow card adds a mode-aware section:

```
Mode: <reads from MODE.md, defaults to behavioral-block>

Recommended order of writing (this mode):
  <list from §7.4 above>

Mode-specific reader-test bank: <bank file referenced>
```

---

## 8. User-facing surface

After implementation, the user-visible changes:

### 8.1 New flow

```
$ /spec-init my_axi_lite_slave_bfm
> What kind of IP is this?
>   A) Behavioral block (default)
>   B) Protocol BFM / VIP
> Choice: B

> Bus / protocol the BFM speaks?
> AXI4-Lite

> Role (master / slave / both)?
> slave

> Active mode, passive mode, or both?
> both

> [continues with mode-appropriate questions]

[skeleton generated under ./spec/my_axi_lite_slave_bfm/, with MODE.md set
 to protocol-bfm]
```

### 8.2 Same commands, mode-aware behavior

- `/spec-status` reads MODE.md, applies the correct stage-gate checklist
- `/spec-review` reads MODE.md, dispatches the correct reader-test bank to the spec-reader subagent
- `/spec-lint` reads MODE.md, applies common + mode-specific lint rules
- `/spec-gate` reads MODE.md, evaluates mode-conditional gate items

### 8.3 No regression for existing specs

Existing specs (without MODE.md) default to `behavioral-block`. All current behavior preserved.

---

## 9. Worked example

The AXI-Lite slave BFM example (§7.8) is the acceptance artifact. A reader who is asked to implement this BFM in C++ or SystemVerilog should be able to do so without any external reference beyond the AXI-Lite protocol spec and the example itself.

Concretely:

- A C++ implementer reading the example knows exactly which member functions to expose (Transaction API), what they do, what wires get driven and in what order
- An SV/UVM implementer reading the example knows the agent's driver/monitor split, the transaction class fields, the sequencer interface
- A DV engineer reading the example knows which protocol rules to write assertions for, and what coverage to collect

If any of these readers need to ask questions of the example author to fill gaps, the example failed and should be revised before the BFM mode ships.

This is the same standard `wctmr/` is held to in behavioral-block mode.

---

## 10. Validation and acceptance criteria

The BFM mode work is complete when:

1. ☐ `/spec-init` offers the mode choice and persists it to MODE.md
2. ☐ All seven new template files exist with required-structure / writing-rules / anti-patterns / mini-example sections (matching existing template style)
3. ☐ `process/bfm_reader_test_bank.md` covers at least the eight categories listed in §7.5
4. ☐ `/spec-lint` LINT-BFM-001..005 implemented and tested against the AXI-Lite example
5. ☐ Stage gates have BFM-conditional rows at D1, D2, D3
6. ☐ `examples/axi_lite_slave_bfm/` is complete and passes its own `/spec-review` and `/spec-lint` cleanly (with any waivers documented in WAIVERS.md)
7. ☐ Existing `examples/wctmr/` continues to pass its checks unchanged (no regression)
8. ☐ SKILL.md and `/spec-help` describe the mode flag honestly
9. ☐ This draft (`BFM_MODE_DESIGN.md`) is checked in alongside the implementation as design history

---

## 11. Open questions

Items that should be resolved during implementation, not now:

### Q1. Does `registers.md` apply to BFM mode?

Some BFMs (e.g., a BFM that happens to be a CSR-controllable test infrastructure block) do have register interfaces. Default in §7.4 is to omit `registers.md` for BFM mode, but should it be optional rather than excluded?

**Tentative resolution:** make it optional. If user writes a `registers.md`, lint and stage gate include it; if absent, no error.

### Q2. How should mixed-purpose IPs (e.g., a NoC NI that is both a behavioral block and exposes BFM-style interfaces) be handled?

**Tentative resolution (deferred):** v1 picks one mode. Mixed IPs write two specs that cross-reference. Revisit in v2 if pattern recurs.

### Q3. Should the plugin ship pre-canned AXI / AXI-Lite / APB rule tables for users to start from?

**Tentative resolution: NO.** Out of scope per §5. The plugin teaches structure; protocol content is the user's. Otherwise the plugin must keep up with protocol revisions, which is a maintenance burden out of proportion with its value.

### Q4. Format for protocol rule severity — three levels (FAIL/WARN/RECOMMEND) or simpler binary?

**Resolution (revised after implementation cross-check):** Ship with **two levels — `FAIL` and `RECOMMEND`** — in a separate column.

The original draft claimed "ARM uses three levels", which was incorrect. Cross-checking the ARM AMBA Protocol Checker User Guides (DUI 0534B for AXI4, DUI 0576A for AXI3) and multiple SVA-AXI implementations (YosysHQ SVA-AXI4-FVIP, AMD/Xilinx LogiCORE PG101) confirms ARM Protocol Checker uses exactly two severity classes: **`ERR` (mandatory)** and **`REC` (recommended)**, encoded as a substring of the rule ID itself (`AXI4_ERRM_AWVALID_STABLE`, `AXI4_RECM_RREADY_MAX_WAIT`).

Decisions taken in `02b_protocol_rules.md`:

- **Two severity levels** (`FAIL`, `RECOMMEND`) — matches ARM's `ERR`/`REC` count and intent. The earlier middle level `WARN` is dropped: rules promoted to `FAIL` if violation breaks BFM determinism; demoted to `RECOMMEND` if it is quality-of-implementation only.
- **Severity as a separate column**, not embedded in the rule ID — diverges from ARM. Rationale: (a) rule IDs are stage-gate-anchor-stable contracts; renaming a rule when severity changes would invalidate `WAIVERS.md` references. (b) Per-project severity overrides (e.g., promoting a `RECOMMEND` to `FAIL` for a stricter test suite) are common; column-based severity supports this without rule renaming.
- **Optional `ARM SVA equivalent` column** in the rule table — for rules that have a 1:1 ARM `<PROTO>_<SEV><ROLE>_<NAME>` counterpart, the column lists it. Implementers writing or cross-validating against SVA libraries get the lookup; rules with no ARM equivalent (BFM-specific extensions like configuration-knob rules, reset-duration rules, cross-channel rules under `XCH`) are explicitly tagged `(none)`.

This combination keeps our format auditable and gates-stable while preserving 1:1 ARM tooling lookup where it exists.

### Q5. Should `spec-reader` subagent be given protocol-specific knowledge (e.g., "you know AXI") for BFM-mode reader tests?

**Tentative resolution: NO.** The subagent's value is its isolation from authoring context, not its protocol expertise. If the spec relies on the reader knowing AXI, that's a documentation gap, not a strength.

### Q6. How does `/spec-import` (brownfield path) interact with BFM mode?

**Status: known limitation, not a deferred design decision.**

This question was not in the original draft. The draft predates the existence of `/spec-import`, which was added to the plugin in a prior release. `/spec-import` (brownfield path) currently understands only the behavioral-block 6-file layout. BFM-style RTL — pure interface modules, SystemVerilog `interface` + `modport` declarations, modules with no software-visible register file — will be force-fitted into the behavioral-block layout if imported through this command, producing a draft that is structurally wrong for the source material.

**v1 handling (in scope of this BFM mode work):**

- `commands/spec-import.md` opens with a "Known limitation: BFM mode" section steering BFM users to `/spec-init <name>` with `protocol-bfm` selected, then manual port population from the RTL.
- If the input RTL exhibits clear BFM-style indicators (no `always_ff` block writing software-visible state; no hjson register file present; presence of `interface` / `modport` constructs; pure assign / wire-routing dominates over sequential logic), `/spec-import` emits the redirect prompt and stops rather than generating a behavioral-block skeleton. Detection should err toward false negatives (proceed with import) over false positives (block valid behavioral-block input).
- Generated specs from `/spec-import` write `MODE.md` with `mode: behavioral-block` for explicit forward-compat with mode-aware tooling.

**v2 directions (not promised, listed for future-author orientation):**

- Dispatch BFM templates from `references/templates/bfm/` after RTL classification, instead of redirecting.
- Reverse-extract protocol rules from `modport` direction signs and SVA bodies. High error rate expected; output must be tagged `[draft — protocol rule extraction is heuristic; review every row]` and require manual review before any stage gate.
- Extend `references/process/rtl_extraction.md` with a BFM-mode section enumerating which BFM artifacts are extractable, which are heuristic, and which are out of reach.

### Q7. How does the spec cover RTL internal architecture when BFM has an RTL counterpart?

**Status: resolved during v0.3.0 implementation review (2026-05-05).**

The original BFM mode design assumed BFM specs document only the BFM implementation (driver / monitor / sequencer). This works for pure verification IP (BFM with no RTL counterpart). It fails for the user's actual workflow: each block has both a BFM/C-model implementation AND an RTL implementation, both behaviorally equivalent at the bus interface but with different internal architectures.

**Decision:** BFM-mode `theory_of_operation.md` has a **two-section structure**:
- `## BFM internal architecture` — **always required** in BFM mode. Driver, monitor, sequencer, configuration store, implementation algorithms, reset entry sequencing, performance commitments.
- `## RTL internal architecture` — **conditional**. Required when `MODE.md` declares `has-rtl-counterpart: yes`. Contains RTL block structure, pipeline / timing, RTL reset behavior, **RTL-vs-BFM behavioral equivalence** sub-section (the load-bearing contract: states which BFM features are test-only vs which have RTL counterparts), optional implementation notes.

**Implementation:**

- New BFM-mode-specific template: `references/templates/bfm/02_theory_of_operation.md` with the two-section required structure.
- `MODE.md` format extended with optional `has-rtl-counterpart: yes | no` field (BFM mode only; defaults to `no` if absent).
- `/spec-init` BFM interview asks the question explicitly: "Will this block also have an RTL counterpart implementation?" The answer flows into MODE.md and into ToO generation (whether the RTL section is generated or omitted).
- New stage gate anchor: `D1.bfm.too` — covers BFM-internal section always, plus RTL-internal section when has-rtl-counterpart is yes.
- Both worked examples (`axi_lite_slave_bfm`, `apb_slave_bfm`) updated: MODE.md sets `has-rtl-counterpart: yes`; theory_of_operation.md restructured with both sections; the RTL section includes the RTL-vs-BFM behavioral equivalence table making the abstraction gap (BFM-only knobs like `set_response_delay` vs RTL fixed timing) explicit.

**Why two sections in one file (not a separate `rtl_architecture.md` file):**

- "Same block, one spec" principle. The block is one logical unit; its multiple implementations are different views of the same thing.
- BFM internal architecture and RTL internal architecture both describe "how this block works internally" — natural to colocate.
- Avoids file-count proliferation in BFM mode (already 9–10 files; would become 11).
- The RTL-vs-BFM behavioral equivalence sub-section makes most sense when adjacent to the BFM internal architecture description it contrasts against.

**Out of scope (not in v0.3.0):**

- A `behavioral-block` mode equivalent. behavioral-block ToO is already RTL-centric; the BFM concern doesn't apply.
- Lint enforcement of the conditional section. `/spec-lint` does not currently parse `has-rtl-counterpart` to enforce section presence/absence; the stage gate `D1.bfm.too` is the enforcement point.
- Reverse engineering: starting from RTL and producing the BFM section. v2 direction.

---

## 12. Future extensions explicitly out of scope

These are real needs but belong to separate design drafts when the time comes:

- **Protocol Reference Library** (**v0.4.0 candidate, primary** — see §13 below for sketch). Industry-standard protocols (AXI4 / AXI4-Lite / APB / TileLink / CHI / ACE) have fixed wire-level rules per their published specifications. Currently every BFM spec re-generates these rules inline, with risk of drift between specs and verification burden every time. A built-in protocol library — verified once against ARM official documents + open-source SVA implementations — would let new BFM specs **inherit** the standard rules and only describe block-specific extensions. Drift eliminated, verification amortised, spec writing accelerated.
- **AT-level TLM model spec extension** — for performance modeling separate from BFM. Per workflow discussion, the designer's "performance modeling" use case may want this. Different from BFM mode (which is CA at the pin boundary). Separate concern.
- **Co-simulation orchestration spec** — describing how the BFM, the RTL, and the test harness link together (DPI-C bridges, SystemC adapters, simulator command-line flows). Currently outside any spec the plugin produces.
- **Coverage closure tracking** — beyond the per-rule coverage hooks listed in DV plan. A separate skill for VLSI coverage management could exist later.
- **Protocol conformance test suite generation** — the plugin produces a spec, not tests. Generating an executable conformance suite from the protocol_rules.md table is a tempting v2 feature but a different problem (codegen, not authoring).

---

## 13. Protocol Reference Library — v0.4.0 design sketch

**Status: planned for v0.4.0; not in scope of v0.3.0.** This section captures the sketch so v0.4.0 implementation can pick up cold.

### Motivation

In v0.3.0, BFM specs inline every protocol rule. The same AXI4 rule (e.g. `AXI4_ERRM_AWVALID_STABLE`) gets re-generated each time someone runs `/spec-init protocol-bfm` against an AXI4-based BFM. Three problems:

1. **Drift across specs**: rule wording, IDs, severity assignments, and ARM SVA equivalent column populations vary between specs that should describe the same protocol.
2. **Per-spec verification burden**: the user must check correctness of ~30+ rules per spec, every time.
3. **Spec bloat**: each `protocol_rules.md` is 150+ lines mostly re-stating the standard, with only the last few rows specifying block-specific extensions (RoB, ECC, custom QoS) that are the actually-novel content.

### Architecture

Add a built-in protocol library at:

```
plugins/hw-spec-author/skills/hw-spec-author/references/protocols/
├── axi4/
│   ├── README.md                  # Coverage scope, ARM document version reference, verification log entry point
│   ├── signal_interface.md        # Canonical wire tables for master / slave / monitor roles
│   ├── protocol_rules.md          # Canonical AXI4 rules — every row cross-checked against ARM IHI 0022 + open-source SVA libraries
│   ├── channel_handshake.md       # Standard AXI4 may/must dependency diagrams
│   └── VERIFIED.md                # Verification provenance: source-by-source cross-check log, reviewer + date
├── axi4_lite/                     # Subset of AXI4
├── apb/                           # APB3 / APB4 (per ARM IHI 0024)
├── tilelink_ul/                   # TileLink-UL (per FlooNoC spec)
├── chi/                           # ARM CHI (Issue X)
└── README.md                      # Library index, version policy, contribution flow
```

### BFM specs become thin wrappers

Instead of inlining ~30 standard rules, a BFM spec's `protocol_rules.md` becomes:

```markdown
# Protocol Rules

This BFM uses **AXI4 master** as defined in [`references/protocols/axi4/`](...).
All `AXI4_MST_*` rules from that reference apply unmodified.

## Block-specific extensions

| ID | Condition | Required behavior | Severity | ARM SVA equivalent |
|---|---|---|---|---|
| NI_RST_OUTPUTS_LOW_AXI | ... | ... | FAIL | (none) |
| NI_CFG_QOS_REGULATOR | ... | ... | FAIL | (none) |
...
```

A typical BFM `protocol_rules.md` shrinks from ~150 lines to ~30 lines. `signal_interface.md` and `channel_handshake.md` shrink similarly.

### Verification flow (the load-bearing part)

The library's value depends on its rules being **verified** against authoritative sources. Every entry in `protocols/<proto>/protocol_rules.md` must be cross-checked against at least 2 of:

- **ARM official documents** (IHI 0022 for AXI4, IHI 0024 for APB, IHI 0050 for CHI, etc.)
- **ARM official SVA assertion libraries** (DUI 0534B for AXI4 protocol assertions; DUI 0576A for AXI3; equivalent for other protocols)
- **Open-source SVA implementations**: YosysHQ SVA-AXI4-FVIP, jkopanski Axi4PC.sv, AMD/Xilinx LogiCORE PG101, others as identified

For every rule:
- ID, Condition, Required behavior, Severity all cross-checked
- Source citations recorded in `VERIFIED.md` (file + line for SVA libraries; section + revision for ARM documents)
- ARM SVA equivalent column populated with verified IDs
- Disputed or unverified rules tagged `(disputed)` or `(unverified)`; resolution required before merge

VERIFIED.md format (per protocol):

```markdown
# Verification Log — AXI4

**Verified against**:
- ARM IHI 0022 Issue F (date: ...)
- ARM DUI 0534B Issue B (date: ...)
- YosysHQ SVA-AXI4-FVIP commit <hash> (date: ...)

**Reviewer**: <name>
**Date**: <date>

**Rule-by-rule verification**: each rule row in `protocol_rules.md` has a row here listing source(s) cross-checked. Disputed rows flagged.
```

### Integration with `/spec-init`

`/spec-init protocol-bfm` adds a new question:

> Q: Which protocol does this BFM speak?
>   (A) AXI4 (full) — uses library
>   (B) AXI4-Lite — uses library
>   (C) APB3 / APB4 — uses library
>   (D) TileLink-UL — uses library
>   (E) CHI — uses library
>   (F) Custom protocol — full inline spec (current v0.3.0 behavior)

For options A-E: spec-init writes a thin `protocol_rules.md` / `signal_interface.md` / `channel_handshake.md` that **inherits** from the library and includes a `## Block-specific extensions` section for designer-specific additions. The library is referenced via relative path; its content is not duplicated.

For option F: current v0.3.0 behavior unchanged — full inline spec.

### Versioning policy

Each library entry is tied to a specific revision of the upstream specification (e.g. AXI4 against IHI 0022 Issue F). When ARM publishes a new revision:

- Library entry retained at old revision
- New library directory created (e.g., `axi4/issueG/`) at the new revision
- BFM specs that pin to a specific revision get reproducible behavior

Default: BFM specs reference the latest verified revision unless explicitly pinned.

### Variance handling

A BFM may legitimately diverge from the standard (e.g., supports only a subset, or adds custom restrictions). The library reference syntax should support:

- `inherit_all`: all library rules apply unmodified
- `inherit_except: [list of IDs]`: most rules apply; specific rules are explicitly excluded (with rationale)
- `inherit_with_overrides`: most rules apply; a small subset is overridden with BFM-specific severity / behavior

`/spec-lint` LINT-BFM-006 (new) would validate that an exclusion or override has a rationale comment in the spec.

### Implementation order (v0.4.0)

1. **Pre-1**: Write `plan/PROTOCOL_LIBRARY_DESIGN.md` formalising this sketch (~2 hr).
2. **Pre-2**: Verify and produce the first library entry — AXI4-Lite (~3 hr; axi_lite_slave_bfm already covers most rules).
3. **v0.4.0 base**: AXI4 + APB library entries (~6 hr).
4. **v0.4.0 base**: `/spec-init` protocol question + thin-wrapper skeleton generation (~2 hr).
5. **v0.4.0 base**: Refactor existing examples to use library references (~2 hr).
6. **v0.4.x**: Additional protocols (TileLink-UL, CHI, AXI3) — ~4 hr per protocol.

Total v0.4.0 base: ~15 hr.

### Out of scope (defer to v0.5.0+)

- Auto-generation of library entries from RTL `interface` declarations + SVA assertions (reverse-engineering)
- Protocol conformance test suite generation from library entries (codegen)
- Library-backed `/spec-import` BFM mode (closes v0.3.0 §11.Q6 known limitation)

---

## Appendix A — Summary of files added

```
plugins/hw-spec-author/
├── BFM_MODE_DESIGN.md                              # ← THIS FILE (design history)
├── skills/hw-spec-author/
│   ├── SKILL.md                                    # MODIFIED (mode section)
│   └── references/
│       ├── templates/
│       │   └── bfm/                                # NEW directory
│       │       ├── 02b_protocol_rules.md
│       │       ├── 02c_channel_handshake.md
│       │       ├── 02d_pin_level_reset.md
│       │       ├── 03b_transaction_api.md
│       │       ├── 03c_channel_api.md
│       │       ├── 03d_active_passive_mode.md
│       │       └── 04b_signal_interface.md
│       └── process/
│           ├── stage_gates.md                      # MODIFIED (BFM-conditional rows)
│           └── bfm_reader_test_bank.md             # NEW
├── commands/
│   ├── spec-init.md                                # MODIFIED (mode question)
│   ├── spec-status.md                              # MODIFIED (mode-aware)
│   ├── spec-review.md                              # MODIFIED (mode-aware bank)
│   ├── spec-lint.md                                # MODIFIED (LINT-BFM-*)
│   ├── spec-gate.md                                # MODIFIED (mode-aware checklist)
│   └── spec-help.md                                # MODIFIED (mode-aware card)
└── examples/
    ├── wctmr/                                      # unchanged
    └── axi_lite_slave_bfm/                         # NEW worked example
        ├── README.md
        ├── MODE.md
        ├── doc/
        │   ├── theory_of_operation.md
        │   ├── signal_interface.md
        │   ├── pin_level_reset.md
        │   ├── protocol_rules.md
        │   ├── channel_handshake.md
        │   ├── transaction_api.md
        │   ├── channel_api.md
        │   └── active_passive_mode.md
        └── dv/
            └── plan.md
```

## Appendix B — Estimated scope

For sizing future implementation work:

| Item | Estimated lines |
|---|---|
| 7 new template files | ~1500 (similar density to existing templates) |
| bfm_reader_test_bank.md | ~250 |
| Stage gate additions | ~50 |
| LINT-BFM-* additions to spec-lint.md | ~150 |
| SKILL.md mode section + workflow updates | ~80 |
| All 6 commands' mode-awareness | ~100 |
| AXI-Lite slave BFM example (8 spec files) | ~1500 |
| **Total estimated additions** | **~3600 lines** |

Existing plugin is ~1800 lines (excluding wctmr example which is ~970). BFM mode roughly doubles plugin size. Worth flagging — the user may want this split across multiple commits or releases rather than one big merge.

---

*End of design draft.*

# hw-spec-author-plugin

> **Languages**: [English](./README.md) | [繁體中文](./README.zh-TW.md)

A Claude Code plugin marketplace for digital hardware design tooling. The marketplace currently ships one plugin: **`hw-spec-author`** — an opinionated workflow for authoring clean, complete, audience-segregated hardware design specifications for digital IP blocks. Two modes are supported:

- **`behavioral-block`** mode (default) — the OpenTitan / lowRISC Comportability layout for IP blocks with internal logic and software-visible registers.
- **`protocol-bfm`** mode — for blocks with rigorous wire-level protocol contracts (AXI, AHB, APB, TileLink, NoC, custom). Suits both verification-only BFMs and real RTL+C-model paired implementations.

---

## Why this exists

Hardware design specifications drift in predictable ways:

- **Audience mixing.** A single paragraph addresses HW designer, DV engineer, and SW driver author in the same breath, serving none of them well.
- **Duplicated facts.** The same register behavior is described in three files; over six months the three descriptions diverge silently.
- **Implicit assumptions.** "Reset clears everything" — does it clear RAM? Scrambling state? Scrubbing counters? The author knows; the spec doesn't say.
- **No discipline on completeness.** "The spec is mostly done" is not a falsifiable claim. Review meetings devolve into preference votes.
- **Late-found ambiguity.** Bugs surface during V1 testing as "the spec didn't say what should happen here," costing weeks per occurrence.

`hw-spec-author` enforces four properties that, together, eliminate this class of drift:

1. **Audience-segregated.** Every section states its primary reader (HW / DV / SW / SoC integrator / testbench author) and stays in that lane.
2. **Single source of truth.** Every fact lives in exactly one file; other files cross-reference.
3. **Stage-gated D0 → D3.** Each gate has an explicit checklist with stable anchors. "Done enough for the next phase" is a yes/no question.
4. **Reader-testable.** A fresh-context subagent reads the spec and answers concrete corner-case questions; gaps are mechanical findings, not opinions.

The framework is silicon-proven for behavioral-block mode: OpenTitan uses it across Google, ETH Zurich, Western Digital, Seagate, and Nuvoton. `protocol-bfm` mode extends the same four properties into the protocol-rigorous space (cycle-level rules, channel handshake dependencies, pin-level reset, transaction/channel API).

---

## What's shipped in this marketplace (v0.3.1)

| Plugin | Description |
|---|---|
| [`hw-spec-author`](./plugins/hw-spec-author/) | Authors hardware design specs for digital IP blocks. Two modes: **behavioral-block** (OpenTitan-Comportability 6-file structure) and **protocol-bfm** (9–11-file structure with pin-level rigor, protocol rules, transaction/channel API). D0→D3 stage gates and reader-testing in both modes. |

**v0.3.1** is a documentation-only patch release that aligns `TODO(designer):` format guidance across the generation side (SKILL.md, /spec-init, /spec-import) and the verification side (writing_principles.md §6, /spec-lint LINT-002, /spec-gate `D1.cross.todos_tracked`). The two canonical formats — `(see #N)` for tracked work and `(no issue yet — <reason>)` for sandbox/pending decisions — are now uniformly enforced from generation through gate, eliminating a documented dogfood-rework class. No spec content changes; existing v0.3.0 specs remain valid as long as their TODO markers were already in canonical form.

---

## Install

### Inside Claude Code (recommended)

```
/plugin marketplace add changlukas/hw-spec-author-plugin
/plugin install hw-spec-author@changlukas
```

Restart Claude Code, then verify with:

```
/spec-help
```

If the workflow card appears (with mode line at the top), you're ready.

### Persistent install via `settings.json`

For team-wide or persistent install, add to `~/.claude/settings.json` (global) or `.claude/settings.json` (project):

```json
{
  "extraKnownMarketplaces": {
    "changlukas": {
      "source": {
        "source": "github",
        "repo": "changlukas/hw-spec-author-plugin"
      }
    }
  },
  "enabledPlugins": {
    "hw-spec-author@changlukas": true
  }
}
```

---

## Choosing a mode: `behavioral-block` vs `protocol-bfm`

Mode is selected when you run `/spec-init` and recorded in the spec root's `MODE.md`. All downstream commands (`/spec-status`, `/spec-review`, `/spec-lint`, `/spec-gate`, `/spec-help`) read it.

### Decision tree

Walk through these questions in order:

1. **Does the block speak a wire-level handshake protocol** (AXI4 / AXI4-Lite / AHB / APB / TileLink / CHI / custom valid-ready) **as its primary external interface?**
   - **No** (it's a peripheral with software-visible CSR but no protocol-cycle rigor needed; or it's a purely internal sub-block) → **`behavioral-block`**.
   - **Yes** → continue to Q2.

2. **Will you (or someone else) write a cycle-accurate C/C++/SystemC/UVM model** of this block, **alongside an RTL implementation, both behaviorally equivalent at the wire boundary?**
   - **Yes** → **`protocol-bfm`** with `has-rtl-counterpart: yes`. Spec serves both implementations: shared protocol contract + separate internal architecture sections (BFM-internal: driver/monitor/sequencer; RTL-internal: register file/decoder/pipeline).
   - **No, only an RTL implementation** → use **`behavioral-block`** for now. (Future v0.4.0 may add a "rtl-protocol-block" variant; for v0.3.0 the `behavioral-block` template's `theory_of_operation.md` is closer fit.)

3. **Is this a pure verification artifact** (a BFM/VIP that has no RTL counterpart, written purely to verify someone else's RTL)?
   - **Yes** → **`protocol-bfm`** with `has-rtl-counterpart: no`. Spec describes BFM internal architecture, testbench-API surface, and protocol contract.

### Concrete examples

| IP type | Mode | `has-rtl-counterpart` | Notes |
|---|---|---|---|
| Wallclock timer (CSR-driven, single bus, ~10 registers) | `behavioral-block` | n/a | See `examples/wctmr/`. |
| Generic peripheral (UART, SPI, I2C controller) with CSR | `behavioral-block` | n/a | Bus interface handled abstractly; protocol-cycle rigor not the focus. |
| AXI4 master IP design + paired C++ model for performance / co-sim verification | `protocol-bfm` | yes | Spec covers wire-level AXI4 contract once; both RTL and C model implement against it. Internal architecture documented in two sections. |
| AXI-Lite slave BFM (verification IP only, no RTL counterpart) | `protocol-bfm` | no | See `examples/axi_lite_slave_bfm/`. |
| APB slave BFM | `protocol-bfm` | yes / no | See `examples/apb_slave_bfm/` for the structural template. |
| Network Interface bridging AXI4 ↔ NoC packet substrate | `protocol-bfm` | yes | Bus-attached, dual-protocol, with both RTL and C model implementations expected. |

### "BFM" — what this plugin means by the term

Industry usage: "BFM" / "VIP" usually = **pure verification model**, has no RTL counterpart.

This plugin's `protocol-bfm` mode: any block whose external interface is a **wire-level protocol contract** that needs cycle-accurate documentation. The block may have:

- An RTL implementation (e.g., your AXI master design that goes to silicon)
- A C/C++/SystemC/UVM behavior model (e.g., your AXI master's reference C model for co-sim)
- Both (most common in your typical workflow)
- Only a BFM (industry-standard verification IP)

The mode name `protocol-bfm` reflects the **template structure** (cycle-level protocol rigor, transaction/channel API surface) rather than the **artifact category** (verification-only). When `has-rtl-counterpart: yes`, the spec serves both implementations from one source of truth, with explicit documentation of which features (response delay, fault injection, active/passive mode) are test-only BFM knobs vs which behaviors the RTL counterpart implements identically.

---

## Quick start

### Greenfield (no existing material) — `behavioral-block`

```
/spec-init my_timer
```

Pick option (A) Behavioral block when prompted. Runs the Capture-phase interview (asks for IP purpose, bus interface, clock/reset domains, and 3–8 features), writes `MODE.md`, then generates a 6-file OpenTitan-style skeleton at `./spec/my_timer/`.

### Greenfield — `protocol-bfm`

```
/spec-init my_axi_master
```

Pick option (B) Protocol BFM / VIP. Additional questions: protocol & version, role (master/slave/both), active/passive support, configuration knobs, **and whether an RTL counterpart will exist**. Writes `MODE.md` (with `has-rtl-counterpart: yes/no`), then generates a 9–11-file skeleton at `./spec/my_axi_master/`.

### Brownfield (existing spec / RTL / hjson)

```
/spec-import path/to/old_spec_or_rtl/
```

**Behavioral-block mode only** in v0.3.0. Reads existing markdown / SystemVerilog / Verilog / hjson and restructures into the 6-file layout with mechanical extraction (ports, parameters, register reset values, FSM state names, SVA assertions). Conflicts surfaced in `IMPORT_REPORT.md`.

For BFM-mode brownfield (importing existing BFM source), use `/spec-init` greenfield path with manual port population. See [§11.Q6 of the BFM mode design draft](./plan/BFM_MODE_DESIGN.md) for v2 directions.

### Subsequent commands (mode-aware)

| Command | When to run | Effect |
|---|---|---|
| `/spec-status` | Anytime | Reads `MODE.md`. Reports current D-stage against the mode-appropriate checklist. Mode-conditional items (e.g., `D1.bfm.*`) only count when applicable. Read-only. |
| `/spec-lint` | Anytime in Phase 2 | Mechanical drift check. LINT-001..007 always apply; LINT-BFM-001..005 additionally in `protocol-bfm` mode. Read-only. |
| `/spec-review` | Before claiming D1 | Reader test via the `spec-reader` subagent. Question bank is mode-matched: `reader_test.md` for behavioral-block; `bfm_reader_test_bank.md` for protocol-bfm. |
| `/spec-gate D1` | When D1 looks ready | Walks every D1 checklist item (mode-conditional N/A items skipped). Persistent waivers in `WAIVERS.md`. |
| `/spec-gate D2` / `D3` | Later in lifecycle | Same protocol for the later gates. |
| `/spec-help` | Anytime | Workflow card with mode line + suggested next command based on state. |

---

## What `/spec-init` produces

### `behavioral-block` mode (6 files)

```
<ip_name>/
├── MODE.md                     # mode: behavioral-block
├── README.md                   # Summary index — Overview, Features, Description, Compatibility
├── doc/
│   ├── theory_of_operation.md  # Block diagram, datapath, FSM, errors (HW + DV + SW)
│   ├── programmers_guide.md    # Init, use cases, error handling, IRQs (SW + DV)
│   ├── interfaces.md           # Port table, parameters, IRQs (SoC integrator)
│   └── registers.md            # CSR map (SW + DV)
└── dv/
    └── plan.md                 # Testpoints, coverage model, sec_cm (DV)
```

A small block can collapse `theory_of_operation.md` and `programmers_guide.md` into the README; a complex block may split `theory_of_operation.md` across multiple files under `doc/`.

### `protocol-bfm` mode (9–11 files)

```
<ip_name>/
├── MODE.md                     # mode: protocol-bfm; has-rtl-counterpart: yes/no
├── README.md                   # Summary index — Overview, Features, Description, Compatibility
├── doc/
│   ├── theory_of_operation.md  # Two sections: ## BFM internal architecture (always) + ## RTL internal architecture (when has-rtl-counterpart: yes)
│   ├── signal_interface.md     # Every wire fully specified (replaces interfaces.md for BFM mode)
│   ├── pin_level_reset.md      # Every wire's reset value, both before and after reset
│   ├── protocol_rules.md       # Cycle-level rules per channel + cross-channel + reset + config-knob rules
│   ├── channel_handshake.md    # Channel dependency diagrams + may/must arrow conventions
│   ├── transaction_api.md      # High-level testbench API (~95% of tests use only this)
│   ├── channel_api.md          # Phase-level testbench API (5% corner-case tests)
│   ├── active_passive_mode.md  # Active vs Passive capability + config knob
│   └── registers.md            # Optional — only if the block has software-visible CSR
└── dv/
    └── plan.md                 # Testpoints + coverage hooks per protocol-rule ID
```

`registers.md` is optional in BFM mode (most BFMs have no software-visible CSR). When present, the standard register lints/gates apply automatically.

---

## What each file is for: roles and audiences

A spec is multiple files because each file targets a specific reader. The table below maps every file to its problem, audience, and implementation use.

### `behavioral-block` mode

| File | Solves what | Who reads it | Maps to what in implementation |
|---|---|---|---|
| `README.md` | 1-minute "what is this IP" + Features list | Anyone first-touch | Not directly; orients reader. |
| `doc/theory_of_operation.md` | Block-internal architecture (datapath, FSMs, error handling, performance) | RTL designer + DV engineer + sometimes SW author | RTL submodule hierarchy + behavioral algorithms. |
| `doc/programmers_guide.md` | Software-driver perspective: init sequence, use cases, error responses, IRQ handling | Firmware/driver author | C driver code structure. |
| `doc/interfaces.md` | Port table, parameters, clocks, resets, IRQs, alerts | SoC integrator + RTL designer | RTL module port list. |
| `doc/registers.md` | CSR memory map + per-register field decomposition + reset values | SW author + RTL designer | RTL register file + driver register accesses. |
| `dv/plan.md` | Verification scope + testpoints + coverage model + sec_cm | DV engineer | UVM testbench + assertions + covergroups. |

### `protocol-bfm` mode

| File | Solves what | Who reads it | Maps to what in implementation |
|---|---|---|---|
| `MODE.md` | Plugin metadata (mode + has-rtl-counterpart flag) | Plugin tools | Not implementation-facing. |
| `README.md` | 1-minute "what BFM is this" + Features | Anyone first-touch | Not directly. |
| `doc/theory_of_operation.md` (§BFM internal) | How the BFM is structured: driver / monitor / sequencer / configuration store / sub-modules | C model implementer | C++ class internal partitioning. |
| `doc/theory_of_operation.md` (§RTL internal) | When `has-rtl-counterpart: yes` — RTL block structure, pipeline, register file, RTL-vs-BFM equivalence table | RTL designer + C model implementer (cross-check) | RTL submodule hierarchy. |
| `doc/signal_interface.md` | Every wire's contract (name, dir, width, sample edge, clock domain, reset linkage, optional-in-protocol, BFM-supports). Replaces `interfaces.md`. | RTL DUT integrator + C model integrator + DPI-C bridge author | RTL port list / C++ class member variables. |
| `doc/pin_level_reset.md` | Every wire's reset value during and immediately after reset, per reset signal (multi-domain handled) | Both implementers | C model `reset()` function lines / RTL flop reset values. |
| `doc/protocol_rules.md` | All protocol invariants enumerated as testable rules (`<PROTO>_<ROLE>_<CHANNEL>_<NAME>` IDs with severity FAIL/RECOMMEND + optional ARM SVA equivalent) | DV engineer (→ SVA assertions) + implementer (→ if/else conditions) | RTL `assert property` / C model protocol-checker. |
| `doc/channel_handshake.md` | Cross-channel and cross-protocol dependency diagrams + may/must arrows + deadlock-avoidance commentary | Implementer designing FSMs / interlocks | RTL FSM transition guards / C model sequencer ordering. |
| `doc/transaction_api.md` | High-level testbench-facing API (what 95% of tests call): `apply_*`, `expect_*`, `set_*`, `get_*`, `reset_state`. Each method gets Signature / Preconditions / Side effects / Return / Error modes + Equivalent channel API decomposition. | Test author | C++ public method interface. |
| `doc/channel_api.md` | Phase-level fine-grained API (5% corner-case tests): per-channel `begin_phase`, `assert_valid`, `wait_for_ready`, `end_phase`. | Advanced test author | C++ public method interface (fine-grained). |
| `doc/active_passive_mode.md` | Capability table (active drives wires; passive monitors only) + mode-switch knob + common testbench setups | Testbench integrator | `bfm_mode` flag + driver enable/disable. |
| `doc/registers.md` (optional) | CSR memory map (when block has software-visible CSR) | SW author + implementer | Same as behavioral-block. |
| `dv/plan.md` | Testpoints + covergroups + ABV (one SVA per FAIL rule) + FPV strategy | DV engineer | UVM testbench + SVA library + covergroups. |

### Reading routes by audience

**RTL implementer** (writing synthesizable RTL):
1. `signal_interface.md` → module port list
2. `pin_level_reset.md` → flop reset values
3. `protocol_rules.md` → SVA assertions + internal logic conditions
4. `channel_handshake.md` → cross-channel FSM design
5. `theory_of_operation.md §RTL internal` → submodule hierarchy
6. `registers.md` (if present) → register file design

**C/C++ model implementer** (writing BFM):
1. Same files 1–4 above (shared contract)
2. `theory_of_operation.md §BFM internal` → driver/monitor/sequencer partitioning
3. `transaction_api.md` + `channel_api.md` → public method interface
4. `active_passive_mode.md` → mode-switch logic
5. `registers.md` (if present) → CSR file emulation backend

**Testbench / test author** (writing tests):
1. `README.md` → know what features to test
2. `transaction_api.md` → know what methods the BFM exposes
3. `channel_api.md` → for corner-case tests
4. `active_passive_mode.md` → configure single-/multi-BFM topology
5. `dv/plan.md` → know what testpoints to write
6. `registers.md` (if present) → CSR-driven configuration tests

**DV engineer** (writing scoreboards / assertions):
1. `protocol_rules.md` → 1:1 SVA assertion library
2. `dv/plan.md` → covergroup design
3. `channel_handshake.md` → cross-channel assertions

**SoC integrator / firmware author**:
1. `signal_interface.md` (or `interfaces.md` in behavioral-block) → wire connections
2. `pin_level_reset.md` → power-on assumptions
3. `registers.md` → driver code

---

## Recommended writing order

The skill enforces a writing order that avoids circular rewrites.

### `behavioral-block` mode

1. **README** — Overview + Features (5 minutes; forces the elevator pitch)
2. **`interfaces.md`** — pins down ports, parameters, clocks
3. **`registers.md`** — pins down the programming model
4. **`theory_of_operation.md`** — now you have signals + registers to refer to
5. **`programmers_guide.md`** — now you can write usage flows referencing real registers
6. **`dv/plan.md`** — now you can scope coverage

Writing `theory_of_operation.md` first feels natural (it's the "main" doc), but produces re-rewrites every time a port or register name shifts.

### `protocol-bfm` mode

1. **README** — Overview + protocol + role + active/passive
2. **`signal_interface.md`** — every wire pinned down (drives everything else; channel names declared here flow into protocol-rule IDs)
3. **`pin_level_reset.md`** — every wire's reset value (ties to signal_interface)
4. **`protocol_rules.md`** — rule tables per channel; each rule will become an SVA assertion
5. **`channel_handshake.md`** — cross-channel dependencies; catches deadlock modes
6. **`transaction_api.md`** — high-level test-facing API (~95% of tests)
7. **`channel_api.md`** — phase-level API for the remaining 5%
8. **`active_passive_mode.md`** — capability table + mode knob
9. **`theory_of_operation.md`** — synthesizes both BFM internal architecture and RTL internal architecture (if has-rtl-counterpart)
10. **`dv/plan.md`** — coverage hooks tied to protocol-rule IDs

Do not write `protocol_rules.md` before `signal_interface.md` — protocol-rule IDs reference channel names declared in signal_interface.

---

## Workflow (D0 → D3)

### Phase 1 — Capture (target: D0)

Goal: a complete skeleton (6 files for behavioral-block, 9–11 files for protocol-bfm) with `TODO(designer):` markers wherever a detail isn't yet pinned down.

- **Greenfield path**: `/spec-init <ip_name> [output_dir]` runs an interview that **first asks the IP-class question** (behavioral-block vs protocol-bfm), then mode-specific follow-up questions. Writes `MODE.md` and generates the matching skeleton from templates. Will not invent features the user didn't state.
- **Brownfield path**: `/spec-import <input_path> [output_dir]` (behavioral-block only in v0.3.0) reads existing material and restructures into the 6-file layout. RTL is canonical for ports/parameters/registers/FSM; existing prose is canonical for narrative sections. Conflicts surfaced in `IMPORT_REPORT.md` rather than silently resolved. BFM-mode brownfield: use greenfield + manual population.

Both paths converge to the same Phase 2 / 3 / 4 workflow.

### Phase 2 — Iterate (D0 → D1)

Section by section, in the recommended order. Before writing each file, the skill reads the matching template and any worked example for the chosen mode. Use `/spec-status` to track gate progress; use `/spec-lint` to catch drift early.

### Phase 3 — Reader Test (D1 sign-off)

`/spec-review` spawns the `spec-reader` subagent in a fresh context with read-only access to the spec files. Question bank is mode-matched:

- **`behavioral-block`** → 8–12 questions from `reader_test.md` (universal bank + block-specific bank: DMA / FIFO / configurable / security-critical)
- **`protocol-bfm`** → 8–12 questions from `bfm_reader_test_bank.md` (universal-BFM bank: protocol rules / handshake dependencies / pin-level reset / API completeness / mode coverage / outstanding tracking / ID rules / error injection + block-specific banks: master / slave / passive / ID-bearing / burst / cache-coherent)

PASS requires a quoted spec sentence; NOT_ANSWERED and AMBIGUOUS are gaps the author needs to close.

The subagent's isolation matters: a spec author cannot un-know what they wrote. The reader test exists precisely to expose blindspots that author review cannot reach.

### Phase 4 — Stage gating (D1 → D2 → D3)

`/spec-gate <D1|D2|D3>` walks the checklist for the target gate. Items have stable anchors (e.g. `D1.too.fsm`, `D1.bfm.protocol_rules`) so waivers in `WAIVERS.md` survive across sessions and re-orderings of the checklist. Mode-conditional items are tagged with `**Applies when:**` clauses; items that don't apply to the spec's mode are scored N/A and don't count toward or against gate completion.

---

## Stage gate definitions

| Stage | Definition | Triggered when |
|---|---|---|
| **D0** | Concept; rough shape known | After interview / `/spec-init` |
| **D1** | Spec ~90% complete; sufficient for DV to start the testbench, RTL author to start stubs, C model author to start implementation | Before RTL or BFM implementation begins |
| **D2** | RTL/BFM functional and matches spec; spec frozen except clarifications | Implementation exists, smoke test passes |
| **D3** | Sign-off ready for tape-out integration (RTL) or for wide deployment (BFM) | All reviews complete, lint clean, FPV proven where applicable |

Full per-stage checklists with stable anchors (including mode-conditional `D1.bfm.*`, `D2.bfm.*`, `D3.bfm.*` items) are in [`stage_gates.md`](./plugins/hw-spec-author/skills/hw-spec-author/references/process/stage_gates.md).

There is no "D1 minus." A spec at "D1 with one missing FSM" is at D0 until the missing artifact is supplied or explicitly waived with a one-sentence rationale recorded in `WAIVERS.md`.

---

## Subagent: `spec-reader`

The plugin ships with one subagent at [`agents/spec-reader.md`](./plugins/hw-spec-author/agents/spec-reader.md). It is invoked by `/spec-review` via the Task tool.

- **Tools restricted to** `Read`, `Grep`, `Glob` — no writes, no shell, no web access.
- **Fresh context** — no access to authoring history.
- **Strict isolation rules** — answer only from the spec text; quote the supporting sentence(s); report `NOT_ANSWERED` rather than guess from common sense or domain knowledge of similar IPs.

If you find yourself answering reader-test questions yourself instead of dispatching the subagent, you have re-introduced the exact bias the test is designed to surface.

---

## Worked examples

### `behavioral-block` mode: `wctmr`

[`plugins/hw-spec-author/examples/wctmr/`](./plugins/hw-spec-author/examples/wctmr/) — D1-grade spec for a 64-bit Wallclock Timer (two compare slots, programmable prescaler, APB slave).

Demonstrates: canonical 6-file layout, audience-segregated prose (ToO for HW/DV/SW; programmer's guide for SW; interfaces for SoC integrator), single-source-of-truth discipline, concrete corner-case coverage (counter wrap, software write atomicity, IRQ re-fire after clear-while-true, out-of-range prescale clamping, mid-APB-transaction reset), and a [`READER_TEST_LOG.md`](./plugins/hw-spec-author/examples/wctmr/READER_TEST_LOG.md) recording the actual `/spec-review` run that took the spec from 5 PASS / 5 gaps to **10 / 10 PASS**.

### `protocol-bfm` mode: `axi_lite_slave_bfm`

[`plugins/hw-spec-author/examples/axi_lite_slave_bfm/`](./plugins/hw-spec-author/examples/axi_lite_slave_bfm/) — D1-grade BFM spec for an AXI-Lite slave with `has-rtl-counterpart: yes`.

Demonstrates: 9-file BFM layout, ARM-style protocol-rule rows with severity + ARM SVA equivalent column, channel-handshake Mermaid dependency diagrams, Transaction API + Channel API split, BFM-internal driver/monitor/sequencer architecture in `theory_of_operation.md` §BFM, plus RTL counterpart architecture in `theory_of_operation.md` §RTL with explicit RTL-vs-BFM behavioral equivalence table.

### `protocol-bfm` mode: `apb_slave_bfm`

[`plugins/hw-spec-author/examples/apb_slave_bfm/`](./plugins/hw-spec-author/examples/apb_slave_bfm/) — Portability test of the BFM templates on a no-channel protocol (APB has no AXI-style channel grouping; it's two phases SETUP→ACCESS on a shared signal set).

Demonstrates: rule ID format `<PROTO>_<ROLE>_<NAME>` (3-field variant for channel-less protocols), SETUP/ACCESS phase state machine in `channel_api.md` (per-phase variant of the AXI per-channel state machine), wait-state ASCII timing diagram for `CFG_WAIT_STATES_BOUND` rule.

---

Read the matching example **before** authoring a new spec. Prose tone, table density, and audience separation are easier to imitate from a concrete reference than to derive from templates alone.

---

## Output language

Spec content defaults to **English**. This aligns with industry convention (OpenTitan, ARM AMBA, SiFive, RISC-V references are all in English) and downstream uses (cross-team review, paper, patent). The conversation around the spec can be in any language; only the produced files default to English. State a different language in conversation to override.

---

## Auto-activation

The skill also activates from natural language without a slash command:

- "write a spec for ..."
- "draft a micro-arch spec"
- "design document for the ... module"
- "write a BFM spec for ..."
- "幫我寫一個 ... 的 spec"

Both paths reach the same workflow logic (and ask the IP-class question early).

---

## Design philosophy

The plugin is built around four properties, checked at every gate:

1. **Audience-segregated.** Every section states its primary reader and is written for them. No paragraph mixes audiences.
2. **Single source of truth.** Every fact lives in exactly one place; other places cross-reference.
3. **Stage-gated.** D0 / D1 / D2 / D3 have explicit checklists with stable anchors. No "D1 minus."
4. **Reader-testable.** A fresh-context reader can answer concrete corner-case questions from the spec alone.

For the full rationale, read [`SKILL.md`](./plugins/hw-spec-author/skills/hw-spec-author/SKILL.md) and the documents under [`references/process/`](./plugins/hw-spec-author/skills/hw-spec-author/references/process/). For BFM-mode design rationale specifically, see [`plan/BFM_MODE_DESIGN.md`](./plan/BFM_MODE_DESIGN.md).

---

## Components shipped

- 1 skill: `hw-spec-author` (workflow engine + 13 templates + 5 process docs)
- 1 subagent: `spec-reader` (isolated-context reader for the reader test)
- 7 slash commands: `/spec-init`, `/spec-import`, `/spec-status`, `/spec-review`, `/spec-lint`, `/spec-gate`, `/spec-help`
- 3 worked examples: `wctmr` (behavioral-block), `axi_lite_slave_bfm` (protocol-bfm with RTL counterpart), `apb_slave_bfm` (protocol-bfm portability test on channel-less protocol)

---

## Repository layout

```
hw-spec-author-plugin/
├── .claude-plugin/
│   └── marketplace.json
├── plugins/
│   └── hw-spec-author/
│       ├── .claude-plugin/
│       │   └── plugin.json
│       ├── README.md
│       ├── agents/
│       │   └── spec-reader.md
│       ├── commands/
│       │   ├── spec-init.md
│       │   ├── spec-import.md
│       │   ├── spec-status.md
│       │   ├── spec-review.md
│       │   ├── spec-lint.md
│       │   ├── spec-gate.md
│       │   └── spec-help.md
│       ├── examples/
│       │   ├── wctmr/                          # behavioral-block worked example
│       │   ├── axi_lite_slave_bfm/             # protocol-bfm with RTL counterpart
│       │   └── apb_slave_bfm/                  # protocol-bfm portability test
│       └── skills/
│           └── hw-spec-author/
│               ├── SKILL.md
│               └── references/
│                   ├── process/
│                   │   ├── stage_gates.md
│                   │   ├── reader_test.md           # behavioral-block question bank
│                   │   ├── bfm_reader_test_bank.md  # protocol-bfm question bank
│                   │   ├── writing_principles.md
│                   │   └── rtl_extraction.md
│                   └── templates/
│                       ├── 01_summary.md            # behavioral-block templates
│                       ├── 02_theory_of_operation.md
│                       ├── 03_programmers_guide.md
│                       ├── 04_interfaces.md
│                       ├── 05_registers.md
│                       ├── 06_dv_plan.md
│                       └── bfm/                     # protocol-bfm templates
│                           ├── 02_theory_of_operation.md
│                           ├── 02b_protocol_rules.md
│                           ├── 02c_channel_handshake.md
│                           ├── 02d_pin_level_reset.md
│                           ├── 03b_transaction_api.md
│                           ├── 03c_channel_api.md
│                           ├── 03d_active_passive_mode.md
│                           └── 04b_signal_interface.md
├── plan/
│   └── BFM_MODE_DESIGN.md                      # protocol-bfm mode design history
├── LICENSE
└── README.md
```

---

## License

Apache 2.0. See [LICENSE](./LICENSE).

---

## Acknowledgments

`behavioral-block` mode is heavily inspired by the [OpenTitan / lowRISC Comportability framework](https://opentitan.org/book/doc/contributing/hw/comportability/index.html). Stage-gate concept adapted from the OpenTitan project signoff checklist. `protocol-bfm` mode draws structural ideas from ARM AMBA Protocol Checker User Guides (rule-by-rule format), ARM AMBA AXI Specification §A2 (channel handshake dependencies), UVM agent architecture (driver/monitor/sequencer + active/passive), and OSVVM / Aldec / Cadence three-layer BFM API conventions (signal interface → channel API → transaction API).

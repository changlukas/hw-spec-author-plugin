# hw-spec-author

> **Languages**: [English](./README.md) | [繁體中文](./README.zh-TW.md)

A Claude Code plugin for authoring clean, complete, audience-segregated hardware design specifications for digital IP blocks. Modeled on the [OpenTitan / lowRISC Comportability framework](https://opentitan.org/book/doc/contributing/hw/comportability/index.html).

> **Install**: see the [marketplace top-level README](../../README.md). This document focuses on plugin internals, workflow, and the full command reference.

---

## What this plugin produces

A six-file, audience-segregated hardware spec at the directory you choose:

```
<ip_name>/
├── README.md                       # Summary index — Overview, Features, Description, Compatibility
├── doc/
│   ├── theory_of_operation.md      # Block diagram, datapath, FSM, errors  (HW + DV + SW)
│   ├── programmers_guide.md        # Init, use cases, error handling, IRQs (SW + DV)
│   ├── interfaces.md               # Port table, parameters, IRQs, alerts  (SoC integrator)
│   └── registers.md                # CSR map                               (SW + DV)
└── dv/
    └── plan.md                     # Testpoints, coverage model, sec_cm    (DV)
```

The plugin enforces:

- **Audience separation** per section
- **Single source of truth** discipline (cross-reference, do not duplicate)
- **Stage gates D0 → D3** with explicit checklists and stable anchors
- **Reader testing** for ambiguity (run by an isolated subagent, 8–12 universal questions plus per-block-type banks)
- **OpenTitan-style prose tone** (concrete, declarative, no marketing)

---

## Slash commands (full reference)

### `/spec-init <ip_name> [output_dir]`

Phase 1 (Capture / D0). Runs the Capture interview and generates the 6-file skeleton.

- Asks for IP name, one-sentence purpose, bus interface, clock/reset domains, top-level features (3–8 bullets), prior art if any, and target output directory (default `./spec/<ip_name>/`).
- Reads each template under `references/templates/` before generating its corresponding file.
- Fills in known information from the interview; marks unknowns as `TODO(designer): <what's missing>`.
- Will **not** invent features. If the user did not state it, it stays a TODO.

After completion, it reports where the skeleton was created, how many TODO markers remain (per file), and a recommended next action (typically: fill in `interfaces.md` first).

### `/spec-status [path]`

Inspect an existing spec. Reports the current D-stage and what's missing for the next gate.

- Locates the spec root from the argument or by scanning CWD for a `README.md` + `doc/` pattern.
- Reads all six files (notes any that are missing).
- Walks the D0 → D1 → D2 → D3 checklists from `stage_gates.md` and marks each item ✓ / ✗ / ?.
- Determines current stage as the highest stage where every item is ✓.
- Outputs a structured report with up to 3 recommended next actions and a TODO marker count per file.

Read-only. Does not modify the spec.

### `/spec-review [path]`

Run the reader test (Phase 3, D1 sign-off).

- Reads `references/process/reader_test.md` for the protocol and question bank.
- Locates and inventories the spec.
- Builds an 8–12 question set from the universal bank (reset, CDC, bus, software interaction, IRQs, errors, performance) plus a block-specific bank (DMA / FIFO / configurable / security-critical) chosen from the README features.
- **Spawns the `spec-reader` subagent** via the Task tool with the spec file paths and one question at a time. Critical: the orchestrating Claude does **not** answer the questions itself — that would defeat the test's isolation.
- Aggregates results into PASS / NOT_ANSWERED / AMBIGUOUS, with the spec-reader's quoted-source for each PASS.
- Triages each gap with the user as: in-scope (must add), out-of-scope (must reference), or genuinely undefined (must annotate).
- Optionally applies fixes (with confirmation per edit).

### `/spec-lint [path]`

Mechanical consistency check. Complements `/spec-status` (gate completeness) and `/spec-review` (semantic ambiguity).

The seven lint rules:

| Rule | Check |
|---|---|
| LINT-001 | Broken cross-references (`see Theory of Operation §X`, markdown links, etc.) |
| LINT-002 | Untracked TODO markers — bare `TBD`, `TODO(designer):` without an issue link, undocumented `IMPLEMENTATION-DEFINED:` |
| LINT-003 | Feature ↔ testpoint mapping: every README feature should map to at least one testpoint in `dv/plan.md` |
| LINT-004 | Register name casing — canonical-driven against `registers.md` (case-variants of canonical names only; does not flag generic uppercase tokens) |
| LINT-005 | Port name casing — canonical-driven against `interfaces.md` |
| LINT-006 | Reset value format (`0x...`, `—` for WO, `strap-dependent` allowed; flags bare `0`, "all zero", missing entries) |
| LINT-007 | Stale block-diagram SVG references (file existence) |

Read-only.

### `/spec-gate <D1|D2|D3> [path]`

Stage-gate ceremony. Walks every checklist item for the target gate.

- Verifies that the previous gate is fully met before checking the requested one. (D2 requires D1; D3 requires D2.)
- Reads `<spec_root>/WAIVERS.md` if present; matches each waiver's `## H2` heading against the checklist anchor (e.g. `(id: D2.sec_cm_list)`).
- Marks each item ✓ / ✗ / ⚠ waived. Out-of-band, missing, or misspelled waiver anchors are surfaced as separate report lines.
- Outputs a verdict: PASS (all items ✓ or ⚠) or FAIL with prioritized open items.
- If the user explicitly waives an item, the command asks for who/why/rationale and appends to `WAIVERS.md`. Bare "skip that" is rejected.

Items in `stage_gates.md` carry stable `(id: ...)` anchors — see [`stage_gates.md`](./skills/hw-spec-author/references/process/stage_gates.md). The anchors are the contract; tools never derive anchors from text.

### `/spec-help`

Workflow card. Lists the four phases, all commands, and (if a spec is found in CWD) the suggested next command based on the spec's apparent state.

---

## Auto-activation from natural language

The skill activates without an explicit command when the user's prompt contains:

- "write a spec for ..."
- "draft a micro-arch spec"
- "design document for the ... module"
- "幫我寫一個 ... 的 spec"

Slash commands and natural-language activation reach the same workflow. Use whichever is more natural at the moment.

---

## Workflow

```
Phase 1 — Capture (D0)             /spec-init
   ↓
Phase 2 — Iterate by section       natural conversation, /spec-status, /spec-lint
   ↓
Phase 3 — Reader test (D1)         /spec-review
   ↓
Phase 4 — Stage gates              /spec-gate D1, D2, D3
```

### Recommended writing order within Phase 2

1. **README** — Overview + Features (5 minutes; forces the elevator pitch)
2. **`interfaces.md`** — pins down ports, parameters, clocks
3. **`registers.md`** — pins down the programming model
4. **`theory_of_operation.md`** — now you have signals + registers to refer to
5. **`programmers_guide.md`** — now you can write usage flows referencing real registers
6. **`dv/plan.md`** — now you can scope coverage

Writing `theory_of_operation.md` first is tempting (it's the "main" doc) but produces circular rewrites — every port or register name change forces a rewrite. Avoid.

---

## Subagent: `spec-reader`

Defined at [`agents/spec-reader.md`](./agents/spec-reader.md). Invoked by `/spec-review` via the Task tool.

The subagent runs in a fresh context with `Read`, `Grep`, `Glob` only — no writes, no shell, no web. It is given the spec file paths and one question at a time, with strict rules:

- Answer **only** from the spec text. No common-sense extrapolation. No domain knowledge of similar IPs.
- Quote the supporting sentence(s) for every PASS, citing file path and section.
- Report `NOT_ANSWERED` when the spec is silent and `AMBIGUOUS` when the spec partially answers but does not unambiguously resolve.
- Cross-references count as PASS only if the target section actually contains the answer.

The subagent's isolation is the entire reason the reader test is rigorous. The spec author cannot un-know what they wrote — so the author's reading of the spec fills gaps unconsciously. A fresh subagent cannot.

---

## Worked example: `wctmr`

[`examples/wctmr/`](./examples/wctmr/) is a complete D1-grade spec for a 64-bit Wallclock Timer with two compare slots and a programmable prescaler over an APB slave. It demonstrates:

- The canonical 6-file layout
- Audience-segregated prose (ToO for HW/DV/SW; programmer's guide for SW; interfaces for SoC integrator)
- Single-source-of-truth discipline (registers reference ToO; ToO references interfaces)
- Concrete corner-case coverage: counter wrap, software write atomicity, IRQ re-fire after clear-while-true, out-of-range prescale clamping, mid-APB-transaction reset
- A [`READER_TEST_LOG.md`](./examples/wctmr/READER_TEST_LOG.md) recording the actual `/spec-review` run that took the spec from 5 PASS / 5 gaps to **10 / 10 PASS** — a real dogfood demonstration that the reader test catches what author review misses.

Read the wctmr example **before** authoring a new spec. Prose tone, table density, and audience separation are easier to imitate from a concrete reference than to derive from templates alone.

---

## Stage gates

| Stage | Definition |
|---|---|
| **D0** | Concept; rough shape known. The block has been proposed; one-sentence purpose, three+ feature bullets, bus interface named, clock/reset roughly identified, skeleton files exist. |
| **D1** | Spec ~90% complete. Sufficient for DV to start the testbench and RTL author to start stubs. Reader test has been run and gaps closed. |
| **D2** | RTL functional and matches spec. RTL exists, compiles, elaborates without X propagation; smoke test passes; spec frozen except clarifications. |
| **D3** | Sign-off. DV at V3, lint sign-off-grade, FPV proven where applicable, signed off by HW/DV/SW leads. |

Full per-stage checklists with stable `(id: ...)` anchors are in [`skills/hw-spec-author/references/process/stage_gates.md`](./skills/hw-spec-author/references/process/stage_gates.md).

There is no "D1 minus." A spec at "D1 with one missing FSM" is at D0 until the missing artifact is supplied or explicitly waived in `WAIVERS.md` with a one-sentence rationale.

---

## Output language

Spec content defaults to **English**. This aligns with industry convention (OpenTitan, ARM AMBA, SiFive, RISC-V references are all English) and downstream uses (cross-team review, paper, patent). The conversation around the spec can be in any language; only the produced files default to English. State a different language in conversation to override.

---

## Design philosophy

The plugin is built around four properties:

1. **Audience-segregated.** Every section states its primary reader and is written for them. No paragraph mixes audiences.
2. **Single source of truth.** Every fact lives in exactly one place; other places cross-reference.
3. **Stage-gated.** D0 / D1 / D2 / D3 have explicit checklists with stable anchors. No "D1 minus."
4. **Reader-testable.** A fresh-context reader can answer concrete corner-case questions from the spec alone.

For full rationale, read [`skills/hw-spec-author/SKILL.md`](./skills/hw-spec-author/SKILL.md) and the documents under [`references/process/`](./skills/hw-spec-author/references/process/).

---

## Components shipped

- 1 skill: `hw-spec-author` (workflow engine + 6 templates + 3 process documents)
- 1 subagent: `spec-reader` (isolated-context reader for the reader test)
- 6 slash commands
- 1 worked example: `wctmr`

---

## Plugin layout

```
plugins/hw-spec-author/
├── .claude-plugin/
│   └── plugin.json
├── README.md, README.zh-TW.md
├── agents/
│   └── spec-reader.md
├── commands/
│   ├── spec-init.md
│   ├── spec-status.md
│   ├── spec-review.md
│   ├── spec-lint.md
│   ├── spec-gate.md
│   └── spec-help.md
├── examples/
│   └── wctmr/
│       ├── README.md
│       ├── READER_TEST_LOG.md
│       ├── doc/
│       └── dv/
└── skills/
    └── hw-spec-author/
        ├── SKILL.md
        └── references/
            ├── process/
            │   ├── stage_gates.md
            │   ├── reader_test.md
            │   └── writing_principles.md
            └── templates/
                ├── 01_summary.md
                ├── 02_theory_of_operation.md
                ├── 03_programmers_guide.md
                ├── 04_interfaces.md
                ├── 05_registers.md
                └── 06_dv_plan.md
```

---

## License

Apache 2.0. See [LICENSE](../../LICENSE).

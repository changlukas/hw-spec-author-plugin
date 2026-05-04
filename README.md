# hw-spec-author-plugin

> **Languages**: [English](./README.md) | [繁體中文](./README.zh-TW.md)

A Claude Code plugin marketplace for digital hardware design tooling. The marketplace currently ships one plugin: **`hw-spec-author`** — an opinionated workflow for authoring clean, complete, audience-segregated hardware design specifications for digital IP blocks, modeled on the [OpenTitan / lowRISC Comportability framework](https://opentitan.org/book/doc/contributing/hw/comportability/index.html).

---

## Why this exists

Hardware design specifications drift in predictable ways:

- **Audience mixing.** A single paragraph addresses HW designer, DV engineer, and SW driver author in the same breath, serving none of them well.
- **Duplicated facts.** The same register behavior is described in three files; over six months the three descriptions diverge silently.
- **Implicit assumptions.** "Reset clears everything" — does it clear RAM? Scrambling state? Scrubbing counters? The author knows; the spec doesn't say.
- **No discipline on completeness.** "The spec is mostly done" is not a falsifiable claim. Review meetings devolve into preference votes.
- **Late-found ambiguity.** Bugs surface during V1 testing as "the spec didn't say what should happen here," costing weeks per occurrence.

`hw-spec-author` enforces four properties that, together, eliminate this class of drift:

1. **Audience-segregated.** Every section states its primary reader (HW / DV / SW / SoC integrator) and stays in that lane.
2. **Single source of truth.** Every fact lives in exactly one file; other files cross-reference.
3. **Stage-gated D0 → D3.** Each gate has an explicit checklist with stable anchors. "Done enough for the next phase" is a yes/no question.
4. **Reader-testable.** A fresh-context subagent reads the spec and answers concrete corner-case questions; gaps are mechanical findings, not opinions.

The framework is silicon-proven: OpenTitan uses it across Google, ETH Zurich, Western Digital, Seagate, and Nuvoton.

---

## What's shipped in this marketplace

| Plugin | Description |
|---|---|
| [`hw-spec-author`](./plugins/hw-spec-author/) | Authors hardware design specs for digital IP blocks. OpenTitan-Comportability-style 6-file structure with D0→D3 stage gates and reader-testing. |

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

If the workflow card appears, you're ready.

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

## Quick start

```
/spec-init my_timer
```

This runs the Capture-phase interview (asks for IP purpose, bus interface, clock/reset domains, and 3–8 features), then generates a 6-file OpenTitan-style skeleton at `./spec/my_timer/`. Each file is populated from a template; any unanswered detail becomes a `TODO(designer):` marker rather than a guess.

From there:

| Command | When to run | Effect |
|---|---|---|
| `/spec-status` | Anytime | Reports current D-stage and what's missing for the next gate. Read-only. |
| `/spec-lint` | Anytime in Phase 2 | Mechanical drift check: broken cross-references, untracked TODOs, Feature ↔ testpoint coverage gaps, register/port casing. Read-only. |
| `/spec-review` | Before claiming D1 | Reader test via the `spec-reader` subagent (fresh, isolated context). Asks 8–12 ambiguity-targeting questions; quotes a passing answer or reports a gap. |
| `/spec-gate D1` | When D1 looks ready | Walks every D1 checklist item; persistent waivers in `WAIVERS.md`. |
| `/spec-gate D2` / `D3` | Later in lifecycle | Same protocol for the later gates. |
| `/spec-help` | Anytime | Workflow card + suggested next command based on state. |

---

## What `/spec-init` produces

```
<ip_name>/
├── README.md                   # Summary index — Overview, Features, Description, Compatibility
├── doc/
│   ├── theory_of_operation.md  # Block diagram, datapath, FSM, errors (HW + DV + SW)
│   ├── programmers_guide.md    # Init, use cases, error handling, IRQs (SW + DV)
│   ├── interfaces.md           # Port table, parameters, IRQs (SoC integrator)
│   └── registers.md            # CSR map (SW + DV)
└── dv/
    └── plan.md                 # Testpoints, coverage model, sec_cm (DV)
```

A small block can collapse `theory_of_operation.md` and `programmers_guide.md` into the README; a complex block may split `theory_of_operation.md` across multiple files under `doc/`. Default to the canonical 6-file layout above.

---

## Recommended writing order

The skill enforces a writing order that avoids circular rewrites:

1. **README** — Overview + Features (5 minutes; forces the elevator pitch)
2. **`interfaces.md`** — pins down ports, parameters, clocks
3. **`registers.md`** — pins down the programming model
4. **`theory_of_operation.md`** — now you have signals + registers to refer to
5. **`programmers_guide.md`** — now you can write usage flows referencing real registers
6. **`dv/plan.md`** — now you can scope coverage

Writing `theory_of_operation.md` first feels natural (it's the "main" doc), but produces re-rewrites every time a port or register name shifts.

---

## Workflow (D0 → D3)

### Phase 1 — Capture (target: D0)

Interview-driven skeleton creation. Goal: a complete 6-file scaffold with `TODO(designer):` markers wherever a detail isn't yet pinned down. Run via `/spec-init <ip_name> [output_dir]`. Do **not** invent features the user didn't state; leave a TODO instead.

### Phase 2 — Iterate (D0 → D1)

Section by section, in the recommended order. Before writing each file, the skill reads the matching template (under `references/templates/`) and the `wctmr` worked example. Use `/spec-status` to track gate progress; use `/spec-lint` to catch drift early.

### Phase 3 — Reader Test (D1 sign-off)

`/spec-review` spawns the `spec-reader` subagent in a fresh context with read-only access to the six spec files. It answers 8–12 questions drawn from a universal bank (reset, CDC, bus, IRQ, errors, performance) plus a block-specific bank chosen from the IP category (DMA / FIFO / configurable / security-critical). PASS requires a quoted spec sentence; NOT_ANSWERED and AMBIGUOUS are gaps the author needs to close.

The subagent's isolation matters: a spec author cannot un-know what they wrote. The reader test exists precisely to expose blindspots that author review cannot reach.

### Phase 4 — Stage gating (D1 → D2 → D3)

`/spec-gate <D1|D2|D3>` walks the checklist for the target gate. Items have stable anchors (e.g. `(id: D1.too.fsm)`) so waivers in `WAIVERS.md` survive across sessions and across re-orderings of the checklist.

---

## Stage gate definitions

| Stage | Definition | Triggered when |
|---|---|---|
| **D0** | Concept; rough shape known | After interview / `/spec-init` |
| **D1** | Spec ~90% complete; sufficient for DV to start the testbench, RTL author to start stubs | Before RTL implementation begins |
| **D2** | RTL functional and matches spec; spec frozen except clarifications | RTL exists, smoke test passes |
| **D3** | Sign-off ready for tape-out integration | All reviews complete, lint clean, FPV proven where applicable |

Full per-stage checklists with stable anchors are in [`stage_gates.md`](./plugins/hw-spec-author/skills/hw-spec-author/references/process/stage_gates.md).

There is no "D1 minus." A spec at "D1 with one missing FSM" is at D0 until the missing artifact is supplied or explicitly waived with a one-sentence rationale recorded in `WAIVERS.md`.

---

## Subagent: `spec-reader`

The plugin ships with one subagent at [`agents/spec-reader.md`](./plugins/hw-spec-author/agents/spec-reader.md). It is invoked by `/spec-review` via the Task tool.

- **Tools restricted to** `Read`, `Grep`, `Glob` — no writes, no shell, no web access.
- **Fresh context** — no access to authoring history.
- **Strict isolation rules** — answer only from the spec text; quote the supporting sentence(s); report `NOT_ANSWERED` rather than guess from common sense or domain knowledge of similar IPs.

If you find yourself answering reader-test questions yourself instead of dispatching the subagent, you have re-introduced the exact bias the test is designed to surface.

---

## Worked example: wctmr

[`plugins/hw-spec-author/examples/wctmr/`](./plugins/hw-spec-author/examples/wctmr/) is a complete D1-grade spec for a 64-bit Wallclock Timer — two compare slots, programmable prescaler, APB slave. It demonstrates:

- The canonical 6-file layout
- Audience-segregated prose (ToO for HW/DV/SW; programmer's guide for SW; interfaces for SoC integrator)
- Single-source-of-truth discipline (registers reference ToO; ToO references interfaces)
- Concrete corner-case coverage: counter wrap, software write atomicity, IRQ re-fire after clear-while-true, out-of-range prescale clamping, mid-APB-transaction reset
- A [`READER_TEST_LOG.md`](./plugins/hw-spec-author/examples/wctmr/READER_TEST_LOG.md) recording the actual `/spec-review` run that took the spec from 5 PASS / 5 gaps to **10 / 10 PASS** — a real dogfood demonstration that the reader test catches what author review misses.

Read the wctmr example **before** authoring a new spec. The prose tone, table density, and audience separation are easier to imitate from a concrete reference than to derive from the templates alone.

---

## Output language

Spec content defaults to **English**. This aligns with industry convention (OpenTitan, ARM AMBA, SiFive, RISC-V references are all in English) and downstream uses (cross-team review, paper, patent). The conversation around the spec can be in any language; only the produced files default to English. State a different language in conversation to override.

---

## Auto-activation

The skill also activates from natural language without a slash command:

- "write a spec for ..."
- "draft a micro-arch spec"
- "design document for the ... module"
- "幫我寫一個 ... 的 spec"

Both paths reach the same workflow logic.

---

## Design philosophy

The plugin is built around four properties, checked at every gate:

1. **Audience-segregated.** Every section states its primary reader and is written for them. No paragraph mixes audiences.
2. **Single source of truth.** Every fact lives in exactly one place; other places cross-reference.
3. **Stage-gated.** D0 / D1 / D2 / D3 have explicit checklists with stable anchors. No "D1 minus."
4. **Reader-testable.** A fresh-context reader can answer concrete corner-case questions from the spec alone.

For the full rationale, read [`SKILL.md`](./plugins/hw-spec-author/skills/hw-spec-author/SKILL.md) and the documents under [`references/process/`](./plugins/hw-spec-author/skills/hw-spec-author/references/process/).

---

## Components shipped

- 1 skill: `hw-spec-author` (workflow engine + 6 templates + 3 process docs)
- 1 subagent: `spec-reader` (isolated-context reader for the reader test)
- 6 slash commands: `/spec-init`, `/spec-status`, `/spec-review`, `/spec-lint`, `/spec-gate`, `/spec-help`
- 1 worked example: `wctmr` (64-bit timer with two compare slots, APB slave, programmable prescaler)

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
│       │   ├── spec-status.md
│       │   ├── spec-review.md
│       │   ├── spec-lint.md
│       │   ├── spec-gate.md
│       │   └── spec-help.md
│       ├── examples/
│       │   └── wctmr/
│       │       ├── README.md
│       │       ├── READER_TEST_LOG.md
│       │       ├── doc/
│       │       └── dv/
│       └── skills/
│           └── hw-spec-author/
│               ├── SKILL.md
│               └── references/
│                   ├── process/
│                   │   ├── stage_gates.md
│                   │   ├── reader_test.md
│                   │   └── writing_principles.md
│                   └── templates/
│                       ├── 01_summary.md
│                       ├── 02_theory_of_operation.md
│                       ├── 03_programmers_guide.md
│                       ├── 04_interfaces.md
│                       ├── 05_registers.md
│                       └── 06_dv_plan.md
├── LICENSE
└── README.md
```

---

## License

Apache 2.0. See [LICENSE](./LICENSE).

---

## Acknowledgments

Heavily inspired by the [OpenTitan / lowRISC Comportability framework](https://opentitan.org/book/doc/contributing/hw/comportability/index.html). Stage-gate concept adapted from the OpenTitan project signoff checklist.

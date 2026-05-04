# hw-spec-author

A Claude Code plugin for authoring clean, complete, audience-segregated hardware design specifications for digital IP blocks. Modeled on the [OpenTitan / lowRISC Comportability framework](https://opentitan.org/book/doc/contributing/hw/comportability/index.html).

> **Install instructions are in the [repo top-level README](../../README.md).**

## What it does

Produces deliverable hardware specs across six audience-segregated files:

```
<ip_name>/
├── README.md
├── doc/
│   ├── theory_of_operation.md      # HW + DV + SW
│   ├── programmers_guide.md        # SW + DV
│   ├── interfaces.md               # SoC integrator
│   └── registers.md                # SW + DV
└── dv/
    └── plan.md                     # DV
```

Enforces:
- **Audience separation** per section
- **Single source of truth** discipline (cross-reference, don't duplicate)
- **Stage gates** D0 → D3 with explicit checklists
- **Reader testing** for ambiguity (run by an isolated subagent, 8–12 universal questions plus per-block-type banks)
- **OpenTitan-style prose tone** (concrete, declarative, no marketing)

## Slash commands

| Command | Purpose |
|---|---|
| `/spec-init <ip_name> [output_dir]` | Start a new spec. Runs Capture interview, generates 6-file skeleton. |
| `/spec-status [path]` | Inspect existing spec. Reports current D-stage and what's missing. |
| `/spec-review [path]` | Run reader test (via isolated `spec-reader` subagent). 8–12 ambiguity-targeting questions. |
| `/spec-lint [path]` | Mechanical consistency: cross-references, TODO format, Feature↔testpoint mapping. |
| `/spec-gate <D1\|D2\|D3>` | Check whether spec meets gate criteria. Persistent waivers in `WAIVERS.md`. |
| `/spec-help` | Workflow overview, current state, suggested next command. |

## Auto-activation

The skill also activates from natural language. Phrases that trigger it:

- "write a spec for ..."
- "draft a micro-arch spec"
- "design document for the ... module"
- "幫我寫一個 ... 的 spec"

Slash commands and auto-activation reach the same logic. Use whichever feels more natural per situation.

## Workflow

```
Phase 1 — Capture (D0)            /spec-init
   ↓
Phase 2 — Iterate by section       natural conversation, /spec-status
   ↓
Phase 3 — Reader test (D1)         /spec-review
   ↓
Phase 4 — Stage gates (D1→D2→D3)   /spec-gate D1, D2, D3
```

Recommended writing order within Phase 2:
1. README (Overview + Features) — forces the elevator pitch
2. interfaces.md — pins down ports, parameters, clocks
3. registers.md — pins down programming model
4. theory_of_operation.md — now you have signals + registers to refer to
5. programmers_guide.md — now you can write usage flows referencing real registers
6. dv/plan.md — now you can scope coverage

Writing `theory_of_operation.md` first is tempting but produces circular rewrites — avoid.

## Worked example

[`examples/wctmr/`](./examples/wctmr/) is a complete D1-level spec for a 64-bit Wallclock Timer. It demonstrates:
- The canonical 6-file layout
- Audience-segregated prose (ToO for HW/DV/SW; programmer's guide for SW; interfaces for SoC integrator)
- Single-source-of-truth discipline (registers reference ToO, ToO references interfaces)
- Concrete corner-case coverage (counter wrap, software write atomicity, IRQ re-fire after clear-while-true)
- Useful prose tone: declarative, no marketing, RTL identifiers throughout

Read the wctmr example before authoring a new spec — it grounds the templates in something concrete.

## Design philosophy

The plugin is built around four properties:

1. **Audience-segregated.** Every section states its primary reader and is written for them. No paragraph mixes audiences.
2. **Single source of truth.** Every fact lives in exactly one place. Other places cross-reference.
3. **Stage-gated.** D0 / D1 / D2 / D3 have explicit checklists. No "D1 minus."
4. **Reader-testable.** A reader new to the design can answer concrete corner-case questions from the spec alone.

For details, read [`skills/hw-spec-author/SKILL.md`](./skills/hw-spec-author/SKILL.md) and the `references/process/` documents under it.

## Output language

Spec content defaults to English (industry convention; aligns with OpenTitan, ARM AMBA, SiFive, RISC-V references). The conversation around the spec can be in any language; only the produced spec files are English by default. Override by stating the desired language in the conversation.

## Components shipped

- 1 skill: `hw-spec-author` (workflow engine + references)
- 1 subagent: `spec-reader` (isolated-context reader for the reader test)
- 6 slash commands
- 1 worked example: `wctmr`

## License

Apache 2.0.

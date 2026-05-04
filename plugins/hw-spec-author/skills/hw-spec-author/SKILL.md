---
name: hw-spec-author
description: Authors clean, complete, audience-segregated hardware design specifications for digital IP blocks (RTL modules). Use whenever the user wants to write, draft, scaffold, restructure, or review a hardware design spec, micro-architecture spec, IP datasheet, RTL spec, register spec, or design document for a digital block. Triggers include phrases like "write a spec for", "design spec", "micro-arch spec", "draft a hardware spec", "IP datasheet", "module specification", "RTL specification", or any request to document a hardware module before or during implementation. The skill enforces an OpenTitan-Comportability-style structure (summary / theory of operation / programmer's guide / interfaces / registers / DV plan), staged completion gates (D0→D3), and reader-testing for ambiguity. Use this skill even when the user does not explicitly say "skill" — if they are writing or planning to write a hardware design document, this is the right skill.
---

# Hardware Design Spec Author

A skill for producing clean, complete, audience-segregated hardware design specs for digital IP blocks. Modeled on the OpenTitan/lowRISC Comportability framework, which is silicon-proven and used across Google, ETH Zurich, Western Digital, Seagate, and Nuvoton.

## What "good" means here

A great hw spec has four properties. The whole skill is organized around enforcing them.

1. **Audience-segregated.** Every section states who its primary reader is (HW designer, DV, SW, or SoC integrator) and is written for that reader. No paragraph mixes audiences.
2. **Single source of truth.** A signal, a register, or a parameter is defined in exactly one place. Other sections cross-reference it.
3. **Stage-gated.** The spec has a defined completion level (D0 → D3). The author and reviewers know what "done enough for the next phase" means.
4. **Reader-testable.** The spec can be handed to a reader who has never seen the design, and they can answer concrete questions about reset behavior, error handling, and corner cases without asking the author.

## File structure produced

The skill produces this layout (matching OpenTitan convention so the resulting docs render cleanly in any standard hw doc pipeline):

```
<ip_name>/
├── README.md                       # Summary index — Overview, Features, Description, Compatibility
├── doc/
│   ├── theory_of_operation.md      # Block diagram, datapath, FSM, corner cases (HW + DV + SW audience)
│   ├── programmers_guide.md        # Init, use cases, error handling, IRQs (SW + DV audience)
│   ├── interfaces.md               # Port table, parameters, IRQs, alerts (SoC integrator audience)
│   └── registers.md                # CSR map (SW + DV audience)
└── dv/
    └── plan.md                     # Testpoints, coverage model, sec_cm (DV audience)
```

A small block can collapse `theory_of_operation.md` and `programmers_guide.md` into the `README.md`. A complex block may grow `theory_of_operation.md` into multiple files under `doc/`. Default to the canonical layout above.

## Workflow

The author goes through four phases. Do not skip ahead.

### Phase 1 — Capture (target: D0)

Goal: produce a skeleton with all six files, each containing accurate placeholders, before writing any prose.

Interview the user for:

- IP name and one-sentence purpose
- Bus/host interface (e.g., AXI4 slave, APB, TileLink-UL, custom)
- Clock and reset domains (number of clocks, async/sync resets, retention/AON requirements if any)
- Top-level features as a 3–8 item bullet list
- Any existing register interface this IP must be compatible with (e.g., "16550 UART-compatible")
- Any existing block diagram or rough sketch (ask the user to paste or describe)

If the user has prior art (a draft, a similar IP's spec, scribbled notes), ask for it before drafting. **Do not hallucinate features.** If a detail isn't given, leave a `TODO(designer):` placeholder rather than invent.

After the interview, generate the skeleton files using the templates in `references/templates/`. Read each template before generating its corresponding file.

### Phase 2 — Iterate by section (D0 → D1)

Go section by section. For each section, before writing, read the corresponding template file and follow its structure exactly. Templates are in `references/templates/`:

| When writing... | Read first |
|---|---|
| `README.md` | `references/templates/01_summary.md` |
| `doc/theory_of_operation.md` | `references/templates/02_theory_of_operation.md` |
| `doc/programmers_guide.md` | `references/templates/03_programmers_guide.md` |
| `doc/interfaces.md` | `references/templates/04_interfaces.md` |
| `doc/registers.md` | `references/templates/05_registers.md` |
| `dv/plan.md` | `references/templates/06_dv_plan.md` |

Recommended order of writing:
1. `README.md` Overview + Features (5 minutes — forces the elevator pitch)
2. `interfaces.md` (forces commitment on ports, parameters, clocks)
3. `registers.md` (forces commitment on programming model)
4. `theory_of_operation.md` (now you have signals + registers to refer to)
5. `programmers_guide.md` (now you can write usage flows referencing real registers)
6. `dv/plan.md` (now you can scope coverage)

Do not write `theory_of_operation.md` first. The temptation is strong because it's the "main" doc, but writing it before pinning down ports and registers leads to circular rewrites.

### Phase 3 — Reader Test (D1 sign-off)

Once the spec reaches D1 completeness, run the reader-test protocol described in `references/process/reader_test.md`. Briefly: act as a reader who has never seen the design, ask 8–12 concrete questions targeting reset, error, IRQ priority, write-during-busy, CDC, and corner cases, and try to answer them from the spec alone. Each question that can't be answered or is ambiguous = a fix.

Read `references/process/reader_test.md` before running this phase.

### Phase 4 — Stage gating (D1 → D2 → D3)

Spec completeness is gated. Each gate has explicit checklist items. Read `references/process/stage_gates.md` for the D0/D1/D2/D3 criteria. Do not claim a stage is reached unless every box is ticked or explicitly waived with a note.

## Universal writing principles

Read `references/process/writing_principles.md` for tone, prose conventions, naming, and anti-patterns. Apply on every paragraph, not just at the end.

Three rules to internalize before writing anything:

1. **Name things by their RTL identifier.** Write `TLUL host port`, `INTR_ENABLE register`, `cfg_q[7:0]` — not "the host bus" or "the enable bit." Match the case used in RTL.
2. **No unstated assumptions.** Every behavior must be reachable from a sentence in the spec. If the user has to "just know" that writes are dropped during reset, that sentence is missing.
3. **Cross-reference, don't duplicate.** A register's behavior lives in `theory_of_operation.md`; the register table in `registers.md` says "see Theory of Operation §X." A signal's protocol lives in `interfaces.md`; the FSM description references the signal name without redefining it.

## Reference index

```
references/
├── templates/
│   ├── 01_summary.md
│   ├── 02_theory_of_operation.md
│   ├── 03_programmers_guide.md
│   ├── 04_interfaces.md
│   ├── 05_registers.md
│   └── 06_dv_plan.md
└── process/
    ├── stage_gates.md
    ├── reader_test.md
    └── writing_principles.md
```

Read templates as needed when writing the corresponding section. Read process files at the appropriate phase transitions.

## Worked example

A complete worked example exists at `${CLAUDE_PLUGIN_ROOT}/examples/wctmr/` — a 64-bit Wallclock Timer with two compare slots, APB slave, programmable prescaler. The example demonstrates the canonical 6-file layout, audience-segregated prose, single-source-of-truth cross-references, and the level of concrete detail expected at D1.

When writing a new spec, **read the wctmr example before drafting any section**. The prose tone, table formats, and density are easier to imitate than to derive from the templates alone. Specifically:

- For `theory_of_operation.md` style: see `examples/wctmr/doc/theory_of_operation.md`. Note how block diagram, datapath, FSM/control logic, resets, errors, and performance each get a dedicated heading at the same level.
- For `registers.md` density: see `examples/wctmr/doc/registers.md`. Each register has its own subsection with field-level table and per-field cross-reference to ToO.
- For `programmers_guide.md` use-case format: see `examples/wctmr/doc/programmers_guide.md`. Note the Precondition / Action / Observable result / Failure mode structure for each use case.

## Output language

Default to **English** for the spec content itself. English aligns with industry convention (OpenTitan, ARM AMBA, SiFive, RISC-V specs are all English) and downstream uses (paper, patent, cross-team review). The skill instructions and conversation can be in whatever language the user uses; only the produced spec files are English-by-default. If the user explicitly requests another language for the spec content, follow that.

## Slash commands (Claude Code plugin)

When this skill is installed as part of the `hw-spec-author` Claude Code plugin, the following slash commands are available. They map onto the four phases above:

| Command | Phase | Purpose |
|---|---|---|
| `/spec-init <ip_name> [output_dir]` | Phase 1 | Run the Capture interview, generate the 6-file skeleton |
| `/spec-status [path]` | Phase 2 | Inspect current spec, report current D-stage, list open items |
| `/spec-review [path]` | Phase 3 | Run reader test via the `spec-reader` subagent (isolated context); produce ambiguity-gap report |
| `/spec-lint [path]` | any | Mechanical consistency check: broken xrefs, untracked TODOs, Feature↔testpoint mapping, register/port casing |
| `/spec-gate <D1\|D2\|D3>` | Phase 4 | Check stage-gate checklist; persistent waivers in `WAIVERS.md` |
| `/spec-help` | any | Workflow overview, current state, suggested next command |

Commands are explicit invocations; the skill itself activates ambiently from natural language. Both paths reach the same underlying logic — commands just give the user a predictable handle when they want one.

The plugin also provides a `spec-reader` subagent (under `agents/`) which is invoked by `/spec-review` via the Task tool. The subagent is given only `Read`, `Grep`, `Glob` access — no writes, no shell — and runs in fresh context so it cannot draw on authoring history when answering reader-test questions. This isolation is what makes the reader test rigorous.

## Diagram formats

Acceptable diagram formats, in order of preference:

1. **Mermaid** — renders inline in Markdown viewers (GitHub, GitLab, VS Code preview, Claude Code). Best for FSM state diagrams, dependency graphs, CDC paths, hierarchies. Use ` ```mermaid ` fenced blocks.
2. **SVG** — for hand-drawn or tool-exported block diagrams. Higher visual fidelity, version-controllable.
3. **WaveJSON / WaveDrom** — for timing diagrams. Renders in supported viewers.
4. **ASCII fallback** — when none of the above are available, ASCII boxes-and-arrows are acceptable. Ugly but unambiguous.

Within a single spec, mix formats based on what each diagram needs (Mermaid for FSM, SVG for block diagram, WaveJSON for timing) — there is no requirement to be uniform.

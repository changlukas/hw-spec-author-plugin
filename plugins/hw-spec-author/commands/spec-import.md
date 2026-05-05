---
description: Import an existing hardware spec (Markdown/text) and/or RTL source (.sv/.v) and restructure into the OpenTitan-Comportability 6-file behavioral-block layout. Mechanically extracts ports/parameters/registers/FSM from RTL; preserves prose from existing docs. **Behavioral-block mode only** — for protocol-bfm specs, use /spec-init protocol-bfm instead (BFM brownfield import is a known limitation).
argument-hint: <input_path> [output_dir]
allowed-tools: Read, Write, Glob, Grep
---

You are running `/spec-import`. The user has an existing artifact — a prior spec in any format, RTL source, or both — and wants a draft of the canonical 6-file layout populated from that material.

This command is **not** a writing-from-scratch tool. It is a restructuring + extraction tool. The output is a *draft* that the user shepherds through Phase 2 → Phase 3 → Phase 4 of the hw-spec-author workflow as if it were a freshly initialized spec.

## Known limitation: BFM mode

`/spec-import` produces only `behavioral-block` mode skeletons. BFM-style RTL — pure interface modules, SystemVerilog `interface` + `modport` declarations, modules with no software-visible register file, modules dominated by pure assign/wire-routing logic rather than sequential state — will be force-fitted into the 6-file behavioral layout and produce an unusable draft.

For BFM specs, run `/spec-init <name>` and select `protocol-bfm`, then manually populate `signal_interface.md` from the existing RTL ports. Reverse-extracting protocol rules from `modport` direction signs and SVA assertion bodies is a v2 direction; not in v1 scope.

If `/spec-import` heuristically detects the input is BFM-style (see Step 3), it stops and emits the redirect prompt instead of generating a misshapen skeleton.

## Steps

1. **Read the skill and the extraction reference**:
   - `${CLAUDE_PLUGIN_ROOT}/skills/hw-spec-author/SKILL.md` for the workflow.
   - `${CLAUDE_PLUGIN_ROOT}/skills/hw-spec-author/references/process/rtl_extraction.md` for what is and is not extractable from RTL.
   - `${CLAUDE_PLUGIN_ROOT}/skills/hw-spec-author/references/process/writing_principles.md` §6 for the canonical `TODO(designer):` format. Every `TODO(designer):` emitted by this command MUST end in `(see #N)` or `(no issue yet — <reason>)` per §6 — free-form rationale (e.g. `confirm — assumed default`) is rejected by `/spec-lint` LINT-002 and `/spec-gate` `D1.cross.todos_tracked` and forces rework at gate time. Default to `(no issue yet — <reason>)` when no issue tracker is supplied; the reason can be as short as `(no issue yet — extraction placeholder)` for mechanical-extraction gaps.
   - The 6 templates under `${CLAUDE_PLUGIN_ROOT}/skills/hw-spec-author/references/templates/` for the target structure.

2. **Parse the invocation** `/spec-import <input_path> [output_dir]`:
   - `<input_path>` (required) is a file or directory containing the source material.
   - `[output_dir]` (optional) is where the 6-file spec is written. Default: `./spec/<inferred_ip_name>/`.
   - Infer `<ip_name>` from the input: top-level RTL module name if RTL is present; else first-heading or filename if doc-only; else ask the user.

3. **Detect the input mode**. Glob `<input_path>` recursively:
   - **`.sv` / `.v` files present** → RTL mode (extract via `rtl_extraction.md` heuristics)
   - **`.md` / `.txt` files present** → doc mode (restructure prose into the 6-file layout)
   - **Both present** → combined mode (RTL canonical for tables; doc canonical for prose; cross-validate)
   - **`.hjson` register definitions** present → use as canonical for `registers.md`
   - **Verilator XML** (`*.xml` with `<module>` elements) → use as structural index instead of native LLM parsing of `.sv`

   ### 3a. BFM-style RTL screening (before proceeding)

   Before generating a behavioral-block skeleton, screen the input RTL for BFM-style indicators. If any of the following are true, stop and emit the redirect prompt:

   - Input contains SystemVerilog `interface` declarations with `modport` definitions and **no** module-level RTL with `always_ff` blocks writing software-visible state.
   - Input contains modules whose port list is dominated by valid/ready handshake signals (e.g., `*VALID`/`*READY` pairs) AND whose body is dominated by pure `assign` / wire-routing statements rather than sequential logic.
   - No hjson register file is present AND no `always_ff` block in any input module assigns to a software-visible register array.

   Emit this message and stop:

   ```
   /spec-import detected BFM-style RTL: <one-line reason — e.g., "interface + modport
   only, no sequential state assignments to software-visible registers".>

   /spec-import currently produces only behavioral-block mode skeletons. BFM-style
   RTL force-fitted into that layout produces an unusable draft.

   Recommended action:
     1. Run /spec-init <name> and select protocol-bfm.
     2. Manually populate doc/signal_interface.md from the existing RTL ports.
     3. Iterate the rest of the BFM spec following Phase 2 of the workflow.

   If you believe this detection is a false positive (the RTL is a real
   behavioral block that happens to look BFM-shaped), re-run with --force to
   bypass screening: /spec-import --force <input_path> [output_dir]
   ```

   Detection should err toward false negatives (proceed with import) over false positives (block valid behavioral-block input). When in doubt, proceed and let the user catch the mismatch via the IMPORT_REPORT.

   ### 3b. Confirm detected mode

   Tell the user which input mode you detected and what files are in scope. Wait for confirmation before proceeding.

4. **Build the skeleton**. Create the canonical layout at `<output_dir>` using the templates as scaffolding:

```
<output_dir>/
├── MODE.md
├── README.md
├── doc/
│   ├── theory_of_operation.md
│   ├── programmers_guide.md
│   ├── interfaces.md
│   └── registers.md
└── dv/
    └── plan.md
```

Write `MODE.md` with `mode: behavioral-block` (the only mode `/spec-import` produces; BFM-mode input was rejected at step 3a if detected). Include `created: <today's ISO date>` and `spec-author-plugin-version: <plugin version>` per the MODE.md format defined in SKILL.md.

5. **Extract** based on the detected mode (see sections below). For every extracted item, append a **provenance comment** in the produced markdown:

```markdown
| psel_i | input | 1 | APB select |
<!-- source: rtl/wctmr.sv:14 -->
```

Provenance comments make later audits cheap: the reader can jump from any spec table row to the originating RTL line in one step.

6. **Generate `<output_dir>/IMPORT_REPORT.md`**. This artifact is the audit trail of the import. Its structure is in the **IMPORT_REPORT format** section below.

7. **Run `/spec-status` automatically** after the import completes, and surface the result inline. The current D-stage will almost always be `pre-D0` (skeleton not yet at D0) or `D0` if every D0 item happens to be present. The user uses this to gauge how much manual work remains.

8. **Recommend next actions** in the order:
   1. Triage the high-priority TODOs in the IMPORT_REPORT (typically: README Features, performance commitments, programmer's guide use cases).
   2. Run `/spec-lint` to surface mechanical drift introduced by the import.
   3. Run `/spec-review` once the spec reaches D1-completeness candidate.
   4. `/spec-gate D1` to formalize.

## Mode-specific extraction

### RTL mode

For each `.sv`/`.v` file in scope:

#### a. Module port list → `interfaces.md`

Parse the `module ... ( ... );` header. Bucket each port into one of the canonical interface tables based on naming convention (see `rtl_extraction.md` §Naming-convention assumptions). Extract direction, width, and signal name verbatim from RTL. **Never invent descriptions** — leave a `TODO(designer):` for any signal whose semantic role is not explicit in its name.

#### b. Parameters → `interfaces.md` Parameters table

Extract name, type, and default. Constraint columns get `TODO(designer):`.

#### c. Reset block → `theory_of_operation.md` §Resets and `registers.md` Reset column

Find every `always_ff @(posedge clk_* or negedge rst_*ni)` block. Extract the assignments under the `if (!rst_ni)` branch. Map RTL `*_q` register names to spec register names (best effort: `ctrl_q` → `CTRL`).

#### d. FSM → `theory_of_operation.md` §Control/FSM

Find every `typedef enum` block whose values are state names. Extract states verbatim. For each FSM:
- Emit a Mermaid `stateDiagram-v2` skeleton with `[*] --> <reset_state>` (where `<reset_state>` is the value assigned in the reset block).
- Emit a transition table by tracing `state_d` assignments in `always_comb` / `always_ff`. Each row: From state, To state, Condition (RTL expression). Tag the table `[draft — verify transitions against design intent]`.

#### e. Datapath → `theory_of_operation.md` §Block Diagram and §Datapath

Extract module instantiations to a Mermaid `flowchart TB` skeleton. Tag as `[draft]`. The Datapath prose is left as `TODO(designer):` — datapath narrative is design intent, not extractable from instance graph alone.

#### f. SVA → `dv/plan.md` ABV section

Extract assertion name and property body.

#### g. hjson register definitions

If present (e.g., under `data/<ip>.hjson`), parse and emit the full register map directly to `registers.md`. This is the highest-fidelity path and overrides any inference from the RTL itself.

### Doc mode

For each input doc:

#### a. Heading-based section classification

Read the doc top to bottom. For each section heading, classify into one of the 6 target files based on content keywords:

| Keywords / patterns in section | Target file |
|---|---|
| "Overview", "Introduction", "Features", "Compatibility" | `README.md` |
| "Block diagram", "Datapath", "FSM", "State machine", "Reset", "Clock", "CDC", "Errors", "Performance" | `doc/theory_of_operation.md` |
| "Initialization", "Use case", "Driver", "Software flow", "Interrupt servicing", "Polling" | `doc/programmers_guide.md` |
| "Ports", "Signals", "Parameters", "Bus", "Sideband", "Pin" | `doc/interfaces.md` |
| "Register map", "CSR", "Field", "Offset", "Address" | `doc/registers.md` |
| "Verification", "Testpoint", "Coverage", "Testbench", "DV" | `dv/plan.md` |

Sections that don't classify cleanly: leave as `TODO(designer): classify section "<heading>"` block at the end of the most likely target file.

#### b. Audience separation triage

For each prose paragraph, identify the implied audience(s):
- Mentions registers / driver flow → SW reader
- Mentions ports / parameters / clocks → SoC integrator
- Mentions FSM / pipeline / RTL identifiers → HW designer
- Mentions tests / coverage / scoreboards → DV engineer

Paragraphs addressing multiple audiences get **split** into per-audience prose, each landing in the right file. The IMPORT_REPORT records every split with a "before/after" snippet so the user can verify.

#### c. Duplicate-fact detection

For every register name, port name, or parameter mentioned in the source doc:
- Note all locations of mention.
- Pick the canonical home per `writing_principles.md` §2 (Single source of truth):
  - Register offset/width/reset/access → `registers.md`
  - Register field *behavior* → `theory_of_operation.md`
  - Port direction/width → `interfaces.md`
  - FSM state list → `theory_of_operation.md`
- At every non-canonical mention, replace with a cross-reference: "see Registers §X" / "see Theory of Operation §Y".
- IMPORT_REPORT lists every canonicalization so the user can verify the choice.

### Combined mode (RTL + doc)

Build the RTL-extracted skeleton first. Then layer the doc-extracted prose on top:

1. RTL extraction is **canonical** for: ports, parameters, register offsets/widths/reset values, FSM state names, signal connectivity.
2. Doc extraction is **canonical** for: prose tone, programmer's guide use cases, performance claims, error semantics not extractable from RTL.
3. **Conflict resolution**: when RTL and doc disagree on extractable data (e.g., RTL says register CTRL is at offset 0x00 but doc says 0x04), record both in the IMPORT_REPORT under "Conflicts" and emit the RTL-derived value into the spec with a `<!-- conflict: doc says 0x04 — please reconcile -->` comment. Do **not** silently pick one.

## IMPORT_REPORT format

Write `<output_dir>/IMPORT_REPORT.md`:

```markdown
# Import Report — <ip_name>

Generated by `/spec-import` on <date>. This file documents what was extracted, with what confidence, and what the designer must still review.

## Source manifest

| File | Lines | Last modified | Used for |
|---|---|---|---|
| rtl/wctmr.sv | 312 | 2026-04-30 | interfaces.md (ports), theory_of_operation.md (FSM, resets), registers.md (reset values) |
| docs/wctmr_old.md | 540 | 2026-03-12 | README prose, programmer's guide use cases |
| ... | | | |

## Extraction summary

| Spec file | High-confidence items | Medium-confidence | Low-confidence (TODO) |
|---|---|---|---|
| README.md | 0 | 1 (block one-liner) | 4 (Features, Description, Compatibility, etc.) |
| doc/theory_of_operation.md | 12 (resets, FSM states) | 6 (FSM transitions, datapath) | 3 (performance, sec_cm) |
| ... | | | |

## Confidence legend

- **High**: 1-to-1 mapping from RTL syntax to spec content. Verified against source line.
- **Medium**: pattern-matched, may have false positives or miss cases. The designer should review every medium-confidence row.
- **Low / TODO**: not extractable from source; designer must fill in.

## Conflicts (RTL vs doc)

(Empty if no conflicts. Otherwise:)

### CTRL register offset

- **RTL** (rtl/wctmr.sv:148): `localparam CTRL_OFFSET = 6'h00;` → 0x00
- **Doc** (docs/wctmr_old.md §3.1): "CTRL is at offset 0x04"
- **Emitted**: 0x00 (from RTL)
- **Action required**: confirm correct value; update doc or RTL accordingly.

## TODO inventory

| File | TODO count |
|---|---|
| README.md | 4 |
| doc/theory_of_operation.md | 3 |
| doc/programmers_guide.md | 6 |
| doc/interfaces.md | 1 |
| doc/registers.md | 0 |
| dv/plan.md | 4 |

## Recommended next actions

1. Triage TODOs in README.md first — Features and Description gate everything downstream.
2. Resolve any conflicts above before considering D1.
3. Run `/spec-lint <output_dir>` to catch mechanical drift introduced by the import.
4. Run `/spec-status <output_dir>` to see the current D-stage.
5. Reach D1-completeness candidate, then run `/spec-review` for the reader test.

## Provenance index

Every extracted table row has a `<!-- source: <file>:<line> -->` comment in the produced spec. Use these to trace any extracted value back to its origin without re-running the import.
```

## Constraints

- **Do not fabricate.** If the source does not state a value, the produced spec gets a `TODO(designer):` — never a guess. This is the same rule as `/spec-init`; it is doubly important here because import is meant to handle large source material where fabrication can hide.
- **RTL is canonical for structure; doc is canonical for prose.** When in doubt, prefer the RTL-extracted value for any datum that has a syntactic representation (offset, width, signal name); prefer doc prose for any narrative.
- **Provenance is mandatory.** Every extracted table row must have a source-line comment. The IMPORT_REPORT is not a substitute — it summarizes; the inline comments enable line-level audit.
- **Do not claim a D-stage.** The output of `/spec-import` is at most a D0 skeleton; usually pre-D0 because audience-separation and single-source-of-truth haven't yet been verified by the user. Run `/spec-status` to find out and report it; do not editorialize.
- **Honest scope statement to the user**: at the end of the import, plainly tell the user what fraction of the spec was mechanically extracted vs. left as TODO. This sets expectations for the manual work ahead.
- **Read-modify only the output directory.** Never write into `<input_path>`; the source material is read-only as far as this command is concerned.
- **Idempotency**: if `<output_dir>` already exists and contains a previous import, do **not** silently overwrite. Stop and ask the user whether to overwrite, merge into the existing skeleton, or write to a new directory. If overwriting, archive the previous IMPORT_REPORT under `<output_dir>/.import_history/<timestamp>.md` for traceability.

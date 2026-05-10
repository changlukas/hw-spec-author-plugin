# Process: Writing Principles

These principles apply to every paragraph of every spec file. They are the difference between a spec that produces clean RTL and a spec that produces an integration nightmare.

## 1. Audience separation

Every section has a primary audience. State it (mentally or explicitly) before writing. Match the level.

| Audience            | Cares about                                                              | Doesn't care about                              |
|---------------------|--------------------------------------------------------------------------|-------------------------------------------------|
| HW designer         | Internal datapath, FSMs, timing, CDC, exact RTL identifiers              | Software API, register-write code fragments     |
| DV engineer         | Every named behavior, every error condition, every corner case           | Marketing-level descriptions                    |
| SW engineer         | Register layout, init sequence, IRQ handling, error responses            | RTL pipeline depth (unless it affects timing)   |
| SoC integrator      | Port list, clocks, resets, parameters, bus protocol                      | Internal FSM details                            |

A paragraph that mixes audiences fails all of them. If you find yourself writing "for software, … and for hardware, …" in one paragraph, split the section.

## 2. Single source of truth

Every fact about the design lives in exactly one place. Other places reference it.

| Fact                          | Lives in                              | Other places say                          |
|-------------------------------|---------------------------------------|-------------------------------------------|
| Register offset, width, reset | `registers.md` per-register sub-section | "see Registers §X"                       |
| Register field *behavior*     | `theory_of_operation.md`              | `registers.md` says "see ToO §Y"          |
| Port name, direction, width   | `interfaces.md`                       | Other docs reference by name              |
| Init sequence                 | `programmers_guide.md` Initialization | DV plan says "test the documented init order" |
| FSM state list                | `theory_of_operation.md` Control / FSM | DV plan testpoints reference state names  |

If you find the same fact stated in two files, one of them is wrong (eventually). Pick a home and cross-reference.

## 3. Concrete naming

Always name things by their RTL identifier, in the case used in RTL.

- ✅ `INTR_STATE.done`, `s_axi.awvalid`, `tx_ctrl_fsm`, `wdata_fifo`
- ❌ "the done bit", "the AXI write address valid signal", "the main state machine", "the transmit buffer"

The reader should be able to grep the RTL for any name in the spec.

This rule has one exception: when introducing a concept for the first time in a Description paragraph, prose names are acceptable. From the second mention on, use the RTL name.

## 4. No unstated assumptions

Every behavior the design exhibits must be reachable from a sentence in the spec. The test: hand the spec to a reader, ask any question about the block's response to any input, and the answer must be in the spec — not "obvious", not "follows from common sense", not "you can figure it out from the diagram."

The reader test (see `reader_test.md`) is the systematic way to enforce this.

Common unstated assumptions to watch for:

- "Reset clears everything" → which everything? RAM too? Scrambling state? Scrubbing counters?
- "Software polls until done" → polls *what register, what bit*?
- "Errors are reported" → reported *where, with what bit, with what mechanism*?
- "Standard handshake" → AMBA standard? Custom? Ready-before-valid or any-order?

## 5. Cross-reference, don't duplicate

When a fact is in another section of the spec, link to it. Don't restate.

- **Bad:** Both `theory_of_operation.md` and `programmers_guide.md` describe the init sequence. They drift.
- **Good:** `programmers_guide.md` Initialization is the home; `theory_of_operation.md` says "Software-visible initialization sequence is in Programmer's Guide §X. Internally, this corresponds to the ctrl_fsm transition IDLE → READY described above."

## 6. Errata and TBDs are tagged, never silent

A `TODO(designer):` is acceptable in a draft. A bare `TBD` is not. Every `TODO(designer):` MUST use exactly one of two canonical formats:

- `TODO(designer): <text> (see #N)` — for tracked work with an issue-tracker link.
- `TODO(designer): <text> (no issue yet — <reason>)` — for sandbox specs, pending design decisions, future spec revisions, or any case where no issue tracker is in scope.

Both formats end with the parenthesised tracking annotation. Free-form rationale (e.g. trailing dash + reason without parentheses) does not satisfy this rule, even when the rationale is informative — `/spec-lint` LINT-002 and `/spec-gate` `D1.cross.todos_tracked` both parse the parenthesised form literally.

Examples:

- ✅ `TODO(designer): performance commitment for back-to-back reads (see #42).`
- ✅ `TODO(designer): cross-link to router-spec ordering once router DV publishes (no issue yet — router spec under separate ownership).`
- ✅ `TODO(designer): formula for CDC FIFO depth (no issue yet — current 16-entry default is conservative for ratios in [0.1, 10]; formula deferred until ratio range is locked).`
- ✅ `IMPLEMENTATION-DEFINED: order of IRQ servicing when two sources fire same cycle. Software treats as undefined.`
- ❌ `TODO(designer): confirm — performance assumed 1-cycle.`  (free-form rationale; not parenthesised)
- ❌ `TODO(designer): performance commitment for back-to-back reads. Awaiting synthesis numbers.`  (no issue link and no `(no issue yet — …)` annotation)
- ❌ `TBD`
- ❌ "More details to follow."

When generating a spec from scratch (`/spec-init`), from existing material (`/spec-import`), or while iterating in conversation, prefer the `(no issue yet — <reason>)` form unless the user has already committed to an issue tracker and supplied an issue number.

## 7. Prose tone

Match the tone of high-quality reference specs (OpenTitan, RISC-V ISA spec, ARM AMBA spec):

- **Declarative.** "The block raises `intr_done_o` when the operation completes." Not "When the operation completes, the block will raise `intr_done_o`."
- **No hedging.** "If the FIFO is full, the write is silently dropped." Not "Generally, if the FIFO is mostly full, the write may be dropped."
- **No marketing.** "Sustains 1 transaction per cycle on the AXI write channel." Not "High-performance design with optimized throughput."
- **Active voice for hardware actions.** "The arbiter selects the request with the highest priority." Not "The request with the highest priority is selected by the arbiter."
- **Passive voice is acceptable** when the actor is software and the focus is on the hardware effect: "When CTRL.enable is written 1, the block transitions to ACTIVE." (The actor "software" is implied and not the focus.)

## 8. Formatting conventions

- Code: backticks for inline names (`CTRL.enable`), fenced blocks for code fragments.
- Tables for any list of items with multiple attributes (registers, signals, parameters, testpoints).
- Bullet lists for short, attribute-less enumerations (features, valid modes, states).
- Block diagrams: **Mermaid preferred** (renders inline in GitHub/GitLab/VS Code), SVG also acceptable, ASCII fallback when neither is available.
- FSM state diagrams: **Mermaid `stateDiagram-v2`** preferred, transition tables required regardless (tables are greppable, diagrams are not).
- Dependency / hierarchy diagrams: **Mermaid `flowchart TB`** preferred.
- Timing diagrams: WaveJSON or WaveDrom preferred, ASCII fallback.
- File path references: relative paths from spec root, `./doc/theory_of_operation.md` style.

## 9. Don't fabricate

If a number is unknown, write `TODO(designer): ...`. Do not write a plausible-sounding number to make the spec look complete. A spec full of fabricated numbers is worse than a spec full of TODOs because the fabrication won't be questioned in review.

This applies especially to:

- Pipeline depths
- Latency cycles
- Reset values
- Maximum throughputs
- FIFO depths
- Address ranges

If the user gives a rough estimate, mark it: `Pipeline depth: 3 cycles (estimate, to be confirmed in micro-architecture).` That's honest. A bare "Pipeline depth: 3" reads as committed.

## 10. Examples earn their place

A worked example (use case in programmer's guide, mini block diagram, sample register encoding) is worth more than three paragraphs of abstract description. But examples must be correct and consistent with the rest of the spec. An incorrect example is worse than no example because it teaches the wrong thing.

Cross-check every example against the relevant sections before submitting.

## 11. Default to minimum sufficient

When in doubt about adding a clause, mechanism, or edge-case handler the user did not explicitly request, omit it. The cost of a missing clause is one round-trip to add it. The cost of an unrequested clause is much higher: it ships, gets verified, gets locked in, and removing it later is expensive.

Before adding any unrequested feature, write one line: "Adding this because <reason>." If the reason is weak, drop the addition.

Common offenders to watch for:

- Reserved-encoding forward-compat clauses ("encoding 3 is Reserved; BFM accepts as Hybrid")
- Debug-only acceptance paths ("mode=2 SHOULD be avoided in production but no SLVERR")
- CDC mechanism choices the user did not specify (req/ack vs 2FF vs gray-counter)
- Splitting one register into CTRL/STATUS pair where the user wrote one
- Severity bumps (RECOMMEND ↔ FAIL) on existing rules
- Backwards-compatibility shims for unreleased revisions
- Race-semantic and liveness clauses beyond what the user asked for

Counter-pattern: when the spec genuinely needs a constraint to be unambiguous (e.g., "what happens on AXI ID collision?"), state it. The rule is no *unrequested complexity*, not *no rigor*.

Worked example and rationale: see `plan/DOGFOOD_OBSERVATIONS_A5.md` §1.

## 12. No semicolons in narrative prose

Avoid semicolons (`;`) in spec narrative prose. Split into two sentences or use a bullet list.

- ✅ "The arbiter picks one VC per cycle. Wormhole-lock holds per `NOC_FLIT_VC_HARDLOCK`."
- ❌ "The arbiter picks one VC per cycle; wormhole-lock holds per `NOC_FLIT_VC_HARDLOCK`."

Reason: spec readers skim. Two short sentences scan more easily than one compound sentence chained by a semicolon.

Exception: semicolons inside dense table cells (where line breaks would distort the layout). Code blocks (SystemVerilog, C++, etc.) unaffected.

## 13. External attribution requires verbatim citation

When the spec attributes a design choice to an external standard (AMD pg313, ARM AMBA AXI, RISC-V, FlooNoC, etc.), include a verbatim quote with section reference. Do not summarize what the standard "essentially says" — paraphrases drift across multi-round editing.

Format:

> *"<verbatim quote from external source>"* — <source name> §<section>

Example:

> *"The NPP packet (DST ID + LAST) field is also protected by 1-bit even parity."* — AMD pg313 §Parity

Failure mode this prevents: across multi-round editing, "AMD says X" drifts to "AMD-aligned X" to "industry-standard X". Eventually X no longer matches the source. Verbatim quotes are the mechanical safeguard.

Worked example: see `plan/DOGFOOD_OBSERVATIONS_A5.md` §4.

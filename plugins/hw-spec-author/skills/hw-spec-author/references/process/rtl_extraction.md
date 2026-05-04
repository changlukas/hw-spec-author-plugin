# Process: RTL Extraction

`/spec-import` can read SystemVerilog/Verilog source and produce a draft spec. This document defines what is mechanically extractable, what is heuristic, and what is structurally impossible to extract — so the produced spec is honest about its origin.

## Core principle: do not fabricate

If RTL does not state something explicitly, the extracted spec **must not** invent a plausible value. Every uncertain item becomes one of:

- `TODO(designer): <what's missing> (auto-extracted: <hint>)` — for items where RTL gave a clue but not a definitive answer
- `TODO(designer): <what's missing>` — for items entirely outside RTL's reach (use cases, performance commitments, programmer intent)

The cost of a fabricated value is much higher than the cost of a TODO. A TODO surfaces a question; a fabricated value silently lies.

---

## What is mechanically extractable (HIGH confidence)

These items have a 1-to-1 mapping from RTL syntax to spec content. The extractor should output them with no qualifiers.

### Module port list → `interfaces.md`

```systemverilog
module wctmr (
    input  logic        clk_i,
    input  logic        rst_ni,
    input  logic [5:0]  paddr_i,
    output logic [31:0] prdata_o,
    output logic        intr_cmp0_o,
    ...
);
```

Extracted directly to:
- `Clocks` table (entries matching `clk_*_i`)
- `Resets` table (entries matching `rst_*_ni`, with active-low inferred from the `_n` suffix; bare `rst_i` defaults to active-high)
- `Bus interface` signal table (groups by prefix: `s_apb`, `s_axi`, `m_axi`, etc.)
- `Sideband` table (everything else not matching IRQ/alert pattern)
- `Interrupts` table (`intr_*_o`)
- `Alerts` table (`alert_*_o` if present)

Direction (input/output), width, and signal name are taken verbatim from RTL.

### Parameters → `interfaces.md` Parameters table

```systemverilog
module wctmr #(
    parameter int ADDR_WIDTH = 6,
    parameter int PRESCALE_W = 15
) (...);
```

Extracted to:
| Name | Type | Default | Description |
| `ADDR_WIDTH` | int | 6 | `TODO(designer): describe ADDR_WIDTH purpose and constraints` |
| `PRESCALE_W` | int | 15 | `TODO(designer): describe PRESCALE_W purpose and constraints` |

Constraints (range, dependencies) are **not** extractable from RTL syntax — the spec author must add them.

### Reset values → `registers.md` Reset column

```systemverilog
always_ff @(posedge clk_i or negedge rst_ni) begin
    if (!rst_ni) begin
        ctrl_q       <= 32'h0;
        intr_state_q <= 2'b00;
        counter_q    <= 64'h0;
    end else begin
        ...
    end
end
```

Extract reset values for every register `*_q` referenced in the reset-branch of an `always_ff` block. Map RTL identifiers to register table entries by name (e.g., `ctrl_q` → CTRL register).

### FSM state names → ToO §Control/FSM

```systemverilog
typedef enum logic [2:0] {
    IDLE     = 3'b000,
    LOAD     = 3'b001,
    TRANSMIT = 3'b010,
    DRAIN    = 3'b011,
    ERROR    = 3'b100
} state_e;

state_e state_q, state_d;
```

Extract:
- FSM name from the variable: `state_q` → "Main FSM" (placeholder) or, better, extract from a hierarchical context (instance name in always block prefix)
- State list verbatim
- State encoding (binary) to a code-fragment subsection if the encoding is one-hot or gray-coded (relevant for DV)

### SVA assertions → `dv/plan.md` ABV section

```systemverilog
a_intr_propagation: assert property (
    @(posedge clk_i) disable iff (!rst_ni)
    intr_state_q[i] && intr_enable_q[i] |-> intr_o[i]
);
```

Extract assertion name and property text into the ABV table. Spec-level vs. implementation-level distinction is human-judgment; tag everything as `TODO(reviewer): classify as spec-level vs. RTL-implementation` for the designer to triage.

### OpenTitan-style hjson register definitions → `registers.md`

If the RTL package contains a `data/<ip>.hjson` (OpenTitan reggen format), parse it and emit the register map directly. This is the highest-fidelity extraction path because hjson is *itself* a spec.

---

## What is heuristically extractable (MEDIUM confidence)

These need pattern matching plus judgment. Always tag the output with the extraction confidence in the IMPORT_REPORT so the designer can verify.

### FSM transitions

Trace `always_comb` or `always_ff` blocks that drive `state_d`:

```systemverilog
always_comb begin
    state_d = state_q;
    case (state_q)
        IDLE: if (cmd_start) state_d = LOAD;
        LOAD: begin
            if (descriptor_valid && length > 0) state_d = TRANSMIT;
            else if (descriptor_valid && length == 0) state_d = DRAIN;
            else if (descriptor_error) state_d = ERROR;
        end
        ...
    endcase
end
```

Extract a transition table:

| From | To | Condition (RTL expression) |
|---|---|---|
| IDLE | LOAD | `cmd_start` |
| LOAD | TRANSMIT | `descriptor_valid && length > 0` |
| ... | ... | ... |

The "Condition" column is RTL-grounded; the spec author should rewrite it in plain prose for ToO §Control/FSM.

### Datapath structure

Extract from `module` instantiations:

```systemverilog
prescaler u_prescaler (...);
counter   u_counter   (...);
cmp_arm   u_cmp0_arm  (...);
cmp_arm   u_cmp1_arm  (...);
```

The instance hierarchy hints at the block diagram. Output a Mermaid `flowchart TB` with the instances as nodes and the connection signals as edges. Tag the output as `[draft — verify against design intent]`.

### Error conditions

Search for patterns that mark error reporting:

- `*_err`, `*_error`, `*_fault` signal names
- `_alert_*` signal names
- `intr_*` signals named after errors (`intr_overflow_o`, `intr_underrun_o`)
- `assert` that match standard error patterns

Each hit becomes a row in the ToO §Error and fault handling table with the trigger condition (extracted) and a `TODO(designer):` for the block reaction and software-visible effect.

### Sub-word / unmapped access policy

Detect from APB / AXI register-file logic:

```systemverilog
assign pslverr_o = psel_i && (
    paddr_i[1:0] != 2'b00 ||  // sub-word
    !addr_in_map               // unmapped
);
```

Extract the conditions and emit corresponding entries in the Error table. If the logic is intricate (multi-cycle), tag for review.

---

## What is structurally NOT extractable (LOW confidence — TODO only)

The extractor must not attempt these. They require designer input and become explicit TODOs.

| Spec section | Why RTL can't tell us |
|---|---|
| README Features (capability list) | RTL does what it does; a feature list is *what we promise* in human-meaningful terms. Implementation ≠ promise. |
| README Description (prose) | The "shape of the design" needs design intent, not just connectivity. |
| README Compatibility | RTL doesn't know about external standards it claims to be compatible with. |
| Programmer's guide use cases | Sequence-of-operations is software intent, not hardware structure. |
| Performance commitments | Even if you could measure throughput, *committing* to a number is a contract. RTL has behavior; the spec has commitment. |
| Out-of-scope items in DV plan | Negative knowledge. Not extractable. |
| Security countermeasure rationale | Why a sec_cm exists requires threat-model context outside RTL. |

For these sections, `/spec-import` produces the section heading, the template scaffolding, and `TODO(designer):` markers. Nothing is invented.

---

## Naming-convention assumptions

The extractor relies on a few common conventions. If an RTL deviates, the import is less precise — the IMPORT_REPORT will flag this.

| Convention | Default expectation |
|---|---|
| Active-low reset suffix | `_ni` (e.g., `rst_ni`). Without `_n` infix, assume active-high. |
| Direction suffix | `_i` for input, `_o` for output. Other RTLs may use `_in`/`_out` or no suffix. |
| Internal-only signals | `_q` (registered), `_d` (next-state), `_t` (combinational). The extractor does **not** put `_q`/`_d`/`_t` in the public interface tables. |
| Bus interface prefix | `s_<protocol>_<signal>` for slave (e.g., `s_apb_paddr_i`), `m_<protocol>_<signal>` for master. If absent, the extractor groups bus signals by inferred protocol from signal names (`paddr_i` + `psel_i` + ... → APB). |
| IRQ output | `intr_<source>_o`. |
| Alert output | `alert_<name>_o`. |

If the project uses different conventions, document the mapping in `<spec_root>/IMPORT_NAMING.md` before re-running `/spec-import`, or accept that the imported tables will need manual relabelling.

---

## Cross-validation: when both RTL and an existing doc are given

This is the most powerful mode. The extractor:

1. Builds the RTL-derived skeleton.
2. Reads the existing doc(s) and matches each prose section to one of the six target files based on audience signals.
3. For overlapping content (e.g., both RTL and doc describe register CTRL), uses **RTL as canonical** for the table data (offset, width, reset, access type) and **doc as canonical** for the prose (description, behavior, programmer's guide use cases).
4. Reports conflicts in IMPORT_REPORT.md: "RTL: CTRL.enable reset = 0; doc: 'CTRL.enable defaults to disabled' — consistent. RTL: CTRL.mode width = 4 bits; doc: 'mode is a 3-bit field' — **CONFLICT**, please reconcile."

Conflicts are surfaced as ✗ items the user must resolve before claiming D1.

---

## Limits of native LLM RTL parsing

The default `/spec-import` reads RTL files via the `Read` and `Grep` tools — no external SystemVerilog parser. This works well for:

- Single-module IPs up to a few hundred lines
- Standard coding styles (typedef enum FSMs, `always_ff`/`always_comb` separation, OpenTitan / lowRISC conventions)

It works poorly on:

- Mega-IPs where the relevant logic is spread over thousands of lines and dozens of files
- Heavily macro'ed code (Verilog `define` chains)
- Generated code without a hjson source
- Non-standard FSM encodings (e.g., distributed state across multiple flip-flop groups without an enum)

For mega-IPs, run a structural pre-pass with Verilator (`verilator --xml-only --xml-output design.xml ...`) and point `/spec-import` at the XML; the extractor will use the XML for connectivity and use the source files only for prose hints. This pre-pass is optional but recommended for IPs over ~2000 lines of RTL.

---

## Output: the IMPORT_REPORT.md artifact

Every `/spec-import` invocation produces `<spec_root>/IMPORT_REPORT.md` with:

1. **Source manifest**: which files were read, their line counts, modification dates.
2. **Extraction summary** per spec file: high/medium/low-confidence item count.
3. **Per-extraction provenance**: each table row in the produced spec links back to the source file and line range it came from. Format: `<!-- source: rtl/wctmr.sv:42-58 -->` after each table.
4. **Conflict log** (if cross-validating with existing doc): list of mismatches with both sides quoted.
5. **TODO inventory**: total `TODO(designer):` count by file.
6. **Recommended next actions**: typically `/spec-status` then targeted designer review of the medium/low-confidence sections.

The IMPORT_REPORT plays the same role as READER_TEST_LOG.md plays for the reader test: it documents how the spec got to its current state, so a reviewer can audit the import without re-running it.

---

## When NOT to use `/spec-import`

- The IP is greenfield with no RTL or prior doc → use `/spec-init` instead. Forcing `/spec-import` on nothing produces a spec full of TODOs with no anchor to verify against.
- The existing doc is a marketing brochure (no register tables, no port lists) → import will not produce useful output. Treat it as `/spec-init` input instead.
- The RTL is a behavioral model not intended for synthesis → the extracted ports may not match the synthesizable interface; the spec author should treat the import as a draft and write `interfaces.md` from scratch.

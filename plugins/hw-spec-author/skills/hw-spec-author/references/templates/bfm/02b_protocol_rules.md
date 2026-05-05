# Template: protocol_rules.md

**Primary audience:** BFM implementer, RTL designer of the DUT facing the BFM, DV engineer writing the assertion / coverage suite.
**Goal:** Every protocol-cycle-level rule the BFM must enforce or respect is listed exactly once, with a stable ID, a testable condition, the required behavior, and a severity. Each rule maps 1:1 to an SVA assertion in `dv/plan.md` and 1:1 to a reader-test question in `bfm_reader_test_bank.md`.

**Protocol applicability:** Protocol-agnostic. Applies to AXI, AXI-Lite, AXI-Stream, APB, AHB, TileLink, CHI, and any custom valid/ready handshake protocol. For protocols without channels (e.g., APB), omit the `<CHANNEL>` field from the rule ID. Mini-example uses AXI-Lite.

This file is the BFM's contract with the protocol. If a rule is missing, the BFM is silently allowed to violate it; if a rule is fabricated, the DUT under test will be over-constrained.

Format origin: ARM AMBA Protocol Checker User Guides (DUI 0534B / DUI 0576A). Format-choice rationale (severity vocabulary, ID structure, divergence from ARM, ARM SVA equivalent column) documented in `plan/BFM_MODE_DESIGN.md` §11.Q4.

## Required structure

```
# Protocol Rules

**Protocol:** <name and version, must match signal_interface.md>
**Role:** <master | slave | passive monitor>
**ID format:** Two legal variants — pick one based on whether the protocol has channels in signal_interface.md §Channel grouping.

  - **Channel-based protocols** (AXI, AXI-Lite, AXI-Stream, AHB, TileLink, CHI, etc.): `<PROTO>_<ROLE>_<CHANNEL>_<SHORT_NAME>`
  - **Channel-less protocols** (APB, simple custom protocols): `<PROTO>_<ROLE>_<SHORT_NAME>`

Component definitions:
  - PROTO: short uppercase tag for the protocol, e.g. `AXI4LITE`, `AXI4`, `APB`, `TLUL`
  - ROLE: `MST` (master), `SLV` (slave), or `MON` (monitor-only)
  - CHANNEL (channel-based variant only): matches a row in signal_interface.md §Channel grouping; or `RST` for reset rules; or `XCH` for cross-channel rules; or `CFG` for configuration-knob rules
  - SHORT_NAME: snake_case, distinctive token (~3–6 words)

For channel-less protocols, RST/CFG sub-sections still apply but the rule IDs collapse to `<PROTO>_<ROLE>_RST_<name>` and `<PROTO>_<ROLE>_CFG_<name>` (no `<CHANNEL>` field — `RST` and `CFG` take its place visually). XCH does not apply (no cross-channel concept).

**Severity legend (2 levels, matching ARM Protocol Checker):**
  - **FAIL** — protocol violation. BFM must signal an error (assertion fail, log entry, scoreboard mismatch). Active BFMs must not produce stimulus that violates this rule. Maps to ARM `ERR` severity.
  - **RECOMMEND** — quality-of-implementation hint. Not a protocol violation; the BFM may log the observation and contribute to coverage but does not fail the test. Maps to ARM `REC` severity.

**Per-project overrides:** A test suite may promote `RECOMMEND` rules to `FAIL` (stricter testing), or demote `FAIL` rules to `RECOMMEND` (relaxed testing of a known-broken DUT). Override mechanism is documented in `dv/plan.md`; the rule ID and table here remain stable.

## Reset rules

| ID | Condition | Required behavior | Severity | ARM SVA equivalent |
|----|-----------|-------------------|----------|--------------------|
| <PROTO>_<ROLE>_RST_<name> | <when does this apply> | <what must happen> | <FAIL/RECOMMEND> | <ARM ID or "(none)"> |
| ...                       | ...                    | ...                  | ...              | ...                  |

## <Channel name 1> rules

(One sub-section per channel listed in signal_interface.md §Channel grouping. For protocols without channels, this section becomes a single flat rule table and the `<CHANNEL>` field is dropped from rule IDs.)

| ID | Condition | Required behavior | Severity | ARM SVA equivalent |
|----|-----------|-------------------|----------|--------------------|
| <PROTO>_<ROLE>_<CHAN>_<name> | <condition> | <required behavior> | <severity> | <ARM ID or "(none)"> |
| ...                          | ...         | ...                 | ...        | ...                  |

## <Channel name 2> rules

(...)

## Cross-channel rules

For rules that span multiple channels (e.g. "BVALID must not assert before WLAST has been observed"), list under `<PROTO>_<ROLE>_XCH_<name>`. Cross-channel ordering also appears in `channel_handshake.md` as dependency arrows; the rule row here is the formal predicate, the diagram there is the visual aid.

| ID | Condition | Required behavior | Severity | ARM SVA equivalent |
|----|-----------|-------------------|----------|--------------------|
| ... | ... | ... | ... | ... |

## Configuration-knob rules

Rules that depend on a configuration knob (response delay, fault injection, outstanding limit). Use channel `CFG`. The Condition column references the knob by name (defined in `transaction_api.md` or `active_passive_mode.md`).

These rules are BFM-specific extensions and have no ARM SVA equivalent — ARM Protocol Checker is a passive checker without configuration knobs. Tag the ARM SVA column `(none)` for every CFG row.

| ID | Condition | Required behavior | Severity | ARM SVA equivalent |
|----|-----------|-------------------|----------|--------------------|
| <PROTO>_<ROLE>_CFG_<name> | <e.g. "set_response_fault(SLVERR) is active for the next write"> | <e.g. "BRESP must be 2'b10 (SLVERR) on the next BVALID handshake; BFM must clear the one-shot fault after firing"> | FAIL | (none) |
```

## Writing rules for this file

1. **One rule per row. No "and"/"or" splits.** Compound rules ("AWVALID must remain HIGH until AWREADY observed AND must not change during the wait") split into two rows. Each row maps to one assertion; compound rows map to two assertions and create maintenance ambiguity.
2. **The Condition column is a testable predicate.** Phrased as "when <observable event>" — "when AWVALID rises", "when ARESETn deasserts", "when set_response_fault is active". If the condition is not directly observable from the wire signals plus configuration state, rewrite until it is.
3. **The Required behavior column is a testable predicate.** Phrased as a wire-level constraint or a deterministic action — "must remain HIGH until AWREADY observed HIGH", "must not change", "the next BVALID must carry BRESP = 2'b10". Phrases like "should generally" or "may" indicate the rule belongs at `RECOMMEND` severity, not `FAIL`.
4. **Every rule ID is unique.** `/spec-lint` LINT-BFM-002 enforces the ID format and uniqueness. Renaming a rule retires the old ID — do not reuse.
5. **Channel names match `signal_interface.md` §Channel grouping verbatim.** `/spec-lint` LINT-BFM-004 cross-checks. Inventing a channel name here that doesn't exist there is a violation.
6. **Severity is judged per protocol spec, not per implementation taste.** If the protocol spec marks a rule as mandatory, severity is `FAIL` even if "the DUT we're testing doesn't care." If the spec marks a rule as recommended-but-not-required, severity is `RECOMMEND`. Implementation-quality opinions belong at `RECOMMEND`.
7. **Populate the `ARM SVA equivalent` column for every row.** Three legal values:
   - **Verified ARM ID** (e.g. `AXI4_ERRM_AWVALID_STABLE`) — confirmed against ARM Protocol Checker User Guides (DUI 0534B / DUI 0576A) or the YosysHQ SVA-AXI4-FVIP open-source library.
   - **`<ARM-pattern ID> (unverified)`** — ID follows ARM `<PROTO>_<SEV><ROLE>_<NAME>` naming pattern but has not been cross-checked against the source. Use when you suspect ARM has the rule but cannot confirm immediately; mark for later verification.
   - **`(none)`** — confirmed ARM has no equivalent. Typically true for `RST` duration rules, `XCH` cross-channel rules, and all `CFG` configuration-knob rules.

   Do not leave the column blank: empty cells read as "the author didn't check"; explicit `(none)` or `(unverified)` reads as "checked, with this status."

## Anti-patterns

- **Anti-pattern:** "See the AXI4-Lite spec, §B" instead of stating the rule. The spec is too long to read inline; restate the rule in this row, with the spec section reference in the Notes column if useful. The rule must be self-contained.
- **Anti-pattern:** Rules that depend on configuration knobs without naming the knob. "If outstanding count is large enough, drop the next request" — name the knob (`set_outstanding_limit`) and put the rule under `CFG`.
- **Anti-pattern:** Combining two unrelated channels in the same row. Cross-channel rules go to `XCH`; same-channel rules stay in their channel sub-section.
- **Anti-pattern:** Missing reset rules. Every BFM has at least one reset rule (e.g., "while reset is asserted, all output VALID signals must be LOW"). A protocol_rules.md without a `## Reset rules` sub-section is incomplete.
- **Anti-pattern:** Choosing severity based on what's convenient for the BFM implementation. Severity reflects the protocol contract, not the implementation roadmap.
- **Anti-pattern:** Twisting our `<PROTO>_<ROLE>_<CHANNEL>_<NAME>` ID format to chase 1:1 ARM matching (e.g. dropping the `<CHANNEL>` field for AXI rules so the ID looks like ARM's `<PROTO>_<SEV><ROLE>_<NAME>`). Our format is the canonical contract; the `ARM SVA equivalent` column carries the ARM lookup. Format consistency across protocols matters more than visual similarity to ARM IDs.
- **Anti-pattern:** Rules with cycle-count semantics ("hold X for N cycles", "wait M cycles before Y") stated only in prose. Implementer must guess the count semantics (does N=0 mean "0 cycles of hold" or "raise on cycle 0"? does the count include the trigger cycle or start from the next?). For every such rule, include either an ASCII timing diagram or two worked examples (e.g. N=0 and N=2) showing PCLK/ACLK edges, the trigger event, and the resulting signal transitions. Three lines of waveform save 5–10 minutes of implementer guessing per rule.
- **Anti-pattern:** Conditional rules ("unless X happens") without stating what X looks like at the wire level. Example: "PSEL must deassert after PREADY observed HIGH, unless another back-to-back transaction begins." A conformant implementer needs to know: what wire combination defines "back-to-back"? Spell it out in the same row or with a cross-reference to a definition in `channel_handshake.md` or `theory_of_operation.md`.

## Mini-example: AXI-Lite slave BFM (write-address channel rules only — full file enumerates all channels)

```
# Protocol Rules

**Protocol:** AXI4-Lite (ARM IHI0022, ID022613, §B).
**Role:** slave (SLV).
**ID format:** AXI4LITE_SLV_<channel>_<name>.

**Severity legend:**
  - FAIL — slave BFM must reject violation (drive an assertion fail and log; never produce stimulus that violates).
  - RECOMMEND — quality-of-implementation; covered in coverage only.

**ARM SVA equivalent column convention:** Verified ARM IDs are listed verbatim. IDs suffixed `(unverified)` follow ARM naming pattern but await cross-check against ARM DUI 0534B / DUI 0576A. Rules with no ARM equivalent (BFM extensions) are tagged `(none)`.

## Reset rules

| ID | Condition | Required behavior | Severity | ARM SVA equivalent |
|----|-----------|-------------------|----------|--------------------|
| AXI4LITE_SLV_RST_OUTPUTS_LOW | ARESETn is asserted (LOW) | All BFM-driven VALID outputs (AWREADY, WREADY, BVALID, ARREADY, RVALID) must be LOW. | FAIL | AXI4_ERRS_RESET_ALL_OUTPUTS_LOW (unverified) |
| AXI4LITE_SLV_RST_DURATION    | ARESETn pulse begins      | ARESETn must be held LOW for at least 16 ACLK cycles before the BFM enters its defined reset state. Shorter pulses leave the BFM in an undefined state — BFM cannot guarantee any subsequent rule. | FAIL | (none) |

## AW rules

| ID | Condition | Required behavior | Severity | ARM SVA equivalent |
|----|-----------|-------------------|----------|--------------------|
| AXI4LITE_SLV_AW_AWVALID_STABLE   | AWVALID rises HIGH                                | AWVALID must remain HIGH until AWREADY is observed HIGH on a rising ACLK edge. The slave samples this implicitly; a master that violates this rule is detected. | FAIL | AXI4_ERRM_AWVALID_STABLE |
| AXI4LITE_SLV_AW_AWADDR_STABLE    | AWVALID is HIGH                                   | AWADDR must not change between the cycle AWVALID rises and the cycle AWREADY is observed HIGH. | FAIL | AXI4_ERRM_AWADDR_STABLE (unverified) |
| AXI4LITE_SLV_AW_AWPROT_STABLE    | AWVALID is HIGH                                   | AWPROT must not change between the cycle AWVALID rises and the cycle AWREADY is observed HIGH. | FAIL | AXI4_ERRM_AWPROT_STABLE (unverified) |
| AXI4LITE_SLV_AW_AWPROT_IGNORED   | AWVALID + AWREADY handshake completes              | The slave BFM samples AWPROT but does not enforce protection-attribute checks; AWPROT values are recorded for monitor mode but never cause the slave to refuse a transaction. | RECOMMEND | (none) |
| AXI4LITE_SLV_AW_AWREADY_NO_LATCH | AWREADY is HIGH and AWVALID is LOW on the same cycle | AWREADY may be HIGH before AWVALID; this is legal per AXI4-Lite. The slave must not interpret a one-cycle AWREADY assertion as completing a transaction. | FAIL | AXI4_ERRS_AWREADY_NO_LATCH (unverified) |
| AXI4LITE_SLV_AW_AWREADY_MAX_WAIT | AWVALID has been HIGH for 1024 consecutive ACLK cycles without AWREADY observed HIGH | The slave BFM logs a back-pressure observation (suggests the DUT is starved by the configured response delay; see CFG_RESPONSE_DELAY_BOUND). The 1024-cycle threshold is implementation choice for monitoring health, not a protocol violation. | RECOMMEND | AXI4_RECS_AWREADY_MAX_WAIT (unverified) |

## Cross-channel rules

| ID | Condition | Required behavior | Severity | ARM SVA equivalent |
|----|-----------|-------------------|----------|--------------------|
| AXI4LITE_SLV_XCH_B_AFTER_AW_AND_W | BVALID is asserted | The corresponding AW handshake AND the corresponding W handshake must both have completed in an earlier or the same cycle. The slave must not assert BVALID for a write whose address or data has not yet arrived. | FAIL | (none) |
| AXI4LITE_SLV_XCH_R_AFTER_AR       | RVALID is asserted | The corresponding AR handshake must have completed in an earlier or the same cycle. | FAIL | (none) |

## Configuration-knob rules

| ID | Condition | Required behavior | Severity | ARM SVA equivalent |
|----|-----------|-------------------|----------|--------------------|
| AXI4LITE_SLV_CFG_RESPONSE_DELAY_BOUND | Configuration `set_response_delay(min, max)` has been called with min=N, max=M | For every transaction, the BFM must hold off the response handshake (BVALID for writes, RVALID for reads) by a randomly chosen number of ACLK cycles in [N, M]. Bounds are inclusive. | FAIL | (none) |
| AXI4LITE_SLV_CFG_FAULT_BRESP          | Configuration `set_response_fault(SLVERR)` is active for the next write | The next BVALID handshake must carry BRESP = 2'b10 (SLVERR). The fault is one-shot: the BFM clears the configured fault after the next write completes. | FAIL | (none) |
| AXI4LITE_SLV_CFG_FAULT_RRESP          | Configuration `set_response_fault(SLVERR)` is active for the next read | The next RVALID handshake must carry RRESP = 2'b10 (SLVERR). One-shot. | FAIL | (none) |
```

The full AXI-Lite slave protocol_rules.md continues in the same shape across W, B, AR, R sub-sections — typically ~30 rules across all channels for AXI4-Lite, ~80 for full AXI4. Roughly half map 1:1 to ARM SVA assertions; the remainder (RST duration, all XCH cross-channel rules, all CFG configuration-knob rules) are BFM extensions tagged `(none)`.

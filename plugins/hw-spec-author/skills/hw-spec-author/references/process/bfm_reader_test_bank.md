# Process: Reader Test Bank — protocol-bfm mode

Reader-test question bank for `protocol-bfm` specs. Used by `/spec-review` when `MODE.md` declares `mode: protocol-bfm`.

For `behavioral-block` specs, see `reader_test.md`. The protocol, isolation rules, and "PASS / NOT_ANSWERED / AMBIGUOUS" output format are identical across both banks; only the question content differs.

## Protocol

1. **Prepare the spec.** All BFM-mode files at D1 completeness, no `TODO(designer):` markers. Files in scope: `README.md`, `MODE.md`, `doc/theory_of_operation.md`, `doc/signal_interface.md`, `doc/pin_level_reset.md`, `doc/protocol_rules.md`, `doc/channel_handshake.md`, `doc/transaction_api.md`, `doc/channel_api.md`, `doc/active_passive_mode.md`, `dv/plan.md`. `doc/registers.md` if present.
2. **Generate questions.** Use the universal BFM bank below (cover at least one question per category) plus the matching block-specific bank.
3. **Run the reader.** Spawn the `spec-reader` subagent. Feed the spec files and one question at a time. The reader must:
   - Quote the spec sentence(s) that answer the question, OR
   - Report `NOT_ANSWERED` if the spec is silent, OR
   - Report `AMBIGUOUS` if the spec text is unclear.
4. **Triage results.** For each gap:
   - In-scope, must add → fix the spec.
   - Out-of-scope, must reference → add a "see X" sentence.
   - Genuinely implementation-defined → add an explicit annotation.
5. **Re-run** until all questions resolve cleanly.

## Question bank — universal BFM

Apply to every protocol-bfm spec. 8 categories.

### Protocol rules
1. For every channel listed in `signal_interface.md` §Channel grouping, is there a corresponding rule sub-section in `protocol_rules.md`?
2. Does every rule in `protocol_rules.md` have a unique ID matching the format `<PROTO>_<ROLE>_<CHANNEL>_<NAME>` (or `<PROTO>_<ROLE>_<NAME>` for protocols without channels)?
3. Does every rule have a non-empty Condition column AND a non-empty Required behavior column?
4. Does every rule have a Severity (FAIL or RECOMMEND)? Is the severity choice consistent with the protocol spec the rule cites?

### Handshake dependencies
5. For every transaction type listed in `channel_handshake.md`, is there a dependency diagram (Mermaid or equivalent) AND a textual dependency list?
6. Are deadlock-prone channel pairs explicitly called out with a "deadlock-avoidance commentary" paragraph? If no deadlock mode applies, is "(none)" stated explicitly rather than omitted?
7. Does every "must precede" arrow in the handshake diagrams correspond to an `XCH` rule in `protocol_rules.md`?

### Pin-level reset
8. Does the during-reset table in `pin_level_reset.md` include every wire from `signal_interface.md` §Wire table? Does the after-reset table do the same?
9. Are post-reset values stated for both BFM-driven outputs AND BFM-observed inputs? (Inputs typically state "as driven by the DUT", which is acceptable.)
10. Is the minimum reset assertion duration stated, including the unit (ACLK cycles or wall-clock time)?

### API completeness
11. Can every common test scenario for this protocol — single read, single write, burst (if supported), back-pressure recovery, error injection — be expressed using only methods from `transaction_api.md`?
12. Does every method in `transaction_api.md` have all five sub-sections: Signature, Preconditions, Side effects, Return value, Error modes?
13. Does every method in `transaction_api.md` reference its Equivalent channel API decomposition (a step-by-step mapping to `channel_api.md` calls, or "(none)" if Transaction-API-only)?
14. Are Channel API methods sufficient for out-of-order driving, mid-transaction pausing, and intentional protocol-rule violation injection (for DUT protocol-checker testing)?

### Mode coverage
15. Is the active-vs-passive switch a single named configuration knob (one knob, two values)?
16. Does passive mode definitively not drive any wire? List which wires the BFM stops driving when switching from active to passive.
17. What happens to in-flight transactions held by `transaction_api.md` calls when the mode is switched mid-test?

### Outstanding-transaction tracking
18. What is the maximum outstanding transaction count the BFM tracks? (For protocols without outstanding semantics, e.g. AXI4-Lite, the answer should be "1" or "n/a — single transaction at a time.")
19. What happens if a test exceeds the outstanding limit? Is the BFM's behavior FAIL, back-pressure, or undefined?

### ID rules (only for ID-bearing protocols, e.g. AXI4 with `AWID` / `ARID`)
20. Does the BFM enforce same-ID ordering at the response side? Quote the rule.
21. Does the BFM allow different-ID transactions to complete out of order? Quote the rule.
22. What is the BFM's behavior on a duplicate outstanding ID (a second transaction issued with the same ID before the first response returns)?

### Error injection (configuration-knob effects)
23. Every configuration knob in `transaction_api.md` and `active_passive_mode.md` — is there a corresponding `CFG` rule in `protocol_rules.md` describing the knob's wire-level effect?
24. Are configuration knobs one-shot (clear after firing) or persistent? State for each.
25. What is the precedence when multiple knobs would drive conflicting behavior on the same transaction?

## Question bank — add per BFM type

Append the matching block-specific bank to the universal bank.

**For master BFMs (drive a slave DUT):**
- What stimulus patterns can the BFM generate without a sequencer (built-in primitives)?
- Can the BFM throttle stimulus rate? Via what knob?
- Does the BFM check the slave DUT's response correctness, or only its protocol compliance?

**For slave BFMs (respond to a master DUT):**
- Is the BFM's internal address-indexed memory pre-loadable? Via what method?
- What is the default response (BRESP/RRESP equivalent) for an unmapped address access?
- How is the response delay modelled — fixed, configurable per address, or random within bounds?

**For monitor-only / passive BFMs:**
- Does the BFM reconstruct full transactions from observed pin activity, or only individual handshakes?
- What coverage hooks does the BFM expose? List by name.
- Does the BFM log every protocol-rule violation, or only violations above a configurable severity threshold?

**For ID-bearing protocols (AXI4, AXI3, ACE, CHI, custom with IDs):**
- (questions 20–22 in the universal bank above are mandatory; not optional)
- Is the ID width parameterized? What is the maximum ID value the BFM supports?
- How does the BFM handle ID wraparound — does it FAIL, log, or continue?

**For burst-capable protocols (AXI4, AXI3, AHB):**
- What burst types are supported (FIXED, INCR, WRAP)?
- What is the maximum burst length the BFM can issue or accept?
- Does the BFM enforce 4KB-boundary crossing (for AXI) or analogous wraparound rules?

**For cache-coherent protocols (CHI, ACE):**
- What snoop response types does the BFM generate? List by name.
- Does the BFM model cache state transitions, or only the wire-level snoop protocol?
- What is the BFM's behavior on a CleanInvalid + WriteBack ordering corner case?

## How to run the reader

When `/spec-review` runs in `protocol-bfm` mode:

1. The command reads `MODE.md`. If `mode: protocol-bfm`, it dispatches the `spec-reader` subagent (fresh context, isolated, `Read`/`Grep`/`Glob` only) with questions drawn from this bank.
2. Question count: 8–12 questions per run, sampled to cover at least one question per universal category plus the matching block-specific bank.
3. The subagent answers each question independently using the format defined in `agents/spec-reader.md` (PASS / NOT_ANSWERED / AMBIGUOUS, quoting supporting text).
4. `/spec-review` aggregates results and reports the gap list to the user for triage.

## What "passing" looks like

A `protocol-bfm` spec passes the reader test when:

- Every universal-bank question (1–25, applicable to the protocol) resolves to a quoted sentence in the spec.
- Every block-specific question for the matching BFM type resolves to a quoted sentence.
- No question resolves to `NOT_ANSWERED` or `AMBIGUOUS` without an explicit out-of-scope or implementation-defined annotation in the spec.

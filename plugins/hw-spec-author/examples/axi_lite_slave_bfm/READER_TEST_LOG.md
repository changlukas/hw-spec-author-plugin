# axi_lite_slave_bfm — Reader Test Log

This log substantiates the D1-completeness claim for `examples/axi_lite_slave_bfm/`. Per `stage_gates.md` item `D1.cross.bfm_reader_test`, a `protocol-bfm` mode spec at D1 must have run a reader test of at least 8 questions covering all 8 universal-BFM categories applicable to the protocol; gaps fixed.

The reader test was executed twice: an initial run, which surfaced two gaps; the spec was edited to close them; a re-run confirmed every question now resolves to a quoted sentence in the spec.

Both runs were performed by spawning a fresh-context subagent (the `spec-reader` role) with read-only access to the eleven axi_lite_slave_bfm spec files. The subagent had no access to authoring history, design intent, or external references — only the spec text. This isolation is the entire point of the reader test: the spec author cannot un-know what they wrote, and the test exists to expose that gap.

---

## Protocol

- **Files in scope** (treated as the only source of truth):
  - `MODE.md`
  - `README.md`
  - `doc/theory_of_operation.md`
  - `doc/signal_interface.md`
  - `doc/pin_level_reset.md`
  - `doc/protocol_rules.md`
  - `doc/channel_handshake.md`
  - `doc/transaction_api.md`
  - `doc/channel_api.md`
  - `doc/active_passive_mode.md`
  - `dv/plan.md`

- **Question selection** (10 questions, drawn from `references/process/bfm_reader_test_bank.md`):
  - 8 universal-BFM questions covering all 8 categories (protocol rules, handshake dependencies, pin-level reset, API completeness, mode coverage, outstanding tracking, ID rules — N/A for AXI-Lite which has no IDs, error injection).
  - 2 block-specific questions for the slave BFM bank (default response for unmapped address; response delay model).

- **Answer rule**: PASS only if a verbatim spec sentence answers the question. NOT_ANSWERED when the spec is silent. AMBIGUOUS when relevant text exists but does not unambiguously answer.

---

## Run 1 — initial

| #   | Question summary                                                                | Outcome       |
|-----|---------------------------------------------------------------------------------|---------------|
| Q1  | Every channel from signal_interface has a rule sub-section in protocol_rules?   | PASS          |
| Q2  | Every protocol rule has unique ID matching `<PROTO>_<ROLE>_<CHANNEL>_<NAME>`?    | PASS          |
| Q3  | Every transaction type in channel_handshake has dependency diagram + textual list? | PASS       |
| Q4  | During-reset and after-reset tables both list every wire from signal_interface? | PASS          |
| Q5  | Can every common test scenario be expressed using only Transaction API methods? | NOT_ANSWERED  |
| Q6  | Active-vs-passive switch is a single named knob?                                | PASS          |
| Q7  | Maximum outstanding transaction count for AXI-Lite (ID rules N/A)               | PASS          |
| Q8  | What is the default response (RDATA) for an unmapped address read?              | NOT_ANSWERED  |
| Q9  | How is response delay modelled — fixed, configurable per address, or random?    | PASS          |
| Q10 | Every config knob has a corresponding CFG rule in protocol_rules?               | PASS          |

**Outcome: 8 PASS, 2 NOT_ANSWERED.**

### Gap 1: Q5 — common test scenario coverage (NOT_ANSWERED)

The spec lists 8 transaction-API methods (`expect_write`, `expect_read`, `set_register_value`, etc.) but does not enumerate which "common test scenarios" are coverable using only this surface. The reader cannot determine whether burst tests are expressible (they're not — AXI-Lite has no bursts, but the spec doesn't say so in the API context), or whether back-pressure recovery tests are expressible (they require Channel API).

**Fix**: added explicit list of "common test scenarios" and which API surface covers each in `transaction_api.md` §When to use Channel API instead. (This was found to already be partially covered in `channel_api.md` §When to use this API; cross-reference added in transaction_api.md.)

### Gap 2: Q8 — unmapped address read response (NOT_ANSWERED)

The spec describes `set_register_value(addr, value)` but does not state what the BFM returns when the master DUT reads an address for which no `set_register_value` has been called.

**Fix**: added one sentence to `transaction_api.md` §expect_read Side effects: "RDATA defaults to 0x0 if no `set_register_value(addr, ...)` has been issued for the address being read." Same fix mirrored in `theory_of_operation.md` §Internal architecture > Configuration store.

---

## Run 2 — after fixes

| #   | Question summary                                                                | Outcome       |
|-----|---------------------------------------------------------------------------------|---------------|
| Q1  | Every channel from signal_interface has a rule sub-section in protocol_rules?   | PASS          |
| Q2  | Every protocol rule has unique ID matching `<PROTO>_<ROLE>_<CHANNEL>_<NAME>`?    | PASS          |
| Q3  | Every transaction type in channel_handshake has dependency diagram + textual list? | PASS       |
| Q4  | During-reset and after-reset tables both list every wire from signal_interface? | PASS          |
| Q5  | Can every common test scenario be expressed using only Transaction API methods? | PASS — quoted from transaction_api.md cross-reference to channel_api.md "When to use this API" listing back-pressure recovery / phase-level fault injection / illegal handshake injection as Channel-API-only scenarios |
| Q6  | Active-vs-passive switch is a single named knob?                                | PASS          |
| Q7  | Maximum outstanding transaction count for AXI-Lite                               | PASS          |
| Q8  | What is the default response (RDATA) for an unmapped address read?              | PASS — quoted from transaction_api.md §expect_read: "RDATA defaults to 0x0 if no set_register_value has been issued" |
| Q9  | How is response delay modelled — fixed, configurable per address, or random?    | PASS          |
| Q10 | Every config knob has a corresponding CFG rule in protocol_rules?               | PASS          |

**Outcome: 10 / 10 PASS.**

The spec is at D1.

---

## Notes for future reviewers

This log was produced as the canonical example of the BFM-mode reader test in action. Future protocol-bfm specs should follow the same protocol: 8–12 questions covering all 8 universal-BFM categories applicable to the protocol, run via the `spec-reader` subagent in fresh context, gaps fixed before claiming D1.

The two gaps surfaced here are characteristic of BFM-mode authoring blind spots:
- Gap 1 (Q5) — the API surface was internally consistent but lacked a top-level orientation telling the reader which surface to use for which scenario.
- Gap 2 (Q8) — a default behavior was implicit in the implementation but never explicit in the spec. The fix is one sentence.

Both fixes took under 5 minutes once the gap was identified. The cost of finding them via the reader test is much lower than the cost of finding them via a confused implementer six months later.

# Protocol Rules

**Protocol:** APB3 / APB4 (ARM AMBA APB Protocol Specification, IHI 0024).
**Role:** Completer (slave; SLV).
**ID format:** APB_SLV_<NAME> (no `<CHANNEL>` field — APB has no channels per signal_interface.md §Channel grouping).

**Severity legend (2 levels, matching ARM Protocol Checker):**
  - FAIL — slave BFM must reject violation.
  - RECOMMEND — quality-of-implementation; covered in coverage only.

**ARM SVA equivalent column convention:** Verified ARM IDs are listed verbatim. IDs suffixed `(unverified)` follow ARM naming pattern but await cross-check against ARM DUI 0534B / DUI 0576A. Rules with no ARM equivalent are tagged `(none)`.

## Reset rules

| ID | Condition | Required behavior | Severity | ARM SVA equivalent |
|----|-----------|-------------------|----------|--------------------|
| APB_SLV_RST_OUTPUTS_LOW | PRESETn asserted (LOW)         | All BFM-driven outputs must be at their during-reset values per pin_level_reset.md (PREADY=0, PRDATA=0, PSLVERR=0). | FAIL | (none) |
| APB_SLV_RST_DURATION    | PRESETn pulse begins           | PRESETn must be held LOW for at least 4 PCLK cycles. Shorter pulses leave the BFM in an undefined state. | FAIL | (none) |

## Phase rules

APB has two phases per transaction: SETUP and ACCESS. Phase rules govern transitions between them.

| ID | Condition | Required behavior | Severity | ARM SVA equivalent |
|----|-----------|-------------------|----------|--------------------|
| APB_SLV_SETUP_PSEL_PENABLE     | First cycle PSEL rises HIGH                            | PENABLE must be LOW on the same cycle. PSEL=1 + PENABLE=0 defines the SETUP phase.                                                                                  | FAIL | (none) |
| APB_SLV_ACCESS_PSEL_PENABLE    | Second cycle of a transaction                          | PENABLE must rise HIGH on the cycle following SETUP. PSEL=1 + PENABLE=1 defines the ACCESS phase.                                                                   | FAIL | (none) |
| APB_SLV_PADDR_STABLE           | PSEL is HIGH                                           | PADDR must not change between the SETUP cycle and the ACCESS cycle of the same transaction.                                                                        | FAIL | (none) |
| APB_SLV_PWRITE_STABLE          | PSEL is HIGH                                           | PWRITE must not change between the SETUP cycle and the ACCESS cycle of the same transaction.                                                                        | FAIL | (none) |
| APB_SLV_PWDATA_STABLE          | PSEL && PWRITE && PENABLE                              | PWDATA must not change during the ACCESS phase of a write transaction.                                                                                              | FAIL | (none) |
| APB_SLV_PSEL_DEASSERT_AFTER_ACCESS | PREADY observed HIGH during ACCESS phase           | PSEL and PENABLE must both be deasserted (LOW) on the cycle immediately following the ACCESS-phase completion, unless another back-to-back transaction begins. **Back-to-back transaction** = on the cycle following PREADY observed HIGH, PSEL stays HIGH and PENABLE drops to LOW (entering a new SETUP phase for a new transaction). PSEL deassert is required only if no new transaction follows; if a new transaction starts immediately, PSEL stays HIGH continuously. | FAIL | (none) |

## Wait-state rules

| ID | Condition | Required behavior | Severity | ARM SVA equivalent |
|----|-----------|-------------------|----------|--------------------|
| APB_SLV_PREADY_LOW_IN_SETUP   | PSEL is HIGH and PENABLE is LOW (SETUP cycle) | PREADY may be either HIGH or LOW during SETUP; the master ignores it. The slave is recommended to drive PREADY LOW during SETUP for waveform clarity. | RECOMMEND | (none) |
| APB_SLV_PREADY_TIMEOUT        | PENABLE has been HIGH for 1024 consecutive PCLK cycles without PREADY observed HIGH | The slave BFM logs a back-pressure observation. Implementation choice for monitoring health, not a protocol violation. | RECOMMEND | (none) |
| APB_SLV_PREADY_NO_LATCH       | PSEL && PENABLE && PREADY observed HIGH | The transaction completes on this cycle. PREADY may stay HIGH or drop on the next cycle; either is legal. | FAIL | (none) |

## Response rules

| ID | Condition | Required behavior | Severity | ARM SVA equivalent |
|----|-----------|-------------------|----------|--------------------|
| APB_SLV_PSLVERR_VALID_WITH_PREADY | PSLVERR is asserted | PREADY must also be asserted on the same cycle. PSLVERR is meaningful only when PREADY is HIGH (i.e., during ACCESS-phase completion). | FAIL | (none) |
| APB_SLV_PRDATA_VALID_WITH_PREADY  | PSEL && !PWRITE && PREADY        | PRDATA must carry valid read data. The master samples PRDATA on the cycle PREADY rises during ACCESS. | FAIL | (none) |
| APB_SLV_PSTRB_BYTE_LANES          | PSTRB[i] HIGH on the ACCESS cycle of a write transaction (APB4) | Only bytes of PWDATA whose corresponding PSTRB bit is HIGH are written to the BFM's internal memory at PADDR. Bytes whose PSTRB bit is LOW are unchanged. | FAIL | (none) |
| APB_SLV_PPROT_IGNORED             | APB4 transaction with PPROT supplied | The slave BFM samples PPROT but does not enforce protection-attribute checks; PPROT values are recorded for monitor mode but never cause the slave to refuse a transaction. | RECOMMEND | (none) |

## Configuration-knob rules

| ID | Condition | Required behavior | Severity | ARM SVA equivalent |
|----|-----------|-------------------|----------|--------------------|
| APB_SLV_CFG_WAIT_STATES_BOUND  | Configuration `set_wait_states(min, max)` has been called with min=N, max=M (PCLK cycles, inclusive) | For every transaction, the BFM must hold PREADY LOW for a randomly chosen number K ∈ [N, M] (inclusive) of ACCESS-phase cycles before raising PREADY HIGH. K is the count of ACCESS cycles where PREADY is observed LOW; PREADY rises on ACCESS cycle K+1. Default if uncalled: min=0, max=0 — PREADY rises on the same cycle PENABLE rises (zero LOW cycles). See timing diagram below. | FAIL | (none) |

### Wait-state timing examples

```
N=0 (zero wait states):
  PCLK     ▁▁▁▁┐_┌─┐_┌─┐_┌─┐_┌─┐_┌
  PSEL     ────┐         ┌──────
  PENABLE  ────────┐     ┌──────
  PREADY   ────────┐     ┌──────   ← rises on same cycle as PENABLE
                  ^SETUP ^ACCESS

N=2 (two wait-state cycles):
  PCLK     ▁▁▁▁┐_┌─┐_┌─┐_┌─┐_┌─┐_┌─┐_┌─┐_┌─┐_┌
  PSEL     ────┐                     ┌──────
  PENABLE  ────────┐                 ┌──────
  PREADY   ────────────────────┐     ┌──────   ← LOW for cycles 1,2 of ACCESS, HIGH on cycle 3
                  ^SETUP ^ACCESS-1 ^ACCESS-2 ^ACCESS-3
```
| APB_SLV_CFG_FAULT_PSLVERR      | Configuration `set_response_fault()` is active for the next transaction | The next ACCESS-phase completion must drive PSLVERR=1 alongside PREADY=1. The fault is one-shot: BFM clears the configured fault after the transaction completes. | FAIL | (none) |
| APB_SLV_CFG_REGISTER_PRELOAD   | Configuration `set_register_value(addr, value)` has been called | A subsequent read whose PADDR equals addr must return PRDATA = value, regardless of any prior write to that address. The configured value persists until reset_state() or a subsequent write to addr. | FAIL | (none) |
| APB_SLV_CFG_MODE_SWITCH        | Configuration `bfm_mode` transitions from ACTIVE to PASSIVE | All BFM-driven outputs (PREADY, PRDATA, PSLVERR) must transition to their during-reset values within one PCLK cycle. Any in-flight transaction_api.md call must unblock with status MODE_SWITCHED_TO_PASSIVE. | FAIL | (none) |

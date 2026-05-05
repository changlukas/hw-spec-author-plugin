# Template: transaction_api.md

**Primary audience:** Test author writing driver code or sequence code against the BFM.
**Goal:** The high-level test-facing API. ~95% of tests use only this surface; the remaining 5% use `channel_api.md`. Every method's contract — preconditions, side effects, return values, error modes — is explicit enough that a test author can write a sequence without reading the BFM source.

**Protocol applicability:** Structure is protocol-agnostic. Suggested method names (`apply_write`, `apply_read`, `burst_*`, `expect_*`) assume memory-mapped protocols. For streaming protocols (AXI4-Stream): use `apply_packet` / `expect_packet` / `apply_beat`. For cache-coherent protocols (CHI, ACE): use `apply_snoop` / `expect_snoop` and protocol-specific verbs. Per-method sub-section structure is unchanged across protocols. Mini-example uses AXI-Lite (memory-mapped).

This file exists separately from `channel_api.md` because the two have different audiences and different specificity. Transaction-API methods abstract whole transactions ("write 4 bytes to this address, expect OKAY"); channel-API methods control individual handshake phases ("assert AWVALID with AWADDR=0x40, then wait for AWREADY"). Conflating the two produces an API that is both too verbose for typical tests and too coarse for fault-injection tests.

Modeled on the OSVVM and Aldec/Cadence three-layer separation: signal interface (`signal_interface.md`) → channel API (`channel_api.md`) → transaction (function) API (this file).

## Required structure

```
# Transaction API

## API conventions

State language-agnostic conventions used in the signatures below:
- Pseudocode style: <e.g. "C-like; types map to SystemVerilog, SystemC, C++ as appropriate">
- Naming: <e.g. "snake_case; verb-noun for actions; expect_<noun> for monitor-mode assertions">
- Error reporting: <e.g. "all methods return a status enum; status `OK` on success, status `<NAME>` on each defined error mode">
- Blocking discipline: <e.g. "all methods are blocking until the transaction completes (BVALID handshake for writes); use channel_api.md for non-blocking control">

## Method index

A flat list of every method, grouped by purpose. The index is what a test author scans first; details follow below.

| Group | Method | One-line summary |
|-------|--------|------------------|
| Stimulus  | apply_write(addr, data)           | Drive a single write; block until response. |
| Stimulus  | apply_read(addr)                  | Drive a single read; block until data returns. |
| Monitor   | expect_write(addr, data)          | Wait for a write to address; assert data matches; fail otherwise. |
| Monitor   | expect_read(addr)                 | Wait for a read from address; return observed data. |
| Configuration | set_response_delay(min, max) | Configure response latency bounds. |
| Configuration | set_response_fault(kind)     | Inject a one-shot response fault on the next transaction. |
| Configuration | reset_state()                | Drop outstanding state, restore defaults; does not toggle ARESETn. |

## Method details

(One sub-section per method. Order: stimulus methods first, monitor methods second, configuration methods third.)

### <method_name>(<args>)

**Signature:** `<return_type> <method_name>(<arg_type> <arg_name>, ...)`

**Preconditions:**
- <e.g. "BFM has been instantiated and clocked; reset has deasserted at least once">
- <e.g. "No conflicting concurrent call to channel_api.md for the channels this transaction touches">
- <list every precondition; do not assume readers infer them>

**Side effects:**
- <which wires get driven>
- <which configuration knobs change>
- <whether outstanding state changes>

**Return value:**
- <return type semantics; what fields the return struct carries; on what conditions each value is set>

**Error modes:**
- <list every error mode that can be returned; one row each in a table or list>
- For each error: condition that triggers it, the value returned, whether the BFM logs the error, whether the transaction is retried automatically.

**Example:**
```c
status_t s = apply_write(0x40, 0xDEADBEEF);
if (s != OK) {
    /* handle the specific status */
}
```

**Equivalent channel API decomposition:**
- State which `channel_api.md` calls this method maps to internally. Either a step-by-step breakdown ("`apply_write` is equivalent to `begin_phase(AW, ...) ; assert_valid(AW) ; wait_for_ready(AW) ; end_phase(AW) ; begin_phase(W, ...) ; ...`") or a brief "(none — this is a Transaction-API-only method, no Channel-API equivalent)".
- This decomposition is the contract that `/spec-lint` LINT-BFM-005 checks: every Transaction API method must reference Channel API equivalents or note their absence.

(Repeat for every method.)

## Behavior under reset

State explicitly what every API method does when the protocol reset signal asserts during a call:
- <e.g. "All in-flight transactions are dropped. The blocking caller is unblocked with status `RESET_DURING_TRANSACTION`. Outstanding configuration knobs are preserved (set_response_delay, set_response_fault); they survive reset.">

## Concurrency rules

State what is and is not safe to call concurrently:
- <e.g. "apply_write and apply_read may be called concurrently from separate testbench threads. Concurrent apply_write calls to overlapping addresses are an error and produce status `CONCURRENT_OVERLAP`.">
- <e.g. "Transaction API and Channel API may not be mixed for an in-flight transaction. Specifically: once apply_write begins, channel_api calls on AW, W, B are forbidden until apply_write returns. Mixing produces undefined behavior and the BFM logs an error.">
```

## Writing rules for this file

1. **Every method has all five sub-sections.** Signature, Preconditions, Side effects, Return value, Error modes — and the Equivalent channel API decomposition. Missing any of these makes the API ambiguous from the test author's perspective.
2. **Preconditions are exhaustive, not exemplary.** "BFM has been instantiated" is one of typically 3–5 preconditions. List them all. A test author hits an undocumented precondition once and never trusts the API again.
3. **Side effects include configuration state.** If `apply_write` clears a one-shot fault (`set_response_fault`), that's a side effect — list it. Hidden configuration interactions are the #1 source of "the BFM behaves randomly" tickets.
4. **Error modes are an exhaustive list, not a sample.** Every status value the method can return is documented. "And other errors as appropriate" is unacceptable.
5. **The Equivalent channel API decomposition is mandatory.** Either a step-by-step mapping to channel_api.md calls, or "(none — this is Transaction-API-only)". `/spec-lint` LINT-BFM-005 will fail if this row is missing or empty.
6. **State the blocking discipline once.** All-blocking, all-non-blocking, or mixed — pick one and state it in API conventions. Do not describe blocking behavior method-by-method (it drifts).
7. **Concurrency contract is explicit, not implied.** The §Concurrency rules section must state for every method pair (or category): (a) which combinations are safe to call from concurrent threads, (b) which combinations produce undefined behavior, and (c) for unsafe combinations, **whether the BFM enforces serialisation (e.g., via internal mutex) or relies on the test author to serialise externally**. "Racy, behavior undefined" without saying who's responsible for prevention leaves implementers guessing whether to add a mutex.
8. **State-mutating methods (`reset_state`, `clear_*`, `flush_*`) enumerate every state field.** The Side effects sub-section must list every internal state field the method touches with explicit "cleared / preserved / reset to default" annotation. Implicit "everything else preserved" leaves edge cases (configuration knobs, mode flags, observation lists) ambiguous. List them all, even ones that seem obviously preserved.

## Anti-patterns

- **Anti-pattern:** "apply_write returns status; common values are OK and ERROR." What about the others? Enumerate.
- **Anti-pattern:** Documenting only the success path. Test authors care about failure paths more than success paths — that is what they are checking the DUT against.
- **Anti-pattern:** Describing internal implementation in the side effects ("calls the internal driver task `axi_drive_aw`"). The reader does not have the BFM source. Side effects are observable from outside: which wires move, which config knobs change.
- **Anti-pattern:** Methods whose names suggest different behavior than they have. `apply_write` that does not block is a trap. Either rename to `start_write` or document the blocking behavior loudly.
- **Anti-pattern:** A "Pre-conditions: see API conventions" row instead of listing the per-method preconditions. Conventions cover the global preconditions; per-method preconditions still exist (e.g. "no concurrent overlap on this address") and must appear here.

## Mini-example: AXI-Lite slave BFM (apply_write only — full file enumerates apply_read, expect_*, set_*, reset_state)

```
# Transaction API

## API conventions

- Pseudocode style: C-like; maps cleanly to SystemVerilog tasks, SystemC functions, and C++ methods via DPI-C.
- Naming: snake_case. Verb-noun for actions (`apply_write`); `expect_<noun>` for monitor-mode assertions.
- Error reporting: every method returns `status_t`. `OK` on success; one of the documented error enums otherwise.
- Blocking discipline: all methods block until the transaction completes (BVALID handshake for writes; RVALID handshake for reads). For non-blocking control, use channel_api.md.

## Method index

| Group         | Method                              | One-line summary |
|---------------|-------------------------------------|------------------|
| Stimulus      | (n/a — slave BFM does not initiate transactions)              |  |
| Monitor       | expect_write(addr, data)            | Block until a write to addr is seen; assert WDATA matches data; fail otherwise. |
| Monitor       | expect_read(addr) -> data            | Block until a read from addr is seen; return the BFM-supplied RDATA. |
| Configuration | set_response_delay(min_cycles, max_cycles) | Configure ACLK-cycle delay bounds for response handshakes. |
| Configuration | set_response_fault(kind)            | Inject a one-shot response fault on the next handshake of the matching direction. |
| Configuration | set_register_value(addr, value)     | Pre-load a value into the BFM's internal address-indexed memory; subsequent reads to addr return value. |
| State         | reset_state()                       | Drop outstanding-transaction trackers and reset configuration to defaults. Does not toggle ARESETn. |

## Method details

### expect_write(addr, data) -> status_t

**Signature:** `status_t expect_write(uint32_t addr, uint32_t data)`

**Preconditions:**
- BFM has been instantiated and clocked; ARESETn has deasserted at least once since instantiation.
- No concurrent expect_write call for the same addr is pending. (Concurrent calls for *different* addresses are legal.)
- No Channel-API call is in progress on AW or W channels.

**Side effects:**
- Blocks the caller until the master DUT issues a write to addr. The BFM waits passively; it does not back-pressure the AW/W channels beyond the configured response delay.
- Once the write arrives: the BFM drives BVALID/BRESP per `set_response_delay` and `set_response_fault` configuration. WDATA is captured and compared against data.
- The configured response fault (if any) is consumed (one-shot semantics).

**Return value:**
- `OK`: the write was observed and WDATA matched data.
- `DATA_MISMATCH`: a write to addr was observed, but WDATA did not equal data. The BFM logs the actual WDATA value and the expected data.
- `STRB_PARTIAL`: a write to addr was observed with WSTRB != all-ones; the data comparison was performed only over the strobed bytes; remaining bytes were not checked. (RECOMMEND-severity warning; not a failure.)
- `RESET_DURING_TRANSACTION`: ARESETn asserted before the write was observed; expect_write returns without observing a write. The caller may retry after reset deasserts.
- `TIMEOUT`: configurable timeout (default: 10000 ACLK cycles) elapsed before any write to addr was seen.

**Error modes:**
| Status                     | Trigger                                                   | BFM logs? | Auto-retry? |
|----------------------------|-----------------------------------------------------------|-----------|-------------|
| DATA_MISMATCH              | Write to addr observed; WDATA != data                     | yes       | no          |
| STRB_PARTIAL               | Write observed with WSTRB != all-ones                     | yes       | no          |
| RESET_DURING_TRANSACTION   | ARESETn asserted before observation completed             | yes       | no          |
| TIMEOUT                    | No write to addr in 10000 ACLK cycles                     | yes       | no          |

**Example:**
```c
status_t s = expect_write(0x40, 0xDEADBEEF);
if (s == DATA_MISMATCH) {
    /* DUT wrote a different value; log captured WDATA */
}
```

**Equivalent channel API decomposition:**
The expect_write method is equivalent to:

```
wait_for_phase(AW, addr_match=addr) ;
wait_for_phase(W) ;
end_phase(AW) ;
end_phase(W) ;
begin_phase(B, BRESP=<per fault config>) ;
wait_for_ready(B) ;
end_phase(B) ;
```

Plus the WDATA-vs-data comparison and status assignment, which have no channel-API equivalent (the channel API is data-agnostic).
```

The full transaction_api.md continues with sub-sections for each remaining method, following the same five-sub-section template.

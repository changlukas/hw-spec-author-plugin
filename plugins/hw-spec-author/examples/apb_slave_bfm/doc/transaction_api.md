# Transaction API

## API conventions

- Pseudocode style: C-like; maps cleanly to SystemVerilog tasks, SystemC functions, and C++ methods via DPI-C.
- Naming: snake_case. `expect_<noun>` for monitor-mode assertions; `set_<knob>` for configuration.
- Error reporting: every method returns `status_t`. `OK` on success.
- Blocking discipline: `expect_*` methods block until the matching DUT activity occurs (or a configurable timeout fires). All `set_*` and `reset_state` are non-blocking.

## When the Transaction API is insufficient

Transaction-API methods cover ~95% of common test scenarios: single read, single write with PSTRB, error response injection, monitor-only observation. Use channel_api.md instead for:

- Holding PREADY LOW for an extended interval to test master DUT timeout behavior.
- Driving PSLVERR=1 with PREADY=0 (illegal handshake) to verify master DUT's protocol-checker assertions.
- Driving per-cycle deterministic patterns on PREADY rather than the random wait-state range that `set_wait_states` provides.

## Method index

| Group         | Method                                       | One-line summary |
|---------------|----------------------------------------------|------------------|
| Monitor       | expect_write(addr, data, [timeout])          | Block until a write to addr; assert PWDATA matches data. |
| Monitor       | expect_read(addr, [timeout]) -> data         | Block until a read from addr; return BFM-supplied PRDATA. |
| Monitor       | get_observed_writes() -> list                | Return all observed writes since last reset_state. |
| Monitor       | get_observed_reads() -> list                 | Return all observed reads since last reset_state. |
| Configuration | set_wait_states(min_cycles, max_cycles)      | Configure wait-state count for ACCESS-phase PREADY assertion. |
| Configuration | set_response_fault()                         | Inject a one-shot PSLVERR on the next ACCESS-phase completion. |
| Configuration | set_register_value(addr, value)              | Pre-load BFM internal memory at addr. |
| Configuration | set_bfm_mode(mode)                           | Switch ACTIVE / PASSIVE; see active_passive_mode.md. |
| State         | reset_state()                                | Drop trackers + observed lists; restore configuration to defaults. |

## Method details

### expect_write(addr, data, timeout=10000) -> status_t

**Signature:** `status_t expect_write(uint32_t addr, uint32_t data, uint32_t timeout_cycles = 10000)`

**Preconditions:**
- BFM instantiated, clocked, post-reset.
- BFM mode is ACTIVE (in PASSIVE, expect_write monitors but BFM does not drive PREADY/PSLVERR).
- No concurrent expect_write to the same addr.

**Side effects:**
- Blocks until master DUT issues a write to addr (PSEL && PWRITE && PADDR == addr observed in SETUP+ACCESS sequence).
- In ACTIVE: BFM drives PREADY per `set_wait_states`, drives PSLVERR per `set_response_fault`. The configured fault (if any) is consumed (one-shot).
- PWDATA captured and compared against data. If APB4 PSTRB is present and != all-ones, comparison is performed only over strobed bytes.
- Observed write appended to get_observed_writes list.

**Return value:**
- `OK`: write observed, PWDATA matched.
- `DATA_MISMATCH`: write observed, PWDATA did not equal data.
- `STRB_PARTIAL`: write observed with PSTRB != all-ones; comparison was over strobed bytes only.
- `RESET_DURING_TRANSACTION`: PRESETn asserted before observation completed.
- `MODE_SWITCHED_TO_PASSIVE`: bfm_mode set to PASSIVE during call.
- `TIMEOUT`: timeout_cycles elapsed.

**Error modes:**
| Status                     | Trigger                                              | BFM logs? |
|----------------------------|------------------------------------------------------|-----------|
| DATA_MISMATCH              | Write to addr observed; PWDATA != data               | yes       |
| STRB_PARTIAL               | Write observed with PSTRB != all-ones                | yes       |
| RESET_DURING_TRANSACTION   | PRESETn asserted before observation completed        | yes       |
| MODE_SWITCHED_TO_PASSIVE   | bfm_mode set to PASSIVE during call                  | yes       |
| TIMEOUT                    | timeout_cycles PCLK cycles elapsed                   | yes       |

**Example:**
```c
status_t s = expect_write(0x40, 0xDEADBEEF);
```

**Equivalent channel API decomposition:**
```
begin_phase_setup() ;
wait_for_setup(addr_match=addr, write=true) ;  /* observes PSEL=1 + PENABLE=0 */
end_phase_setup() ;
begin_phase_access(PSLVERR=<per fault config>) ;
wait_for_enable() ;                            /* observes PENABLE rising */
assert_pready() ;                              /* after wait-state countdown */
wait_for_master_observe() ;                    /* one PCLK after PREADY */
end_phase_access() ;
```
Plus the PWDATA-vs-data comparison and status assignment, which have no channel-API equivalent.

### expect_read(addr, timeout=10000) -> (status_t, data)

**Signature:** `(status_t, uint32_t) expect_read(uint32_t addr, uint32_t timeout_cycles = 10000)`

**Preconditions:** BFM instantiated, post-reset, ACTIVE mode (PASSIVE does not supply PRDATA).

**Side effects:**
- Blocks until master DUT issues a read to addr.
- BFM drives PRDATA per `set_register_value` (if pre-loaded for addr) or **defaults to 0x0 if no `set_register_value(addr, ...)` has been issued for the address being read**; drives PREADY per `set_wait_states`; drives PSLVERR per `set_response_fault`.
- Observed read appended to get_observed_reads.

**Return value:** `(OK, prdata)` on success; `(RESET_DURING_TRANSACTION, 0)`, `(MODE_SWITCHED_TO_PASSIVE, 0)`, `(TIMEOUT, 0)` as applicable.

**Error modes:** as for expect_write minus DATA_MISMATCH and STRB_PARTIAL.

**Example:**
```c
status_t s; uint32_t rdata;
set_register_value(0x100, 0xABCD1234);
(s, rdata) = expect_read(0x100);
assert(s == OK && rdata == 0xABCD1234);
```

**Equivalent channel API decomposition:**
```
begin_phase_setup() ;
wait_for_setup(addr_match=addr, write=false) ;
end_phase_setup() ;
begin_phase_access(PRDATA=<from set_register_value or 0x0>, PSLVERR=<per fault config>) ;
wait_for_enable() ;
assert_pready() ;
wait_for_master_observe() ;
end_phase_access() ;
```

### set_wait_states(min_cycles, max_cycles)

**Signature:** `void set_wait_states(uint32_t min_cycles, uint32_t max_cycles)`

**Preconditions:** min_cycles ≤ max_cycles. Both ≤ 65535.

**Side effects:** BFM internal configuration updated. The next ACCESS phase will hold PREADY LOW for a uniformly random number of PCLK cycles in [min_cycles, max_cycles] inclusive before raising it. Persists across transactions until reconfigured or reset_state.

**Return value:** none.

**Error modes:** none — illegal arguments trigger immediate assertion failure.

**Example:** `set_wait_states(0, 5);  /* random 0..5 cycles per transaction */`

**Equivalent channel API decomposition:** (none — Transaction-API-only; per-channel wait pattern is set via begin_phase_access timing.)

### set_response_fault()

**Signature:** `void set_response_fault(void)`

**Preconditions:** BFM instantiated, post-reset.

**Side effects:** One-shot fault flag set. The next ACCESS-phase completion (whether read or write) drives PSLVERR=1 alongside PREADY=1. Flag automatically cleared after firing.

**Return value:** none.

**Error modes:** none.

**Example:**
```c
set_response_fault();
expect_write(0x40, 0xDEADBEEF);   /* this write's PSLVERR will be 1 */
expect_write(0x44, 0x12345678);   /* this write's PSLVERR is 0 (one-shot cleared) */
```

**Equivalent channel API decomposition:** (none — Transaction-API-only; the flag is consumed inside begin_phase_access.)

### set_register_value(addr, value)

**Signature:** `void set_register_value(uint32_t addr, uint32_t value)`

**Preconditions:** addr within ADDR_WIDTH range.

**Side effects:** BFM internal memory at addr set to value. Subsequent reads return value (until overwritten).

**Return value:** none. **Error modes:** none.

**Equivalent channel API decomposition:** (none — Transaction-API-only.)

### set_bfm_mode(mode)

**Signature:** `void set_bfm_mode(bfm_mode_t mode)` where `bfm_mode_t ∈ {ACTIVE, PASSIVE}`

See active_passive_mode.md for full semantics.

**Equivalent channel API decomposition:** (none — global state change.)

### reset_state()

**Signature:** `void reset_state(void)`

**Side effects:** Resets internal BFM state. Per-field behavior:

| Internal state field            | reset_state() effect | Notes |
|---------------------------------|----------------------|-------|
| In-flight transaction tracker   | Cleared (→ IDLE)     | Any active SETUP / ACCESS phase is dropped. |
| Observed-writes list            | Cleared              | Past observation history discarded. |
| Observed-reads list             | Cleared              | Past observation history discarded. |
| One-shot response fault flag    | Cleared              | Pending fault discarded. |
| `set_wait_states(min, max)`     | Reset to (0, 0)      | Configuration returns to defaults. |
| `set_register_value(addr, ...)` | **Preserved**        | Pre-loaded memory contents kept across reset_state calls. |
| `bfm_mode` (ACTIVE/PASSIVE)     | **Preserved**        | Mode is testbench-level config, not transaction-level state. |

Does not toggle PRESETn (the wire is testbench-controlled).

**Equivalent channel API decomposition:** (none — internal state reset.)

## Behavior under reset

When PRESETn asserts during any in-flight Transaction-API call: all `expect_*` calls unblock with `RESET_DURING_TRANSACTION`. In-flight transaction tracker dropped. Configuration knobs preserved.

## Concurrency rules

APB has only one transaction in flight, so per-thread concurrency is restricted:

| Combination                                                              | Safety                  | Who enforces |
|--------------------------------------------------------------------------|-------------------------|--------------|
| Two `expect_*` calls from different threads                              | **Unsafe**              | **Test author must serialise externally.** The BFM does NOT add an internal mutex — race condition is observable (e.g., one call may return for the wrong transaction). |
| `expect_*` and `set_*` (config knob) concurrently                        | Safe                    | BFM (set_* writes are atomic). |
| `set_*` and `set_*` concurrently (different knobs)                       | Safe                    | BFM. |
| `set_*` and `set_*` concurrently (same knob)                             | Safe; last-write-wins   | BFM. |
| `expect_*` while a Channel API call is in progress on the same channel  | **Unsafe**              | BFM enforces: Channel API method returns `BUSY_TXN_API` if a Transaction API call is blocked on the bus. |
| Transaction API and Channel API on the same in-flight transaction       | **Forbidden**           | BFM enforces via BUSY_TXN_API. |
| `reset_state()` while `expect_*` is blocked                              | **Unsafe**              | Test author should not call reset_state mid-expect; result is undefined. |

**Why no internal mutex on `expect_*` calls**: APB has fundamentally single-transaction semantics, and the natural usage pattern (one test thread driving expectations sequentially) is single-threaded. Adding a mutex would mask test-author bugs (e.g., accidentally racing two expect_* calls) rather than expose them.

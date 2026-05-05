# Transaction API

## API conventions

- Pseudocode style: C-like; maps cleanly to SystemVerilog tasks, SystemC functions, and C++ methods via DPI-C.
- Naming: snake_case. Verb-noun for actions (`apply_*`); `expect_<noun>` for monitor-mode assertions; `set_<knob>` for configuration.
- Error reporting: every method returns `status_t`. `OK` on success; one of the documented error enums otherwise.
- Blocking discipline: `expect_*` methods block until the matching DUT activity occurs (or a configurable timeout fires). All `set_*` and `reset_state` methods are non-blocking and return immediately.

## When the Transaction API is insufficient

Transaction-API methods cover ~95% of common test scenarios: single read, single write, read with pre-loaded value, write with one-shot fault, monitor-only observation. Scenarios that **require channel_api.md** instead:

- Driving AWREADY / WREADY / ARREADY without an accompanying response (slave back-pressure recovery testing).
- Holding WREADY / BREADY low for extended intervals to verify master DUT timeout behavior.
- Injecting illegal handshake sequences (e.g., assert BVALID before any AW handshake) to test master DUT's protocol-checker assertions.
- Driving per-cycle deterministic patterns on AWREADY / ARREADY (rather than the random delay range that `set_response_delay` provides).

If the test fits one of these patterns, switch to channel_api.md. Otherwise, the methods below are sufficient.

## Method index

| Group         | Method                                | One-line summary |
|---------------|---------------------------------------|------------------|
| Stimulus      | (n/a — slave BFM does not initiate transactions; the master DUT initiates) | |
| Monitor       | expect_write(addr, data, [timeout])   | Block until a write to addr is seen; assert WDATA matches data; fail otherwise. |
| Monitor       | expect_read(addr, [timeout])          | Block until a read from addr is seen; return the BFM-supplied RDATA. |
| Monitor       | get_observed_writes() -> list         | Return the list of all write transactions observed since last reset_state, including those not matched by any expect_write. |
| Monitor       | get_observed_reads() -> list          | Return the list of all read transactions observed since last reset_state. |
| Configuration | set_response_delay(min_cycles, max_cycles) | Configure ACLK-cycle delay bounds for response handshakes. |
| Configuration | set_response_fault(kind)              | Inject a one-shot response fault on the next handshake. |
| Configuration | set_register_value(addr, value)       | Pre-load a value into the BFM's internal address-indexed memory. |
| Configuration | set_bfm_mode(mode)                    | Switch between ACTIVE and PASSIVE; see active_passive_mode.md. |
| State         | reset_state()                         | Drop outstanding-transaction trackers and observed-transaction lists; restore configuration to defaults. Does not toggle ARESETn. |

## Method details

### expect_write(addr, data, timeout=10000) -> status_t

**Signature:** `status_t expect_write(uint32_t addr, uint32_t data, uint32_t timeout_cycles = 10000)`

**Preconditions:**
- BFM has been instantiated and clocked; ARESETn has deasserted at least once since instantiation.
- BFM mode is ACTIVE (passive mode does not drive responses, but expect_write still works in passive mode for monitoring).
- No concurrent expect_write call for the same addr is pending. (Concurrent calls for *different* addresses are legal.)
- No Channel-API call is in progress on AW or W channels.

**Side effects:**
- Blocks the caller until the master DUT issues a write to addr.
- Once the write arrives: in ACTIVE mode, the BFM drives BVALID/BRESP per `set_response_delay` and `set_response_fault` configuration. In PASSIVE mode, the BFM observes the write but does not drive B; an external active slave is assumed.
- WDATA is captured and compared against data; only bytes whose WSTRB bit is HIGH are compared (per protocol_rules.md `AXI4LITE_SLV_W_WSTRB_BYTE_LANES`).
- The configured response fault (if any) is consumed (one-shot semantics).
- The observed write is appended to the get_observed_writes list.

**Return value:**
- `OK`: the write was observed and WDATA matched data.
- `DATA_MISMATCH`: a write to addr was observed, but WDATA did not equal data.
- `STRB_PARTIAL`: a write to addr was observed with WSTRB != all-ones; comparison was performed only over the strobed bytes.
- `RESET_DURING_TRANSACTION`: ARESETn asserted before observation completed.
- `MODE_SWITCHED_TO_PASSIVE`: bfm_mode was switched to PASSIVE mid-call.
- `TIMEOUT`: timeout_cycles ACLK cycles elapsed before any write to addr was seen.

**Error modes:**
| Status                     | Trigger                                                   | BFM logs? | Auto-retry? |
|----------------------------|-----------------------------------------------------------|-----------|-------------|
| DATA_MISMATCH              | Write to addr observed; WDATA != data                     | yes       | no          |
| STRB_PARTIAL               | Write observed with WSTRB != all-ones                     | yes       | no          |
| RESET_DURING_TRANSACTION   | ARESETn asserted before observation completed             | yes       | no          |
| MODE_SWITCHED_TO_PASSIVE   | bfm_mode set to PASSIVE during call                       | yes       | no          |
| TIMEOUT                    | timeout_cycles elapsed                                    | yes       | no          |

**Example:**
```c
status_t s = expect_write(0x40, 0xDEADBEEF);
if (s == DATA_MISMATCH) {
    /* DUT wrote a different value; log captured WDATA via get_observed_writes() */
}
```

**Equivalent channel API decomposition:**
```
wait_for_phase(AW, addr_match=addr) ;
wait_for_phase(W) ;
end_phase(AW) ;
end_phase(W) ;
begin_phase(B, BRESP=<per fault config>) ;
wait_for_ready(B) ;
end_phase(B) ;
```
Plus the WDATA-vs-data comparison and status assignment, which have no channel-API equivalent.

### expect_read(addr, timeout=10000) -> (status_t, data)

**Signature:** `(status_t, uint32_t) expect_read(uint32_t addr, uint32_t timeout_cycles = 10000)`

**Preconditions:**
- BFM instantiated, clocked, post-reset.
- BFM mode is ACTIVE (in PASSIVE mode, no RDATA can be supplied — use get_observed_reads instead).
- No concurrent expect_read call for the same addr is pending.
- No Channel-API call is in progress on AR or R channels.

**Side effects:**
- Blocks until the master DUT issues a read to addr.
- BFM drives RVALID/RDATA/RRESP per: `set_register_value` (if pre-loaded for addr) or **defaults to 0x0 if no `set_register_value(addr, ...)` has been issued for the address being read**; per `set_response_delay`; per `set_response_fault`.
- The configured response fault (if any) is consumed (one-shot).
- The observed read is appended to get_observed_reads.

**Return value:**
- `(OK, rdata)`: read observed; rdata is the value the BFM supplied.
- `(RESET_DURING_TRANSACTION, 0)`, `(MODE_SWITCHED_TO_PASSIVE, 0)`, `(TIMEOUT, 0)`: as for expect_write.

**Error modes:**
| Status                     | Trigger                                                   | BFM logs? | Auto-retry? |
|----------------------------|-----------------------------------------------------------|-----------|-------------|
| RESET_DURING_TRANSACTION   | ARESETn asserted before observation completed             | yes       | no          |
| MODE_SWITCHED_TO_PASSIVE   | bfm_mode set to PASSIVE during call                       | yes       | no          |
| TIMEOUT                    | timeout_cycles elapsed                                    | yes       | no          |

**Example:**
```c
status_t s; uint32_t rdata;
set_register_value(0x100, 0xABCD1234);
(s, rdata) = expect_read(0x100);
assert(s == OK && rdata == 0xABCD1234);
```

**Equivalent channel API decomposition:**
```
wait_for_phase(AR, addr_match=addr) ;
end_phase(AR) ;
begin_phase(R, RDATA=<from set_register_value or 0x0>, RRESP=<per fault config>) ;
wait_for_ready(R) ;
end_phase(R) ;
```

### set_response_delay(min_cycles, max_cycles)

**Signature:** `void set_response_delay(uint32_t min_cycles, uint32_t max_cycles)`

**Preconditions:**
- min_cycles ≤ max_cycles.
- Both ≤ 65535 (BFM internal counter width).

**Side effects:**
- BFM internal configuration updated. The next response handshake (B for writes, R for reads) is delayed by a uniformly random number of ACLK cycles in [min_cycles, max_cycles] inclusive. Persists across transactions until reconfigured or reset_state.

**Return value:** none.

**Error modes:** none — illegal arguments (min > max, value > 65535) trigger an immediate assertion failure rather than returning a status.

**Example:**
```c
set_response_delay(2, 10);  /* random latency 2..10 ACLK cycles */
```

**Equivalent channel API decomposition:** (none — this is Transaction-API-only; there is no per-channel knob to set delay.)

### set_response_fault(kind)

**Signature:** `void set_response_fault(fault_kind_t kind)` where `fault_kind_t ∈ {NONE, SLVERR, DECERR}`

**Preconditions:** BFM instantiated, post-reset.

**Side effects:**
- BFM internal one-shot fault flag set. The next response handshake (B for writes, R for reads — whichever fires first) carries the corresponding BRESP/RRESP value:
  - SLVERR → 2'b10
  - DECERR → 2'b11
  - NONE → no effect (clears any pending fault)
- After the next response, the flag is automatically cleared (one-shot).

**Return value:** none.

**Error modes:** none.

**Example:**
```c
set_response_fault(SLVERR);
expect_write(0x40, 0xDEADBEEF);   /* this write's BRESP will be SLVERR */
expect_write(0x44, 0x12345678);   /* this write's BRESP returns to OKAY (fault was one-shot) */
```

**Equivalent channel API decomposition:** (none — this is Transaction-API-only; the fault flag is consumed inside the next `begin_phase(B, ...)` or `begin_phase(R, ...)`.)

### set_register_value(addr, value)

**Signature:** `void set_register_value(uint32_t addr, uint32_t value)`

**Preconditions:** addr is within the BFM's modelled address space (defined by ADDR_WIDTH).

**Side effects:**
- BFM internal memory at addr is set to value. Subsequent reads to addr return value (until overwritten by a write or by reset_state).
- Does not affect any in-flight transaction.

**Return value:** none.

**Error modes:** none.

**Example:**
```c
set_register_value(0x100, 0xCAFEBABE);
/* When the master DUT issues a read of 0x100, RDATA will be 0xCAFEBABE. */
```

**Equivalent channel API decomposition:** (none — this is Transaction-API-only; the internal memory is read inside `begin_phase(R, RDATA=...)`.)

### set_bfm_mode(mode)

**Signature:** `void set_bfm_mode(bfm_mode_t mode)` where `bfm_mode_t ∈ {ACTIVE, PASSIVE}`

**Preconditions:** none.

**Side effects:**
- See active_passive_mode.md §Mode switch for full semantics. Briefly: ACTIVE → PASSIVE causes all BFM-driven outputs to float to during-reset values within one ACLK cycle and unblocks any in-flight expect_*/Channel-API call with status `MODE_SWITCHED_TO_PASSIVE`.

**Return value:** none.

**Error modes:** none.

**Example:**
```c
set_bfm_mode(PASSIVE);  /* stop driving; observe only */
```

**Equivalent channel API decomposition:** (none — mode switch is a global state change, not a per-channel operation.)

### reset_state()

**Signature:** `void reset_state(void)`

**Preconditions:** none.

**Side effects:** Resets internal BFM state. Per-field behavior:

| Internal state field                  | reset_state() effect | Notes |
|---------------------------------------|----------------------|-------|
| Outstanding-write tracker             | Cleared (→ IDLE)     | Any in-flight write transaction is dropped. |
| Outstanding-read tracker              | Cleared (→ IDLE)     | Any in-flight read transaction is dropped. |
| Observed-writes list                  | Cleared              | Past observation history discarded. |
| Observed-reads list                   | Cleared              | Past observation history discarded. |
| One-shot response fault flag          | Cleared              | Pending fault discarded. |
| `set_response_delay(min, max)`        | Reset to (0, 0)      | Delay configuration returns to defaults. |
| `set_register_value(addr, ...)` map   | **Preserved**        | Pre-loaded memory contents kept across reset_state calls. To clear specific addresses, call `set_register_value` with new values; to clear all, no built-in API exists (re-instantiate the BFM). |
| `bfm_mode` (ACTIVE/PASSIVE)           | **Preserved**        | Mode is testbench-level config, not transaction-level state. |

Does **not** toggle ARESETn (the wire is testbench-controlled).

**Return value:** none.

**Error modes:** none.

**Example:**
```c
reset_state();  /* fresh starting point for the next test phase */
```

**Equivalent channel API decomposition:** (none — internal state reset; no wire-level effect.)

## Behavior under reset

When ARESETn asserts during any in-flight Transaction-API call:
- All `expect_*` calls unblock with status `RESET_DURING_TRANSACTION`.
- Outstanding-transaction trackers are dropped.
- Configuration knobs (`set_response_delay`, `set_response_fault`, `set_register_value`) are **preserved** across ARESETn — they are testbench-side state, not BFM-internal hardware state.
- The wire-level reset behavior is in pin_level_reset.md; this section covers only the API contract.

## Concurrency rules

AXI4-Lite has independent write (AW/W/B) and read (AR/R) channels, so per-direction concurrency is allowed:

| Combination                                                                       | Safety                  | Who enforces |
|-----------------------------------------------------------------------------------|-------------------------|--------------|
| Concurrent `expect_write` + `expect_read` from different threads                  | **Safe**                | BFM (channels are independent). |
| Two concurrent `expect_write` to **different** addresses                          | Safe (sequential observation; both eventually resolve as transactions arrive) | BFM (queues observations on AW match). |
| Two concurrent `expect_write` to **the same / overlapping** address               | **Unsafe**              | BFM enforces: whichever call resolves second returns `CONCURRENT_OVERLAP`. |
| Concurrent `set_*` calls (different knobs)                                        | Safe                    | BFM. |
| Concurrent `set_*` calls (same knob)                                              | Safe; last-write-wins   | BFM. |
| `expect_*` while a Channel API call is in progress on the same channel set       | **Forbidden**           | BFM enforces: Channel API method returns `BUSY_TXN_API` when the relevant channels are owned by an in-flight Transaction API call. |
| Transaction API and Channel API on the same in-flight transaction                | **Forbidden**           | BFM enforces via BUSY_TXN_API. |
| `reset_state()` while `expect_*` is blocked                                       | **Unsafe**              | Test author should not call reset_state mid-expect; result is undefined. |

**Why concurrent writes to overlapping addresses are caught (vs APB-style "BFM doesn't arbitrate")**: AXI4-Lite has independent channels, so multi-threaded test patterns (one thread per direction) are normal and supported. Same-address race is a real test-author bug pattern (two threads both expecting to observe the same write); the BFM detects and reports it.

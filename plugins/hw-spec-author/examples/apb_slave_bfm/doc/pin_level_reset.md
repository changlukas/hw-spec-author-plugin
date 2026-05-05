# Pin-Level Reset Behavior

**Reset signal:** PRESETn (defined in signal_interface.md §Protocol clock and reset).
**Reset assertion (active level):** low.
**Minimum assertion duration:** 4 PCLK cycles.
**Synchronicity to clock:** asynchronous assertion, synchronous deassertion to PCLK rising edge.

## During reset (PRESETn == 0)

| Signal   | Value during reset | Notes |
|----------|--------------------|-------|
| PSEL     | as driven by DUT   | input from master DUT |
| PENABLE  | as driven by DUT   |       |
| PADDR    | as driven by DUT   |       |
| PWRITE   | as driven by DUT   |       |
| PWDATA   | as driven by DUT   |       |
| PREADY   | 0                  |       |
| PRDATA   | 0x0...0            | Driven to all-zero rather than X for waveform readability. |
| PSLVERR  | 0                  |       |
| PPROT    | as driven by DUT   | APB4 only |
| PSTRB    | as driven by DUT   | APB4 only |

## After reset (first PCLK rising edge with PRESETn == 1)

| Signal   | Value first cycle post-reset | Notes |
|----------|------------------------------|-------|
| PSEL     | as driven by DUT             |       |
| PENABLE  | as driven by DUT             |       |
| PADDR    | as driven by DUT             |       |
| PWRITE   | as driven by DUT             |       |
| PWDATA   | as driven by DUT             |       |
| PREADY   | 0                            | Returns to 1 only after the configured wait-state count elapses for the next ACCESS phase; see channel_handshake.md. |
| PRDATA   | 0x0...0                      | Holds at 0 until a read transaction completes ACCESS phase. |
| PSLVERR  | 0                            | Holds at 0 until a write or read with active fault injection completes. |
| PPROT    | as driven by DUT             | APB4 only |
| PSTRB    | as driven by DUT             | APB4 only |

## Reset entry sequencing

1. PRESETn asserts (asynchronous). All BFM-driven outputs (PREADY, PRDATA, PSLVERR) drive their during-reset values combinationally.
2. While PRESETn is low: any in-flight transaction tracker (currently in SETUP or ACCESS phase) is dropped; pending wait-state countdowns cancelled; pending one-shot fault flag cleared. Testbench API calls during reset are silently dropped with a warning logged.
3. PRESETn deasserts on a rising PCLK edge. On that same rising edge, BFM outputs transition to their "After reset" values; PREADY remains 0 until the BFM accepts the next SETUP phase.

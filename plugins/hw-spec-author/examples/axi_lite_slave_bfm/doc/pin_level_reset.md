# Pin-Level Reset Behavior

**Reset signal:** ARESETn (defined in signal_interface.md §Protocol clock and reset).
**Reset assertion (active level):** low.
**Minimum assertion duration:** 16 ACLK cycles.
**Synchronicity to clock:** asynchronous assertion, synchronous deassertion to ACLK rising edge.

## During reset (ARESETn == 0)

| Channel | Signal   | Value during reset | Notes |
|---------|----------|--------------------|-------|
| AW      | AWVALID  | as driven by DUT   | input from master DUT — BFM samples but does not act |
| AW      | AWREADY  | 0                  |       |
| AW      | AWADDR   | as driven by DUT   |       |
| AW      | AWPROT   | as driven by DUT   |       |
| W       | WVALID   | as driven by DUT   |       |
| W       | WREADY   | 0                  |       |
| W       | WDATA    | as driven by DUT   |       |
| W       | WSTRB    | as driven by DUT   |       |
| B       | BVALID   | 0                  |       |
| B       | BREADY   | as driven by DUT   |       |
| B       | BRESP    | 0x0 (OKAY)         | Driven to OKAY rather than X to keep waveforms readable; receiver ignores BRESP unless BVALID is high. |
| AR      | ARVALID  | as driven by DUT   |       |
| AR      | ARREADY  | 0                  |       |
| AR      | ARADDR   | as driven by DUT   |       |
| AR      | ARPROT   | as driven by DUT   |       |
| R       | RVALID   | 0                  |       |
| R       | RREADY   | as driven by DUT   |       |
| R       | RDATA    | 0x0...0            | Driven to all-zero rather than X for waveform readability. |
| R       | RRESP    | 0x0 (OKAY)         |       |

## After reset (first ACLK rising edge with ARESETn == 1)

| Channel | Signal   | Value first cycle post-reset | Notes |
|---------|----------|------------------------------|-------|
| AW      | AWVALID  | as driven by DUT             |       |
| AW      | AWREADY  | 0                            | Returns to 1 when the BFM is ready to accept the next address; see channel_handshake.md §Write transaction. |
| AW      | AWADDR   | as driven by DUT             |       |
| AW      | AWPROT   | as driven by DUT             |       |
| W       | WVALID   | as driven by DUT             |       |
| W       | WREADY   | 0                            | Returns to 1 only after AW phase completes for the corresponding write; rule `AXI4LITE_SLV_XCH_W_AFTER_AW`. |
| W       | WDATA    | as driven by DUT             |       |
| W       | WSTRB    | as driven by DUT             |       |
| B       | BVALID   | 0                            | Asserts only after both AW and W phases complete. |
| B       | BREADY   | as driven by DUT             |       |
| B       | BRESP    | 0x0 (OKAY)                   | Holds OKAY until BVALID asserts with the response for an actual transaction. |
| AR      | ARVALID  | as driven by DUT             |       |
| AR      | ARREADY  | 0                            | Returns to 1 when the BFM is ready to accept the next address. |
| AR      | ARADDR   | as driven by DUT             |       |
| AR      | ARPROT   | as driven by DUT             |       |
| R       | RVALID   | 0                            | Asserts only after AR phase completes and read data is available. |
| R       | RREADY   | as driven by DUT             |       |
| R       | RDATA    | 0x0...0                      |       |
| R       | RRESP    | 0x0 (OKAY)                   |       |

## Reset entry sequencing

1. ARESETn asserts (asynchronous). All BFM-driven outputs in the "During reset" table assert their stated values combinationally — no clock cycle of delay.
2. While ARESETn is low: any outstanding-transaction trackers (write-address-not-yet-acked, read-address-not-yet-acked, response-pending queues) are cleared. Testbench API calls during reset are silently dropped with a warning logged; see transaction_api.md §Behavior under reset.
3. ARESETn deasserts on the rising edge of ACLK. On that same rising edge, BFM-driven outputs transition to their "After reset" values; in particular, AWREADY and ARREADY remain 0 until the testbench enables stimulus acceptance via channel_api.md (default: enabled at reset deassertion in active mode).

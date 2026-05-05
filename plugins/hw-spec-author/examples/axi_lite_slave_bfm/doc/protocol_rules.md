# Protocol Rules

**Protocol:** AXI4-Lite (ARM IHI0022, ID022613, §B).
**Role:** slave (SLV).
**ID format:** AXI4LITE_SLV_<channel>_<name>.

**Severity legend (2 levels, matching ARM Protocol Checker):**
  - FAIL — slave BFM must reject violation (drive an assertion fail and log; never produce stimulus that violates).
  - RECOMMEND — quality-of-implementation; covered in coverage only.

**ARM SVA equivalent column convention:** Verified ARM IDs are listed verbatim. IDs suffixed `(unverified)` follow ARM naming pattern but await cross-check against ARM DUI 0534B / DUI 0576A. Rules with no ARM equivalent (BFM extensions: RST duration, all XCH cross-channel, all CFG configuration-knob) are tagged `(none)`.

## Reset rules

| ID | Condition | Required behavior | Severity | ARM SVA equivalent |
|----|-----------|-------------------|----------|--------------------|
| AXI4LITE_SLV_RST_OUTPUTS_LOW | ARESETn is asserted (LOW) | All BFM-driven VALID outputs (AWREADY, WREADY, BVALID, ARREADY, RVALID) must be LOW. | FAIL | AXI4_ERRS_RESET_ALL_OUTPUTS_LOW (unverified) |
| AXI4LITE_SLV_RST_DURATION    | ARESETn pulse begins      | ARESETn must be held LOW for at least 16 ACLK cycles before the BFM enters its defined reset state. Shorter pulses leave the BFM in an undefined state — BFM cannot guarantee any subsequent rule. | FAIL | (none) |

## AW rules

| ID | Condition | Required behavior | Severity | ARM SVA equivalent |
|----|-----------|-------------------|----------|--------------------|
| AXI4LITE_SLV_AW_AWVALID_STABLE   | AWVALID rises HIGH                                | AWVALID must remain HIGH until AWREADY is observed HIGH on a rising ACLK edge. | FAIL | AXI4_ERRM_AWVALID_STABLE |
| AXI4LITE_SLV_AW_AWADDR_STABLE    | AWVALID is HIGH                                   | AWADDR must not change between the cycle AWVALID rises and the cycle AWREADY is observed HIGH. | FAIL | AXI4_ERRM_AWADDR_STABLE (unverified) |
| AXI4LITE_SLV_AW_AWPROT_STABLE    | AWVALID is HIGH                                   | AWPROT must not change between the cycle AWVALID rises and the cycle AWREADY is observed HIGH. | FAIL | AXI4_ERRM_AWPROT_STABLE (unverified) |
| AXI4LITE_SLV_AW_AWPROT_IGNORED   | AWVALID + AWREADY handshake completes              | The slave BFM samples AWPROT but does not enforce protection-attribute checks; AWPROT values are recorded for monitor mode but never cause the slave to refuse a transaction. | RECOMMEND | (none) |
| AXI4LITE_SLV_AW_AWREADY_NO_LATCH | AWREADY is HIGH and AWVALID is LOW on the same cycle | AWREADY may be HIGH before AWVALID; this is legal per AXI4-Lite. The slave must not interpret a one-cycle AWREADY assertion as completing a transaction. | FAIL | AXI4_ERRS_AWREADY_NO_LATCH (unverified) |
| AXI4LITE_SLV_AW_AWREADY_MAX_WAIT | AWVALID has been HIGH for 1024 consecutive ACLK cycles without AWREADY observed HIGH | The slave BFM logs a back-pressure observation. The 1024-cycle threshold is implementation choice for monitoring health, not a protocol violation. | RECOMMEND | AXI4_RECS_AWREADY_MAX_WAIT (unverified) |

## W rules

| ID | Condition | Required behavior | Severity | ARM SVA equivalent |
|----|-----------|-------------------|----------|--------------------|
| AXI4LITE_SLV_W_WVALID_STABLE | WVALID rises HIGH | WVALID must remain HIGH until WREADY is observed HIGH on a rising ACLK edge. | FAIL | AXI4_ERRM_WVALID_STABLE (unverified) |
| AXI4LITE_SLV_W_WDATA_STABLE  | WVALID is HIGH    | WDATA must not change between the cycle WVALID rises and the cycle WREADY is observed HIGH. | FAIL | AXI4_ERRM_WDATA_STABLE (unverified) |
| AXI4LITE_SLV_W_WSTRB_STABLE  | WVALID is HIGH    | WSTRB must not change between the cycle WVALID rises and the cycle WREADY is observed HIGH. | FAIL | AXI4_ERRM_WSTRB_STABLE (unverified) |
| AXI4LITE_SLV_W_WSTRB_BYTE_LANES | WSTRB[i] is HIGH on the cycle WVALID + WREADY handshake completes | Only the bytes of WDATA whose corresponding WSTRB bit is HIGH are written to the BFM's internal memory at AWADDR. Bytes whose WSTRB bit is LOW are unchanged. | FAIL | (none) |

## B rules

| ID | Condition | Required behavior | Severity | ARM SVA equivalent |
|----|-----------|-------------------|----------|--------------------|
| AXI4LITE_SLV_B_BVALID_STABLE | BVALID rises HIGH | BVALID must remain HIGH until BREADY is observed HIGH on a rising ACLK edge. | FAIL | AXI4_ERRS_BVALID_STABLE (unverified) |
| AXI4LITE_SLV_B_BRESP_STABLE  | BVALID is HIGH    | BRESP must not change between the cycle BVALID rises and the cycle BREADY is observed HIGH. | FAIL | AXI4_ERRS_BRESP_STABLE (unverified) |
| AXI4LITE_SLV_B_BRESP_VALUES  | BVALID + BREADY handshake completes | BRESP must be one of: 2'b00 (OKAY), 2'b10 (SLVERR), 2'b11 (DECERR). 2'b01 (EXOKAY) is reserved in AXI-Lite and must not be used. | FAIL | AXI4_ERRS_BRESP_VALUES (unverified) |

## AR rules

| ID | Condition | Required behavior | Severity | ARM SVA equivalent |
|----|-----------|-------------------|----------|--------------------|
| AXI4LITE_SLV_AR_ARVALID_STABLE   | ARVALID rises HIGH                                | ARVALID must remain HIGH until ARREADY is observed HIGH on a rising ACLK edge. | FAIL | AXI4_ERRM_ARVALID_STABLE (unverified) |
| AXI4LITE_SLV_AR_ARADDR_STABLE    | ARVALID is HIGH                                   | ARADDR must not change between the cycle ARVALID rises and the cycle ARREADY is observed HIGH. | FAIL | AXI4_ERRM_ARADDR_STABLE (unverified) |
| AXI4LITE_SLV_AR_ARPROT_STABLE    | ARVALID is HIGH                                   | ARPROT must not change between the cycle ARVALID rises and the cycle ARREADY is observed HIGH. | FAIL | AXI4_ERRM_ARPROT_STABLE (unverified) |
| AXI4LITE_SLV_AR_ARPROT_IGNORED   | ARVALID + ARREADY handshake completes              | The slave BFM samples ARPROT but does not enforce protection-attribute checks. | RECOMMEND | (none) |
| AXI4LITE_SLV_AR_ARREADY_NO_LATCH | ARREADY is HIGH and ARVALID is LOW on the same cycle | ARREADY may be HIGH before ARVALID; this is legal per AXI4-Lite. The slave must not interpret a one-cycle ARREADY assertion as completing a transaction. | FAIL | AXI4_ERRS_ARREADY_NO_LATCH (unverified) |
| AXI4LITE_SLV_AR_ARREADY_MAX_WAIT | ARVALID has been HIGH for 1024 consecutive ACLK cycles without ARREADY observed HIGH | The slave BFM logs a back-pressure observation. Implementation choice, not a protocol violation. | RECOMMEND | AXI4_RECS_ARREADY_MAX_WAIT (unverified) |

## R rules

| ID | Condition | Required behavior | Severity | ARM SVA equivalent |
|----|-----------|-------------------|----------|--------------------|
| AXI4LITE_SLV_R_RVALID_STABLE | RVALID rises HIGH | RVALID must remain HIGH until RREADY is observed HIGH on a rising ACLK edge. | FAIL | AXI4_ERRS_RVALID_STABLE (unverified) |
| AXI4LITE_SLV_R_RDATA_STABLE  | RVALID is HIGH    | RDATA must not change between the cycle RVALID rises and the cycle RREADY is observed HIGH. | FAIL | AXI4_ERRS_RDATA_STABLE |
| AXI4LITE_SLV_R_RRESP_STABLE  | RVALID is HIGH    | RRESP must not change between the cycle RVALID rises and the cycle RREADY is observed HIGH. | FAIL | AXI4_ERRS_RRESP_STABLE (unverified) |
| AXI4LITE_SLV_R_RRESP_VALUES  | RVALID + RREADY handshake completes | RRESP must be one of: 2'b00 (OKAY), 2'b10 (SLVERR), 2'b11 (DECERR). 2'b01 (EXOKAY) is reserved in AXI-Lite and must not be used. | FAIL | AXI4_ERRS_RRESP_VALUES (unverified) |

## Cross-channel rules

| ID | Condition | Required behavior | Severity | ARM SVA equivalent |
|----|-----------|-------------------|----------|--------------------|
| AXI4LITE_SLV_XCH_W_AFTER_AW       | WREADY is asserted          | The corresponding AW handshake must have completed in an earlier or the same cycle. The slave does not accept W phase before AW phase for the same write. | FAIL | (none) |
| AXI4LITE_SLV_XCH_B_AFTER_AW_AND_W | BVALID is asserted          | Both the corresponding AW handshake AND the corresponding W handshake must have completed in an earlier or the same cycle. The slave must not assert BVALID for a write whose address or data has not yet arrived. | FAIL | (none) |
| AXI4LITE_SLV_XCH_R_AFTER_AR       | RVALID is asserted          | The corresponding AR handshake must have completed in an earlier or the same cycle. | FAIL | (none) |
| AXI4LITE_SLV_XCH_NO_OUTSTANDING_WRITE | A second AWVALID rises while the previous write's BVALID handshake has not yet completed | The slave back-pressures by holding AWREADY low for the second write until the first write's B phase completes. The BFM enforces single-outstanding-write semantics for AXI-Lite. | FAIL | (none) |
| AXI4LITE_SLV_XCH_NO_OUTSTANDING_READ  | A second ARVALID rises while the previous read's RVALID handshake has not yet completed | The slave back-pressures by holding ARREADY low for the second read until the first read's R phase completes. | FAIL | (none) |

## Configuration-knob rules

| ID | Condition | Required behavior | Severity | ARM SVA equivalent |
|----|-----------|-------------------|----------|--------------------|
| AXI4LITE_SLV_CFG_RESPONSE_DELAY_BOUND | Configuration `set_response_delay(min, max)` has been called with min=N, max=M (both in ACLK cycles, inclusive) | For every transaction, the BFM must hold off the response handshake (BVALID for writes, RVALID for reads) by a randomly chosen number of ACLK cycles in [N, M]. Bounds are inclusive. Default if uncalled: min=0, max=0 (no delay). | FAIL | (none) |
| AXI4LITE_SLV_CFG_FAULT_BRESP          | Configuration `set_response_fault(SLVERR)` or `set_response_fault(DECERR)` is active for the next write | The next BVALID handshake must carry BRESP = 2'b10 (SLVERR) or 2'b11 (DECERR) respectively. The fault is one-shot: the BFM clears the configured fault after the next write completes. | FAIL | (none) |
| AXI4LITE_SLV_CFG_FAULT_RRESP          | Configuration `set_response_fault(SLVERR)` or `set_response_fault(DECERR)` is active for the next read | The next RVALID handshake must carry RRESP = 2'b10 (SLVERR) or 2'b11 (DECERR) respectively. One-shot. | FAIL | (none) |
| AXI4LITE_SLV_CFG_REGISTER_PRELOAD     | Configuration `set_register_value(addr, value)` has been called | A subsequent read whose ARADDR equals addr must return RDATA = value, regardless of any prior write to that address. The configured value persists across multiple reads until either reset_state() is called or a subsequent write to addr overwrites it. | FAIL | (none) |
| AXI4LITE_SLV_CFG_MODE_SWITCH          | Configuration `bfm_mode` transitions from ACTIVE to PASSIVE | All BFM-driven outputs (AWREADY, WREADY, BVALID, BRESP, ARREADY, RVALID, RDATA, RRESP) must transition to their during-reset values within one ACLK cycle. Any in-flight transaction_api.md call must unblock with status MODE_SWITCHED_TO_PASSIVE. | FAIL | (none) |

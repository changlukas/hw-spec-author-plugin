# AXI-Lite Slave BFM (axi_lite_slave_bfm)

## Overview

This document specifies the AXI-Lite Slave Bus Functional Model (axi_lite_slave_bfm). The BFM responds to an AXI-Lite master DUT on the configuration bus, providing a software-controllable target with configurable response delay, fault injection, and a pre-loadable internal memory. The BFM supports both active mode (driving response wires) and passive mode (monitoring only). It is implemented in C++ with DPI-C bridges to SystemVerilog testbenches; the same spec also applies to SystemC, native-SV/UVM, and pure-VHDL implementations.

## Features

- **AXI-Lite (ARM IHI0022 §B) compliance** as a slave (subordinate) — five channels: AW, W, B, AR, R.
- **Configurable response delay** via `set_response_delay(min, max)` — bounds in ACLK cycles, randomised per transaction, inclusive.
- **One-shot response fault injection** via `set_response_fault(SLVERR | DECERR)` — inject the next BVALID's BRESP or RVALID's RRESP.
- **Pre-loadable internal address-indexed memory** via `set_register_value(addr, value)` — subsequent reads to addr return value.
- **Active and passive modes** switchable at runtime via `bfm_mode` knob — passive mode floats all driven outputs and observes only.
- **Configurable address and data widths** — `ADDR_WIDTH ∈ {1..32}`, `DATA_WIDTH ∈ {32, 64}` per AXI-Lite restriction.

## Description

The BFM behaves as a single-outstanding AXI-Lite slave: at most one write transaction (AW + W → B) and at most one read transaction (AR → R) may be in flight at any time, and the second of either is back-pressured at AWREADY / ARREADY until the first completes. AWPROT and ARPROT are sampled and recorded for monitor mode but do not influence response generation. Sub-word writes via WSTRB are honoured: only the strobed bytes of WDATA are written to the BFM's internal memory at the addressed location.

The BFM is implemented as three internal blocks: a **driver** (drives BFM-controlled outputs), a **monitor** (samples DUT-driven inputs and reconstructs transactions), and a **sequencer** (the testbench-API surface that translates `set_*`/`expect_*` calls into driver and monitor activity). In passive mode, the driver is disabled; monitor and sequencer continue.

## Compatibility

This BFM implements AXI4-Lite as defined in ARM AMBA AXI Specification IHI0022 §B. It does not implement the full AXI4 (no bursts, no ID channels, no exclusive access, no atomic operations, no QoS or REGION channels). It is suitable as a stand-in for any AXI-Lite-compliant slave; behavioral equivalence to a vendor-specific slave (e.g., Xilinx AXI BRAM Controller, ARM CoreLink slave components) requires the testbench to mirror that vendor's behaviour through `set_register_value` pre-loads and per-test response delay configuration.

## Further Reading

- [Theory of Operation](./doc/theory_of_operation.md)
- [Signal Interface](./doc/signal_interface.md)
- [Pin-Level Reset](./doc/pin_level_reset.md)
- [Protocol Rules](./doc/protocol_rules.md)
- [Channel Handshake & Dependencies](./doc/channel_handshake.md)
- [Transaction API](./doc/transaction_api.md)
- [Channel API](./doc/channel_api.md)
- [Active vs Passive Mode](./doc/active_passive_mode.md)
- [DV Plan](./dv/plan.md)
- [Reader Test Log](./READER_TEST_LOG.md)

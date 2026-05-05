# APB Slave BFM (apb_slave_bfm)

## Overview

This document specifies the APB Slave Bus Functional Model (apb_slave_bfm). The BFM responds to an APB master DUT (the "requester") on a peripheral bus, providing a software-controllable target with configurable PSLVERR injection, configurable wait-state insertion, and a pre-loadable internal memory. The BFM supports both active and passive modes. APB3 (ARM IHI 0024) is the canonical reference; the BFM also accepts APB4 superset signals (PPROT, PSTRB) when present.

## Features

- **APB3/APB4 (ARM IHI 0024) compliance** as a Completer (slave). Two-phase access: SETUP (one cycle) → ACCESS (one or more cycles, gated by PREADY).
- **Configurable wait-state count** via `set_wait_states(min, max)` — number of ACCESS-phase cycles before PREADY rises. Random per transaction in [min, max] inclusive.
- **One-shot PSLVERR injection** via `set_response_fault()` — assert PSLVERR on the next ACCESS-phase completion.
- **Pre-loadable internal address-indexed memory** via `set_register_value(addr, value)`.
- **Active and passive modes** via `bfm_mode` knob.
- **APB4 PSTRB support** for byte-granular writes (when PSTRB signal is present in the testbench wiring).
- **PPROT recorded but not enforced** (APB4 only).

## Description

APB is a strictly serial peripheral bus — no outstanding transactions, no channels in the AXI sense. A transaction is two phases on a shared signal set: SETUP (PSEL high, PENABLE low — master presents address and write data) and ACCESS (PSEL high, PENABLE high — slave responds with PREADY and either PSLVERR or successful PRDATA). The BFM enforces APB's strict alternation: SETUP must always be followed by exactly one ACCESS, no overlap, no concurrent transactions.

The BFM is implemented as three internal blocks: a **driver** (drives PREADY, PRDATA, PSLVERR), a **monitor** (samples PSEL/PENABLE/PADDR/PWDATA/PWRITE and reconstructs transactions), and a **sequencer** (the testbench-API surface).

## Compatibility

This BFM implements APB3 and the APB4 superset as defined in ARM AMBA APB Protocol Specification IHI 0024. APB4 features (PPROT, PSTRB) are recognised when present; PPROT is sampled-only (not enforced for access control), PSTRB drives byte-granular writes when the master DUT provides it.

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

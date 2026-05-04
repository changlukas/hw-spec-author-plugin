# Hardware Interfaces

Naming convention: `_i` = input, `_o` = output, `_n` = active-low. Internal-only signals (e.g., `*_q`, `*_d`) are not exposed at this interface.

## Parameters

| Name            | Type / Range  | Default | Description                                                |
|-----------------|---------------|---------|------------------------------------------------------------|
| ADDR_WIDTH      | int (4..12)   | 6       | APB address width. Must cover at least the 0x00..0x28 register range. |
| PRESCALE_W      | int (1..15)   | 15      | Internal prescaler counter width. The valid range of `CTRL.prescale_log2` is 0..PRESCALE_W; with the default, the maximum prescaler period is 2^15 (32768 cycles). |

## Clocks

| Signal | Direction | Description           |
|--------|-----------|-----------------------|
| clk_i  | input     | Primary functional clock. APB and counter share this clock. |

## Resets

| Signal  | Direction | Active | Sync | Description                              |
|---------|-----------|--------|------|------------------------------------------|
| rst_ni  | input     | low    | sync | Synchronous reset for clk_i domain. See Theory of Operation §Resets for the post-reset state. |

## Bus interface

### s_apb (configuration slave)

- Protocol: APB3 (AMBA APB v3.0 specification).
- Role: completer (slave).
- Address width: `ADDR_WIDTH` bits (default 6, covering offsets 0x00..0x3F).
- Data width: 32 bits.
- Sub-word access: not supported. Byte and half-word writes return `PSLVERR` and leave register state unchanged. Reads return the full 32-bit register; the requestor is responsible for byte selection.
- Wait-state insertion: none. Every transfer completes in 2 PCLK cycles (setup + access).
- `pprot` and `pstrb` are ignored.

| Signal       | Direction | Width        | Description                          |
|--------------|-----------|--------------|--------------------------------------|
| psel_i       | input     | 1            | APB select                           |
| penable_i    | input     | 1            | APB enable phase indicator           |
| pwrite_i     | input     | 1            | 1 = write, 0 = read                  |
| paddr_i      | input     | ADDR_WIDTH   | Register offset (byte address)       |
| pwdata_i     | input     | 32           | Write data                           |
| prdata_o     | output    | 32           | Read data, valid when penable & ~pwrite |
| pready_o     | output    | 1            | Always 1 — no wait states            |
| pslverr_o    | output    | 1            | 1 indicates protocol error (sub-word, unmapped offset) |

## Sideband signals

wctmr has no sideband signals beyond the bus, clocks, and resets.

## Interrupts

| Signal        | Type  | Description                                             |
|---------------|-------|---------------------------------------------------------|
| intr_cmp0_o   | level | Asserted while `INTR_STATE.cmp0_match & INTR_ENABLE.cmp0_match`. Cleared by writing 1 to `INTR_STATE.cmp0_match`. |
| intr_cmp1_o   | level | Asserted while `INTR_STATE.cmp1_match & INTR_ENABLE.cmp1_match`. Cleared by writing 1 to `INTR_STATE.cmp1_match`. |

Interrupt outputs deassert combinationally when their underlying state bit is cleared. Software must clear the state bit and not rely on `INTR_ENABLE` masking alone, since masking only suppresses the output but leaves the state pending.

## Alerts

wctmr has no alert outputs.

## Inter-IP signals

wctmr has no direct inter-IP connections. All software interaction is via the APB slave port; all hardware-event interaction is via `intr_cmp0_o` / `intr_cmp1_o` to the interrupt controller.

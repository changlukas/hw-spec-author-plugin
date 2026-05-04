# Registers

Reserved fields read as zero. Writes to reserved fields are ignored (do not cause `PSLVERR`). Software should write reserved fields as zero to remain forward-compatible with future revisions.

## Register map

| Offset | Name        | Access | Reset       | Description                          |
|--------|-------------|--------|-------------|--------------------------------------|
| 0x00   | CTRL        | RW     | 0x00000000  | Top-level control                    |
| 0x04   | STATUS      | RO     | 0x00000000  | Operational status                   |
| 0x08   | INTR_STATE  | RW1C   | 0x00000000  | Interrupt status (write 1 to clear)  |
| 0x0C   | INTR_ENABLE | RW     | 0x00000000  | Interrupt enable mask                |
| 0x10   | INTR_TEST   | WO     | —           | Interrupt force-set (DV / diagnostic) |
| 0x14   | CNT_LO      | RW     | 0x00000000  | Counter bits [31:0]                  |
| 0x18   | CNT_HI      | RW     | 0x00000000  | Counter bits [63:32]                 |
| 0x1C   | CMP0_LO     | RW     | 0x00000000  | Compare slot 0 threshold [31:0]      |
| 0x20   | CMP0_HI     | RW     | 0x00000000  | Compare slot 0 threshold [63:32]     |
| 0x24   | CMP1_LO     | RW     | 0x00000000  | Compare slot 1 threshold [31:0]      |
| 0x28   | CMP1_HI     | RW     | 0x00000000  | Compare slot 1 threshold [63:32]     |

Offsets 0x2C..0x3F are unmapped. APB access to these returns `PSLVERR`.

## Register details

### CTRL

- Offset: 0x00
- Width: 32
- Access: RW
- Reset: 0x00000000

| Bits   | Field         | Access | Reset | Description                                                   |
|--------|---------------|--------|-------|---------------------------------------------------------------|
| [0]    | enable        | RW     | 0x0   | Block-level enable. When 0, the counter is frozen. See Theory of Operation §Datapath. |
| [5:1]  | prescale_log2 | RW     | 0x0   | Prescaler divider = 2^prescale_log2. Valid range 0..PRESCALE_W (default 0..15 → 1..32768). Software writes outside the valid range are accepted in the register (read-back returns the written value), but the prescaler hardware clamps the effective period to 2^PRESCALE_W cycles. No `PSLVERR` is raised for out-of-range values; the register acts as a soft contract. See Theory of Operation §Datapath. |
| [16]   | cmp0_en       | RW     | 0x0   | Enable compare slot 0. When 0, `CMP0_LO` / `CMP0_HI` are still readable but cannot fire `intr_cmp0_o`. |
| [17]   | cmp1_en       | RW     | 0x0   | Enable compare slot 1. As above for slot 1.                   |
| [15:6], [31:18] | — | RO     | 0x0   | Reserved. Reads as 0; writes ignored.                          |

### STATUS

- Offset: 0x04
- Width: 32
- Access: RO
- Reset: 0x00000000

| Bits   | Field        | Access | Reset | Description                          |
|--------|--------------|--------|-------|--------------------------------------|
| [0]    | running      | RO     | 0x0   | Mirror of `CTRL.enable`.             |
| [1]    | cmp0_armed   | RO     | 0x0   | Mirror of `CTRL.cmp0_en`.            |
| [2]    | cmp1_armed   | RO     | 0x0   | Mirror of `CTRL.cmp1_en`.            |
| [31:3] | —            | RO     | 0x0   | Reserved.                            |

Writes to STATUS return `PSLVERR`.

### INTR_STATE

- Offset: 0x08
- Width: 32
- Access: RW1C (write-1-to-clear)
- Reset: 0x00000000

| Bits   | Field      | Access | Reset | Description                                                              |
|--------|------------|--------|-------|--------------------------------------------------------------------------|
| [0]    | cmp0_match | RW1C   | 0x0   | Set by hardware when compare slot 0 fires (counter ≥ CMP0 with cmp0_en=1). Software writes 1 to clear. See Theory of Operation §Error and fault handling for behavior when both slots fire same cycle. |
| [1]    | cmp1_match | RW1C   | 0x0   | Set by hardware when compare slot 1 fires. Software writes 1 to clear.   |
| [31:2] | —          | RO     | 0x0   | Reserved.                                                                |

If software writes 1 to clear a bit while the underlying compare condition is still continuously satisfied, the bit clears for one cycle and is re-set on the next. Software must change the compare value or disable the slot to permanently clear the interrupt.

### INTR_ENABLE

- Offset: 0x0C
- Width: 32
- Access: RW
- Reset: 0x00000000

| Bits   | Field      | Access | Reset | Description                                                          |
|--------|------------|--------|-------|----------------------------------------------------------------------|
| [0]    | cmp0_match | RW     | 0x0   | When 1, `INTR_STATE.cmp0_match` propagates to `intr_cmp0_o`.         |
| [1]    | cmp1_match | RW     | 0x0   | When 1, `INTR_STATE.cmp1_match` propagates to `intr_cmp1_o`.         |
| [31:2] | —          | RO     | 0x0   | Reserved.                                                            |

### INTR_TEST

- Offset: 0x10
- Width: 32
- Access: WO
- Reset: —

| Bits   | Field      | Access | Reset | Description                                                          |
|--------|------------|--------|-------|----------------------------------------------------------------------|
| [0]    | cmp0_match | WO     | —     | Writing 1 sets `INTR_STATE.cmp0_match`. For DV and self-test only. Reads return 0. |
| [1]    | cmp1_match | WO     | —     | Writing 1 sets `INTR_STATE.cmp1_match`.                              |
| [31:2] | —          | WO     | —     | Reserved.                                                            |

### CNT_LO

- Offset: 0x14
- Width: 32
- Access: RW
- Reset: 0x00000000

Low 32 bits of the 64-bit counter. Software write order matters: write `CNT_LO` first, then `CNT_HI`. The counter increment is suppressed for one cycle following each write to `CNT_LO`. See Theory of Operation §Software write atomicity.

| Bits    | Field   | Access | Reset      | Description                  |
|---------|---------|--------|------------|------------------------------|
| [31:0]  | cnt_lo  | RW     | 0x00000000 | Counter bits [31:0]          |

### CNT_HI

- Offset: 0x18
- Width: 32
- Access: RW
- Reset: 0x00000000

| Bits    | Field   | Access | Reset      | Description                  |
|---------|---------|--------|------------|------------------------------|
| [31:0]  | cnt_hi  | RW     | 0x00000000 | Counter bits [63:32]         |

### CMP0_LO

- Offset: 0x1C
- Width: 32
- Access: RW
- Reset: 0x00000000

| Bits    | Field    | Access | Reset      | Description                          |
|---------|----------|--------|------------|--------------------------------------|
| [31:0]  | cmp0_lo  | RW     | 0x00000000 | Compare slot 0 threshold [31:0]      |

### CMP0_HI

- Offset: 0x20
- Width: 32
- Access: RW
- Reset: 0x00000000

| Bits    | Field    | Access | Reset      | Description                          |
|---------|----------|--------|------------|--------------------------------------|
| [31:0]  | cmp0_hi  | RW     | 0x00000000 | Compare slot 0 threshold [63:32]     |

### CMP1_LO

- Offset: 0x24
- Width: 32
- Access: RW
- Reset: 0x00000000

| Bits    | Field    | Access | Reset      | Description                          |
|---------|----------|--------|------------|--------------------------------------|
| [31:0]  | cmp1_lo  | RW     | 0x00000000 | Compare slot 1 threshold [31:0]      |

### CMP1_HI

- Offset: 0x28
- Width: 32
- Access: RW
- Reset: 0x00000000

| Bits    | Field    | Access | Reset      | Description                          |
|---------|----------|--------|------------|--------------------------------------|
| [31:0]  | cmp1_hi  | RW     | 0x00000000 | Compare slot 1 threshold [63:32]     |

# Programmer's Guide

## Initialization

Verified initialization sequence after `rst_ni` deassertion:

```c
// 1. Confirm block is in reset-default state (counter == 0, all compares disabled)
assert(read(STATUS) == 0);

// 2. Clear any stale interrupt state from prior sessions (defensive)
write(INTR_STATE, 0x3);  // write 1 to both bits to clear

// 3. Optionally configure the prescaler. prescale_log2=0 → no division.
// Example: divide-by-256 for a slow tick.
uint32_t ctrl = (8 << 1);  // prescale_log2 = 8
write(CTRL, ctrl);

// 4. Enable the block. Counter begins incrementing on the next prescaler tick.
write(CTRL, ctrl | 0x1);  // OR in CTRL.enable

// 5. Block is now running.
```

Other orderings may work but only the above is verified by DV.

## Use case A: One-shot timer

**Goal:** Receive an interrupt N counter ticks from now.

**Precondition:** Block initialized as in §Initialization.

**Action:**

```c
// Read the current counter (atomic 64-bit read pattern: see Use case D)
uint64_t now = read_counter_atomic();

// Compute target threshold
uint64_t target = now + N;

// Program compare slot 0
write(CMP0_LO, (uint32_t)(target));
write(CMP0_HI, (uint32_t)(target >> 32));

// Enable the slot and arm the interrupt
uint32_t ctrl = read(CTRL);
ctrl |= (1 << 16);   // CTRL.cmp0_en
write(CTRL, ctrl);
write(INTR_ENABLE, 0x1);  // INTR_ENABLE.cmp0_match
```

**Observable result:** Approximately N counter ticks later (subject to prescaler), `intr_cmp0_o` rises and `INTR_STATE.cmp0_match` reads 1.

**On interrupt service:**

```c
// Disable the slot first to prevent re-firing
uint32_t ctrl = read(CTRL);
ctrl &= ~(1 << 16);
write(CTRL, ctrl);

// Then clear the interrupt state
write(INTR_STATE, 0x1);
```

If software clears `INTR_STATE.cmp0_match` first without disabling the slot, the bit re-sets on the next cycle because the underlying condition (`counter >= CMP0`) is still satisfied. See Theory of Operation §Error and fault handling.

## Use case B: Periodic timer

**Goal:** Receive an interrupt every N counter ticks.

**Precondition:** Block initialized.

**Action:** As Use case A for the first interrupt.

**On interrupt service:**

```c
// Read current counter, set next compare to current + N
uint64_t now = read_counter_atomic();
uint64_t next = now + N;
write(CMP0_LO, (uint32_t)(next));
write(CMP0_HI, (uint32_t)(next >> 32));

// Clear the interrupt state.
// CTRL.cmp0_en stays asserted; the new threshold is in the future,
// so the underlying condition is no longer satisfied.
write(INTR_STATE, 0x1);
```

The handler must run before the next compare value is reached, otherwise the interrupt is missed (no queueing). For high-rate periodic timers, choose N large enough to absorb worst-case interrupt latency.

## Use case C: Independent timers using both slots

The two compare slots are independent. A common pattern is one for an OS tick and the other for a watchdog or one-shot.

```c
write(CMP0_LO, tick_threshold_lo); write(CMP0_HI, tick_threshold_hi);
write(CMP1_LO, wdog_threshold_lo); write(CMP1_HI, wdog_threshold_hi);

uint32_t ctrl = read(CTRL);
ctrl |= (1 << 16) | (1 << 17);   // cmp0_en and cmp1_en
write(CTRL, ctrl);

write(INTR_ENABLE, 0x3);  // both interrupts enabled
```

If both compares fire on the same cycle, both `INTR_STATE` bits set in the same cycle. Software cannot determine the order beyond "both fired." Interrupt-priority resolution is the host's responsibility; wctmr emits both `intr_cmp0_o` and `intr_cmp1_o` simultaneously.

## Use case D: Coherent 64-bit counter read

A two-word read of `CNT_LO` then `CNT_HI` can be torn if the counter rolls over between the two reads (`CNT_LO` becomes small while `CNT_HI` increments). The standard idiom:

```c
uint32_t hi1, hi2, lo;
do {
    hi1 = read(CNT_HI);
    lo  = read(CNT_LO);
    hi2 = read(CNT_HI);
} while (hi1 != hi2);
uint64_t value = ((uint64_t)hi2 << 32) | lo;
```

The loop retries when a rollover happened mid-read. Worst-case loop count is bounded by 2 in normal operation.

Alternatively, software can disable the block (`CTRL.enable = 0`) for the duration of the read, but this introduces a measurement-induced gap that must then be compensated.

## Error handling

| Error                                  | Software detection                                          | Recovery                                                  |
|----------------------------------------|-------------------------------------------------------------|-----------------------------------------------------------|
| APB sub-word write attempted           | APB returns `PSLVERR`                                       | Re-issue as full 32-bit transaction.                      |
| APB write to unmapped offset           | APB returns `PSLVERR`                                       | Verify register map; correct offset and retry.            |
| Counter wrap-around                    | None — software must check by tracking deltas               | If timestamps are needed across wrap, software handles by extending to 128-bit in software. |
| Interrupt fails to deassert            | Software clears `INTR_STATE` but `intr_*_o` remains high    | Likely the underlying compare condition is still true; either advance the compare value or disable the slot. |

## Interrupt handling

- Both interrupts (`intr_cmp0_o`, `intr_cmp1_o`) are level type. They follow `INTR_STATE.cmp*_match & INTR_ENABLE.cmp*_match`.
- Servicing order when both are pending is unspecified by hardware; the host's interrupt controller decides.
- To clear an interrupt: write 1 to the corresponding `INTR_STATE` bit.
- Disabling `INTR_ENABLE` masks the output but does **not** clear `INTR_STATE`. Software must clear `INTR_STATE` separately.

## Register accesses during operation

| Register write          | Effect during normal operation                             |
|-------------------------|------------------------------------------------------------|
| `CTRL.enable` 1 → 0     | Counter freezes on the same cycle. In-flight prescaler tick is dropped. |
| `CTRL.enable` 0 → 1     | Counter resumes from its current value on the next prescaler tick. No tick is "lost" or replayed. |
| `CTRL.prescale_log2`    | Takes effect on the next prescaler tick boundary; prescaler is not reset by the change, so the first new-period tick may arrive earlier or later than expected. |
| `CTRL.cmp*_en` 0 → 1    | If the counter already exceeds the compare value, the interrupt fires within the next 2 cycles. |
| `CTRL.cmp*_en` 1 → 0    | The corresponding `INTR_STATE` bit is **not** cleared. Software must clear it explicitly. |
| Write to `CNT_LO`       | Counter increment suspended for 1 cycle. Write `CNT_HI` next cycle. |
| Write to `CNT_HI` only  | High 32 bits update; low 32 bits keep incrementing if enabled. |
| Write to `CMPx_LO`/`HI` while `CTRL.cmpx_en` = 1 | Comparison uses the new value from the cycle after the write. If the new value is already exceeded, the interrupt fires. |
| Write to `INTR_STATE`   | Bits with 1 in `pwdata` are cleared. Other bits unchanged. |

Status reads (`STATUS`, `INTR_STATE`) are racy with hardware updates: a read on the same cycle as a hardware update returns the pre-update value. The next read returns the post-update value. There is no torn-read hazard for single-register reads; see Use case D for the 64-bit counter.

## Reset handling from software

wctmr has no software-issued block reset. Equivalent operations:

- **Clear the counter:** write 0 to `CNT_LO`, then 0 to `CNT_HI`.
- **Disable everything:** write 0 to `CTRL`. This freezes the counter, disables both compare slots, and clears `INTR_ENABLE` is not — `INTR_ENABLE` stays at its previously-written value because writing 0 to `CTRL` does not affect `INTR_ENABLE`. Software clears `INTR_ENABLE` separately.
- **Clear pending interrupts:** write `0x3` to `INTR_STATE`.

To return to full reset state, assert `rst_ni` from the SoC reset controller.

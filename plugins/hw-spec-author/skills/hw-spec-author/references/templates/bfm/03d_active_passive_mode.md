# Template: active_passive_mode.md

**Primary audience:** Testbench integrator, DV lead deciding which BFMs to spawn for each test.
**Goal:** The same BFM source supports two roles — active (driving stimulus into the DUT) and passive (silently observing the DUT's wires for protocol compliance and coverage). Both roles, the capability difference, and the configuration knob that switches between them are explicit.

**Protocol applicability:** Protocol-agnostic and language-agnostic (UVM, SystemC, OSVVM, C++/DPI-C, pure VHDL). Capability rows and mode-switch semantics are the same shape for any BFM. Mini-example uses AXI-Lite.

This file is short — typically one capability table, one config knob description, one paragraph on how to switch — but it is mandatory because passive mode is the foundation of every "monitor-only" testbench setup, and conflating active and passive in the documentation guarantees integration bugs.

Modeled on the UVM agent architecture: an active agent contains driver + monitor + sequencer; a passive agent contains monitor only. The UVM concept generalises to any BFM regardless of language — every BFM should declare its active/passive split.

## Required structure

```
# Active vs Passive Mode

## Capability table

| Capability | Active | Passive |
|------------|--------|---------|
| Drives outputs to the DUT | yes | no |
| Samples inputs from the DUT | yes | yes |
| Reconstructs transactions from observed pin activity | <yes / no — typically yes for both, since both monitor inbound activity> | yes |
| Generates stimulus from a sequencer or testbench API | yes | no |
| Reports protocol-rule violations (per protocol_rules.md) | yes | yes |
| Provides coverage hooks (per dv/plan.md) | yes | yes |
| Honors configuration knobs (response delay, fault injection) | yes | <yes / no — typically no, since the BFM is not driving anything> |

## Mode switch

State the configuration knob name, type, default, and the cycle-level effect of changing it:

- **Knob name**: <e.g. `bfm_mode`>
- **Type / values**: <e.g. enum {ACTIVE, PASSIVE}>
- **Default**: <e.g. ACTIVE>
- **Effect of switching from active to passive**: <state explicitly. Typical: "All BFM-driven outputs immediately tristate (or float to their during-reset values, see pin_level_reset.md). In-flight transactions held by transaction_api.md calls are unblocked with status `MODE_SWITCHED_TO_PASSIVE`. The BFM continues to monitor and report violations, but no longer drives.">
- **Effect of switching from passive to active**: <typical: "BFM-driven outputs return to their reset-deassertion values. The first transaction_api.md call after the switch begins normally.">
- **Switching mid-transaction**: <typical: "Permitted, but produces a logged warning. In-flight transactions are unblocked as described above; the test author should usually call reset_state before switching back.">

## Common testbench setups

Brief examples of where each mode applies:

- **Single-master testbench**: one active master BFM drives the DUT; no passive BFM needed.
- **Multi-master testbench**: one active master BFM per DUT input port; passive monitor BFMs may also be attached for cross-channel checking.
- **DUT integration regression**: passive BFMs only; the DUT's existing internal masters generate stimulus; passive BFMs catch protocol violations and contribute to coverage closure.
- **Mixed**: active BFM on one port + passive BFM on the same port (forbidden — only one BFM may drive a port). Or active on one port + passive on a different port (legal and common).

## Reset interaction

State whether the active/passive mode survives ARESETn:
- <e.g. "Mode is preserved across reset. ARESETn asserts → BFM-driven outputs follow pin_level_reset.md regardless of mode → on deassertion, active mode resumes driving, passive mode resumes observing.">
```

## Writing rules for this file

1. **One capability table, one mode-switch paragraph, one common-setups paragraph.** Resist the urge to over-explain; this file is meant to be scannable in one minute.
2. **The capability table is the authoritative summary.** If a capability is "yes" in active and "no" in passive (or vice versa), the table row says so explicitly. No "varies" or "see notes."
3. **Switch effect is wire-level explicit.** "All BFM-driven outputs immediately tristate" is testable; "the BFM stops driving" is interpretable.
4. **Switch effect on in-flight transactions is explicit.** Test authors will switch mid-test; they need to know what happens to a `apply_write` that is currently blocked on BVALID.
5. **The forbidden setup ("two BFMs driving the same port") is stated explicitly.** Without it, integrators will try.

## Anti-patterns

- **Anti-pattern:** Treating active vs passive as a runtime detail buried in the API documentation. It is a top-level architectural distinction; it deserves its own file.
- **Anti-pattern:** Capability table with merged cells or "yes/no/it depends." Pick yes or no for each capability per mode; if it depends, decompose the capability into sub-capabilities until each row is a clear yes/no.
- **Anti-pattern:** Mode switch with no documented effect on in-flight transactions. Test authors *will* switch mid-transaction (intentionally or by mistake); the silent behavior is exactly the corner case that breaks tests in the field.
- **Anti-pattern:** Two configuration knobs ("driving_enabled" and "monitoring_enabled") for what is really one binary mode. Use one knob with two values; document the implications.
- **Anti-pattern:** Documenting "active" without documenting "passive" (or vice versa). Both modes exist; both deserve a row in every table.

## Mini-example: AXI-Lite slave BFM

```
# Active vs Passive Mode

## Capability table

| Capability                                                     | Active | Passive |
|----------------------------------------------------------------|--------|---------|
| Drives outputs to the DUT (AWREADY, WREADY, BVALID, BRESP, ARREADY, RVALID, RDATA, RRESP) | yes | no |
| Samples inputs from the DUT (AWVALID, AWADDR, AWPROT, WVALID, WDATA, WSTRB, BREADY, ARVALID, ARADDR, ARPROT, RREADY) | yes | yes |
| Reconstructs transactions from observed pin activity (write addr/data pairs, read addr/return-data pairs) | yes | yes |
| Generates stimulus via transaction_api.md (set_register_value pre-load, set_response_delay, set_response_fault) | yes | no |
| Reports protocol-rule violations per protocol_rules.md          | yes | yes |
| Contributes to coverage hooks per dv/plan.md                    | yes | yes |
| Honors configuration knobs (response delay, fault injection)    | yes | no — knobs accepted but have no effect since BFM is not driving |

## Mode switch

- **Knob name**: `bfm_mode`
- **Type / values**: enum `{ACTIVE, PASSIVE}`
- **Default**: `ACTIVE`
- **Effect of switching from ACTIVE to PASSIVE**: All BFM-driven outputs (AWREADY, WREADY, BVALID, BRESP, ARREADY, RVALID, RDATA, RRESP) immediately float to their during-reset values per `pin_level_reset.md` (i.e., AWREADY/WREADY/BVALID/ARREADY/RVALID = 0; BRESP/RRESP = 0x0; RDATA = 0x0). In-flight transaction_api.md calls (e.g., a blocked `expect_write`) are unblocked with status `MODE_SWITCHED_TO_PASSIVE`; the test author may either resume monitoring via continued expect_* calls or terminate the test phase. The BFM continues to monitor inbound transactions and continues to log protocol-rule violations — only the driving stops.
- **Effect of switching from PASSIVE to ACTIVE**: BFM-driven outputs return to their reset-deassertion values per `pin_level_reset.md` (AWREADY/WREADY/BVALID/ARREADY/RVALID = 0 until the BFM accepts the next transaction). Configuration knobs that were modified during passive mode become effective on the first transaction after the switch.
- **Switching mid-transaction**: Permitted, but the BFM logs a warning. The test author should usually call `reset_state()` (per transaction_api.md) before switching back to active to drop any partial-transaction trackers and start clean.

## Common testbench setups

- **Single-master testbench**: one active AXI-Lite slave BFM at address 0x0000_0000–0x0000_FFFF; the master DUT drives reads and writes; the slave BFM responds and observes.
- **Multi-master testbench**: one active slave BFM per address segment; passive slave BFMs may be attached at the same address segments to cross-check that two parallel masters do not both observe the same write (illegal in the test scenario being checked).
- **DUT integration regression**: passive slave BFM attached to an existing slave port that the DUT-internal master already drives; the passive BFM contributes to coverage and catches protocol violations without changing the DUT's behavior.
- **Forbidden**: two BFMs driving the same port. The protocol does not allow multiple drivers; even with one in active mode and one in passive, the active one is unique and the passive one must be explicitly declared as monitor-only.

## Reset interaction

Mode is preserved across ARESETn. Reset asserts → all BFM-driven outputs follow pin_level_reset.md (regardless of mode) → on deassertion, the BFM returns to whatever mode was last set; default is ACTIVE on first reset deassertion after instantiation.
```

That's the entire active/passive contract — one capability table with seven rows, one mode-switch paragraph, four common-setup descriptions, one reset note.

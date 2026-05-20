Review result: mostly OK, but I see two spec-level sharp edges.

- **LINT-010 is precise enough.** It correctly scopes extraction to `## Testpoints` definition rows with `^\| *TP-?\d+ *\|`, so it handles both `TP-01` in `wctmr` and `TP1` in `axi_lite_slave_bfm`. It also avoids the `wctmr` stages row `All TP-01..TP-15 sanity-passing`, which would otherwise be a false duplicate source.

- **`spec-stats` testpoint handling is inconsistent.** The main text says use `TP-?\d+`, which handles both conventions. But the derivation trailer still says `grep -oE 'TP[0-9]+'`, which misses `TP-01`. Fix that trailer. Also, stats says “extract every testpoint ID token”; for lower false-positive risk, it should mirror LINT-010 and count definition rows only, or prose/range mentions can inflate future specs.

- **LINT-013 is directionally good but needs one explicit parser rule.** `wctmr/doc/registers.md` has reserved rows like `| [15:6], [31:18] | — | ... |`. The lint spec should explicitly accept comma-separated multiple ranges in one Bits cell and treat reserved/`—` fields as valid coverage. Otherwise a naive parser may report false INFO holes, or worse mishandle overlap. No actual overlap/overflow exists in `wctmr`; `axi_lite_slave_bfm` has no `doc/registers.md`, so LINT-013 must be skipped/N/A there.

- **LINT-BFM-001 looks safe for the AXI-Lite example.** `signal_interface.md` has explicit channel grouping, and `pin_level_reset.md` lists each wire in both reset tables. No grouped rows or channel-less interface edge is exercised by this example, but the wording is adequate if the implementation splits grouped signal cells on `/` and probably comma too.

- **LINT-002 loosenings should not trip either example.** I saw no TODO/TBD pattern issue in the read files.

Net: examples should lint cleanly for these changed rules after fixing `spec-stats` derivation pattern and making LINT-013’s multi-range reserved-row handling explicit.

---
description: Report aggregate counts for a hardware spec (mode-aware) — rule count with FAIL/RECOMMEND split, distinct testpoint count and max-ID, ABV property count, register count, and parameter defaults. Read-only.
argument-hint: "[path]"
allowed-tools: Read, Glob, Grep
---

You are activating the spec-stats report of the **hw-spec-author** skill. This command surfaces the canonical counts authors otherwise extract by hand for slide decks, READMEs, status reports, and `NEXT_SESSION_*.md` handoffs. It is observational and read-only.

Every number you report MUST come from a grep/count over the actual spec file, never from memory. If a count is unavailable (file missing), report it as N/A, not zero.

## Steps

1. **Locate the spec**. Parse `/spec-stats [path]`:
   - If a path is given (in `$ARGUMENTS`), use it as the spec root.
   - If no path, look for a spec root in CWD: a directory containing `README.md` plus `doc/`, or `./spec/`. If multiple candidates, ask which one.
   - If nothing found, tell the user and suggest `/spec-init <ip_name>`.

2. **Detect mode**. Read `<spec_root>/MODE.md` and parse the `mode:` line. If absent, default to `behavioral-block`. The mode selects which counts apply.

3. **Compute the counts** by grepping the actual files. Use these canonical patterns. Report N/A for any file that does not exist.

   **Rule count** (protocol-bfm only — from `doc/protocol_rules.md`):
   - A rule is a table row whose first cell is an UPPER_SNAKE rule ID. Match rows like `| <PROTO>_<...>_<NAME> | ... | <FAIL|RECOMMEND> | ... |`.
   - Count total rule rows. Split by the severity cell into FAIL and RECOMMEND.
   - Report `FAIL=<n>, RECOMMEND=<m>, total=<n+m>`. Do not count the header/separator rows or prose mentions of "FAIL".

   **Testpoint count** (both modes — from `dv/plan.md`):
   - Extract testpoint IDs from **testpoint-definition rows only** (`^\| *TP-?\d+ *\|` in the `## Testpoints` table), matching **both** conventions with `TP-?\d+`: protocol-bfm specs write `TP1`, behavioral-block specs write `TP-01`. Two cautions: the no-hyphen pattern alone reports 0 for a behavioral-block spec; counting every occurrence (not just definition rows) inflates the count via prose range mentions like `TP-01..TP-15`. Mirror LINT-010's scoping.
   - Compute the **distinct** count and the **max ID** separately.
   - If `distinct != max-ID`, report a gap warning: `distinct=K, max-ID=K' — gap (missing IDs: <list>)`. The two are different invariants (see LINT-010). A non-contiguous set is not an error here — just surface both numbers so a "we have N testpoints" claim is not silently a max-ID claim.

   **ABV property count** (protocol-bfm — from `dv/plan.md`):
   - Count `assert property` + `cover property` occurrences, or the explicit ABV-count line if the plan states one. If the plan only states a prose count, report that and note it is author-stated, not grepped.

   **Register count** (either mode — from `doc/registers.md` if present):
   - Count register-map table rows whose first cell is an offset (`0x...`). Report the count. Register count is N/A when `registers.md` is absent (a BFM with no CSR).

   **Parameter defaults** (protocol-bfm — from `doc/signal_interface.md` §Parameters):
   - Extract the parameter table; report a one-line summary of each parameter and its default value. Behavioral-block specs that declare parameters in `interfaces.md` use that file instead.

4. **Output the report** in this structure (omit lines that are N/A for the mode):

```
Spec: <path>
Mode: <behavioral-block | protocol-bfm>

Rule count        : FAIL=<n>, RECOMMEND=<m>, total=<n+m>      (protocol_rules.md)
Testpoint count   : distinct=<K>, max-ID=<K'>  [gap: missing <ids>]   (dv/plan.md)
ABV property count: <n>                                        (dv/plan.md)
Register count    : <n>                                        (registers.md)
Parameter defaults:
  <PARAM> = <default>
  ...
```

5. **Cite the command for each count** in a short trailer so the number is reproducible, e.g.:

```
Derivation:
  rule count    : grep '^| <ID-pattern>' doc/protocol_rules.md | count, split on severity cell
  testpoint     : grep -oE '^\| *TP-?[0-9]+' dv/plan.md | grep -oE 'TP-?[0-9]+' | sort -u   (definition rows; handles TP1 and TP-01)
  register count: grep -E '^\| *0x' doc/registers.md | count
```

## Constraints

- Never report a count from memory. Grep the file, then report. If the grep and your expectation disagree, trust the grep.
- The testpoint **distinct count** and **max-ID** are reported separately, always. Conflating them is the documented A6-wave error (a deck claimed "51 testpoints" when the real count was 50, max-ID 51).
- Do not flag a testpoint numbering gap as an error. `/spec-stats` reports it; `/spec-lint` LINT-010 decides severity (gaps are INFO, duplicates FAIL).
- This command does not modify the spec. For count auto-fill into `dv/plan.md`, that is `/spec-status --update-counts` (separate, BFM-only).
- `/spec-status --stats` is an equivalent alias for this command.

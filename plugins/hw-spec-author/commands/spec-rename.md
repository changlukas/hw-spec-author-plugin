---
description: Rename a term across a hardware spec, interactively. Case-sensitive by default. Tags each hit with its structural context (register / signal / heading / body) so register-name overlaps are flagged, not clobbered. Supports --dry-run.
argument-hint: "<old> <new> [--dry-run] [-i]"
allowed-tools: Read, Grep, Glob, Edit
---

You are running the **rename utility** of the hw-spec-author skill. Renaming a term across a spec (an internal coinage, a register, a signal) is error-prone with bulk `sed`: hits need per-instance judgement, and a careless replace clobbers a register-table row or a provenance note. This command lists every hit with context, tags what kind of hit it is, and lets the author decide per hit.

## Steps

1. **Locate the spec**. Same logic as `/spec-status`. If nothing found, stop. Parse `$ARGUMENTS` into `<old>`, `<new>`, and flags `--dry-run`, `-i`.

2. **Find every hit**. grep `<old>` across all spec files. **Case-sensitive by default** — hardware identifiers are case-significant (`awready` ≠ `AWREADY` ≠ the prose word "ready"). Only widen to case-insensitive when `-i` is passed.

3. **Tag each hit with structural context** so the author is warned, not left to eyeball line numbers:
   - `REGISTER NAME` — hit is in the first column of a `registers.md` register-map or field table → flag "review: renaming a register touches every cross-reference".
   - `SIGNAL NAME` — hit is in a `signal_interface.md` / `interfaces.md` signal table.
   - `RULE ID` — hit is a rule ID in `protocol_rules.md`.
   - `HEADING` — hit is in a Markdown heading (cross-references may point at it).
   - `BODY` — ordinary prose.

4. **`--dry-run`**: print the full tagged hit list and stop. No edits.

5. **Interactive apply** (no `--dry-run`): for each hit, show `file:line` with before/after context and its tag, then offer:
   - `[a]pply` — replace this occurrence
   - `[s]kip` — leave unchanged
   - `[q]uit` — stop
   Do **not** auto-classify which hits to apply or skip. The author decides per hit. A register/signal/heading/rule-ID hit deserves extra scrutiny because renaming it implies updating its cross-references too — surface that, but do not decide it.

6. **Audit trail**: at the end, print a summary of the applied/skipped decisions (count + each hit's tag) to the user. The user records it where appropriate — typically the commit message body, since the rename is already the diff. The command itself does not commit and does not create a separate log file; it edits spec files in place via `Edit` (consistent with its `allowed-tools`).

7. **After applying**: suggest running `/spec-lint` (broken cross-references) and, if a register/signal/rule-ID was renamed, `/spec-impact "<new>"` to confirm no stale reference to `<old>` remains.

## Constraints

- **Case-sensitive default.** Default-insensitive would clobber every `*ready` signal and the English word "already" on a rename of "ready". `-i` is opt-in and deliberate.
- **No auto-classification.** The plugin tags context; the author judges. Pre-deciding which hits to apply is the failure this command exists to prevent.
- **One term per invocation.** Renaming overlapping terms in one pass risks order-dependent clobbering. Run sequentially.
- **Not a vocabulary linter.** This is the active-cleanup tool. There is no built-in list of "terms to rename" — the user supplies `<old>` and `<new>` per project. (Project-specific vocabulary is a user-AI conversation concern, not a shipped plugin default.)

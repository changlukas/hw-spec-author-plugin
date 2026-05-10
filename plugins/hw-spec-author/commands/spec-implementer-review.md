---
description: Run a multi-agent implementer review of a hardware spec to surface bit-equivalence ambiguities. Each paradigm-paired reviewer (default 2; e.g., c-bfm + rtl) reads the spec independently then peer-reviews the other(s). Synthesizes a converged ambiguity list. Cost ~2N agent dispatches, ~5–10 min wall. v1 supports `protocol-bfm + has-rtl-counterpart=yes` only.
argument-hint: "[path] [--paradigms <p1>,<p2>,...]"
allowed-tools: Read, Grep, Glob, Task, Edit, Write
---

You are activating the **implementer review** protocol of the hw-spec-author skill. This command's value depends on **paradigm isolation**: each paradigm-specific reviewer is a fresh subagent with no memory of how the spec was authored, the other reviewer's perspective, or the user's prior judgments. Do not answer prompts yourself.

## Steps

1. **Locate the spec**. Parse `/spec-implementer-review [path] [--paradigms ...]`:
   - If a path is in `$ARGUMENTS`, use it.
   - If no path, look for a spec root in CWD: a directory containing `MODE.md` plus a `doc/` subfolder, or `./spec/`. If multiple candidates, ask the user which one.
   - If nothing found, tell the user and stop.

2. **Detect mode and gate**. Read `<spec_root>/MODE.md`:
   - `mode: behavioral-block` → tell the user this command does not apply in v1 (reader test via `/spec-review` is sufficient for behavioral-block — no paradigm pair to compare). Stop.
   - `mode: protocol-bfm` AND `has-rtl-counterpart: no` → tell the user this command requires `has-rtl-counterpart: yes` in v1 (single-paradigm variant deferred to v2). Stop.
   - `mode: protocol-bfm` AND `has-rtl-counterpart: yes` → proceed.
   - `MODE.md` absent → tell the user `/spec-init` should be run first. Stop.

3. **Read the process doc**. Open `${CLAUDE_PLUGIN_ROOT}/skills/hw-spec-author/references/process/implementer_review.md` for paradigm presets, Round 1/2 prompt templates, and output format.

4. **Determine paradigms**:
   - If user passed `--paradigms p1,p2[,p3...]` → use that list. Validate against paradigm presets in the process doc. If any paradigm name is unknown, ask the user inline for persona / reading-focus / concerns.
   - If user did NOT pass `--paradigms`, infer the BFM paradigm from `theory_of_operation.md` §BFM internal architecture opening sentences:
     - Mentions C++ / SystemC behavior model → default `c-bfm,rtl`.
     - Mentions UVM / SystemVerilog BFM → default `uvm-bfm,rtl`.
     - Mentions TLM-2.0 / loosely-timed model → default `systemc-tlm,rtl`.
     - None clearly inferrable → tell the user the BFM paradigm could not be inferred and ask them to pass `--paradigms` explicitly. Stop.

5. **Surface plan to user**. Briefly tell the user:
   - Paradigm pair selected: `<p1>` + `<p2>`.
   - Estimated cost: 2N agent dispatches (N Round 1 in parallel + N Round 2 in parallel), so 4 total at the N=2 default. Wall ~5–10 min.
   - Output: `<spec_root>/IMPLEMENTER_REVIEW_LOG.md` (overwritten if exists).
   - Ask whether to proceed.

6. **Round 1 — independent reads**. For each paradigm, render the Round 1 prompt template (substitute `{{PERSONA}}`, `{{SPEC_PATH}}`, `{{MODE_INFO}}`, `{{READING_LIST}}`, `{{CONCERNS}}`, `{{OTHER_PARADIGM_NAMES}}`, `{{PARADIGM_NAME}}`). Use the Task tool to spawn `implementer-reviewer` subagent per paradigm, in parallel.

   Critical: do not answer prompts yourself. Even if you have spec context loaded from prior commands, the value of this protocol depends on subagent isolation.

7. **Wait for all Round 1 outputs**. Collect them as `R1_<paradigm>` strings.

8. **Round 2 — cross-paradigm peer review**. For each paradigm, render the Round 2 prompt template (substitute placeholders, and insert ALL OTHER paradigms' Round 1 outputs concatenated as `{{OTHER_ROUND1_OUTPUTS}}`). Spawn `implementer-reviewer` subagent per paradigm, in parallel.

9. **Wait for all Round 2 outputs**. Collect as `R2_<paradigm>` strings.

10. **Synthesize**. In the main context (no agent), aggregate Round 1 + Round 2 outputs. Build:
    - **Converged ambiguity list** ranked by cross-paradigm appearance count and bit-level impact.
    - **NEW DISCOVERIES** — ambiguities surfaced only by Round 2 cross-review.
    - **DISAGREEMENTS** — places where Round 2 reviewers proposed different resolutions for the same ambiguity. Flag for designer ruling.
    - **SPEC CONTRADICTIONS** — places where two spec files state different things.

11. **Write `<spec_root>/IMPLEMENTER_REVIEW_LOG.md`**. Use the format in `implementer_review.md` §IMPLEMENTER_REVIEW_LOG.md output format. Overwrite if exists. Previous run lives in git history.

12. **Surface verdict to user**. Brief summary:
    - Number of converged ambiguities by category (NEW / DISAGREEMENT / CONTRADICTION / single-side / cross-side).
    - Top 3 by impact, with proposed resolution.
    - Recommended next actions: (a) accept and resolve in spec, (b) defer to `WAIVERS.md`, (c) escalate to designer ruling.

## Constraints

- **Subagent isolation is the test.** If the `implementer-reviewer` subagent is unavailable (Task tool absent), tell the user this command cannot run reliably and stop. Do not silently fall back to main-context inference — that defeats paradigm isolation.
- **N=2 is the v1 default.** When N≥3, warn the user about cost (2N agent dispatches) before proceeding.
- **Round 1 failure handling (v1)**: if any Round 1 agent fails to return a usable plan, abort the whole run and surface the failure to the user. Do not proceed with partial Round 2. v2 may add resilience modes.
- **No silent overwrite of `WAIVERS.md`.** If ambiguities flow to `WAIVERS.md`, the user must approve each entry.
- This command is for cross-paradigm bit-equivalence ambiguity detection — not completeness checking (use `/spec-status`) or stage transitions (use `/spec-gate`).
- Reader test (`/spec-review`) and implementer review are complementary. Both should run before D1 sign-off in `protocol-bfm + has-rtl-counterpart=yes` mode.

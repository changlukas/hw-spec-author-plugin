# Plugin Improvement Plan — Post v0.4.0

**Status**: revised after two-reviewer cross-AI review (Codex GPT-5.5 + fresh-context Claude). See `plan/REVIEW_AGGREGATE.md` for the review synthesis. This revision applies every §5 recommendation from that aggregate.
**Predecessor**: `plan/BFM_MODE_DESIGN.md` (drove v0.3.0 protocol-bfm mode and v0.4.0 implementer-review).
**Backflow source**: `plan/DOGFOOD_OBSERVATIONS_A1-A6.md` plus dogfood log re-review.

**Revision log**:
- v1 (initial): 12 items A–L, P0 = A+B+C+D.
- v2 (this, post cross-review): D demoted out of P0 (verified false-positive on `examples/wctmr/doc/registers.md:104`). B revised to uniqueness-not-contiguity (B′). Three items added: M (register bit-overlap, promoted to P0), N (cross-file scalar consistency), O (stale WAIVERS anchor). K gated on a measured false-positive rate. Nine open questions resolved. §2 rationale tightened.

---

## 1. Filter principle: generality first

This plan applies a strict generality filter.

Every retained item passes five guards:

1. Applies to any IP class (GPIO controller, DMA, PCIe controller, NoC, Crypto, DSP). No assumption about block category.
2. Applies to any vendor reference (ARM AMBA, RISC-V, OpenTitan, AMD, custom). No vendor-specific word list shipped in the plugin.
3. Project-specific configuration is opt-in only. Plugin ships zero project-specific defaults.
4. Worked-example regression: ships clean on both `examples/wctmr/` (behavioral-block) and `examples/axi_lite_slave_bfm/` (protocol-bfm). **This guard is run, not asserted** — the cross-review caught two items (D, K) whose "examples pass" claim was false on inspection.
5. Project-specific cleanup needs are addressed via two generic conversation surfaces (`/spec-rename`, `/spec-impact`) rather than project-specific lint rules.

Principle in one line: **plugin ships the mechanism, user-AI conversation supplies the project content**.

---

## 2. Discarded candidates

Three items from `DOGFOOD_OBSERVATIONS_A1-A6.md` fail the filter and are discarded.

| Candidate | Discard reason |
|---|---|
| LINT-008 vocab residue with built-in word list | Any word list is project-specific. `(B)-philosophy` or `MetaBuffer` mean nothing to a GPIO spec author. The active-cleanup need is served by `/spec-rename` (item F). The passive-recurrence half (a renamed term creeping back in a later wave) is salvageable as an **opt-in user-supplied** word list loaded only if present in the spec dir — worth a one-line follow-up note, not a rebuild. |
| Stage-gate ↔ protocol-rule explicit `rules:` mapping | **Primary reason: semantic granularity mismatch.** `DOGFOOD §3` showed gates are architectural-granularity and rules are line-item — a 1:1 mapping is semantically wrong, not merely risky. Secondary reason: editing the `stage_gates.md` anchor schema breaks user-side `WAIVERS.md` contracts. An advisory non-schema coverage check (every rule appears in some gate's prose, every gate-cited rule exists) needs no schema change but has weak ROI given the granularity mismatch. Parked. |
| Dogfood mirror sync helper | Plugin-author internal workflow only. Regular users have no mirror. A generic `/spec-sync` would be a thin `robocopy` wrapper with zero spec-domain value. |

---

## 3. Retained items — 15 total

Grouped by phase.

### Phase P0 — mechanical, generic, low-risk, runs clean on both examples

Revised bundle: **A + B′ + C + M**. (Original D removed; see P1. M added.)

---

#### A. `/spec-stats` — aggregate counts

**Problem**. Authors building slide decks, READMEs, status reports, or `NEXT_SESSION_*.md` handoffs grep the same counts every time. `NEXT_SESSION_A5.md:21` cites "rule count 138, testpoint count 51"; the 51 was wrong (real count 50, max-ID 51, gap at TP45). Wave A3 cited "~135 rules" — off by 6.

**Design**.

```
/spec-stats <spec-dir> [--mode]

Output (mode-aware):
  Rule count       : FAIL=N, RECOMMEND=M, total=N+M   (BFM mode; from protocol_rules.md)
  Testpoint count  : distinct=K, max-ID=K' (warn if delta != 0)
  ABV count        : ...  (from dv/plan.md cover-property table)
  Parameter defaults: <one-line summary of signal_interface.md §Parameters>
  Register count   : N   (behavioral-block mode; from registers.md)
  Per-file git age : <file> last changed <relative time>   (rides on M5 staleness signal)
```

`/spec-status --stats` is an equivalent alias.

**Generality**. Counts what exists in the spec. Mode dispatch reuses `MODE.md` infrastructure. For a behavioral-block IP with no protocol rules, the rule/ABV counts cleanly report N/A — the command degrades, not breaks.

**Files**.
- New: `commands/spec-stats.md`
- Update: `commands/spec-help.md` (workflow card)

**Worked-example acceptance**.
- `examples/wctmr/` reports register count and TP count.
- `examples/axi_lite_slave_bfm/` reports rule count, TP count, and ABV count.
- Numbers match canonical grep output.

**Effort**: ~3 hr.

---

#### B′. LINT-010 — testpoint ID uniqueness (revised: uniqueness, not contiguity)

**Problem**. Wave A6 of noc-sim-ni-bfm shipped TP1..TP51 with TP45 missing. LINT-003 verifies Feature ↔ TP mapping but not ID integrity.

**Design (revised after cross-review)**.

```
LINT-010 (testpoint ID uniqueness)
  Extract every TP\d+ from dv/plan.md
  FAIL: duplicate IDs
  INFO: numbering gaps (not FAIL)
  Report: distinct count and max-ID separately (different invariants)
```

**Why revised**. The original "contiguous 1..N" check baked in a *numbering policy*, not a *uniqueness check*. An IP that numbers testpoints by category (AES core: TP1xx datapath / TP2xx key-schedule / TP3xx side-channel) is well-organised yet would FAIL contiguity. Cross-review (`REVIEW_AGGREGATE §4`) flagged this as smuggling a convention under a mechanical-check label. The generic invariant is uniqueness; gaps are informational.

**Generality**. `TP\d+` numbering is the established plugin convention for both modes' DV plans. Uniqueness is policy-free.

**Files**. Update `commands/spec-lint.md`.

**Worked-example acceptance**. Both examples pass (no duplicate TP IDs).

**Effort**: ~1 hr. Shares grep logic with A.

---

#### C. LINT-002 / LINT-BFM-001 — loosen false-warns

**Problem**. Two current rules produce noise without underlying ambiguity.

`LINT_REPORT.md:14` flags 14 markers of form `TODO(designer): confirm — <rationale>` as non-canonical because they are not wrapped in `(...)`. The form is semantically equivalent.

`LINT_REPORT.md:124` flags `pin_level_reset.md` rows that collapse `axi_in_awlen / awsize / awburst / awcache / awprot / awqos / awuser` (same reset value, same channel) into one row. The grouped row is reader-clear.

**Design**.
- LINT-002 accepts both forms: `TODO(designer): X — Y` and `TODO(designer): X (no issue yet — Y)`.
- LINT-BFM-001 accepts a grouped row when the `(reset value, channel, direction)` tuple is identical for every wire listed. **The loosened rule must no-op gracefully on channel-less IPs** (eFuse, ADC have no channel axis) — key on whatever axes exist, do not require a `channel` column.

**Generality**. Rule definition tweaks. All users benefit.

**Files**. Update `commands/spec-lint.md`.

**Worked-example acceptance**. Both examples still pass `/spec-lint`. No new false-passes introduced.

**Effort**: ~1 hr.

---

#### M. LINT-013 — register field bit-overlap / width overflow (NEW, promoted to P0)

**Problem**. Register field tables drift. Two fields claim overlapping bit ranges, or fields sum past the declared register width. A 32-bit register with fields summing to 33 bits ships silently. Surfaced by cross-review (`REVIEW_CLAUDE §5 M1`) as the strongest missing mechanical check — same low-risk profile as the rest of P0, more universal than TP numbering.

**Design**.

```
LINT-013 (register bit-overlap)
  Parse registers.md field tables (Bits column)
  FAIL: two fields overlap within a register
  FAIL: a field range exceeds declared register width
  INFO: unused bit holes within a register
```

**Generality**. Every register-bearing IP (GPIO, DMA, crypto, PHY — all behavioral-block specs and most BFMs) has bit-packed registers. Pure mechanical, zero domain assumption — the cleanest P0 profile on the table.

**Files**. Update `commands/spec-lint.md`.

**Worked-example acceptance**. Both examples' `registers.md` tables parse with no overlaps and no width overflow.

**Effort**: ~3 hr.

---

### Phase P1 — workflow tools and design-dependent lints

---

#### D. LINT-011 — attribution verbatim (DEMOTED from P0 to P1 after cross-review)

**Problem**. `DOGFOOD §4` records wave A4.6: prose "AMD: 每個 64-bit data granule 一個 8-bit ECC" attributed to AMD pg313. The pg313 verbatim is whole-flit SECDED. The paraphrase originated in the author's own draft and was mis-attributed across editing rounds.

**Why demoted**. Cross-review **verified** the proposed pattern `following <X> §Y` produces a false positive on `examples/wctmr/doc/registers.md:104` ("...one cycle following each write to `CNT_LO`. See Theory of Operation §Software write atomicity") — an internal cross-reference, not a vendor attribution. The P0 charter requires zero false positives against the worked examples. D fails it as specified, so it cannot ship in the P0 bundle.

**P1 design constraint**. The pattern must distinguish vendor attribution ("per AMD pg313 §...") from internal cross-reference ("See Theory of Operation §..."). Candidate: require a recognized external-source token (a vendor or standard name) adjacent to the `§`, not a bare "following ... §". Default severity WARN. Override via `(<vendor>-derived; no verbatim citation needed: <reason>)`.

**Files**. Update `commands/spec-lint.md`; optional `references/process/attribution_style.md`.

**Worked-example acceptance**. The reworked pattern must produce zero hits on `wctmr/registers.md:104` and on any internal `§` cross-reference in either example.

**Effort**: ~4 hr (regex design + example regression).

---

#### E. `/spec-impact <change-description>` — cross-file ripple analysis

**Problem**. `DOGFOOD §2`: wave A4.7 "drop VR mode" change touched 12+ files. Each wave required manual grep to discover stale references.

**Design**. Two phases, read-only:

```
/spec-impact <change-description>

Phase 1 — mechanical (no LLM)
  Extract keywords from <change-description>.
  grep across spec files → file:line:context list.
  Emit the extracted keyword list: "extracted keywords: [...]; if wrong, rephrase."
  (so a vague input does not masquerade as "no impact found")
  Report only.

Phase 2 — LLM analysis (opt-in via --analyse, per Q-E1)
  For each file matched, judge whether the section is materially affected.
  Output: list of (file, section, expected-action) tuples.
  Advisory only; user decides what to act on.
```

No file modification. No automatic propagation.

**Generality**. Grep and LLM analysis are domain-agnostic. Degenerate on conceptual (non-identifier) changes — the keyword-echo guard makes that visible rather than silent.

**Files**. New `commands/spec-impact.md`.

**Worked-example acceptance**. Demonstrate on a synthetic "rename register X to Y" change against both worked examples. Manual review confirms the impact list is correct.

**Effort**: ~4 hr.

---

#### F. `/spec-rename <old> <new> [--dry-run]`

**Problem**. `DOGFOOD §12`: wave A6 industry-vocab fix renamed `(B)-philosophy` → `log-and-forward fabric error reporting` across 5 files. Each hit needed per-instance judgement. Casing and hyphenation variants confound bulk `sed -i`.

**Design**. Interactive, no auto-classification, **case-sensitive by default** (per Q-F2):

```
/spec-rename <old> <new> [--dry-run] [-i]

For each hit:
  Display: file:line:before-context | match | after-context
  Tag the hit with its context (registers.md table row → flagged "REGISTER NAME — review";
    signal table row → "SIGNAL NAME — review"; heading; body; citation)
  Choices: [a]pply / [s]kip / [q]uit
  Apply:   in-place edit
  Skip:    no change; logged
  Dry-run: list all hits, no edits applied
  -i:      case-insensitive (opt-in; default is case-sensitive)
```

The plugin tags each hit with structural context (per `REVIEW_AGGREGATE §4` edge-case finding) so the user is warned when `<old>` overlaps a register or signal name, rather than left to eyeball line numbers. It does not auto-decide.

Audit log (per Q-F1): record rename decisions in the **commit message body** when in a git repo; fall back to `RENAME_LOG.md` only when git is absent.

**Generality**. Pure string match plus interactive choice. Zero domain knowledge.

**Files**. New `commands/spec-rename.md`.

**Worked-example acceptance**. Dry-run on both worked examples for a synthetic rename produces the expected tagged hit list.

**Effort**: ~4 hr.

---

#### G. `/spec-handoff <wave-name>` — wave handoff template

**Problem**. `DOGFOOD §13`: four `NEXT_SESSION_*.md` files in the noc-sim-ni-bfm dogfood each used a different structure. Each author reinvents.

**Design**. **Four fixed sections** (per Q-G1; subsections may say "none"):

```
# NEXT_SESSION_<wave-name>.md

## 1. Status snapshot
   (auto-filled) /spec-stats output
   (auto-filled) git log -1 HEAD
   (manual)      mode, current D-stage

## 2. Deferred tasks
   For each: priority / blocker / first concrete step / acceptance / effort

## 3. Quick re-entry checklist
   Five-minute sanity checks for the next session's start

## 4. Reminders to future self
   Gotchas, deliberate deferrals, scope boundaries
```

Fixed structure kills the drift the item exists to fix. Sections are allowed to say "none."

**Generality**. Section structure is workflow-only. Content is user-filled.

**Files**.
- New `commands/spec-handoff.md`
- New `references/process/wave_handoff.md` (template body)
- Optional `commands/spec-help.md` workflow-card update.

**Worked-example acceptance**. Not applicable (handoff docs are user-spec artifacts).

**Effort**: ~2 hr.

---

#### H. `D1.delta_from_plan` — gate item for unrequested clauses

**Problem**. `DOGFOOD §1` — over-design tendency. Wave A4 VC_ARB_MODE: plan said one line "runtime arbiter selection". Spec shipped four unrequested mechanisms. CLAUDE.md "delta-from-plan rule" is prompt-level and does not survive long sessions without a gate.

**Design**. New gate anchor:

```
D1.delta_from_plan
  Applies when: always (both modes)
  Plan artifact (per Q-H1): requires a written PLAN.md or a /spec-init interview
    transcript. Never freeform — a freeform "no delta detected" is unfalsifiable.
    If no written plan exists, the gate item is N/A and SAYS SO LOUDLY (not a silent pass).
  Scope (per Q-H2): check only sections changed since the previous gate.
  Check: author enumerates every clause or mechanism added that was not in the plan.
         Each item: revert or get per-item user confirmation.
  Acceptance: written confirmation log in PLAN_DELTA.md, or explicit N/A declaration.
```

**Why the artifact requirement matters**. The greenfield-by-conversation case (no written plan) is exactly where the A4 over-design happened. Without a written artifact to diff against, the gate becomes a rubber-stamp. The N/A-said-loudly rule makes the gap visible.

**Generality**. Self-audit. No IP-class or vendor assumption.

**Files**.
- Update `references/process/stage_gates.md` (new anchor)
- Optional `commands/spec-gate.md` reminder.

**Worked-example acceptance**. Neither worked example has a written `PLAN.md` or interview transcript, so per Q-H1 both carry an explicit **N/A declaration** (stated loudly), not a "no delta detected" annotation. A "no delta detected" annotation is valid only when a written plan artifact exists to diff against. Existing PASS state preserved.

**Effort**: ~2 hr.

---

#### N. LINT-015 — cross-file scalar consistency (NEW, P1)

**Problem**. The same signal or parameter described in 2+ files disagrees on width or value. `signal_interface.md` declares a port `[31:0]` while `theory_of_operation.md` describes it as 16-bit; one file says "bytes", another "beats". Item I (implementer review) catches *architectural* pairs but not *scalar-attribute* mismatches. Surfaced by **both** reviewers (`REVIEW_CODEX §5 M2` + `REVIEW_CLAUDE §5 M3`).

**Design**. Extract `signal[high:low]` and parameter values per file; diff the same identifier across files; FAIL on width or value disagreement. Extend LINT-001 or add LINT-015.

**Generality**. Multi-file specs in both modes carry the same identifier in 2+ files. Pure mechanical.

**Files**. Update `commands/spec-lint.md`.

**Worked-example acceptance**. Both examples pass (no cross-file width/value disagreement).

**Effort**: ~4 hr.

---

#### O. LINT-016 — stale WAIVERS anchor (NEW, P1)

**Problem**. `WAIVERS.md` cites a stage-gate anchor that has since been retired or changed. The waiver silently suppresses a check that no longer maps to it. Surfaced by cross-review (`REVIEW_CODEX §5 M1`).

**Design**. Lint `WAIVERS.md` anchors against current `stage_gates.md`; WARN on any waiver citing an anchor absent from the current gate set.

**Generality**. Any spec with waivers can drift. Directly protects the `stage_gates.md` ↔ `WAIVERS.md` contract that CLAUDE.md treats as load-bearing.

**Files**. Update `commands/spec-lint.md`.

**Worked-example acceptance**. Not applicable (worked examples carry no waivers); add a synthetic-waiver fixture test instead.

**Effort**: ~2 hr.

---

### Phase P2 — scoped but in-scope generic

---

#### I. Implementer-review prompt — cross-file architectural check

**Problem**. `IMPLEMENTER_REVIEW_LOG.md:276`: both Round 1 implementer-review agents read the VC partition table but did not flag its contradiction with `signal_interface.md`. The class of ambiguity requires architectural cross-checking across multiple spec files — neither reader-test nor single-paradigm review catches it.

**Design**. Round 1 prompt suffix:

```
After your independent read, list 3-5 pairs (file_A §X, file_B §Y) where
both files describe the same architectural object from different angles.
For each pair, state whether A and B agree mechanically.
```

**Scope**. Protocol-bfm with `has-rtl-counterpart: yes` only. The plan declares this scope explicitly — it is honest scoping, not a hidden generality failure.

**Files**.
- Update `references/process/implementer_review.md`
- Update `agents/implementer-reviewer.md`

**Worked-example acceptance**. Re-run implementer review on `examples/axi_lite_slave_bfm/`; expect at least two architectural pairs surfaced.

**Effort**: ~1 hr.

---

#### J. dv/plan rule count auto-fill

**Problem**. `DOGFOOD §7` — hand-maintained `cg_protocol_rule_hits` and ABV breakdown drift each wave.

**Design**.

```
Template dv_plan.md adds marker:
   <!-- RULE_COUNT_AUTO: regenerated by /spec-status --update-counts -->

/spec-status --update-counts:
   Greps protocol_rules.md FAIL/RECOMMEND counts; replaces the marker block.
   Without --update-counts: lint-warn on inconsistency only.
```

**Scope**. BFM-only. The plan declares this — behavioral-block specs have no `protocol_rules.md`, so the marker is never inserted (correctly N/A).

**Files**.
- Update `references/templates/bfm/dv_plan.md`
- Update `commands/spec-status.md`

**Worked-example acceptance**. `examples/axi_lite_slave_bfm/dv/plan.md` adds the marker; counts auto-fill correctly.

**Effort**: ~3 hr.

---

#### K. LINT-012 — AI-tell vocabulary (revised: WARN + measured precision gate)

**Problem**. `DOGFOOD §8`. CLAUDE.md "Spec writing style" lists banned words (leverage / ensure / robust / facilitate / scalable / comprehensive / utilises). Currently enforced via prompt only.

**Design (revised after cross-review)**.

```
LINT-012 (AI-tell vocabulary)
  Word list: shipped in references/process/ai_tell_vocabulary.md (industry-neutral)
  Action:    WARN per occurrence (never FAIL, per Q-K1)
  Override:  explicit (non-AI-tell context: <reason>) annotation
```

**Ship gate (added after cross-review)**. Before merging, run the word list against `examples/wctmr/`, `examples/axi_lite_slave_bfm/`, and CLAUDE.md's own banned list; record the false-positive rate. Cross-review **verified** the list fires on legitimate prose — "active BFM provides cycle-accurate response timing" (`axi_lite/active_passive_mode.md:31`); "well-understood ... provides adequate confidence" (`axi_lite/dv/plan.md:63`), plus several more `provides` hits. The original "both examples lint clean" claim was false. WARN severity plus a measured precision number are merge prerequisites.

**Generality**. Word list is plugin-maintained and project-neutral.

**Files**.
- New `references/process/ai_tell_vocabulary.md`
- Update `commands/spec-lint.md`

**Effort**: ~3 hr (plus precision measurement).

---

#### L. Y → X review tooling

**Problem**. `NEXT_SESSION_A4.md:159-169` — user's "先 Y 再 X" pattern (spawn an independent review agent before each commit) is orchestrated manually every wave.

**Design (per Q-L1, Q-L2)**. **Command first, report-only**:

- `/spec-review-pre-commit` wraps a fresh-context review subagent with a templated prompt. **Reports findings; never blocks the commit.**
- Git pre-commit hook example shipped later only if demanded.

**Generality**. Workflow-only.

**Files**.
- New `commands/spec-review-pre-commit.md`
- New `references/process/pre_commit_review.md`

**Effort**: ~3 hr.

---

## 4. Phased rollout

```
Stage 1 (P0 bundle):    A + B′ + C + M     → ~1.5 day, single PR
                        (D removed from P0; M added)
                        Acceptance: run all four against BOTH examples; zero false positives
                        — the bar is run, not asserted.

Stage 2 (P1 individual): D → E → F → G → H → N → O   → individual PRs
                        D leads: it needs the regex rework before it ships at all.
                        Each lands with worked-example regression evidence.

Stage 3 (P2 opportunistic): I → J → K → L
                        K gated on a measured false-positive rate before merge.
```

Version bumps:
- v0.5.0 at end of Stage 1.
- v0.6.0 at end of Stage 2.
- v0.7.0 candidate at end of Stage 3.

---

## 5. Cross-cutting design insights

### `/spec-rename` plus `/spec-impact` replace project-specific lint

The two utilities are the user-AI conversation surface for project-specific cleanup. Plugin authors are tempted to bake project-observed coinage into a built-in lint list. This plan rejects that. The two commands let the user discover and clean up coinage as it emerges.

### Mode-aware dispatch as established pattern

Every new command reads `MODE.md` first and dispatches accordingly. The same pattern is in `/spec-status`, `/spec-lint`, `/spec-gate`.

### Worked-example regression is run, not asserted

Every change must run cleanly on `examples/wctmr/` and `examples/axi_lite_slave_bfm/` after merge. The cross-review caught two items (D, K) whose "examples pass" claim was false because the linters had never been run against the examples. The acceptance bar is a command output, not a sentence in this plan.

### Lint rule severity matters more than lint coverage

P0 item C loosens two rules that produce noise. P2 item K defaults to WARN, not FAIL, and ships only after a measured precision number. B′ reports gaps as INFO, not FAIL. Lint that fires often without being acted on becomes noise authors learn to ignore. Prefer fewer, higher-signal rules.

### Mechanical checks beat pattern-matching heuristics

The cross-review demoted D (a vendor-attribution regex, domain-fragile) out of P0 and promoted M (register bit-overlap, pure structural parse). A check that parses a declared structure and asserts an invariant has near-zero false-positive risk; a check that pattern-matches prose for intent does not.

---

## 6. Generality self-check checklist

Run before any PR lands.

```
[ ] Designed without assuming a protocol family (AMBA / OCP / TileLink / custom)
[ ] Designed without assuming a vendor (AMD / ARM / Intel / SiFive / RISC-V foundation)
[ ] Plugin source contains zero project-specific word, template, or pattern
[ ] All project-specific configuration is opt-in (user-supplied flag or file)
[ ] Worked-example regression RUN (not asserted): both examples pass /spec-status, /spec-lint, /spec-review
[ ] Documentation (READMEs, SKILL.md, /spec-help) updated to reflect new surface
[ ] Mode dispatch: command behaves correctly in both modes, or is explicitly scoped to one
```

---

## 7. Resolved decisions (from cross-AI review)

The nine open questions are resolved per the two-reviewer cross-review (`plan/REVIEW_AGGREGATE.md` §2 consensus + §3 adjudication). Seven had reviewer consensus; two (Q-F1, Q-H1) were adjudicated. Pattern: every decision resolves toward the deterministic / tool-backed / non-blocking option.

| ID | Decision | Basis |
|---|---|---|
| Q-E1 | `/spec-impact` Phase-2 LLM is opt-in via `--analyse`; Phase-1 grep stays deterministic | consensus |
| Q-F1 | rename audit: commit-message body in a git repo; `RENAME_LOG.md` only when git is absent | adjudicated |
| Q-F2 | `/spec-rename` case-sensitive by default, `-i` opt-in | consensus |
| Q-G1 | `/spec-handoff` four fixed sections (may say "none") | consensus |
| Q-H1 | `D1.delta_from_plan` requires a written `PLAN.md` or `/spec-init` transcript; never freeform; else N/A stated loudly | adjudicated (stricter reading) |
| Q-H2 | delta checks only sections changed since the previous gate | consensus |
| Q-K1 | AI-tell lint severity = WARN | consensus |
| Q-L1 | Y→X tooling = command first, hook later | consensus |
| Q-L2 | Y→X review = report-only, never blocks a commit | consensus |

### Review history

- 2026-05-20: two-reviewer cross-AI review executed (Codex GPT-5.5 + fresh-context Claude). Both reviews and the aggregate are in `plan/REVIEW_CODEX.md`, `plan/REVIEW_CLAUDE.md`, `plan/REVIEW_AGGREGATE.md`. Two load-bearing false-positive claims (items D, K) were verified against the repo before this revision.

---

## 8. Provenance

Mapping of plan items to source observations.

| Plan item | Source observation | Source file |
|---|---|---|
| A `/spec-stats` | §11 spec-stats numeric summary | DOGFOOD §11 |
| B′ LINT-010 TP uniqueness | §10 TP sequence gap (revised after cross-review) | DOGFOOD §10; REVIEW_AGGREGATE §4 |
| C LINT loosening | LINT-002 false-warns, LINT-BFM-001 grouped-row | LINT_REPORT.md:14, 124 |
| D LINT-011 attribution (P1) | §4 attribution drift; demoted after verified false-positive | DOGFOOD §4; REVIEW_AGGREGATE §1 |
| E `/spec-impact` | §2 cross-doc sync ripple | DOGFOOD §2 |
| F `/spec-rename` | §12 cross-doc rename not assisted | DOGFOOD §12 |
| G `/spec-handoff` | §13 NEXT_SESSION pattern drift | DOGFOOD §13 |
| H `D1.delta_from_plan` | §1 over-design tendency | DOGFOOD §1 |
| I Implementer-review prompt | VC partition cross-file miss | IMPLEMENTER_REVIEW_LOG.md:276 |
| J Rule count auto-fill | §7 rule count drift | DOGFOOD §7 |
| K AI-tell lint | §8 AI-tells in spec content | DOGFOOD §8 |
| L Y→X review tooling | Y→X workflow undocumented in plugin | NEXT_SESSION_A4.md:159-169 |
| M LINT-013 register bit-overlap (NEW, P0) | cross-review missing-item, strongest candidate | REVIEW_CLAUDE §5 M1 |
| N LINT-015 cross-file scalar consistency (NEW, P1) | cross-review missing-item, both reviewers | REVIEW_CODEX §5 M2 + REVIEW_CLAUDE §5 M3 |
| O LINT-016 stale WAIVERS anchor (NEW, P1) | cross-review missing-item | REVIEW_CODEX §5 M1 |

Observations discarded with rationale in §2: DOGFOOD §3, §5, §6, §9.

Additional cross-review missing-candidates not yet promoted to items: reset-value-vs-encoding consistency (REVIEW_CLAUDE §5 M2), dangling override-annotation drift (REVIEW_CLAUDE §5 M4), normative-clause-without-TP traceability (REVIEW_CODEX §5 M3), citation existence check (REVIEW_CODEX §5 M5). Candidates for a v0.6.0+ plan iteration.

---

## 9. After this plan

Next concrete artifact is a Stage-1 PR implementing **A + B′ + C + M** as a single bundle. The merge gate is worked-example regression **run** on both `examples/wctmr/` and `examples/axi_lite_slave_bfm/` with zero false positives — not an asserted claim.

Per the CLAUDE.md mandatory-codex-cross-check rule, this revised plan should itself pass an independent `codex exec` confirmation before being treated as final. That pass also closes `REVIEW_AGGREGATE §5` step 8 (re-run reviewers after revision).

Versioning bumps the plugin manifest in both `.claude-plugin/marketplace.json` and `plugins/hw-spec-author/.claude-plugin/plugin.json` together, per the CLAUDE.md versioning convention.

# Dogfood Observations from noc-sim NI BFM (waves A1–A6)

This file catalogues plugin failure modes observed while authoring the noc-sim NI BFM spec across waves A1–A6, with each observation matched to a concrete plugin disposition.

This is the *backflow* document — observations originally captured in the dogfood working memory (`memory/project_plugin_improvement_observations.md`) consolidated here as plugin design intent. Where an observation already produced a plugin source change, the entry cites the file and section. Where it remains deferred, the entry notes scope and triggering wave.

## 1. Over-design tendency (plugin generates unrequested complexity)

**Symptom**: spec drafts contain features not in the user's explicit request — Reserved-encoding forward-compat clauses, debug-only acceptance paths, CDC mechanism choices, register splits, severity bumps, race-semantic clauses.

**Worked example (A4 wave)**: User's plan said "runtime selection of arbitration policy" (one line). The shipped spec added four unrequested mechanisms: CDC handshake, Reserved=3 forward-compat, debug-only acceptance, dual-arbiter binding. Required mid-wave revert.

**Disposition**: APPLIED via `references/process/writing_principles.md` §11 ("Default to minimum sufficient").

**Possible follow-on**: `/spec-gate` at D1 could ask the author to enumerate every clause the user did not explicitly request, and require per-item confirmation. Deferred.

## 2. Cross-doc sync ripple is not mechanized

**Symptom**: A4.7 wave's "drop VR mode" change touched 12+ files. Each wave required manual grep to discover stale references.

Items missed in mid-wave:

- `noc_model_guide.md`, `08_simulation.md`, `00_architecture.md`, `03_router.md`, `05_physical_channel.md` (design docs)
- `dv/plan.md` rule count + breakdown scattered in two places
- Cross-repo xref lint behaviour undocumented

**Disposition**: DEFERRED. Two candidate plugin changes:

- LINT-008 (terminology drift detection): user-defined synonym set, lint surfaces all cross-file occurrences.
- New `/spec-impact <change description>` command: LLM-driven dependency analysis identifying every file/section likely affected by a proposed change.

## 3. Stage-gate ↔ protocol-rule coupling is unclear

**Symptom**: A4.7 wave dropped 3 protocol rules (`NOC_MST_VALID_STABLE`, `NOC_MST_FLIT_STABLE`, `NOC_SLV_READY_NO_LATCH`) but no `stage_gates.md` anchor needed updating. The intuition that "every rule maps to a gate item" turns out to be wrong — gates are at architectural-concern granularity, rules are at line-item granularity. The two layers have no explicit mapping.

**Disposition**: DEFERRED. Candidate change: add a `rules:` field to stage-gate anchors listing the protocol-rule IDs each gate covers. `/spec-status` then cross-checks: every rule covered by some gate, every gate-cited rule exists.

## 4. Attribution drift across multi-round editing

**Symptom**: A4.6 wave produced a paraphrase "AMD: 每個 64-bit data granule 一個 8-bit ECC" attributed to AMD pg313. The actual AMD verbatim is whole-flit SECDED. The paraphrase originated in v0.3.0's own design but got mis-attributed to AMD across multiple authoring rounds. User had to grep the source to spot it.

**Root cause**: `CLAUDE.md` "numeric and completeness claims must cite tool output" rule applies to numbers, not to attribution claims.

**Disposition**: PARTIALLY APPLIED via `references/process/writing_principles.md` §13 ("External attribution requires verbatim citation").

**Possible follow-on**: LINT-010 — flag any "per <vendor>" / "aligns with <standard>" claim not paired with a verbatim quote within ~2 lines. Deferred.

## 5. Spec ↔ external design-doc sync strategy missing

**Symptom**: Plugin-generated BFM spec coexists with user's original design docs (e.g., `noc-sim/docs/design/`). Each wave required manual judgement on whether changes should ripple outward to design docs. Plugin offers no guidance: which document is the source of truth?

**Disposition**: DEFERRED. Candidate change:

- Plugin README §"Brownfield integration" describing the source-of-truth contract. Recommended: spec is the verification contract layer, design docs are the design source. Spec xref design docs but not the reverse. Design changes propagate spec → design, not design → spec.
- `/spec-import` brownfield mode requires declaring "design source repo path". Cross-repo xref list captured in `MODE.md`.

## 6. Multi-batch wave workflow lacks support

**Symptom**: A4.7 wave grew from 8 → 12 → 14 files as stale references surfaced mid-wave. Each "discover stale ref → ask user → expand scope" step consumed conversation turns. Plugin has no "wave plan + dynamic scope tracking" tooling.

**Disposition**: OUT OF SCOPE for current plugin direction. Would require new `/wave-init <name> <plan>` + `/wave-discover <stale-refs>` + `/wave-finalize` commands, similar to `git rebase --continue` but for spec-iteration sessions. Significant user-facing workflow change.

## 7. `dv/plan.md` rule count drifts

**Symptom**: Each wave that changes `protocol_rules.md` requires manually updating the rule count in `dv/plan.md` (`cg_protocol_rule_hits` covergroup count, ABV count, breakdown by category). Hand-maintained counts drift. A3 wave's "~135 rules" claim was off by 6.

**Disposition**: DEFERRED. Candidate change: `dv_plan.md` template marks rule-count fields with `<!-- RULE_COUNT_AUTO -->`. `/spec-status` then runs the canonical grep and either auto-updates or lint-warns on inconsistency.

## 8. AI-tells in plugin source artifacts

**Symptom**: Plugin's own `SKILL.md` and template prose unconsciously use the same AI-tell vocabulary that `CLAUDE.md` "Spec writing style" section bans (leverage / ensure / facilitate / robust / scalable / comprehensive / etc.). User flagged the pattern multiple times.

**Disposition**: PARTIALLY APPLIED — the plugin's `CLAUDE.md` (project-level) already bans these terms, and the global rule applies to plugin source authoring.

**Possible follow-on**: LINT-009 — vocabulary scanner over both spec content and plugin source files (templates, SKILL.md, command frontmatter). Deferred.

## 9. Industry-vocab residue not detected (A6 wave — concrete evidence for §2)

**Symptom**: Internal-coined labels persist in spec source across multiple files without lint detection. A6 wave web research (AMD PG313 / ARM IHI 0022 H.b+ / FlooNoC arXiv papers / Arteris FlexNoC docs / Dally & Towles canon) surfaced 4 internal terms as non-canonical industry vocabulary. The terms accumulated across A1–A5 waves; no existing lint rule caught them.

Empirical evidence in `spec/ni/doc/` (grep'd 2026-05-16 post-A6 wave):

| Label | Files affected |
|---|---|
| `(B)-philosophy` | 5 (`02_flit.md`, `protocol_rules.md`, `signal_interface.md`, `theory_of_operation.md`, `transaction_api.md`) |
| `single-chimney` | 2 (`02_flit.md`, `signal_interface.md`) |
| `wormhole-locked` | 4 (`channel_handshake.md`, `protocol_rules.md`, `theory_of_operation.md`, `transaction_api.md`) |
| `MetaBuffer` | 4 (`channel_handshake.md`, `protocol_rules.md`, `signal_interface.md`, `theory_of_operation.md`) |

13 file-occurrences total. Industry-canonical replacements per A6 research:

- `(B)-philosophy` → `log-and-forward fabric error reporting` (AMD-PG313-aligned)
- `single-chimney` → `one NMU + NSU per tile` (industry-neutral; FlooNoC term retained only in FlooNoC-cited context)
- `wormhole-locked` → `VC held for write-burst duration (wormhole)`
- `MetaBuffer` → `transaction context buffer` (FlooNoC spelling retained only with `floo_meta_buffer.sv` attribution)

**Root cause**: spec authoring tooling enforces local consistency (LINT-001 cross-refs, LINT-004/005 casing) but not industry-vocab alignment. Internal coinage normalises during design conversations; once landed in spec source no mechanical check questions it.

**Disposition**: DEFERRED. Strengthens §2 deferred LINT-008.

**Candidate plugin change**:

- New process doc `references/process/industry_vocabulary.md` — vendor-canonical terms table with provenance (AMD PG313 / ARM IHI 0022 H.b+ / FlooNoC / Arteris / Dally), plus known internal-coinage list with recommended replacements.
- LINT-009 (vocab residue): scan spec source for known internal-coinage labels. Allow Speaker-notes / explicit-provenance context (e.g., `*"FlooNoC `floo_meta_buffer.sv` style"*` references retain the label). Body-text occurrences flagged.
- Cross-ref §2 — same lint infrastructure handles both rename-drift and vocab-canonicalisation.

## 10. Testpoint ID sequence gap undetected by LINT-003

**Symptom**: A6 wave found `dv/plan.md` had TP IDs ranging TP1..TP51 but TP45 was absent (`grep -c "TP45" dv/plan.md` → 0). Real distinct count = 50. Numbering jumps TP44 → TP46. The gap accumulated across waves when a testpoint was renamed or deleted without renumbering subsequent IDs.

LINT-003 verifies the Feature ↔ Testpoint *mapping* presence but ignores ID sequence integrity.

**Worked-example impact**: A6 wave `SLIDES.md` Closing-slide carried "51 testpoints" — a max-ID claim, not a count claim. Caught manually during a content-spec consistency audit. Without the audit, the claim would have shipped to internal review.

**Disposition**: DEFERRED.

**Candidate plugin change**: LINT-003 extension OR new LINT-010 — extract every `TP\d+` ID from `dv/plan.md`, verify sequence is contiguous (no gaps, no duplicates). Report includes both the count and the max ID, distinguishing the two values.

## 11. Spec-stats numeric summary not surfaced by /spec-status

**Symptom**: A6 wave required repeated ad-hoc grep commands to extract canonical statistics (rule count 136, testpoint count 50, ABV count 136, parameter defaults). `/spec-status` reports D-stage progression but does not surface these aggregate numbers. Authors producing summaries — slide-deck closings, READMEs, status reports, NEXT_SESSION handoff docs — extract the same numbers manually each session.

Repeated greps observed in A6 wave:

```
grep "^| AXI4_\|NOC_\|NI_" dv/plan.md | ...   # rule count
grep -oE "^| TP[0-9]+" dv/plan.md | sort -u | wc -l   # testpoint count
grep "126 FAIL\|10 RECOMMEND" dv/plan.md      # ABV breakdown
```

**Disposition**: DEFERRED.

**Candidate plugin change**: extend `/spec-status` with `--stats` flag, or new `/spec-stats` command:

- Rule count from `protocol_rules.md` (FAIL / RECOMMEND breakdown).
- Testpoint count from `dv/plan.md` (distinct `TP\d+`, with max-ID separately reported per §10 distinction).
- ABV count from `dv/plan.md`.
- Parameter default summary from `signal_interface.md`.
- Cross-cited by LINT-010 (sequence-gap detection from §10) — `/spec-stats` provides the raw counts, LINT-010 enforces the invariants.

## 12. Cross-doc rename not assisted

**Symptom**: A6 wave's industry-vocab fix (§9) requires renaming `(B)-philosophy` → `log-and-forward fabric error reporting` across 5 spec files. No plugin utility helps:

- Each file's hit needs context judgement (body / Speaker-notes provenance / inline-historical-note).
- Casing or hyphenation variants might appear (`(B)-philosophy` vs `B philosophy` vs `b-philosophy`).
- Dry-run preview before bulk apply is manual.

Authors fall back to `git grep` plus iterative `Edit` calls. Error-prone for terms appearing in dense prose where mechanical replace would clobber provenance context.

**Disposition**: DEFERRED.

**Candidate plugin change**: new `/spec-rename <old> <new> [--dry-run]` command:

- Scan spec source files. List each hit with `file:line:context` snippet.
- Classify each hit: body (apply), Speaker-notes / explicit-provenance (skip with reason), heading / register name (review needed).
- Produce impact report. Author confirms before `--apply`.
- Complement to §2 LINT-008 (passive detection) and §9 LINT-009 (passive enforcement). Active rename is a separate command.

## 13. `NEXT_SESSION_<wave>.md` handoff pattern undocumented

**Symptom**: Handoff doc convention used in A4 wave (post-A4.7), A5 wave (post-A5 closure), and A6 wave (`spec/ni/NEXT_SESSION_A6.md`). Each wave's author reinvents the structure. Some have "Status snapshot + Deferred tasks" sections, others have "Quick re-entry checklist + Reminders to future self". The pattern works — but inconsistency makes cross-session handoff brittle.

**Disposition**: DEFERRED.

**Candidate plugin change**: new `references/process/wave_handoff.md` template with fixed sections:

- Status snapshot (mode, lint status, deferred-task state, last-commit hash)
- Deferred tasks (each with priority, blocker analysis, concrete first step, acceptance criteria, estimated effort)
- Quick re-entry checklist (5-minute sanity checks for next-session start)
- Reminders to future self (gotchas, design decisions deliberately deferred, scope boundaries)

OR new `/spec-handoff <wave>` command that scaffolds the file from a current-spec snapshot (auto-populates Status snapshot via /spec-stats from §11).

## Cross-cutting plugin enhancements (from A5 multi-agent implementer review)

The A5 wave introduced a new validation method beyond the existing `/spec-review` reader test: spawning paired implementer agents (one C-model, one RTL) and comparing their independent reads of the spec.

Both agents converged on the same top-priority ambiguities (flit bit layout missing from `spec/ni/`, Hamming-vs-Hsiao SECDED contradiction, `vc_id` mapping function not specified, CDC partial-reset flush algorithm undefined). Each also identified ~5 paradigm-specific ambiguities the other missed.

This validates the proposition that **cross-paradigm implementer review surfaces a different class of ambiguity than reader test**. Reader test catches "is this answerable from spec text". Implementer review catches "would two paradigm-different implementations produce bit-identical wire behaviour."

**Disposition**: APPLIED in plugin v0.4.0 via `/spec-implementer-review` command, `agents/implementer-reviewer.md` subagent, and `references/process/implementer_review.md`. Stage-gate item `D1.cross.implementer_review` added.

## Priority for plugin work

When the user invests in plugin improvement (vs continuing dogfood):

1. **Prevent systemic errors first**: §1, §4, §7 (writing-principles + lint rules).
2. **Improve workflow next**: §2, §5, §6 (cross-doc impact tooling).
3. **Stabilise infrastructure**: §3, §8 (gate ↔ rule mapping, vocabulary lint).
4. **Cross-cutting innovation**: multi-agent implementer review (APPLIED v0.4.0).
5. **A6 dogfood — concrete and immediate (post-v0.4.0)**: §9, §10, §11 (industry-vocab residue lint, TP sequence gap, /spec-stats). Small, immediate value with concrete evidence.
6. **A6 dogfood — quality-of-life**: §12, §13 (/spec-rename utility, wave-handoff template).

Each item should pair with a worked example pulled from the noc-sim dogfood (e.g., A4 VC_ARB_MODE for §1, A4.6 ECC attribution for §4, A4.7 14-file scope expansion for §2, A6 `(B)-philosophy` 5-file residue for §9, A6 TP45 gap for §10) when the plugin change lands.

## Source waves

A1 (initial spec), A2 (v0.4.0 header restructure), A3 (runtime CSR), A4 (runtime control), A4.5 (FlooNoC port_id removal), A4.6 (AMD ECC alignment), A4.7 (drop VR mode), A4.8 (review-driven precision fixes), A5 (pre-presentation polish + multi-agent implementer review), A6 (slide deck refactor + industry-vocab research; plugin observations §9–§13).

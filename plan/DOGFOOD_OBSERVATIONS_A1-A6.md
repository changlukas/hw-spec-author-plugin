# Dogfood Observations from noc-sim NI BFM (waves A1–A5)

This file catalogues plugin failure modes observed while authoring the noc-sim NI BFM spec across waves A1–A5, with each observation matched to a concrete plugin disposition.

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

## Cross-cutting plugin enhancements (from A5 multi-agent implementer review)

The A5 wave introduced a new validation method beyond the existing `/spec-review` reader test: spawning paired implementer agents (one C-model, one RTL) and comparing their independent reads of the spec.

Both agents converged on the same top-priority ambiguities (flit bit layout missing from `spec/ni/`, Hamming-vs-Hsiao SECDED contradiction, `vc_id` mapping function not specified, CDC partial-reset flush algorithm undefined). Each also identified ~5 paradigm-specific ambiguities the other missed.

This validates the proposition that **cross-paradigm implementer review surfaces a different class of ambiguity than reader test**. Reader test catches "is this answerable from spec text". Implementer review catches "would two paradigm-different implementations produce bit-identical wire behaviour."

**Disposition**: PROPOSED for plugin v0.4.0. Tracked design candidate: new `/spec-implementer-review` command spawning paradigm-paired agents, generating a cross-diff of ambiguities, optionally feeding into a `D1.cross.implementer_review` gate item.

## Priority for plugin work

When the user invests in plugin improvement (vs continuing dogfood):

1. Prevent systemic errors first: §1, §4, §7 (writing-principles + lint rules)
2. Improve workflow next: §2, §5, §6 (cross-doc impact tooling)
3. Stabilise infrastructure: §3, §8 (gate ↔ rule mapping, vocabulary lint)
4. Cross-cutting innovation: multi-agent implementer review (proposed v0.4.0)

Each item should pair with a worked example pulled from the noc-sim dogfood (e.g., A4 VC_ARB_MODE for §1, A4.6 ECC attribution for §4, A4.7 14-file scope expansion for §2) when the plugin change lands.

## Source waves

A1 (initial spec), A2 (v0.4.0 header restructure), A3 (runtime CSR), A4 (runtime control), A4.5 (FlooNoC port_id removal), A4.6 (AMD ECC alignment), A4.7 (drop VR mode), A4.8 (review-driven precision fixes), A5 (pre-presentation polish + multi-agent implementer review).

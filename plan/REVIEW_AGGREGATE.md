# Cross-AI Review Aggregate — `plan/PLUGIN_IMPROVEMENT_PLAN.md`

**Method**: two independent fresh-context reviews, neither saw the authoring conversation.
- `plan/REVIEW_CODEX.md` — Codex GPT-5.5 (ChatGPT subscription, separate process).
- `plan/REVIEW_CLAUDE.md` — fresh-context Claude subagent (no authoring bias).

**Verification status**: the two load-bearing concrete claims were checked against the repo before aggregation (CLAUDE.md trust-but-verify). Both hold.
- ✅ `examples/wctmr/doc/registers.md:104` contains "...following each write to `CNT_LO`. See Theory of Operation §Software write atomicity" — an internal cross-reference, not a vendor attribution. Item D's regex would WARN on it.
- ✅ `examples/axi_lite_slave_bfm/doc/active_passive_mode.md:31` and `dv/plan.md:63` contain `provides` / `well-understood` in legitimate technical prose. Item K's word list fires on them. At least 5 `provides` hits exist across the example.

---

## 1. Headline finding (both reviewers, verified)

The plan states "worked-example regression: both examples pass" as the Stage-1 merge gate. **For items D and K that claim is false on inspection.** D's attribution regex and K's AI-tell word list both fire on the existing worked examples. An acceptance bar asserted but never run is the exact failure mode CLAUDE.md's working protocol exists to prevent.

Action: run the proposed linters against `examples/wctmr/` and `examples/axi_lite_slave_bfm/` *before* declaring any lint item P0-ready.

---

## 2. Consensus (both reviewers independently agree — treat as high-confidence)

| # | Finding |
|---|---|
| C1 | The generality *principle* is sound. The failures are in mechanical specification, not in IP-class scope. |
| C2 | All three discarded items (LINT-008 vocab, gate↔rule mapping, mirror sync) are correctly discarded. |
| C3 | LINT-008 is salvageable as an **opt-in user-supplied** word list (loaded only if present in the user's spec dir). Low residual risk. Worth a one-line note, not a rebuild. |
| C4 | Item D (attribution lint) is the weakest retained item. Should not ship in P0 as specified. |
| C5 | Item K (AI-tell lint) must default to WARN, not FAIL, and needs a measured false-positive rate before shipping. |
| C6 | A **cross-file consistency lint** (same signal/parameter described in 2+ files must agree on width/value) is a valuable missing item. Codex M2 + Claude M3 converge here. |

### Open-question consensus (7 of 9 agree)

| Q | Agreed answer |
|---|---|
| Q-E1 | `/spec-impact` Phase-2 LLM is **opt-in** (`--analyse`). Phase-1 grep stays deterministic. |
| Q-F2 | `/spec-rename` **case-sensitive by default**, `-i` opt-in. Hardware identifiers are case-significant. |
| Q-G1 | `/spec-handoff` **4 fixed sections** (subsections may say "none"). Fixed structure kills the drift the item exists to fix. |
| Q-H2 | `D1.delta_from_plan` checks **only changed sections** since the previous gate. Full re-read gets skipped. |
| Q-K1 | AI-tell lint severity = **WARN**. |
| Q-L1 | Y→X tooling = **command first**, hook later only if demanded. |
| Q-L2 | Y→X review = **report-only**, never block a commit. |

Pattern both reviewers note: every open question resolves toward the **deterministic / tool-backed / non-blocking** option — the same discipline CLAUDE.md mandates.

---

## 3. Disagreements (with adjudication)

| Topic | Codex | Claude | Adjudication |
|---|---|---|---|
| **Q-F1** rename audit log | Dedicated `RENAME_LOG.md` (rename decisions are spec history; non-git users need it) | Commit message body (`RENAME_LOG.md` is drift-prone, the anti-pattern A/J fight) | **Commit body when in a git repo; `RENAME_LOG.md` only when not.** Captures both concerns. Default to git history, fall back to file only when git is absent. |
| **Q-H1** plan artifact | Accept `PLAN.md` / interview transcript / explicit "no written plan" declaration | `PLAN.md` or transcript, **never freeform** | **Claude's stricter reading.** A freeform "no delta detected" is unfalsifiable — the exact greenfield-by-conversation case where the A4 over-design failure arose. Require a written artifact, or the gate item is N/A and says so loudly. |
| **Items I / J** generality | "**No**" — fails guard 1 (BFM-only) | "Scoped / declared — honest, no guard failure" | **Claude's reading.** The plan explicitly declares I and J as BFM-only inside P2. A declared scope limit is not a hidden generality failure. Codex applied guard 1 mechanically without crediting the plan's own scoping. (Codex's underlying fact — they are not universal — is true; the framing is the disagreement.) |
| **Verdict strictness** | Sign off Stage 1, split D if noisy. Confidence medium. | Do **not** sign off `A+B+C+D` as-is; sign off revised `A+C+B′`, demote D. Confidence medium-high. | **Claude's position.** It is backed by the verified false-positives. Codex's "sign off" predates running the linters against the examples. |

---

## 4. Unique findings worth acting on

### From Claude (not raised by Codex)

| id | finding | value |
|---|---|---|
| **B-policy** | Item B bakes a *numbering policy* (contiguous 1..N), not a *uniqueness check*. An IP that numbers testpoints by category (AES: TP1xx datapath / TP2xx key-schedule / TP3xx side-channel) is well-organised yet would FAIL. | High — B as written smuggles a convention under a "mechanical check" label. Fix: check uniqueness, report gaps as **INFO** not FAIL. |
| **M1 register bit-overlap** | New lint: parse `registers.md` field tables, assert non-overlapping bit ranges within declared width, report holes as INFO. | **Strongest missing candidate.** More universal than retained B (every register-bearing IP has it), pure mechanical, exact P0 profile. |
| **M4 dangling override drift** | The plan adds three inline override escape hatches (D / H / K). Nothing checks an override still applies after the guarded line is edited. A stale override silently suppresses a real future warning. | Medium — meta-risk created by the plan's own design. |
| **M2 reset-value vs encoding** | Field documents reset `0x3` but its encodings only define `0,1,2`. Same family as the "Reserved=3" confusion in DOGFOOD §1. | Medium. |
| **M5 spec↔git staleness** | After a multi-file change, some files committed, others forgotten (DOGFOOD §2's 12-file ripple shipped partial twice). | Medium — rides cheaply on item A (`git log -1 --format=%cr` per-file column). |

### From Codex (not raised by Claude)

| id | finding | value |
|---|---|---|
| **M1 stale waiver anchor** | Lint `WAIVERS.md` anchors against current `stage_gates.md`; flag waivers citing retired/changed anchors. | High — directly protects the stage-gate ↔ WAIVERS contract the plan treats as load-bearing. |
| **M3 normative-clause-without-TP** | Lint clauses using "shall"/"must" that have no DV testpoint or rule linkage. | Medium-high — verification-traceability gap, generic to any spec. |
| **M5 citation existence check** | Inventory external citation links / file refs, WARN on unreachable. | Medium — pairs naturally with item D's attribution work. |

---

## 5. Recommended revised plan action

1. **Revise Stage 1 (P0) bundle** from `A + B + C + D` to `A + C + B′`:
   - `B′` = B with **uniqueness check + gaps-as-INFO**, not contiguity-as-FAIL.
   - **Demote D to P1.** Its regex needs design to exclude internal `§` cross-references. Not low-risk, not assumption-free — fails the P0 charter.
2. **Gate K on a measured false-positive rate.** Run the AI-tell list against both examples and CLAUDE.md's own banned list first. Default WARN. Ship only when precision is acceptable.
3. **Promote Claude M1 (register bit-overlap lint) into P0 candidacy.** It is the cleanest mechanical, zero-assumption check on the table — stronger P0 fit than D ever was.
4. **Add a cross-file consistency lint** (C6 consensus) as a P1 item: same identifier's width/value must agree across files.
5. **Add Codex M1 (stale waiver anchor lint)** as a P1 item — protects the WAIVERS contract.
6. **Tighten §2 rationale**: the gate↔rule discard's real reason is the granularity mismatch (DOGFOOD §3: gates are architectural, rules are line-item), not only the schema-break. Both reviewers flagged the stated reason as weaker than the real one.
7. **Resolve the 9 open questions** per §2 consensus + §3 adjudication. Record answers back into the plan §7.
8. **Re-run both reviewers** after the revision to confirm the false-positive findings are closed.

---

## 6. Net assessment

Both reviewers endorse the plan's direction and the generality filter. Neither found an IP-class scope failure in the *principle*. The actionable defects are concentrated in three lint items (D verified-broken, K verified-noisy, B policy-smuggling) and in a handful of high-value missing mechanical checks (register bit-overlap, cross-file consistency, stale-waiver-anchor). The open-question defaults converge cleanly on the safe side. With the §5 revisions, Stage 1 is shippable.

**Confidence**: high. The two reviews converge on the same core defects, and the two load-bearing claims are verified against the repo rather than asserted.

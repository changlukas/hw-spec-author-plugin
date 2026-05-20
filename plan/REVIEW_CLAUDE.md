# Cross-AI Review — `plan/PLUGIN_IMPROVEMENT_PLAN.md`

**Reviewer**: fresh-context Claude session. Has not seen the dogfood logs or the authoring conversation.
**Verdict in one line**: sound direction, but three lint items (D, C, K) ship regex/word-list mechanics that the plan asserts are clean against the worked examples without verifying — I found counter-evidence in the examples for two of them.

---

## Section 1. Generality assessment

| item-id | passes 5 guards? | weakest evidence | counterexample IP class |
|---|---|---|---|
| A `/spec-stats` | Mostly | Rule/ABV/TP counts are BFM-shaped. For behavioral-block it degrades to register count only — thin value. | **eFuse macro** (behavioral): no protocol rules, no TP-per-rule structure. `/spec-stats` reports register count + maybe TP count; the marketed "aggregate dashboard" collapses to one number. Applies cleanly but the value proposition is BFM-weighted. No guard fails. |
| B LINT-010 (TP continuity) | Yes | Assumes `TP\d+` is universal and that contiguous numbering is *desirable*. | **AES core** (behavioral, DV-heavy): legitimately uses TP groups by category (TP1xx datapath, TP2xx key-schedule, TP3xx side-channel). Contiguous 1..N is the wrong invariant; the rule would FAIL a well-organised plan. Guard 1 (no IP-class assumption) is at risk: the rule bakes in a *numbering policy*, not just a *uniqueness* check. |
| C LINT loosening | Yes | The `pin_level_reset` grouped-row loosening is BFM-only; harmless elsewhere. | **USB PHY**: fine. No guard fails. But see §4 — the grouped-row tuple `(reset value, channel, direction)` assumes a *channel* axis that channel-less IPs (eFuse, ADC) do not have; the loosened rule must no-op gracefully, not key on a missing column. |
| D LINT-011 (attribution verbatim) | **No** | The pattern `following <X> §Y` collides with internal cross-references. **Verified**: `examples/wctmr/doc/registers.md:104` — "...one cycle following each write to `CNT_LO`. See Theory of Operation §Software write atomicity." This is an internal xref, not a vendor attribution. LINT-011 as specified would WARN on it. | **Any IP with internal §-cross-refs** (all of them). The regex is the failure, not the IP class. Guard 4 (worked-example regression) is violated as written — the plan claims "must not regress" but the pattern matches non-attribution prose. |
| E `/spec-impact` | Yes | Keyword-grep quality depends on the change description; degenerate on vague input (see §4). | **Crypto core**: "tighten the key-zeroization timing" — no proper-noun keyword to grep. Phase 1 returns near-empty; user assumes "no impact." Applies, but silently weak on conceptual (non-identifier) changes. No hard guard fails. |
| F `/spec-rename` | Yes | Pure string match; cleanest item in the plan. | **I2C controller**: rename `repeated-start` → `Sr`. Works. The one snag is overlap with register names (see §4), not generality. No guard fails. |
| G `/spec-handoff` | Yes | Workflow-only; content user-filled. | **Any**: applies. Risk is adoption, not generality. The §1 auto-fill depends on `/spec-stats` (A) being meaningful — for an eFuse macro the auto-filled snapshot is nearly empty. Soft coupling, no guard failure. |
| H `D1.delta_from_plan` | Conditional | Hinges on a "plan" artifact existing. Q-H1 admits it is undefined. | **SDIO host** authored greenfield by conversation with no written PLAN.md: the gate has nothing to diff against, so "no delta detected" becomes an unfalsifiable rubber-stamp. The mechanism is generic; its *teeth* are not. Guard 1 passes; the gate's value does not survive the no-written-plan case (the exact case CLAUDE.md's A4 violation arose in). |
| I Implementer-review pairing | Scoped (declared) | Explicitly `has-rtl-counterpart: yes` only. Honest scoping. | **ADC front-end** with analog-model counterpart: "two paradigm-different implementations" maps poorly (one side is analog behavioral, not RTL). Within declared scope it is fine; the architectural-pair heuristic assumes two *digital* views. |
| J Rule-count auto-fill | Scoped (BFM-only, declared) | Honest. | **GPIO controller** (behavioral): no `protocol_rules.md`, marker never inserted. Correctly N/A. No guard fails. |
| K LINT-012 (AI-tell vocab) | **At risk** | **Verified**: AI-tell grep over `examples/` returns ~10 hits, several legitimate — `active_passive_mode.md:31` "active BFM **provides** cycle-accurate response timing", `dv/plan.md:63` "the protocol rules are **well-understood**... **provides** adequate confidence." These are correct technical prose. "provides/ensures" have valid uses; a flat word list cannot tell them apart. | **Any IP**: the word list is generic (good), but the false-positive rate is not. Guard 4 at risk: plan says "both examples lint clean" — they do not, under the banned-word list in CLAUDE.md. The hedge "or surface a small set of actionable items" papers over a real precision problem. |
| L Y→X pre-commit review | Yes | Workflow-only. | **Any**: applies. No guard fails. |

**Net**: 8 of 12 pass cleanly. **D fails outright** (regex collides with internal cross-refs, demonstrated). **B and K are at risk** (B bakes a numbering *policy*; K's worked-example-clean claim is false as verified). The generality *principle* is sound; the failures are in mechanical specification, not in IP-class scope.

---

## Section 2. Discarded-items reasoning audit

| candidate | discard reason sound? | alternative framing that saves it? |
|---|---|---|
| LINT-008 vocab residue (built-in list) | **Yes.** A shipped word list is project-specific by definition; `(B)-philosophy` is meaningless to a GPIO author. Discard is correct. | **Salvageable as opt-in.** Frame as `references/process/<user>_vocabulary.md` loaded *only if present in the user's spec dir* — same opt-in pattern the plan already endorses for K's list (except K's is plugin-shipped/neutral; this one is user-supplied). Residual risk: low. The plan already half-builds this via `/spec-rename` (active) — the missing half is *passive recurrence detection* after a rename, so a renamed term does not creep back in a later wave. Worth a one-line note, not a rebuild. |
| Stage-gate ↔ rule `rules:` mapping | **Yes, on contract-break grounds.** Editing anchor schema breaks `WAIVERS.md`. But DOGFOOD §3 showed the deeper reason: gates are architectural-granularity, rules are line-item — the mapping is *semantically wrong*, not just risky. The plan undersells this; it cites only the schema-break, missing the stronger argument. | **Salvageable as advisory, non-schema.** A `/spec-stats`-side check "every rule ID appears in some gate's prose, every gate-cited rule exists" needs no anchor-schema change — it greps existing text. Residual risk: low value (DOGFOOD §3 says the 1:1 intuition is wrong anyway), so even the advisory framing has weak ROI. Correctly parked. |
| Mirror sync helper | **Yes.** Plugin-author-internal; regular users have no mirror. Unambiguous discard. | None worth pursuing. A generic `/spec-sync <dest>` would be a thin wrapper over `robocopy`/`rsync` with zero spec-domain value. Leave discarded. |

All three discards are sound. The plan's *stated reasons* are weaker than the *real* reasons for the first two — recommend tightening the §2 rationale (esp. the granularity-mismatch argument for the gate↔rule mapping).

---

## Section 3. Open questions

| Q-id | recommended answer | rationale (≤2 sentences) | risk if chosen wrong |
|---|---|---|---|
| Q-E1 (`/spec-impact` Phase 2 default) | **Opt-in (`--analyse`).** | Phase-1 grep is deterministic and cheap; Phase-2 LLM judgement is the part that hallucinates "materially affected" verdicts, exactly the unreliability CLAUDE.md warns against. Default to the tool-backed phase; make the judgement phase explicit. | If wrong (default-on): users trust an LLM ripple list, miss a file grep would have caught, ship a stale ref. |
| Q-F1 (rename audit log) | **Commit message body.** | The rename *is* the diff; git already records before/after, and a `RENAME_LOG.md` is one more drift-prone hand-maintained artifact (the exact anti-pattern items A/J fight). | If wrong (RENAME_LOG.md): yet another file to keep in sync; abandoned after two waves. |
| Q-F2 (case-insensitive default) | **Off by default; `-i` opt-in.** | Hardware identifiers are case-significant (`awready` ≠ `AWREADY` ≠ a prose word "ready"); default-insensitive clobbers signal names. Interactive per-hit confirm mitigates, but default should not invite the foot-gun. | If wrong (default-on): a rename of "ready" hits every `*ready` signal and English "already"; high skip burden, user disables the tool. |
| Q-G1 (handoff template strictness) | **4 fixed sections.** | The whole observation (DOGFOOD §13) is that *inconsistency* made handoff brittle; modular opt-in re-introduces the variance the item exists to kill. Fixed structure, sections allowed to say "none." | If wrong (modular): drift returns; item delivers nothing over freeform. |
| Q-H1 (canonical plan artifact) | **User-supplied `PLAN.md` if present, else `/spec-init` interview transcript; never freeform.** | The gate must diff against *something written* or it is unfalsifiable (see §1 H). Freeform = rubber-stamp. | If wrong (freeform allowed): "no delta detected" becomes a checkbox; the A4 over-design failure recurs unguarded. |
| Q-H2 (delta scope) | **Only changed sections since previous gate.** | Full re-read every gate is unaffordable and will be skipped; incremental delta matches how waves actually accrete (CLAUDE.md single-file-batching ethos). | If wrong (full re-read): too costly, authors stub it, gate decays to ceremony. |
| Q-K1 (AI-tell severity) | **WARN.** | Verified false-positive rate on the worked examples (§1 K) makes FAIL a build-blocker on legitimate prose; WARN + override annotation matches CLAUDE.md's "fewer, higher-signal" stance and the plan's own §5. | If wrong (FAIL): every `provides`/`well-understood` blocks a gate; authors blanket-annotate, signal lost. |
| Q-L1 (command / hook / both) | **Command (option a) only, first.** | A pre-commit hook fires on every commit regardless of relevance and is hard to scope to spec dirs; a command is explicit and matches the existing `/spec-*` surface. Ship the hook later only if demanded. | If wrong (hook): fires on doc-typo commits, gets `--no-verify`-bypassed, then ignored. |
| Q-L2 (block vs report) | **Report-only.** | Blocking a commit on a subagent's subjective review contradicts CLAUDE.md's "review-then-extend" and would stall waves; surface findings, let the author decide. | If wrong (block): false-positive review finding blocks a correct commit; trust erodes, tool bypassed. |

Pattern across answers: **default to the deterministic / tool-backed / non-blocking option every time.** This is the same discipline CLAUDE.md's working-protocol section mandates.

---

## Section 4. Stress-test axes

**Generality** — see §1. Hardest counterexamples: AES core (B's numbering policy), eFuse/ADC (channel-less, breaks C's grouped-row tuple and thins A), any IP with internal §-refs (breaks D's regex). Cross-ref §1 table.

**Edge cases** (the three the prompt names):

- *`/spec-rename` when `<old>` overlaps a register name*: the plan's interactive per-hit `[a]/[s]/[q]` handles it **only if the user notices**. There is no signal that a hit is a register-table row vs prose. Recommend: `/spec-rename` should *tag* each hit with its file/section context (registers.md table row → flagged "REGISTER NAME — review") so the user is warned, not left to eyeball line numbers. Currently under-specified.
- *`/spec-impact` when description is vague*: degrades to empty/over-broad grep with no honesty signal. Recommend the command emit "extracted keywords: [...]; if this list is wrong, rephrase" so a vague input does not masquerade as "no impact found." Currently the plan is silent on the empty-result case — the dangerous one.
- *`D1.delta_from_plan` when no written plan exists*: **the central weakness.** As specified, the author writes "no delta detected" with nothing to check against — unfalsifiable. This is precisely the greenfield-by-conversation case where A4's over-design happened. Q-H1 must resolve to "require a written artifact or the gate item is N/A and says so loudly," never a silent pass.

**Discarded items** — rebuttals in §2. Only the user-supplied opt-in vocab list (salvaging LINT-008's *passive recurrence* half) is worth a follow-up note; the other two stay parked.

**Missing items** — see §5.

**Scope creep**:
- **B** drifts hardest: "contiguous 1..N" is a *project numbering convention*, not a universal invariant. An IP that numbers TPs by category (AES) is well-organised yet FAILs. B should check **uniqueness + report gaps as INFO**, not FAIL on non-contiguity. As written it smuggles a policy in under a "mechanical check" label.
- **D** drifts via its vendor-pattern regex: matching `following <X> §Y` reaches into ordinary English ("following each write... §...") — the regex assumes "following + §" implies attribution, a domain assumption that is false.
- **A**'s "parameter defaults one-line summary" implies a `§Parameters` section shape; behavioral-block specs may not name it identically. Minor.

**Sequencing**:
- **D should drop out of P0.** P0 is sold as "mechanical, generic, low-risk, ships zero new domain assumption." D ships a vendor-attribution regex with a demonstrated false-positive against `wctmr/registers.md`. It is neither low-risk nor assumption-free. Demote D to P1 (needs design discussion on the regex), and the Stage-1 bundle becomes **A + B + C** — except B also needs the uniqueness-not-contiguity fix first. Realistically the clean P0 bundle is **A + C** plus a *revised* B.
- **K should not precede a false-positive-rate measurement.** Before shipping LINT-012, run the word list against both examples and CLAUDE.md's own banned list; the §1 evidence says it will fire on legitimate prose. Gate K on a measured precision number, not a hedge.
- Everything else (E→F→G→H, then I→J→L) is ordered sensibly. F before E is defensible too (rename is simpler and higher-immediate-value than impact analysis).

---

## Section 5. Missing-observation candidates

| candidate-id | failure mode | why generic | suggested mechanism |
|---|---|---|---|
| M1 register field-bit overlap / width overflow | Two fields in one register claim overlapping bit ranges, or fields sum past the register width. Hand-drawn bit tables drift; a 32-bit register with fields summing to 33 ships silently. | Every register-bearing IP (GPIO, DMA, crypto, PHY — i.e. *all* behavioral-block specs and most BFMs) has bit-packed registers. More universal than TP numbering (B). | LINT-013: parse `registers.md` field tables, assert non-overlapping ranges within declared width, report holes as INFO. Pure mechanical, zero domain assumption — exactly the P0 profile the plan wants. |
| M2 reset-value vs field-encoding consistency | A field documents reset value `0x3` but its enumerated encodings only define `0,1,2`. Reader cannot tell if reset is a legal state or a typo. Observed class is the same family as DOGFOOD §1's "Reserved=3" confusion. | Any IP with enumerated register fields or mode selectors. | LINT-014: cross-check each field's reset value against its own encoding table; WARN on reset ∉ encodings. |
| M3 cross-file unit / width mismatch | `signal_interface.md` declares a port `[31:0]` while `theory_of_operation.md` describes it as 16-bit, or one file says "bytes" and another "beats." Item I catches *architectural* pairs but not *scalar attribute* mismatches. | Multi-file specs in both modes carry the same signal/parameter described in 2+ files. The drift is purely mechanical. | Extend LINT-001 (cross-ref) or new LINT-015: extract `signal[high:low]` and parameter values per file, diff the same identifier across files, FAIL on width disagreement. |
| M4 dangling override-annotation drift | The plan adds three override annotations (D's `(<vendor>-derived...)`, H's `(no delta...)`, K's `(non-AI-tell context...)`). Nothing checks an override still applies after the line it guarded is edited. A stale override silently suppresses a real future warning. | Any lint with an inline-override escape hatch accumulates orphaned overrides over a multi-wave life. Generic to the override mechanism itself. | LINT-016: flag override annotations whose adjacent matched line no longer matches the rule (override with nothing to override) → likely stale, WARN. |
| M5 spec ↔ git staleness signal | After a multi-file change, some files were touched and committed, others forgotten (DOGFOOD §2's 12-file ripple shipped partial twice). No signal tells the author "you edited protocol_rules.md but dv/plan.md is older than its last rule change." | Any multi-file spec under git. Independent of IP class and mode. | `/spec-stats` (item A) add a per-file `git log -1 --format=%cr` column; flag files whose mtime predates a sibling they cross-reference. Cheap, rides on A. |

M1 is the strongest candidate — more universal than the retained B, same low-risk mechanical profile, and a classic silent-shipping failure.

---

## Section 6. Overall verdict

I would **not** sign off on Stage 1 as `A + B + C + D` exactly as proposed; I would sign off on a **revised** Stage 1 of `A + C + B′` (B with uniqueness-not-contiguity semantics) and **demote D to P1**. The single most important concern: the plan asserts "worked-example regression, both examples pass" as its acceptance bar, but for items **D and K that claim is false on inspection** — D's attribution regex matches an internal cross-reference in `wctmr/registers.md:104`, and K's AI-tell list fires on legitimate prose like "provides cycle-accurate response timing" in the axi_lite example. Acceptance bars asserted-but-not-run are the precise failure mode CLAUDE.md exists to prevent; run the linters against the examples *before* declaring the bar met. The generality principle and the discard reasoning are otherwise sound, and the open-question defaults all resolve toward the safe (deterministic, non-blocking) side. **Confidence: medium-high.**

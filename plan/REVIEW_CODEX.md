## Section 1. Generality assessment

| item-id | passes 5 guards? | weakest evidence | counterexample IP class |
|---|---:|---|---|
| A | yes | Output schema names BFM-specific rule and ABV counts before defining graceful N/A behavior. | ADC front-end: applies if mode dispatch omits rule counts cleanly. |
| B | yes | Assumes `TP\d+` is universal across DV plans. | eFuse macro: applies if DV plan uses plugin TP IDs. |
| C | mostly | Grouped reset rows are BFM-shaped and may not exist in behavioral specs. | AES core: LINT-002 applies. Grouped pin reset may be N/A. |
| D | mostly | Pattern detection for attribution claims may miss non-vendor standards language. | USB PHY: applies to USB-IF attributions if generic pattern covers standards. |
| E | yes | Keyword extraction quality is underspecified. | I2C controller: applies cleanly. |
| F | yes | Pure string match can over-hit semantic identifiers. | SDIO host: applies, but register-name overlap needs guardrails. |
| G | yes | Worked-example acceptance is waived, weakening guard 4. | eFuse macro: applies cleanly as workflow artifact. |
| H | mostly | Requires a canonical plan that may not exist. | ADC front-end: applies if freeform plan evidence is accepted. |
| I | no | Scope is explicitly protocol-bfm with RTL counterpart only. | AES core: fails guard 1 unless reframed as generic cross-file implementer review. |
| J | no | BFM-only rule count auto-fill. | I2C controller behavioral spec: fails guard 1 unless scoped as mode-specific feature. |
| K | yes | “Industry-neutral” word list is asserted, not proven. | USB PHY: applies if list remains style-only, not domain vocabulary. |
| L | mostly | Assumes subagent or git hook workflow exists. | eFuse macro: applies as process, but hook option assumes git. |

## Section 2. Discarded-items reasoning audit

| candidate | discard reason sound? | alternative framing that saves it? |
|---|---|---|
| LINT-008 vocab residue | Partly sound. Built-in project vocabulary would violate the filter. | User-supplied `terminology_watchlist.md` plus `/spec-lint --terms`. Residual risk: medium, because users may treat examples as defaults. |
| Stage-gate to protocol-rule mapping | Sound for mandatory schema change. Too narrow if tied to protocol rules. | Advisory coverage map outside `WAIVERS.md`, with optional `covered-by:` metadata. Residual risk: medium-high due anchor contract confusion. |
| Mirror sync | Sound. It is plugin-author internal. | Generic “external artifact sync checklist” in handoff template. Residual risk: low, but likely low value. |

## Section 3. Open questions

| Q-id | recommended answer | rationale (≤2 sentences) | risk if chosen wrong |
|---|---|---|---|
| Q-E1 | Opt-in via `--analyse`. | Mechanical grep is auditable. LLM impact judgment can overstate scope. | Default-on creates noisy authority and scope creep. |
| Q-F1 | Dedicated `RENAME_LOG.md`, optional summary for commit body. | Rename decisions are spec history, not only git history. Non-git users still need the audit trail. | Commit-only loses context and breaks outside git. |
| Q-F2 | Case-sensitive by default, explicit `-i`. | Identifier casing is often meaningful. Broad matching should be deliberate. | Default-insensitive can rename registers, signals, and acronyms incorrectly. |
| Q-G1 | Fixed four sections, with optional subsections. | Handoff value comes from predictability. Optional subsections preserve local flexibility. | Fully modular templates drift back to current inconsistency. |
| Q-H1 | Accept `PLAN.md`, `/spec-init` transcript, or explicit “no written plan” declaration. | The gate needs evidence, not one artifact shape. Missing plan must be visible. | A single canonical artifact blocks brownfield and ad hoc specs. |
| Q-H2 | Changed sections since previous gate, plus spot-check summary. | Full reread is expensive and will be skipped. Deltas match the failure mode. | Too narrow misses old unrequested clauses. |
| Q-K1 | WARN. | Style lint is subjective. WARN preserves signal without blocking real spec progress. | FAIL causes waiver fatigue and style-lawyering. |
| Q-L1 | Command first, hook example later. | The plugin already exposes workflow as commands. Hooks add environment coupling. | Hook-first makes adoption brittle. |
| Q-L2 | Report-only. | Review agents produce findings for human judgment. Blocking needs a stable severity model. | Blocking on false positives stalls commits. |

## Section 4. Stress-test axes

| axis | findings |
|---|---|
| Generality | A, B, E, F, G, H, K, and L are broadly generic. I and J are explicitly BFM-scoped. C has one generic half and one BFM-shaped half. |
| Edge cases | `/spec-rename` must classify register names, signal names, headings, body text, and citations before apply. `/spec-impact` should reject vague input or ask for sharper nouns. `D1.delta_from_plan` should require a “no written plan” declaration and user confirmation. |
| Discarded items | LINT-008 is salvageable as user-supplied terminology lint. Stage-gate mapping is salvageable as advisory coverage metadata. Mirror sync should stay discarded. |
| Missing items | The plan does not address source-of-truth drift between generated specs and external design docs. It does not address command output determinism. It does not address stale waivers after anchor retirement. |
| Scope creep | D risks becoming a vendor-attribution parser. H risks turning every D1 gate into a broad design audit. L risks becoming CI policy rather than review assistance. |
| Sequencing | Stage 1 is mostly right. D should stay WARN-only and may need more design before P0. E should probably precede F if rename wants impact preview. K could promote before L. |

## Section 5. Missing-observation candidates

| candidate-id | failure mode | why generic | suggested mechanism |
|---|---|---|---|
| M1 | Waiver cites retired or changed gate anchor. | Any spec with waivers can drift. | Lint stale `WAIVERS.md` anchors against current `stage_gates.md`. |
| M2 | Interface table and register field prose disagree on reset, access, or polarity. | Applies to controllers, PHYs, macros, BFMs. | Cross-file consistency lint for named signals, fields, reset values. |
| M3 | Requirement uses “shall” or “must” but has no DV testpoint. | Any hardware spec needs verification traceability. | Lint normative clauses without TP or rule linkage. |
| M4 | Parameter default appears in multiple files with divergent values. | Generic across configurable IP. | `/spec-stats` plus lint for parameter default single-source consistency. |
| M5 | External citation link or file reference becomes unreachable. | Any source-backed spec can rot. | Citation inventory with existence check and unresolved-reference WARN. |

## Section 6. Overall verdict

Sign off Stage 1, but split D if noisy. Main concern: attribution lint precision. Confidence: medium.

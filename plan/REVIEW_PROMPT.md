# Cross-AI Review Prompt — `plan/PLUGIN_IMPROVEMENT_PLAN.md`

This prompt is fed to a fresh-context AI reviewer (Codex / GPT-5.5, Gemini, or another Claude session) to produce an independent review of `plan/PLUGIN_IMPROVEMENT_PLAN.md`.

The reviewer has not seen the dogfood logs (`dogfood/noc-sim-ni-bfm/*`) or the conversation that produced the plan. Its job is to challenge the plan from a fresh-eyes vantage point.

---

## Prompt to feed the reviewer

```
You are a senior hardware-spec tooling reviewer. You have not seen the prior
conversation or the dogfood logs that produced this plan. Your job is to review
`plan/PLUGIN_IMPROVEMENT_PLAN.md` from a fresh-context vantage point and
challenge it where appropriate.

Repository context (read-only):
  - Plan under review:   plan/PLUGIN_IMPROVEMENT_PLAN.md
  - Repo CLAUDE.md:      CLAUDE.md  (working protocol + spec writing style)
  - Predecessor plan:    plan/BFM_MODE_DESIGN.md  (for tone reference)
  - Backflow source:     plan/DOGFOOD_OBSERVATIONS_A1-A6.md  (the observation list this plan filters from)

Read all four files first. Then produce a review under exactly the six
sections below, in this order. Use Markdown. Be terse. One idea per
sentence. Tables for parallel data.

------------------------------------------------------------
## Section 1. Generality assessment

The plan §1 states a five-guard filter (no IP-class assumption / no vendor
assumption / no project-specific defaults / worked-example regression /
project-specific cleanup goes to /spec-rename + /spec-impact).

For each of the 12 retained items in §3 (A through L), evaluate:

| item-id | passes 5 guards? | weakest evidence | counterexample IP class |

Counterexample format: pick an IP class outside noc-sim's protocol-bfm
scope (I2C controller, eFuse macro, AES core, USB PHY, SDIO host, ADC
front-end, etc.) and state whether the item still applies cleanly. If
it does not apply cleanly, name the specific guard that fails.

------------------------------------------------------------
## Section 2. Discarded-items reasoning audit

The plan §2 discards three candidates: LINT-008 vocab residue, stage-gate
↔ protocol-rule mapping, mirror sync. For each:

| candidate | discard reason sound? | alternative framing that saves it? |

If you can propose an alternative framing that resolves the original
discard concern, name the framing and estimate the residual risk.

------------------------------------------------------------
## Section 3. Open questions

The plan §7 lists nine open questions Q-E1 through Q-L2. For each, take a
position:

| Q-id | recommended answer | rationale (≤2 sentences) | risk if chosen wrong |

------------------------------------------------------------
## Section 4. Stress-test axes

§7 invites stress-testing on six axes. Run each axis against §3 items
and report findings:

- Generality: counterexample IP classes (see §1 above; cross-reference)
- Edge cases: what does /spec-rename do when <old> overlaps a register
  name? What does /spec-impact do when the change description is vague?
  What does D1.delta_from_plan do when no written plan exists?
- Discarded items: rebuttal opportunities (cross-reference §2 above)
- Missing items: failure modes the plan does not address
- Scope creep: does any retained item drift toward project-specific
  assumption under closer reading?
- Sequencing: is P0 → P1 → P2 ordering correct? Any item that should
  promote or demote?

------------------------------------------------------------
## Section 5. Missing-observation candidates

List up to five candidate observations from typical hardware spec
authoring (in any mode or IP class) that this plan does not address but
would be worth considering for v0.6.0+. Cite concrete failure modes, not
abstract concerns.

Format:
| candidate-id | failure mode | why generic | suggested mechanism |

------------------------------------------------------------
## Section 6. Overall verdict

One paragraph (≤120 words). Would you sign off on Stage 1 (P0 bundle:
A + B + C + D) as proposed in §4? State the single most important
concern. State your overall confidence (low / medium / high).
```

---

## How to run

### Codex CLI (GPT-5.5 via ChatGPT subscription)

```powershell
codex exec "$(Get-Content -Raw plan/REVIEW_PROMPT.md)" > plan/REVIEW_CODEX.md
```

Or feed via stdin:

```powershell
Get-Content -Raw plan/REVIEW_PROMPT.md | codex exec > plan/REVIEW_CODEX.md
```

Default model in `~/.codex/config.toml` is now `gpt-5.5` (ChatGPT-subscription-compatible). No `-m` flag needed.

### Claude (this session)

In Claude Code, ask Claude to read this prompt file and the plan, then produce a review with the same six sections. Output as `plan/REVIEW_CLAUDE.md`.

### Aggregation

After both reviews land:

1. Diff the two reviews section by section.
2. Identify points of disagreement (one reviewer says X, the other says Y).
3. Identify points of consensus that the plan author should treat as high-confidence findings.
4. Identify findings unique to one reviewer that warrant investigation.
5. Record outcome in `plan/REVIEW_AGGREGATE.md`.

---

## Prompt design notes

- The reviewer is told it has not seen the dogfood logs. This is the
  fresh-context premise — the plan must stand on its own.
- The six sections force structure. Without structure, reviewers tend to
  produce loose prose that is hard to diff across AIs.
- Section 1 (Generality) is the load-bearing section since the plan's
  central claim is generality. The counterexample IP-class column makes
  the reviewer commit to a concrete failure (or admit none exists).
- Section 5 (Missing items) is the open-ended exploration channel. It
  surfaces anything the structured sections miss.
- The verdict in Section 6 is bounded to one paragraph to prevent
  rationalization sprawl.

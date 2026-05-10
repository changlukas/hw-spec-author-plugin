# Process: Implementer Review

The implementer review is a multi-agent cross-check that surfaces ambiguities affecting wire-level equivalence between paradigm-different implementations of the same spec.

This complements `/spec-review` (the reader test). Reader test asks: "Is this answerable from spec text?" Implementer review asks: "Would two paradigm-different implementations produce identical wire behavior?"

## When this protocol applies (v1)

Mandatory at D1 sign-off when:

- `MODE.md` declares `mode: protocol-bfm`
- AND `MODE.md` declares `has-rtl-counterpart: yes`

Other modes are deferred:

- `mode: behavioral-block`: reader test alone is sufficient at v1 (behavioral-block has a single implementer paradigm, RTL — no paradigm pair to compare).
- `mode: protocol-bfm` with `has-rtl-counterpart: no`: single-paradigm BFM has no paired counterpart at v1; a single-paradigm "self-review" variant is deferred to v2.

## Workflow

### Round 1 — independent first read (parallel)

For each paradigm in the review (default 2; user-overridable via `--paradigms`):

1. Spawn one `implementer-reviewer` subagent.
2. Pass it: spec path, paradigm preset (persona + reading list + concerns), Round 1 prompt template, output format.
3. Subagent reads the spec independently and produces a plan + ambiguity list.

Round 1 agents run in parallel, isolated from each other. No agent sees another's output during Round 1.

### Round 2 — cross-paradigm peer review (parallel)

For each paradigm in the review:

1. Spawn one `implementer-reviewer` subagent.
2. Pass it: spec path (for spot-checks), paradigm preset, Round 2 prompt template, **all other paradigms' Round 1 outputs concatenated**.
3. Subagent reads the other(s)' plans, spot-checks key citations against actual spec text, produces peer review.

Round 2 agents run in parallel. Each agent reads all OTHER agents' Round 1 outputs in one shot. No iterative dialogue at v1.

### Synthesis (main context, no agent)

After all Round 2 outputs return, the main slash-command context aggregates:

1. **Converged ambiguity list**, ranked by:
   - Cross-paradigm appearance count (Rank 1 = both/all flagged)
   - Bit-level impact (does it affect wire behavior, or only internal modeling?)
2. **NEW DISCOVERIES** — ambiguities surfaced only in Round 2 (one paradigm reviewing another's plan). Typically the highest-value findings.
3. **DISAGREEMENTS** — places where Round 2 reviewers proposed different resolutions for the same ambiguity. Flag for designer ruling.
4. **SPEC CONTRADICTIONS** — places where two spec files state different things. Both reviewers should have caught these.
5. Write `<spec_root>/IMPLEMENTER_REVIEW_LOG.md` (format below).

### Re-run protocol

After spec edits resolving surfaced ambiguities, re-run `/spec-implementer-review` to confirm:

- Resolved ambiguities no longer appear.
- No new ambiguities surfaced from the edit (regression detection).

The new run replaces the previous `IMPLEMENTER_REVIEW_LOG.md`. Run history lives in git log.

## Paradigm presets

The slash command consumes these presets when rendering prompts.

### `rtl` — synthesizable SystemVerilog RTL

- **Persona**: senior RTL designer building synthesizable SystemVerilog for the spec block.
- **Reading focus**: `theory_of_operation.md` §RTL internal architecture and §RTL-vs-BFM behavioral equivalence, `signal_interface.md`, `pin_level_reset.md`, `protocol_rules.md`, `channel_handshake.md`, `registers.md`, `dv/plan.md`. Skip `transaction_api.md`, `channel_api.md`, `active_passive_mode.md` — BFM-internal API surfaces, not part of the RTL contract.
- **Concerns to emphasize**: pipeline / timing, FSM register state, CDC mechanism choice, RAM macro decisions, reset values per pin, parameter defaults at synthesis time.

### `c-bfm` — C++ / SystemC behavior model BFM

- **Persona**: senior verification engineer building a C++ / SystemC BFM for the spec block.
- **Reading focus**: all spec files. Pay special attention to `theory_of_operation.md` §BFM internal architecture, §RTL-vs-BFM behavioral equivalence, §Implementation-specific algorithms.
- **Concerns to emphasize**: clock-domain scheduler, data-structure choice, API surface, observation lists, test-only knob mechanics, RTL behavioral-equivalence boundaries.

### `uvm-bfm` — SystemVerilog UVM BFM

- **Persona**: senior UVM verification engineer building a SystemVerilog UVM BFM for the spec block.
- **Reading focus**: same as `c-bfm` plus `dv/plan.md` Coverage model section (UVM-specific covergroup mapping).
- **Concerns to emphasize**: UVM phase mapping, sequencer-driver-monitor split, factory overrides, configuration database access, RAL register model integration.

### `systemc-tlm` — SystemC TLM-2.0 BFM

- **Persona**: senior SystemC TLM-2.0 modeler building a TLM model for the spec block.
- **Reading focus**: same as `c-bfm` plus emphasis on transaction-level abstraction.
- **Concerns to emphasize**: TLM-2.0 socket types, blocking vs non-blocking transport, timing annotation mode (LT vs AT), payload extension structure.

### Custom paradigm

If the user passes `--paradigms <name>` for a paradigm not above, the slash command asks for: persona description (one sentence), reading focus (which spec files / sections), concerns to emphasize (paradigm-specific). The custom paradigm is used for that one run only.

## Placeholder definitions

The slash command substitutes the following placeholders when rendering Round 1 / Round 2 prompts:

| Placeholder | Source / value |
|---|---|
| `{{PARADIGM_NAME}}` | The current paradigm being dispatched (e.g., `rtl`, `c-bfm`) |
| `{{PERSONA}}` | The paradigm preset's persona description (e.g., "RTL designer building synthesizable SystemVerilog") |
| `{{SPEC_PATH}}` | Spec root path passed to `/spec-implementer-review` |
| `{{MODE_INFO}}` | One sentence summarising MODE.md (e.g., "Mode: protocol-bfm with has-rtl-counterpart: yes") |
| `{{READING_LIST}}` | The paradigm preset's reading-focus list, formatted as numbered files |
| `{{CONCERNS}}` | The paradigm preset's "concerns to emphasize" list |
| `{{OTHER_PARADIGM_NAMES}}` | Comma-separated names of all other paradigms in the run |
| `{{OTHER_ROUND1_OUTPUTS}}` | (Round 2 only) Concatenated Round 1 outputs of all other paradigms, each prefixed by a `## Round 1 from <name>` header |
| `{{ISO_DATE}}` | (Output log only) ISO-8601 timestamp of the run |
| `{{PLUGIN_VERSION}}` | (Output log only) Plugin version from `plugin.json` |

## Round 1 prompt template

The slash command renders this template per paradigm before dispatch.

```
You are a senior {{PERSONA}}. Spec at `{{SPEC_PATH}}`. {{MODE_INFO}}. The contract is wire-level behavioral equivalence with the parallel implementations: {{OTHER_PARADIGM_NAMES}}.

You have NOT seen this spec before. Read it independently.

**Files to read (in this order)**:
{{READING_LIST}}

**Concerns to emphasize**: {{CONCERNS}}

**Output format** (use these exact section headings, in this order):

# {{PARADIGM_NAME}} implementation plan

## Block summary (my understanding)
[2-3 sentences]

## Top-level architecture
[ASCII hierarchy of modules / classes; clock-domain attribution per component]

## Key implementation decisions (with spec citations)
[5-10 bulleted decisions, each: Decision / Why with spec citation]

## Spec ambiguities I hit
[Numbered. Per item: Question / Where (file:section, verbatim quote when load-bearing) / Why it matters for my implementation / My tentative interpretation]

## Wire-level behaviors I want locked-down with the {{OTHER_PARADIGM_NAMES}}
[5-10 bullets, each a specific bit-equivalence concern]

## My confidence level
[1 paragraph. Where confident, where guessing, what single piece of info would most reduce uncertainty]

**Constraints**:
- Cite file:section. Quote verbatim when load-bearing.
- ~2000 words max.
- Don't write implementation files (no .cpp / .sv).
```

## Round 2 prompt template

```
You are a senior {{PERSONA}}. Spec at `{{SPEC_PATH}}`. The other implementer(s) have just finished Round 1. Their plan(s) are below. **Your job is to peer-review them.**

Spot-check their spec citations where they claim spec text says something specific. Identify points of agreement, disagreement, missed ambiguities, and spec contradictions they may have masked.

---

## Round 1 output(s) from other paradigm(s)

{{OTHER_ROUND1_OUTPUTS}}

---

**Output format** (use these exact section headings, in this order):

# Round 2 — {{PARADIGM_NAME}} perspective on {{OTHER_PARADIGM_NAMES}}'s Round 1

## Points of agreement (high confidence cross-paradigm)
[3-7 bullets]

## Points of disagreement (with my proposed resolution)
[Where I think they're wrong + reasoning + spec citation. If none, say so explicitly.]

## Ambiguities they missed (from {{PARADIGM_NAME}} perspective)
[Up to 5 ambiguities I see that they did not list]

## Spec citations spot-checked
[Pick 3-5 of their cited sections, read those passages, confirm or correct]

## Converged top-N ambiguity list (across both Round 1s)
[Ranked list of highest-priority ambiguities]

## Cross-team meeting agenda I'd propose
[3-5 items the teams must resolve in a synchronous meeting]

**Constraints**:
- ~1500 words max.
- Cite spec file:section when claiming "they missed X" or "their citation is wrong".
- Spot-check load-bearing parts only; don't re-read whole spec.
- Don't write implementation files.
```

## IMPLEMENTER_REVIEW_LOG.md output format

```
# {{spec_name}} — Implementer Review Log

This log substantiates the `D1.cross.implementer_review` claim for `{{spec_path}}`. Per `stage_gates.md`, a `protocol-bfm + has-rtl-counterpart=yes` spec at D1 must have run an implementer review with at least 2 paradigm-paired reviewers; the converged ambiguity list must be either resolved in spec or recorded in `WAIVERS.md`.

**Run timestamp**: {{ISO_DATE}}
**Plugin version**: {{PLUGIN_VERSION}}

## Paradigms

- **{{paradigm_1}}**: {{persona_summary}}
- **{{paradigm_2}}**: {{persona_summary}}

## Round 1 — independent reads

### {{paradigm_1}}

[Round 1 output, full text]

### {{paradigm_2}}

[Round 1 output, full text]

## Round 2 — cross-paradigm peer review

### {{paradigm_1}} reviewing {{paradigm_2}}

[Round 2 output, full text]

### {{paradigm_2}} reviewing {{paradigm_1}}

[Round 2 output, full text]

## Synthesis

### Converged ambiguity list (ranked)

| Rank | Issue | Both flagged? | Affected wire/cycle behavior | Proposed resolution |
|---|---|---|---|---|
| 1 | ... | yes | ... | ... |

### NEW DISCOVERIES (Round 2 only)

[Ambiguities surfaced by cross-review that neither Round 1 found]

### DISAGREEMENTS (need designer ruling)

[Places where Round 2 reviewers proposed different resolutions]

### SPEC CONTRADICTIONS

[Places where two spec files state different things — typically 1-line erratum]

## Verdict

- `D1.cross.implementer_review`: ✓ ran successfully | ✗ failed (reason)
- Resolutions required before D2:
  - {{count}} items flagged for designer ruling
  - {{count}} items flagged as 1-line spec erratum
  - {{count}} items deferred to `WAIVERS.md` (with rationale)
```

## Constraints (cross-cutting)

- **Subagent isolation is the test.** If `implementer-reviewer` is unavailable, `/spec-implementer-review` must stop. Do not silently fall back to main-context inference — that defeats paradigm isolation.
- **N=2 is the v1 default.** N≥3 inflates cost (2N agent dispatches) and synthesis complexity. v2 will support N≥3 with cost-aware throttling.
- **Cost expectation**: per run = 2N agent dispatches × full-spec context. Plan for ~5–10 min wall, ~4× normal session token cost at N=2.
- **Don't auto-fix ambiguities.** The slash command's job is detection + synthesis. Spec edits resolving ambiguities are the user's call (manual, or a future `/spec-implementer-fix` command).

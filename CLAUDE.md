# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repo type

This repo is a **Claude Code plugin marketplace** — pure Markdown + JSON, no build, no test, no lint, no runtime code. The "source" is prompts: skills, subagents, slash commands, templates, and three worked examples. There is nothing to compile or execute. Editing is the workflow; review is by reading.

The marketplace currently ships one plugin: `hw-spec-author` (v0.3.0) under `plugins/hw-spec-author/`.

## Where everything lives

```
.claude-plugin/marketplace.json          # marketplace manifest (lists plugins)
plan/
└── BFM_MODE_DESIGN.md                   # protocol-bfm mode design history (v0.3.0); §13 sketches v0.4.0 Protocol Library
plugins/hw-spec-author/
├── .claude-plugin/plugin.json           # plugin manifest
├── skills/hw-spec-author/
│   ├── SKILL.md                         # skill entry point — workflow logic, mode-aware
│   └── references/
│       ├── templates/                   # behavioral-block mode templates
│       │   ├── 01..06_*.md
│       │   └── bfm/                     # protocol-bfm mode templates (NEW in v0.3.0)
│       │       ├── 02_theory_of_operation.md           # two-section: BFM internal + RTL internal
│       │       ├── 02b_protocol_rules.md
│       │       ├── 02c_channel_handshake.md
│       │       ├── 02d_pin_level_reset.md
│       │       ├── 03b_transaction_api.md
│       │       ├── 03c_channel_api.md
│       │       ├── 03d_active_passive_mode.md
│       │       └── 04b_signal_interface.md
│       └── process/
│           ├── stage_gates.md                          # D0→D3 + mode-conditional D1.bfm.* / D2.bfm.* / D3.bfm.*
│           ├── reader_test.md                          # behavioral-block reader-test bank
│           ├── bfm_reader_test_bank.md                 # protocol-bfm reader-test bank (NEW in v0.3.0)
│           ├── writing_principles.md
│           └── rtl_extraction.md                       # /spec-import RTL extraction (behavioral-block only)
├── agents/spec-reader.md                # isolated-context reader subagent
├── commands/spec-*.md                   # 7 slash commands (all mode-aware in v0.3.0)
└── examples/
    ├── wctmr/                           # behavioral-block worked example (64-bit timer)
    ├── axi_lite_slave_bfm/              # protocol-bfm with has-rtl-counterpart: yes (NEW in v0.3.0)
    └── apb_slave_bfm/                   # protocol-bfm portability test on channel-less protocol (NEW in v0.3.0)
```

The slash commands reference skill assets via `${CLAUDE_PLUGIN_ROOT}/skills/hw-spec-author/...` — these paths are part of the contract. If you move a template or process doc, update every command frontmatter that loads it.

## Big-picture architecture

The plugin enforces a 4-phase workflow that produces hardware specs in one of two modes:

- **`behavioral-block`** mode (default): 6-file OpenTitan-Comportability layout — for IP blocks with internal logic and software-visible registers.
- **`protocol-bfm`** mode: 9-11-file layout with cycle-level protocol rigor — for blocks with wire-level handshake protocol contracts (AXI, AHB, APB, TileLink, NoC, custom). Used for both pure verification BFMs and RTL+C-model paired implementations.

| Phase | Trigger command | What it does |
|---|---|---|
| 1. Capture (D0) | `/spec-init` (greenfield, both modes) or `/spec-import` (brownfield, behavioral-block only) | Asks IP class first; writes `MODE.md`; generates mode-appropriate skeleton |
| 2. Iterate (D0→D1) | natural conversation; `/spec-status`, `/spec-lint` cross-cutting | Mode-specific writing order; mode-conditional checklist scoring |
| 3. Reader Test | `/spec-review` | Spawns `spec-reader` subagent; question bank dispatched by `MODE.md` |
| 4. Stage Gate | `/spec-gate <D1\|D2\|D3>` | Walks mode-conditional checklist in `stage_gates.md`; N/A items skipped; persistent waivers in `WAIVERS.md` |

### Mode infrastructure (load-bearing in v0.3.0)

Every spec has a `MODE.md` at its root:

```markdown
# Spec Mode

mode: <behavioral-block | protocol-bfm>
created: <ISO date>
spec-author-plugin-version: <version>
has-rtl-counterpart: <yes | no>     # protocol-bfm mode only
```

All commands read `MODE.md` to dispatch behavior:
- `/spec-status` walks mode-appropriate checklist; non-applicable items scored N/A
- `/spec-lint` runs LINT-001..007 always + LINT-BFM-001..005 in BFM mode
- `/spec-review` dispatches `reader_test.md` or `bfm_reader_test_bank.md` based on mode
- `/spec-gate` evaluates mode-conditional `**Applies when:**` annotations in `stage_gates.md`
- `/spec-help` shows mode-aware workflow card

### Load-bearing concepts to preserve when editing

- **Stage-gate anchors** `(id: D1.too.fsm)`, `(id: D1.bfm.protocol_rules)` etc. in `references/process/stage_gates.md` are a *stable contract* with `WAIVERS.md` files in user spec directories. Never rename or paraphrase an anchor; if an item changes meaning, retire the old anchor and add a new one. `/spec-gate` parses these literally.
- **Subagent isolation.** `agents/spec-reader.md` declares `tools: Read, Grep, Glob` in its frontmatter — no Write, no Bash, no web. `/spec-review` delegates via the Task tool and must not answer questions itself; that's the entire point of the test.
- **Single-source-of-truth for output structure.** Both 6-file (behavioral-block) and 9-11-file (protocol-bfm) layouts are duplicated across SKILL.md, READMEs (English + zh-TW), every relevant command, and the worked examples. Changes here ripple — if you reorganize the output structure, search the repo for the file list and update everywhere.
- **Source-files-are-framework-only.** Plugin source artifacts (templates, command specs, process docs) carry framework only — required structure, writing rules, anti-patterns, mini-examples. Rationale and "why we chose X" goes in `plan/*.md` design docs and READMEs, never embedded in source files. (See `memory/feedback_source_vs_readme.md`.)
- **BFM-mode template `02_theory_of_operation.md` two-section structure.** When `has-rtl-counterpart: yes`, the BFM-mode ToO must contain both `## BFM internal architecture` (always) and `## RTL internal architecture` (conditional) sections. The RTL-vs-BFM behavioral equivalence sub-section is the load-bearing contract documenting which BFM features are test-only vs which behaviors the RTL counterpart implements identically.

## Common commands (no build system; just git + manual review)

There is no `npm test`, no linter, no formatter. Plugin "testing" is dogfooding:

- **Validate the manifests** — `marketplace.json` and `plugins/hw-spec-author/.claude-plugin/plugin.json` must agree on plugin name, version, and keyword set. Bump both together. Currently at v0.3.0.
- **Validate worked examples still pass** — when editing `references/process/*.md` or any `examples/<name>/` file, re-run `/spec-review examples/<name>/`:
  - `examples/wctmr/` → 10/10 PASS expected (per `READER_TEST_LOG.md`)
  - `examples/axi_lite_slave_bfm/` → 10/10 PASS expected (per `READER_TEST_LOG.md`)
  - `examples/apb_slave_bfm/` → portability test; no formal reader test log, but should pass `/spec-status` to D1 candidate
- **Check command frontmatter `allowed-tools`** — each command's tool list is the security boundary. Don't widen casually (e.g., adding `Bash` to `/spec-status` would defeat its read-only contract).
- **Cross-reference check** — `/spec-lint` checks references inside user specs (LINT-001 cross-refs, LINT-BFM-004 channel cross-refs, etc.), but the *plugin's own* docs (SKILL.md, READMEs, commands) also cross-reference each other. Run `git grep` for moved file paths after any reorganization.
- **Mode-conditional consistency** — when adding a new BFM template file or a new `D1.bfm.*` anchor, also update SKILL.md template tables, spec-init.md generation logic, spec-status.md / spec-gate.md / spec-lint.md mode-aware steps, and both READMEs.

## Working protocol — mechanical verification over judgment

Recurring failure mode in this repo (observed and corrected across the noc-sim NI BFM dogfood waves A1–A4): I add unrequested complexity, miscount, or assert "all X cleaned up" without verifying. Pure-prompt judgment is unreliable for these. **Push every verifiable claim to a tool.** This is the highest-leverage rule in this file.

### Before any edit (delta-from-plan rule)

Before touching files, write a bulleted list of every detail I will add that is NOT in the user's explicit approval. Empty list → proceed. Non-empty list → stop and ask per-item approval. Do not batch-approve.

Examples of "details that count" as unrequested additions and require explicit approval:

- Reserved-encoding forward-compat clauses (e.g., "encoding 3 is Reserved; BFM accepts as Hybrid").
- Debug-only acceptance clauses (e.g., "mode=2 SHOULD be avoided in production but no SLVERR").
- CDC mechanism choices (e.g., "req/ack handshake instead of 2FF").
- Splitting one register into CTRL/STATUS pair where the user wrote one.
- Severity bumps on pre-existing rules (RECOMMEND→FAIL or vice versa).
- New race-semantic rules / liveness rules / accuracy contracts.
- Backwards-compatibility shims, deprecation aliases, parameter range expansion.

A4's VC_ARB_MODE is the canonical violation: the plan said "runtime selection of arbitration policy" (one line), I shipped CDC handshake + Reserved=3 forward-compat + debug-only acceptance + dual-arbiter binding (4 unrequested mechanisms), required mid-wave revert.

### Default to minimum sufficient design

When in doubt about adding a clause / mechanism / forward-compat / handling-the-edge-case, DON'T. Ship the smaller version. Review-then-extend is cheaper than ship-then-revert. If the user wants the addition, they will pull. Do not push.

### Single-file batching

At most 2 file edits per response unless the user explicitly asks for more. After each batch:

- Sync to dogfood mirror if applicable.
- Run `/spec-lint` (or relevant mechanical check).
- Surface the diff or the lint result.
- Then stop and wait.

This catches contradictions at one round-trip cost, vs. cascading errors across 6 files.

### Numeric and completeness claims must cite tool output

Never write a numeric claim or "all X done" claim from memory. Always cite a canonical command and quote its output literally. If tool output contradicts my mental model, trust the tool and update the claim.

| Claim type | Wrong (prompt-only) | Right (tool-backed) |
|---|---|---|
| Register count | "6 new registers" | `grep '^\| 0x13' registers.md \| wc -l` → 6 |
| Rule count | "~135 rules" | `grep -c '^\| \(NI\|AXI4\|NOC\|AXI4LITE\)_' protocol_rules.md` → 141 |
| Cleanup completeness | "all VC_ARB_MODE removed" | `grep -r 'VC_ARB_MODE' spec/` → 0 hits |
| Cross-ref target exists | "section exists in target file" | `grep -i '^####* VC scheduling' theory_of_operation.md` → match found |
| Inheriting a prior count | "per A3 claim ~135" | re-verify with canonical grep before re-citing; A3 was wrong by 6 |
| Markers / TODOs cleared | "no markers in spec" | `grep -rn 'Reviewer assumption\|TODO(designer)' doc/ dv/` → 0 hits |

### Quote, do not paraphrase

When claiming "the spec says X", cite as `<file>:<line> — "<verbatim quote>"`. Paraphrase loses precision and is the source of self-contradictions. A4 produced two through paraphrase: (a) `EXCLUSIVE_MONITOR_CTRL` access "WO" but text said "reads return 0" (strict WO conventionally means reads error), (b) CDC rule "FIFO or 2FF" but new VC_ARB_MODE rule used "req/ack handshake" — neither side referenced the other verbatim.

### Why these rules live in CLAUDE.md

CLAUDE.md is auto-loaded into every session at start and stays in context throughout. Rules written here propagate to every future session without re-training. Memory feedback files (`memory/feedback_*.md`) supplement for narrow patterns; CLAUDE.md carries the cross-cutting working-protocol rules that apply to every wave of every project in this repo.

If a rule above turns out to be wrong or counter-productive in practice, edit it here. The goal is mechanical reliability, not rigid compliance.

## Spec writing style — sound like an IP vendor, not AI

The user has flagged that my spec prose is often hard to parse and "feels AI-generated". The reviewer is the user. If they can't follow the text, they can't sign off — content correctness alone is insufficient. **Match the style of AMD pg313, ARM AMBA AXI, and Intel architecture references**: terse, direct, structurally heavy, one idea per sentence.

This section applies to:

- Spec content in `dogfood/<name>/` and any user-spec generated by the plugin (`registers.md`, `theory_of_operation.md`, `protocol_rules.md`, `dv/plan.md`, etc.).
- Plugin source artifacts (templates, command frontmatter, SKILL.md, READMEs) — same style, for consistency.

This section does NOT apply to chat-style conversational responses or `NEXT_SESSION_*.md` handoff notes (those can be more discursive).

### Sentence-level rules

- **One idea per sentence.** 15–25 words typical. Sentences over 35 words almost always need splitting.
- **Concrete subject + active verb + direct object.** "The NMU drops the flit." Not "The flit is dropped" or "Flit dropping is performed by the NMU".
- **Name actual components and signals.** "NMU's RoB allocator" not "the implementation". `axi_awready_o` not "the ready signal".
- **Replace pronouns with named entities** when ambiguous. "It / this / these" without an unambiguous antecedent within ~8 words → name the thing.

### Banned AI tells

These verbs and modifiers add no information and pattern-match as machine-generated. Replace with the real verb or omit.

- **Empty verbs**: leverages, ensures, facilitates, enables, provides, implements (when it just means "does"), utilises, allows, supports. Use the actual verb — `checks`, `drives`, `asserts`, `drops`, `computes`, `routes`, `forwards`.
- **Empty modifiers**: robust, scalable, comprehensive, holistic, seamless, efficient, optimal — claims without measurable content. Either give the number or omit.
- **Throat-clearing**: "It should be noted that", "Importantly,", "Critically,", "It is worth mentioning". Just say the thing.
- **Hedges**: "may potentially", "could possibly", "tends to", "generally speaking". One hedge per sentence max. Prefer "may" / "can" / "typically" alone.
- **Nominalisation**: "performs the calculation of" → `calculates`; "is responsible for processing" → `processes`; "the implementation of the algorithm" → "the algorithm".
- **Filler phrases**: "in order to" → "to"; "due to the fact that" → "because"; "at this point in time" → "now"; "for the purpose of" → "for".
- **Tricolons (rule of three)**: "robust, scalable, and efficient" — pick one or omit. Three-item lists belong in tables, not adjective stacks.

### Structural rules

- **Lead with the entity, not the precondition.** Wrong: "When `NUM_VC > 1`, the NMU has per-VC FIFOs that..." Right: "NMU has `NUM_VC` per-VC injection FIFOs (active when `NUM_VC > 1`)."
- **Tables for 3+ parallel items.** Three modes? Three error classes? Three reset domains? Use a table, not a bullet list of compound sentences.
- **Numbered procedures for ordered steps.** "Software writes 1, then polls until idle, then..." → numbered 1./2./3. list.
- **`Note:` callouts for asides.** Main text stays on the contract. Tangential context goes to a "**Note:**" line below.
- **No meta-commentary.** Don't write "as discussed in the previous section" or "this section covers". The reader sees the section heading.
- **No semicolons in narrative prose** (per `memory/feedback_no_semicolons.md`). Split into sentences or use bullets. Semicolons are tolerable inside dense table cells where space is tight.

### Concrete contrast examples

**Wrong (AI-style)**:
> Software can request NMU-side quiesce before runtime reconfiguration that requires no in-flight transactions on the NMU path (e.g., software view of routing-table reconfig coordinated through CSR).

**Right (IP-vendor style)**:
> Software requests quiesce before any reconfiguration that must observe a fully-drained NMU. The typical use case is a routing-table update coordinated through CSR.

---

**Wrong**:
> The arbiter is responsible for selecting one VC per cycle for output, ensuring fairness across request and response subsets while maintaining wormhole-lock semantics.

**Right**:
> The arbiter picks one VC per cycle. It alternates between request and response subsets. Wormhole-lock holds per `NOC_FLIT_VC_HARDLOCK`.

---

**Wrong**:
> When the `EXCLUSIVE_MONITOR_CTRL.clear_all` bit is written with a value of 1, this triggers the invalidation of all currently pending NSU Exclusive Monitor entries on the next available `aclk_i` edge.

**Right**:
> Writing 1 to `EXCLUSIVE_MONITOR_CTRL.clear_all` invalidates every pending NSU Exclusive Monitor entry on the next `aclk_i` edge.

### Reference style

When in doubt, model the prose on:

- **AMD pg313 NoC Programmer's Guide** — short paragraphs, tables-heavy, named-component subjects.
- **ARM IHI 0022 (AXI4 protocol spec)** — numbered steps, formal but readable. The rule-row format in this project's `protocol_rules.md` is descended from ARM's protocol-checker tables.
- **Intel architecture references / datasheets** — register-field tables, one-sentence behavior summaries.

The test for whether a paragraph passes: a hardware engineer who skims it once should be able to repeat back the contract. If they have to re-read, the sentence is wrong.

## When making changes

- **Editing templates** (`references/templates/` and `references/templates/bfm/`): templates encode audience-segregation rules, section structure, and lint-rule contracts. Adding a section means updating both the template *and* the matching `D1.*` anchors in `stage_gates.md` so `/spec-gate` will check it. For BFM-mode templates, also update LINT-BFM-* rules in `commands/spec-lint.md` if structural validation is needed.
- **Editing process docs** (`references/process/`): SKILL.md links to these by relative path. If renamed, update SKILL.md and every command that loads them via `${CLAUDE_PLUGIN_ROOT}/skills/hw-spec-author/references/process/`.
- **Adding a new slash command**: place under `commands/`, with frontmatter (`description`, `argument-hint`, `allowed-tools`). Update `/spec-help` so users discover it. Make it mode-aware if behavior should differ between behavioral-block and protocol-bfm.
- **Adding a new BFM template**: place under `references/templates/bfm/`. Update SKILL.md template table for protocol-bfm mode. Add corresponding stage-gate anchor `D1.bfm.<short_name>`. Update `/spec-init` BFM block to generate from the new template. Update both READMEs.
- **Adding a new mode** (beyond behavioral-block / protocol-bfm — e.g., a hypothetical `tlm-at` mode in v0.5.0): see `plan/BFM_MODE_DESIGN.md` for the established mode-infrastructure pattern. Adding a mode is a substantial cross-cutting change touching SKILL.md, every command, stage_gates.md, spec-lint.md, and both READMEs.
- **Versioning**: this is a published Claude Code plugin. Bump `version` in both manifests (`marketplace.json` and `plugin.json`) on user-visible changes. Update both READMEs (`README.md` and `README.zh-TW.md`) — they are kept in lockstep as bilingual documentation, not independent.

## v0.3.0 → v0.4.0 planned direction

Per `plan/BFM_MODE_DESIGN.md` §13: **Protocol Reference Library**. Move standard protocol rules (AXI4 / AXI4-Lite / APB / TileLink / CHI) into verified built-in library entries under `references/protocols/`. BFM specs become thin wrappers that inherit standard rules and only describe block-specific extensions. Eliminates per-spec drift, amortises verification burden. Not in v0.3.0 scope — full design sketch in §13 of the design draft.

## Output language convention (for generated specs, not for this repo)

Specs produced by the plugin default to **English** (industry convention: OpenTitan / ARM AMBA / RISC-V). The conversation around the spec can be any language; only the produced files default to English. This is documented in SKILL.md §"Output language" and the user can override per-spec.

## License

Apache 2.0.

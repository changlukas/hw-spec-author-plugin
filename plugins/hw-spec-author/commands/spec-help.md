---
description: Show the hw-spec-author workflow overview, available commands, and recommended next action based on current state.
argument-hint: ""
allowed-tools: Read, Glob
---

You are showing a **quick reference** for the hw-spec-author skill.

## Steps

1. **Locate the spec** (if any). Same logic as `/spec-status`: try the user-given path, else CWD candidates, else "no spec found yet."

2. **Output a structured help card**:

```
hw-spec-author — Hardware Design Spec Author

Workflow (4 phases, gated D0 → D3):

  Phase 1 — Capture (D0)
    Goal: skeleton with all 6 files, accurate placeholders.
    Greenfield: /spec-init <ip_name>          (interview-driven)
    Brownfield: /spec-import <input_path>     (extract from existing doc / RTL / hjson)

  Phase 2 — Iterate by section (D0 → D1)
    Goal: every section reaches D1 completeness.
    Order: README → interfaces → registers → theory_of_operation → programmers_guide → dv/plan
    Trigger: natural conversation (just describe what to fill in)

  Phase 3 — Reader Test (D1 sign-off)
    Goal: catch ambiguity before implementation starts.
    Trigger: /spec-review

  Phase 4 — Stage Gate (D1 → D2 → D3)
    Goal: verify checklist completeness at each transition.
    Trigger: /spec-gate D1 | /spec-gate D2 | /spec-gate D3

  Cross-cutting (any phase)
    Goal: catch mechanical drift — broken xrefs, untracked TODOs,
          Feature ↔ testpoint gaps, register / port casing.
    Trigger: /spec-lint
    Run periodically while iterating in Phase 2; required-clean before /spec-gate D1.


Available commands:

  /spec-init <ip_name> [output_dir]
      Start a new spec from scratch. Runs the Capture interview, generates
      skeleton.

  /spec-import <input_path> [output_dir]
      Import existing material into the canonical 6-file layout.
      Inputs supported: existing markdown spec, SystemVerilog/Verilog RTL,
      OpenTitan-style hjson register defs, or any combination.
      Produces IMPORT_REPORT.md with provenance and conflict log.

  /spec-status [path]
      Inspect existing spec. Reports current D-stage and what's missing.

  /spec-review [path]
      Run reader test (via spec-reader subagent for true isolation).
      8–12 ambiguity-targeting questions, gap report.

  /spec-lint [path]
      Mechanical consistency check: cross-references, TODO format,
      Feature ↔ testpoint mapping, register casing, reset value format.

  /spec-gate <D1|D2|D3> [path]
      Check whether spec meets criteria for the target gate.
      Reads/writes WAIVERS.md for persistent waivers.

  /spec-help
      This card.
```

3. **If a spec was found in step 1**: append a "Current state" section showing:
   - Spec path
   - Estimated current stage (quick scan of README + 1-2 key files; not the full `/spec-status` analysis)
   - Suggested next command based on state:

   - **No spec, greenfield** → "Start with `/spec-init <ip_name>`."
   - **No spec, but existing doc / RTL available** → "Use `/spec-import <path>` to bootstrap from existing material; review the IMPORT_REPORT.md it produces."
   - **Skeleton only / pre-D0** → "Fill out interfaces.md and registers.md next, or run `/spec-status` for a structured progress check."
   - **Working at D0/D1** → "Run `/spec-status` to see what's still open. When all D1 items look ready, run `/spec-review` then `/spec-gate D1`."
   - **At D1, no review yet** → "Run `/spec-review` before claiming D1."
   - **At D1, RTL exists** → "Run `/spec-gate D2` to check D2 criteria. Sync RTL learnings back into spec where needed."
   - **At D2** → "Final review by HW/DV/SW leads, then `/spec-gate D3`."

4. **If user asks a follow-up like "tell me more about D2"**, read the relevant section from `${CLAUDE_PLUGIN_ROOT}/skills/hw-spec-author/references/process/stage_gates.md` and quote the relevant criteria.

## Constraints

- Keep this short and scannable. The card is reference, not a tutorial.
- Don't run `/spec-status` automatically; the suggestion is enough.
- If the user is clearly stuck (multiple failed gates, repeated questions), proactively offer to walk them through the next blocker rather than just listing commands.

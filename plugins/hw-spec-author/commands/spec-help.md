---
description: Show the hw-spec-author workflow overview, available commands, and recommended next action based on current state. Mode-aware (reads MODE.md if a spec is found).
argument-hint: ""
allowed-tools: Read, Glob
---

You are showing a **quick reference** for the hw-spec-author skill.

## Steps

1. **Locate the spec** (if any). Same logic as `/spec-status`: try the user-given path, else CWD candidates, else "no spec found yet."

2. **Detect mode**. If a spec was found, read `<spec_root>/MODE.md` and parse the `mode:` line. If `MODE.md` is absent, default to `behavioral-block` (legacy spec). Record the mode for use in step 3.

3. **Output a structured help card**. Header always shows the mode line; the writing-order block branches by mode.

```
hw-spec-author — Hardware Design Spec Author

Mode: <behavioral-block | protocol-bfm | (no spec found — defaults to behavioral-block on init)>

Workflow (4 phases, gated D0 → D3):

  Phase 1 — Capture (D0)
    Goal: skeleton with the mode-appropriate file set, accurate placeholders.
    Greenfield: /spec-init <ip_name>           (interview-driven; asks IP class first)
    Brownfield: /spec-import <input_path>      (behavioral-block only — see Note)

  Phase 2 — Iterate by section (D0 → D1)
    Goal: every section reaches D1 completeness.
    Order (behavioral-block): README → interfaces → registers → theory_of_operation
                              → programmers_guide → dv/plan
    Order (protocol-bfm):     README → signal_interface → pin_level_reset
                              → protocol_rules → channel_handshake → transaction_api
                              → channel_api → active_passive_mode → dv/plan
    Trigger: natural conversation (just describe what to fill in)

  Phase 3 — Reader Test (D1 sign-off)
    Goal: catch ambiguity before implementation starts.
    Bank: behavioral-block → reader_test.md
          protocol-bfm     → bfm_reader_test_bank.md
    Trigger: /spec-review

  Phase 4 — Stage Gate (D1 → D2 → D3)
    Goal: verify mode-conditional checklist completeness at each transition.
    Trigger: /spec-gate D1 | /spec-gate D2 | /spec-gate D3

  Cross-cutting (any phase)
    Goal: catch mechanical drift — broken xrefs, untracked TODOs,
          Feature ↔ testpoint gaps, register / port casing,
          plus LINT-BFM-001..005 in BFM mode.
    Trigger: /spec-lint
    Run periodically while iterating in Phase 2; required-clean before /spec-gate D1.

Note on brownfield: /spec-import currently understands only behavioral-block layout.
For BFM specs, run /spec-init <name> and select protocol-bfm.


Available commands:

  /spec-init <ip_name> [output_dir]
      Start a new spec from scratch. Asks the IP-class question first
      (behavioral-block or protocol-bfm), writes MODE.md, generates the
      mode-appropriate skeleton.

  /spec-import <input_path> [output_dir]
      Import existing material into the behavioral-block 6-file layout.
      Inputs supported: existing markdown spec, SystemVerilog/Verilog RTL,
      OpenTitan-style hjson register defs, or any combination.
      Produces IMPORT_REPORT.md with provenance and conflict log.
      BFM-mode brownfield not supported (known limitation).

  /spec-status [path]
      Inspect existing spec (mode-aware). Reports current D-stage against
      the mode-appropriate checklist and what's missing.

  /spec-review [path]
      Run reader test (via spec-reader subagent for true isolation).
      8–12 ambiguity-targeting questions drawn from the mode-matched bank.

  /spec-lint [path]
      Mechanical consistency check (mode-aware). LINT-001..007 always;
      LINT-BFM-001..005 additionally in protocol-bfm mode.

  /spec-gate <D1|D2|D3> [path]
      Check whether spec meets criteria for the target gate
      (mode-conditional rows applied). Reads/writes WAIVERS.md.

  /spec-help
      This card.
```

4. **If a spec was found in step 1**: append a "Current state" section showing:
   - Spec path
   - Detected mode (from MODE.md, or "(default: behavioral-block — no MODE.md present)")
   - Estimated current stage (quick scan of README + 1-2 key files; not the full `/spec-status` analysis)
   - Suggested next command based on state and mode:

   - **No spec, greenfield** → "Start with `/spec-init <ip_name>`. The interview will ask for IP class first."
   - **No spec, but existing doc / RTL available for a behavioral block** → "Use `/spec-import <path>` to bootstrap from existing material; review the IMPORT_REPORT.md it produces."
   - **No spec, but existing BFM RTL** → "BFM brownfield import is a known limitation. Run `/spec-init <name>` and select protocol-bfm; populate signal_interface.md from your RTL ports manually."
   - **Skeleton only / pre-D0 (behavioral-block)** → "Fill out interfaces.md and registers.md next, or run `/spec-status` for a structured progress check."
   - **Skeleton only / pre-D0 (protocol-bfm)** → "Fill out signal_interface.md next — channel names there flow into protocol_rules.md and the API surfaces."
   - **Working at D0/D1** → "Run `/spec-status` to see what's still open. When all D1 items look ready, run `/spec-review` then `/spec-gate D1`."
   - **At D1, no review yet** → "Run `/spec-review` before claiming D1."
   - **At D1, RTL exists** → "Run `/spec-gate D2` to check D2 criteria. Sync RTL learnings back into spec where needed."
   - **At D2** → "Final review by HW/DV/SW leads, then `/spec-gate D3`."

5. **If user asks a follow-up like "tell me more about D2"**, read the relevant section from `${CLAUDE_PLUGIN_ROOT}/skills/hw-spec-author/references/process/stage_gates.md` and quote the relevant criteria. Note any `**Applies when:**` conditional that scopes the item to a specific mode.

## Constraints

- Keep this short and scannable. The card is reference, not a tutorial.
- Don't run `/spec-status` automatically; the suggestion is enough.
- If the user is clearly stuck (multiple failed gates, repeated questions), proactively offer to walk them through the next blocker rather than just listing commands.
- The mode line at the top is mandatory — it's the first thing the user should see, because every downstream recommendation depends on it.

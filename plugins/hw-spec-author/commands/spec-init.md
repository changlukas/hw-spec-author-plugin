---
description: Start a new hardware design spec. Creates the OpenTitan-style 6-file skeleton and begins the Capture-phase interview.
argument-hint: <ip_name> [output_dir]
allowed-tools: Read, Write, Glob, Bash
---

You are activating Phase 1 (Capture / D0) of the **hw-spec-author** skill.

## Steps

1. **Read the skill**: open `${CLAUDE_PLUGIN_ROOT}/skills/hw-spec-author/SKILL.md` to load the full workflow.

2. **Parse the user's invocation**: the command is `/spec-init <ip_name> [output_dir]`.
   - `$ARGUMENTS` contains the user's argument string.
   - First whitespace-delimited token = `ip_name`. Required. If missing, ask the user.
   - Second token (optional) = `output_dir`. If present, use it directly and skip that question in the interview. If absent, default to `./spec/<ip_name>/` and confirm with the user during the interview.

3. **Run the Capture interview**. Ask the user for:
   - One-sentence purpose of the IP
   - Bus / host interface (e.g., AXI4 slave, AXI4-Lite, APB, TileLink-UL, custom)
   - Clock and reset domains (count, async/sync, retention/AON?)
   - Top-level features (3–8 bullets, capabilities not adjectives)
   - Any prior art (existing register interface to be compatible with, similar IP's spec, draft notes)
   - Target output directory — **only ask this if `output_dir` was not given as an argument**. Default `./spec/<ip_name>/`.

   Ask in **batched form** if the user prefers. Don't drag out the interview over 6 separate messages — one consolidated question block is better.

4. **Generate the skeleton**. After answers are in, create the 6-file layout at the target directory:

```
<output_dir>/
├── README.md
├── doc/
│   ├── theory_of_operation.md
│   ├── programmers_guide.md
│   ├── interfaces.md
│   └── registers.md
└── dv/
    └── plan.md
```

For each file, populate it from the corresponding template in `${CLAUDE_PLUGIN_ROOT}/skills/hw-spec-author/references/templates/`. Read the template **before** generating its file. Fill in known information from the interview; mark unknowns as `TODO(designer): <what's missing>`.

5. **Report back**. Tell the user:
   - Where the skeleton was created
   - How many `TODO(designer):` markers remain (per file)
   - The recommended next action: typically "fill in `interfaces.md` first — it forces commitment on ports and parameters, which everything else depends on."
   - Mention `/spec-status` for a structured progress check.

## Constraints

- Do **not** invent features or signals. If the user didn't specify, leave a `TODO`.
- Do **not** skip reading the templates. The templates encode the structure, the audience-segregation rules, and the anti-patterns. Generated content that bypasses them will violate the skill's design.
- Do **not** advance past D0 in this command. `/spec-init` ends with the skeleton in place. Iteration into D1 is done via natural conversation or via `/spec-status` recommendations.

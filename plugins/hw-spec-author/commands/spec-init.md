---
description: Start a new hardware design spec. Asks the IP class first (behavioral-block or protocol-bfm), writes MODE.md, then creates the mode-appropriate skeleton (6-file behavioral or 9-file BFM) and runs the Capture-phase interview.
argument-hint: <ip_name> [output_dir]
allowed-tools: Read, Write, Glob
---

You are activating Phase 1 (Capture / D0) of the **hw-spec-author** skill.

## Steps

1. **Read the skill**: open `${CLAUDE_PLUGIN_ROOT}/skills/hw-spec-author/SKILL.md` to load the full workflow, including the IP-class (mode) section and the per-mode template lists.

2. **Parse the user's invocation**: the command is `/spec-init <ip_name> [output_dir]`.
   - `$ARGUMENTS` contains the user's argument string.
   - First whitespace-delimited token = `ip_name`. Required. If missing, ask the user.
   - Second token (optional) = `output_dir`. If present, use it directly and skip that question in the interview. If absent, default to `./spec/<ip_name>/` and confirm with the user during the interview.

3. **Ask the IP-class question first** (this drives every subsequent step):

   > Q1: What kind of IP is this?
   >   (A) **Behavioral block** — peripheral, accelerator, controller, bus master, or any block whose main concern is internal logic with software-visible registers. (Default.)
   >   (B) **Protocol BFM / VIP** — a verification model that talks to RTL via a defined wire-level protocol (AXI master, AXI slave, custom-protocol BFM, etc.).

   Record the answer as `mode = behavioral-block` or `mode = protocol-bfm`. Default `behavioral-block` if the user doesn't choose explicitly.

4. **Run the mode-appropriate Capture interview**.

   **For `behavioral-block`**, ask:
   - One-sentence purpose of the IP
   - Bus / host interface (e.g., AXI4 slave, AXI4-Lite, APB, TileLink-UL, custom)
   - Clock and reset domains (count, async/sync, retention/AON?)
   - Top-level features (3–8 bullets, capabilities not adjectives)
   - Any prior art (existing register interface to be compatible with, similar IP's spec, draft notes)
   - Target output directory — **only ask this if `output_dir` was not given as an argument**. Default `./spec/<ip_name>/`.

   **For `protocol-bfm`**, ask:
   - One-sentence purpose of the BFM
   - Bus / protocol the BFM speaks (e.g., AXI4-Lite, AXI4, APB, TileLink-UL, custom)
   - Role: master, slave, or both
   - Active mode, passive mode, or both
   - **Will this block also have an RTL counterpart implementation?** (yes / no — affects whether `theory_of_operation.md` requires the `## RTL internal architecture` section and whether `MODE.md` records `has-rtl-counterpart: yes`. Default: ask the user explicitly; do not assume.)
   - Any optional protocol features in/out of scope (exclusive access, ID-bearing channels, low-power handshake, narrow transfers, etc.)
   - Configuration knobs the testbench will need (response delay, outstanding limit, fault injection types)
   - Any prior art (existing BFM source, vendor VIP datasheet, scribbled notes)
   - Target output directory — **only ask this if `output_dir` was not given as an argument**. Default `./spec/<ip_name>/`.

   Ask in **batched form** if the user prefers. Don't drag out the interview over many separate messages — one consolidated question block is better.

5. **Write `MODE.md`** to the spec root before generating any other file:

   ```markdown
   # Spec Mode

   mode: <behavioral-block | protocol-bfm>
   created: <today's ISO date>
   spec-author-plugin-version: <plugin version from plugin.json>
   has-rtl-counterpart: <yes | no>     # protocol-bfm mode only; from interview Q
   ```

   The `has-rtl-counterpart` line is omitted in `behavioral-block` mode (the question doesn't apply — RTL is the primary implementation).

6. **Generate the mode-appropriate skeleton**.

   **For `behavioral-block`**, create the 6-file layout:

   ```
   <output_dir>/
   ├── MODE.md
   ├── README.md
   ├── doc/
   │   ├── theory_of_operation.md
   │   ├── programmers_guide.md
   │   ├── interfaces.md
   │   └── registers.md
   └── dv/
       └── plan.md
   ```

   For each file, populate it from the corresponding template in `${CLAUDE_PLUGIN_ROOT}/skills/hw-spec-author/references/templates/`. Read the template **before** generating its file.

   **For `protocol-bfm`**, create the 9-file layout:

   ```
   <output_dir>/
   ├── MODE.md
   ├── README.md
   ├── doc/
   │   ├── theory_of_operation.md
   │   ├── signal_interface.md
   │   ├── pin_level_reset.md
   │   ├── protocol_rules.md
   │   ├── channel_handshake.md
   │   ├── transaction_api.md
   │   ├── channel_api.md
   │   └── active_passive_mode.md
   └── dv/
       └── plan.md
   ```

   For each BFM-specific file, populate from the matching template under `${CLAUDE_PLUGIN_ROOT}/skills/hw-spec-author/references/templates/bfm/`. For shared files (`README.md`, `dv/plan.md`), use the templates under `references/templates/` (`01_summary.md`, `06_dv_plan.md`) but scope to BFM concerns (README emphasises protocol+role+active/passive). For `theory_of_operation.md`, use the BFM-mode-specific template `${CLAUDE_PLUGIN_ROOT}/skills/hw-spec-author/references/templates/bfm/02_theory_of_operation.md` — it has a two-section structure (BFM internal architecture; optional RTL internal architecture). If the user answered `has-rtl-counterpart: yes` in step 4, generate both sections; otherwise generate only the BFM internal architecture section.

   `registers.md` is **not** generated by default in BFM mode. If the user explicitly says the BFM has a software-visible CSR (rare), generate `doc/registers.md` from `references/templates/05_registers.md`. Downstream tools (`/spec-status`, `/spec-lint`, `/spec-gate`) detect by file existence — a `doc/registers.md` present in a BFM-mode spec activates the `D1.reg.*` checklist items automatically. No additional `MODE.md` field is required.

7. **Fill in known information from the interview**; mark unknowns as `TODO(designer): <what's missing>`.

8. **Report back**. Tell the user:
   - The chosen mode (behavioral-block or protocol-bfm) and where `MODE.md` was written.
   - Where the skeleton was created.
   - How many `TODO(designer):` markers remain (per file).
   - The recommended next action:
     - For `behavioral-block`: "fill in `interfaces.md` first — it forces commitment on ports and parameters, which everything else depends on."
     - For `protocol-bfm`: "fill in `signal_interface.md` first — the channel names declared there flow into protocol-rule IDs, pin-level reset, and the API surfaces."
   - Mention `/spec-status` for a structured progress check.

## Constraints

- Do **not** invent features or signals. If the user didn't specify, leave a `TODO`.
- Do **not** skip reading the templates. The templates encode the structure, the audience-segregation rules, and the anti-patterns. Generated content that bypasses them will violate the skill's design.
- Do **not** advance past D0 in this command. `/spec-init` ends with the skeleton in place. Iteration into D1 is done via natural conversation or via `/spec-status` recommendations.
- **Always write `MODE.md`** — even when defaulting to `behavioral-block`. Downstream commands rely on its presence as the explicit declaration; absent files are tolerated only for back-compat with pre-mode-flag specs.

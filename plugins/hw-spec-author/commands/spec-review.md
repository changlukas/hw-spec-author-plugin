---
description: Run the reader-test protocol on an existing hardware spec. Spawns the spec-reader subagent (fresh context, isolated from authoring history) to detect ambiguity gaps.
argument-hint: "[path]"
allowed-tools: Read, Grep, Glob, Task, Edit
---

You are activating the **reader test** protocol of the hw-spec-author skill. This command's value depends on **isolation**: the questions are answered by a fresh subagent that has no memory of how the spec was authored. Do not answer the questions yourself.

## Steps

1. **Read the protocol**: open `${CLAUDE_PLUGIN_ROOT}/skills/hw-spec-author/references/process/reader_test.md` for the full question bank and execution guidance.

2. **Locate the spec**. Parse `/spec-review [path]`:
   - If a path given (in `$ARGUMENTS`), use it.
   - If no path, look for a spec root in CWD: a directory containing `README.md` plus `doc/` subfolder, or `./spec/`. If multiple candidates, ask the user which one.
   - If nothing found, tell the user and stop.

3. **Inventory the spec**. Use Glob to list which of the canonical six files exist:
   - `<spec_root>/README.md`
   - `<spec_root>/doc/theory_of_operation.md`
   - `<spec_root>/doc/programmers_guide.md`
   - `<spec_root>/doc/interfaces.md`
   - `<spec_root>/doc/registers.md`
   - `<spec_root>/dv/plan.md`

   Also Glob for any block-diagram SVGs the spec references.

4. **Generate the question set**. Read enough of the spec — at minimum the README Features list — to determine the IP's category. Then build 8–12 questions drawn from the question bank in `reader_test.md`:
   - **Universal bank**: include at least one from each category — reset, CDC, bus interface, software interaction, IRQs, errors, performance/saturation.
   - **Block-specific bank**: pick the bank matching this IP's category:
     - Bus master / DMA → outstanding count, response ordering, bus error handling
     - FIFO / buffer → depth, drain on reset, simultaneous read-write boundary
     - Configurable / parameterized → valid parameter combinations, feature dependencies
     - Security-critical → side channels, threat model

   **Show the user the question list briefly** and ask whether to proceed, add, or remove any. Wait for confirmation.

5. **Spawn the spec-reader subagent**. For each question, use the Task tool to invoke the `spec-reader` subagent with:
   - The list of spec file paths
   - The single question to answer
   - Instruction: "Read the spec files. Answer the question using only the spec text, following the spec-reader output format. Quote supporting text or report NOT_ANSWERED."

   You may run questions in parallel batches if the platform supports it.

   Critical: **do not answer the questions yourself**. The whole point of this command is the spec-reader's isolation. If you answer, you have just bypassed the test.

6. **Aggregate the results**. Collect all subagent responses. Build the gap report:

   ```
   Reader Test Report
   Spec: <path>
   Questions asked: <N>

   PASSED: <count>
     Q01: <question> → "<supporting quote>" (<file>)
     ...

   GAPS: <count>
     Q05: <question>
       Status: NOT_ANSWERED
       Probably belongs in: <file/section>
     Q07: <question>
       Status: AMBIGUOUS
       Quoted text: "<unclear sentence>"
       Why ambiguous: <reason>

   Suggested next actions:
     1. Fix in-scope gaps in <file> (priority 1, blocks D1 sign-off).
     2. Add out-of-scope references in <file>.
     3. Annotate undefined behavior in <file>.
   ```

7. **Triage with the user**. For each gap, ask whether:
   - **(a) In-scope, must add** → recommend which file to update.
   - **(b) Out-of-scope, must reference** → recommend "see X" sentence to add.
   - **(c) Genuinely undefined / implementation-defined** → recommend an explicit annotation.

8. **Optionally apply fixes**. If the user agrees on specific changes, edit the relevant spec file(s). Show the proposed edit before writing. Don't apply silently.

## Constraints

- **Subagent isolation is the test**. If the spec-reader subagent is unavailable (Task tool not present), tell the user the test cannot run reliably and stop. Do not silently fall back to answering yourself — that defeats the purpose.
- 8–12 questions is the sweet spot. Fewer → false confidence. More → reader fatigue.
- Ambiguity counts as a gap. A spec is not "passing" until questions resolve to clear quoted answers.
- This command is for ambiguity detection, not completeness checking (use `/spec-status`) or stage transitions (use `/spec-gate`).

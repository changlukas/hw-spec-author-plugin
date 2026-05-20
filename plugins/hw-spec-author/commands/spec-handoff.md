---
description: Scaffold a wave-handoff note (NEXT_SESSION_<wave>.md) with four fixed sections. Auto-fills the status snapshot from /spec-stats and the last git commit; the author fills the rest. Gives multi-session / multi-author spec work a consistent re-entry structure.
argument-hint: "<wave-name> [path]"
allowed-tools: Read, Glob, Grep, Bash, Write
---

You are scaffolding a **wave-handoff note** for the hw-spec-author skill. Multi-session spec work drifts when each session's handoff doc invents its own structure. This command writes a `NEXT_SESSION_<wave-name>.md` with four fixed sections so the next session (or the next author) can pick up cold.

## Steps

1. **Locate the spec**. Same logic as `/spec-status`. Parse `$ARGUMENTS` into `<wave-name>` and optional `[path]`. If nothing found, stop.

2. **Auto-fill the status snapshot**:
   - Run the `/spec-stats` logic over the spec (mode, rule/testpoint/register counts) and embed the result.
   - Run `git log -1 --format='%h %s (%cr)'` in the spec's repo for the last-commit line. If not a git repo, write "not under git".
   - Read `MODE.md` for mode + `has-rtl-counterpart`.

3. **Write `NEXT_SESSION_<wave-name>.md`** at the spec root using exactly these four sections. **Before writing, check whether the file already exists** — if it does, stop and ask the user whether to overwrite. Never clobber an existing handoff silently. Auto-filled fields are populated; author-filled fields carry a clear `<!-- fill -->` placeholder. A section with nothing to say should say "none", not be deleted.

```
# NEXT_SESSION_<wave-name>.md

## 1. Status snapshot
- Mode: <from MODE.md>   has-rtl-counterpart: <if BFM>
- Current D-stage: <!-- fill: run /spec-status -->
- /spec-stats: <auto-filled counts>
- Last commit: <auto-filled git log -1>
- Lint status: <!-- fill: clean / N open -->

## 2. Deferred tasks
For each deferred item:
- **<task>** — priority: <hi/med/lo> · blocker: <what's in the way> ·
  first step: <concrete next action> · acceptance: <how you know it's done> ·
  effort: <estimate>
<!-- fill; "none" if nothing deferred -->

## 3. Quick re-entry checklist
Five-minute sanity checks for the next session's start:
- [ ] git log -1 matches the snapshot above
- [ ] /spec-status stage unchanged
- [ ] <!-- fill: any wave-specific check -->

## 4. Reminders to future self
- <!-- fill: gotchas, deliberately-deferred decisions, scope boundaries -->
```

4. **Report** the path written and which fields still need the author's input (the `<!-- fill -->` ones).

## Constraints

- **Four fixed sections, always.** Do not add, drop, or reorder them — the consistency is the whole point. A section with nothing to report says "none".
- **Auto-fill what is mechanical** (snapshot counts, git line, mode); leave judgement fields (deferred-task analysis, reminders) as explicit placeholders for the author.
- **Write-only scaffold.** This command creates the handoff file; it does not interpret the spec's content beyond the snapshot. The author owns sections 2–4.
- This is a user-spec artifact (like `READER_TEST_LOG.md`), not a plugin example — it lives at the spec root, not in the plugin tree.

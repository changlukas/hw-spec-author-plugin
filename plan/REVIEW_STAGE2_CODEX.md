- `plugins/hw-spec-author/commands/spec-rename.md:27` has an impossible audit-trail step. It says to write `RENAME_LOG.md` when git is absent, but `allowed-tools` lacks `Write`; it also says to put decisions in the commit message body when git exists, but the command neither commits nor has `Bash`. This violates the declared tool contract.

- `plugins/hw-spec-author/commands/spec-handoff.md:18` does not define behavior when `NEXT_SESSION_<wave-name>.md` already exists. With `Write`, this risks clobbering an existing handoff. Existing conventions are explicit about overwrite behavior when intended.

- `plugins/hw-spec-author/skills/hw-spec-author/references/process/stage_gates.md:128` appears to retroactively break the worked examples: neither `wctmr/` nor `axi_lite_slave_bfm/` has `PLAN.md`, `PLAN_DELTA.md`, or a root-level N/A declaration, yet both claim D1 in `READER_TEST_LOG.md`. If the examples are still canonical D1 examples, this needs migration or a mode/version applicability guard.

- `D1.cross.delta_from_plan` is underspecified for mechanical gating. “Prominent N/A declaration at the spec root” has no filename, heading, or parseable token, so `/spec-gate` cannot check it near-zero-false-positive. Define an exact artifact, e.g. `PLAN_DELTA.md` with a fixed N/A heading.

- `plugins/hw-spec-author/commands/spec-lint.md:190-201` is internally inconsistent. The procedure only extracts `<identifier>[hi:lo]` and `<PARAM> = <value>`, but the example compares that against prose “16-bit read data.” If implemented with prose-width matching, LINT-015 risks false positives on the examples, especially `RDATA`/`DATA_WIDTH` and split 64-bit counter prose. Keep it declaration-only or define a strict table/signature grammar.

- `plugins/hw-spec-author/commands/spec-lint.md:209` should extract WAIVER anchors only from `## <anchor>` headings. “Every gate-anchor ID referenced in WAIVERS.md” is too broad and can warn on copied item text, history, or explanatory prose. LINT-016 is correctly N/A when no `WAIVERS.md` exists.

# hw-spec-author-plugin

A Claude Code plugin marketplace for digital hardware design tooling.

## What's inside

| Plugin | Description |
|---|---|
| [`hw-spec-author`](./plugins/hw-spec-author/) | Authors clean, complete, audience-segregated hardware design specifications for digital IP blocks. OpenTitan-Comportability-style structure with D0–D3 stage gates and reader-testing. |

## Install

Inside Claude Code:

```
/plugin marketplace add changlukas/hw-spec-author-plugin
/plugin install hw-spec-author@changlukas
```

Or persist in `~/.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "changlukas": {
      "source": {
        "source": "github",
        "repo": "changlukas/hw-spec-author-plugin"
      }
    }
  },
  "enabledPlugins": {
    "hw-spec-author@changlukas": true
  }
}
```

After install, restart Claude Code and verify with:

```
/spec-help
```

## Quick start

```
/spec-init my_timer
```

Runs the Capture interview, generates a 6-file OpenTitan-style spec skeleton at `./spec/my_timer/`. From there:

- `/spec-status` — see what's left to fill in
- `/spec-review` — run the reader test (via isolated subagent)
- `/spec-lint` — mechanical consistency check
- `/spec-gate D1` — formally close the D1 stage gate

## Plugin design

`hw-spec-author` is built around four properties:

1. **Audience-segregated.** Every section states its primary reader (HW designer, DV, SW, SoC integrator) and is written for them.
2. **Single source of truth.** Every fact lives in exactly one place; other places cross-reference.
3. **Stage-gated.** D0 / D1 / D2 / D3 have explicit checklists. No "D1 minus."
4. **Reader-testable.** A reader new to the design can answer concrete corner-case questions from the spec alone.

For the full design rationale, see [`plugins/hw-spec-author/README.md`](./plugins/hw-spec-author/README.md) and [`plugins/hw-spec-author/skills/hw-spec-author/SKILL.md`](./plugins/hw-spec-author/skills/hw-spec-author/SKILL.md).

## Worked example

[`plugins/hw-spec-author/examples/wctmr/`](./plugins/hw-spec-author/examples/wctmr/) is a complete D1-grade spec for a 64-bit Wallclock Timer. Use it as a reference for prose tone, table density, and audience separation when writing your own specs.

## Components shipped

- 1 skill (`hw-spec-author`) — workflow engine and references
- 1 subagent (`spec-reader`) — isolated-context reader for the reader test
- 6 slash commands — `/spec-init`, `/spec-status`, `/spec-review`, `/spec-lint`, `/spec-gate`, `/spec-help`
- 1 worked example — `wctmr` (64-bit timer with two compare slots)

## License

Apache 2.0. See [LICENSE](./LICENSE).

## Acknowledgments

Heavily inspired by the [OpenTitan / lowRISC Comportability framework](https://opentitan.org/book/doc/contributing/hw/comportability/index.html). Stage-gate concept adapted from the OpenTitan project signoff checklist.

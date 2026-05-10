# Process: Slide Style

Style guidance for hardware IP presentation decks generated from a spec produced by this plugin (e.g., a deck for an internal review of an NI / DMA / RoCE engine spec).

This file applies to deck output (`SLIDES.md` or similar). Spec source documents (`theory_of_operation.md`, `registers.md`, etc.) follow `writing_principles.md` instead.

The patterns are language-agnostic. Worked examples below use mixed English + zh-TW because the dogfood that originated these rules ran in zh-TW. Substitute your narrative language.

## 1. Slide organization — 1 component = 1 slide

Each major component (NMU / NSU / Credit-flow / Data Integrity / RoB / etc.) gets ONE cohesive datasheet-style slide. Sub-features (timeout, write order, errors, IRQ) are bullets within the parent slide, not separate slides.

Slide structure:

- **Bullet capability list** — concrete numeric values and supported feature set (counts, widths, burst types, default parameters).
- **Flowing prose paragraph(s)** — narrative of how the component operates (sub-block hand-offs, response path).
- **Verbatim quotes from external standards** (AMD / ARM / FlooNoC / etc.) embedded inline within bullets or prose where they read naturally. No separate "quotes section" heading.

Avoid:

- Fragmenting one component into multiple sub-aspect slides (architecture / burst / addressing / RoB on separate slides — wrong)
- Meta-template sub-headings inside the slide content. `### On-slide content / ### Speaker notes / ### Visual asset / ### Verbatim quotes` is wrong — speaker notes go in a separate notes pane, not inside slide content.

Style reference: AMD pg313 §NoC Master Unit / §NoC Slave Unit / §Credit-Based Flow Control / §Data Integrity sections each present their component as one continuous content block.

## 2. Slide depth — introduction-level, not implementation-walkthrough

Match an industry datasheet depth (AMD pg313, Xilinx PG, Intel architecture references). Concrete numeric values belong on slides. Specific parameter / signal / register / rule-ID names do not.

| Keep on slide | Drop from slide (move to spec) |
|---|---|
| "Up to 32 outstanding AXI reads + 32 outstanding writes" | `MAX_TXNS = 32` (per-AXI-ID cap `MAX_TXNS_PER_ID = 32`) |
| "Configurable AXI data width: 64 / 128 / 256 / 512 bits" | `DATA_WIDTH` parameter |
| "Three RoB modes balance area against reorder support" | `NoRoB` / `SimpleRoB` / `NormalRoB` with `prev_dest` adaptive bypass per `B_ROB_TYPE` / `R_ROB_TYPE` |
| "Two-layer ECC: per-hop parity + end-to-end SECDED" | `route_par` / `flit_ecc` field names; SECDED Hamming bound formula |

Audience: assume reviewers familiar with the high-level protocol (AXI4, NoC) but not the specific block being presented.

## 3. Takeaway pattern — depends on slide type

Each slide leads with one Takeaway sentence (one line). The sentence pattern depends on slide type.

### Pattern A — Mechanism slide (algorithm / design choice)

`<system requirement> → <component role> → <options>`

Example (Address Map mechanism slide):

> NoC bus 需依賴座標將封包送到目的地。NI 內部提供 3 種 address-to-coordinate 轉換方式：XY-routed、Source-routed、ID-table。

Why: the reader needs to know what system constraint motivates the mechanism before they can judge the trade-offs.

### Pattern B — Element-overview slide (introducing a component)

`<element name> + <primary action> + <responsibility list>`

Example (NMU overview slide):

> NMU 進行 AXI to NoC protocol 轉換，負責 ordering / integrity / flow control 等。

Why: when system context has already been established earlier in the deck, the element-overview slide can start directly from the element.

### When to use which

- Mechanism / algorithm / N-choice design pattern → Pattern A
- Element overview where preceding slide established context → Pattern B
- Standalone slide with no preceding context → Pattern A (lead with system need)

Common origins of weak Takeaways:

- Starting from the element's internal action ("X does A, B, C") instead of the system constraint
- Wrapping operations in white-noise verbs ("把 A 解成 B 兩件事") instead of stating what the constraint forces
- Beginning with "is X — does Y" structure (soft copula chain) instead of an active verb

## 4. Bullet format for enumerated options

When listing 2–4 parallel options (mode / variant / algorithm / interface choice), use the 4-segment bullet:

```
- **<option name>** (default-marker if applicable)：<mechanism> → <output> — <when to use>
```

The 4 segments and their separators:

1. **Option name** — bold, formal name. Optional `(default)` or `(SAM)` annotation immediately after.
2. **`：` separator** then **mechanism** — one phrase describing how it works.
3. **`→` separator** then **output** — what the mechanism produces. Omit this segment when the mechanism IS its own output.
4. **`—` separator** then **applicability** — when to use it / which deployment scenario.

The three separators have distinct semantic roles. Do not interchange them.

Example:

```
- **XY-routed** (default)：抽 awaddr 的 X/Y bit → 直接成 dst_id — 規則 2D mesh
- **Source-routed**：path 預先寫入 flit header — 靜態 topology / 設計階段最佳化
- **ID-table** (SAM)：System Address Map lookup → dst_id — 多區段 address space / 軟體定義 region
```

When this format is wrong:

- 5+ options → use a column-aligned table instead (rows align column-wise for fast comparison)
- Trade-off comparison (A vs B opposing, not A and B parallel) → use 2-col / 3-col table
- Ordered procedure (steps with sequence) → use numbered list
- One option's description is much longer than the others → use table (column widths absorb the asymmetry)

This format is one tool, not a universal rule. Use judgement.

## 5. Style enforcement

`/spec-lint` and `/spec-review` do not currently inspect slide files. Slide style review is a manual pass before the deck is shared.

Future plugin work: a `/spec-deck-review` command that validates slide files against the rules in this document. Tracked in `plan/DOGFOOD_OBSERVATIONS_A5.md`.

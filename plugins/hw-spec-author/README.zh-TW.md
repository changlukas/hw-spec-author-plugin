# hw-spec-author

> **語言**：[English](./README.md) | [繁體中文](./README.zh-TW.md)

Claude Code plugin，用來撰寫**乾淨、完整、依讀者分層**的數位 IP 硬體設計規格書。基於 [OpenTitan / lowRISC Comportability 框架](https://opentitan.org/book/doc/contributing/hw/comportability/index.html)。

> **安裝方式**請見 [marketplace 頂層 README](../../README.zh-TW.md)。本文聚焦在 plugin 內部結構、workflow，以及完整的指令參考。

---

## 這個 plugin 會產出什麼

在你指定的目錄下產生 6 個依讀者分層的檔案：

```
<ip_name>/
├── README.md                       # 摘要索引——Overview, Features, Description, Compatibility
├── doc/
│   ├── theory_of_operation.md      # 方塊圖、datapath、FSM、錯誤處理 (HW + DV + SW)
│   ├── programmers_guide.md        # 初始化、use cases、錯誤處理、IRQ (SW + DV)
│   ├── interfaces.md               # Port table、parameters、IRQ、alerts (SoC integrator)
│   └── registers.md                # CSR map (SW + DV)
└── dv/
    └── plan.md                     # Testpoints、coverage model、sec_cm (DV)
```

Plugin 強制以下原則：

- 每個 section 的 **讀者分層**
- **單一事實源頭**紀律（cross-reference，不要重複）
- **Stage gates D0 → D3** 與穩定 anchor
- 對歧義的 **reader testing**（透過隔離 subagent，8–12 道 universal 題加上分類題）
- **OpenTitan 風格散文**（具體、宣告式、不行銷）

---

## Slash command 完整參考

### `/spec-init <ip_name> [output_dir]`

Phase 1 進入點——**Greenfield**（沒有舊材料）。跑 Capture 訪談並產生 6-file 骨架。

- 詢問 IP 名稱、一句話的目的、bus interface、clock/reset domain、3–8 條 features、是否有 prior art、輸出目錄（預設 `./spec/<ip_name>/`）。
- 產生每個檔案前會先讀對應 template（在 `references/templates/`）。
- 把訪談時拿到的資訊填進去，沒拿到的標 `TODO(designer): <missing description>`。
- **不會**自己腦補 features。使用者沒講就留 TODO。

完成後會回報：骨架建在哪裡、各檔案有幾個 TODO、建議的下一步動作（通常是先填 `interfaces.md`）。

### `/spec-import <input_path> [output_dir]`

Phase 1 進入點——**Brownfield**（有舊 spec、RTL，或兩者皆有）。把既有材料重組成標準 6-file 配置。

支援的輸入模式（依 `<input_path>` 內容自動偵測）：

| 模式 | 偵測條件 | Canonical 來源 |
|---|---|---|
| **Doc** | `.md` / `.txt` 檔 | Prose、敘事段落 |
| **RTL** | `.sv` / `.v` 檔 | Ports、parameters、register reset values、FSM state names、SVA |
| **hjson** | `data/*.hjson`（OpenTitan 風格 register defs）| Register map（offsets、widths、fields、reset values）|
| **Combined** | 上述多種 | RTL/hjson 是 table data 的 canonical；doc 是 prose 的 canonical；衝突會被列出，**不**靜默選邊 |
| **Verilator XML** | `*.xml`（來自 `verilator --xml-only`）| Mega-IP（>2000 LOC）的結構索引 |

萃取規則記錄在 [`references/process/rtl_extraction.md`](./skills/hw-spec-author/references/process/rtl_extraction.md)。三個信心等級：

- **High**：1-to-1 語法對應（port list、parameter、enum FSM state、`always_ff` 中的 reset value）。產出時附行級 provenance 註解（`<!-- source: rtl/wctmr.sv:42 -->`）。
- **Medium**：pattern matching（FSM transitions、datapath、error condition）。標 `[draft — verify against design intent]`。
- **Low / TODO**：結構上萃取不出來（Features capability、programmer's guide use case、performance commitment、design intent）。產出 `TODO(designer):` 標記。

輸出：
- `<output_dir>` 下的 6-file spec（預設 `./spec/<inferred_ip_name>/`）
- `IMPORT_REPORT.md`：source manifest、extraction summary、conflicts（combined mode 時）、TODO inventory、recommended next actions

`/spec-import` **不會**宣告 D-stage；輸出最多到 D0、通常還在 pre-D0。Import 完成後跑 `/spec-status` 看離 D1 還差什麼。

### `/spec-status [path]`

檢查既有 spec。回報目前 D-stage 與下一閘還缺什麼。

- 從參數解析 spec root，沒給就掃 CWD（找含 `README.md` + `doc/` 的目錄）。
- 讀全部 6 個檔案（缺的也記下）。
- 走 `stage_gates.md` 的 D0 → D1 → D2 → D3 checklist，每條標 ✓ / ✗ / ?。
- 取「全部 ✓」的最高階段為當前 stage。
- 輸出結構化報告：最多 3 個建議行動、每檔的 TODO marker 數量。

唯讀，不會修改 spec。

### `/spec-review [path]`

執行 reader test（Phase 3，D1 sign-off）。

- 先讀 `references/process/reader_test.md` 取得協議與題庫。
- 定位並 inventory spec。
- 從 universal bank（reset、CDC、bus、software interaction、IRQs、errors、performance）和 block-specific bank（DMA / FIFO / configurable / security-critical，根據 README features 判定）建出 8–12 題。
- **透過 Task tool 派發 `spec-reader` subagent**，把 spec 檔路徑與一道題一起給它。關鍵：主 Claude **不能**自己回答——那會破壞測試的隔離性。
- 把結果聚合成 PASS / NOT_ANSWERED / AMBIGUOUS，PASS 附 spec-reader 引述的支撐句。
- 與使用者一起 triage 每個 gap：in-scope（要補）、out-of-scope（要 reference）、genuinely undefined（要顯式註記）。
- 可選擇套用修改（每個 edit 前確認）。

### `/spec-lint [path]`

機械性一致性檢查。與 `/spec-status`（gate completeness）、`/spec-review`（語意歧義）互補。

七條 lint 規則：

| 規則 | 檢查項目 |
|---|---|
| LINT-001 | Cross-reference 是否壞掉（`see Theory of Operation §X`、markdown link 等） |
| LINT-002 | 未追蹤的 TODO marker——裸 `TBD`、`TODO(designer):` 沒接 issue link、`IMPLEMENTATION-DEFINED:` 沒附理由 |
| LINT-003 | Feature ↔ testpoint 對應：每條 README feature 都該對到至少一個 `dv/plan.md` 的 testpoint |
| LINT-004 | Register 名稱大小寫——以 `registers.md` 規範清單為基準（只比對 canonical 名稱的大小寫變體；不會誤判一般大寫 token） |
| LINT-005 | Port 名稱大小寫——以 `interfaces.md` 規範清單為基準 |
| LINT-006 | Reset 值格式（`0x...`、WO 用 `—`、可用 `strap-dependent`；裸 `0`、"all zero"、空欄會被抓） |
| LINT-007 | 過期的方塊圖 SVG 引用（檔案是否存在） |

唯讀。

### `/spec-gate <D1|D2|D3> [path]`

Stage-gate 儀式。走目標 gate 的每條 checklist。

- 先驗證前一個 gate 已全部達標（D2 需要 D1；D3 需要 D2）。
- 讀 `<spec_root>/WAIVERS.md`（若存在）；以每個 waiver 的 `## H2` heading 對 checklist 的 anchor（如 `(id: D2.sec_cm_list)`）做 verbatim 比對。
- 每條標 ✓ / ✗ / ⚠ waived。anchor 不存在或拼錯的 waiver 會單獨列出。
- 輸出判決：PASS（全部 ✓ 或 ⚠）或 FAIL 加優先排序的開放項目。
- 若使用者顯式要 waive，指令會問 who / why / rationale 並 append 到 `WAIVERS.md`。裸的「跳過」會被拒絕。

`stage_gates.md` 中每條都帶 `(id: ...)` anchor——詳見 [`stage_gates.md`](./skills/hw-spec-author/references/process/stage_gates.md)。Anchor 是契約；工具不會從文字推導 anchor。

### `/spec-help`

Workflow card。列出四個 phase、所有指令；若 CWD 中能找到 spec，會根據其表面狀態建議下一個指令。

---

## 自然語言觸發

下列字句不需要 slash command，skill 會自行啟動：

- "write a spec for ..."
- "draft a micro-arch spec"
- "design document for the ... module"
- 「幫我寫一個 ... 的 spec」

兩條路徑會走到同一個 workflow。

---

## 工作流程

```
Phase 1 — Capture (D0)             /spec-init           (greenfield)
                                   /spec-import         (brownfield: doc + RTL + hjson)
   ↓
Phase 2 — Iterate by section       自然對話, /spec-status, /spec-lint
   ↓
Phase 3 — Reader test (D1)         /spec-review
   ↓
Phase 4 — Stage gates              /spec-gate D1, D2, D3
```

### Phase 2 中的推薦撰寫順序

1. **README** — Overview + Features（5 分鐘；強迫先寫 elevator pitch）
2. **`interfaces.md`** — 釘住 port、參數、clock
3. **`registers.md`** — 釘住 programming model
4. **`theory_of_operation.md`** — 此時你已經有 signal + register 名字可以引用
5. **`programmers_guide.md`** — 此時可以用真實 register 寫 usage flow
6. **`dv/plan.md`** — 此時可以 scope coverage

先寫 `theory_of_operation.md` 看似自然（畢竟是「主文件」），但會造成循環改寫——port 或 register 名一變就要重寫。避免。

---

## Subagent：`spec-reader`

定義在 [`agents/spec-reader.md`](./agents/spec-reader.md)。由 `/spec-review` 透過 Task tool 派發。

Subagent 在 fresh context 下執行，僅有 `Read`、`Grep`、`Glob`——不能 write、不能 shell、不能上網。每次給它 spec 檔路徑與一道題，配嚴格規則：

- 只能用 spec 文字作答。不能用常識外推，不能用同類 IP 領域知識。
- 每個 PASS 都要引述支撐句並附檔案路徑與 section。
- Spec 沒寫就回 `NOT_ANSWERED`；spec 寫到一半但未明確就回 `AMBIGUOUS`。
- Cross-reference 算 PASS 的條件是：目標 section 真的有答案。

Subagent 的隔離是 reader test 嚴謹的全部理由。Spec 作者**沒辦法把自己寫的東西忘掉**——所以作者讀自己的 spec 時會無意識地填補空缺。Fresh subagent 沒辦法。

---

## 範例：`wctmr`

[`examples/wctmr/`](./examples/wctmr/) 是完整的 D1 級別 spec：64-bit Wallclock Timer，雙 compare slot、可程式化 prescaler、APB slave。它示範了：

- 標準 6-file 配置
- 依讀者分層的散文（ToO 給 HW/DV/SW；programmer's guide 給 SW；interfaces 給 SoC integrator）
- 單一事實源頭紀律（registers 引用 ToO；ToO 引用 interfaces）
- 具體 corner case 涵蓋：counter wrap、software write atomicity、clear-while-true 後 IRQ 重發、out-of-range prescale clamp、mid-APB-transaction reset
- 一份 [`READER_TEST_LOG.md`](./examples/wctmr/READER_TEST_LOG.md)，記錄實際的 `/spec-review` 執行：從 5 PASS / 5 gaps 補完到 **10 / 10 PASS**——dogfood 證明 reader test 真的能抓出 author review 看不見的東西。

**寫新 spec 之前先讀 wctmr**。語氣、表格密度、讀者分層的拿捏，看具體範例比看 template 快。

---

## Stage gates

| 階段 | 定義 |
|---|---|
| **D0** | 概念；粗略形狀已知。Block 已被提案；有一句話的目的、3 條以上 features、bus interface 命名、clock/reset 大致識別、骨架檔案存在。 |
| **D1** | Spec 約 90% 完成。DV 可開始 testbench、RTL author 可開始 stub。Reader test 已跑過且 gaps 已補。 |
| **D2** | RTL 可運作且符合 spec。RTL 存在、編譯與 elaborate 無 X 傳播；smoke test 通過；spec 凍結僅可澄清。 |
| **D3** | Sign-off。DV 達 V3、lint 達 sign-off 級、可適用時 FPV 已證明、HW/DV/SW lead 簽核完成。 |

每個階段的完整 checklist 與 `(id: ...)` 穩定 anchor 在 [`skills/hw-spec-author/references/process/stage_gates.md`](./skills/hw-spec-author/references/process/stage_gates.md)。

**沒有「D1 minus」**。一個「D1 但少一個 FSM」的 spec 還在 D0；除非把缺的東西補齊，或在 `WAIVERS.md` 用一句話的理由顯式 waive。

---

## 輸出語言

Spec 內容預設**英文**。對齊產業慣例（OpenTitan、ARM AMBA、SiFive、RISC-V 規格皆英文）與下游用途（跨團隊 review、論文、專利）。撰寫期間的對話可用任何語言；只有產出檔案預設英文。如果你明確要求其他語言，skill 會跟著調整。

---

## 設計哲學

Plugin 圍繞四個性質打造：

1. **依讀者分層**：每個 section 標明主要讀者並專為他們寫，不混雜。
2. **單一事實源頭**：每個事實活在一個地方，其他地方 cross-reference。
3. **Stage-gated**：D0 / D1 / D2 / D3 各有明確 checklist 與穩定 anchor，沒有「D1 minus」。
4. **可被讀者測試**：新 context 的讀者能單靠 spec 回答 corner case 問題。

完整理由請讀 [`skills/hw-spec-author/SKILL.md`](./skills/hw-spec-author/SKILL.md) 與 [`references/process/`](./skills/hw-spec-author/references/process/) 下的文件。

---

## 出貨內容

- 1 個 skill：`hw-spec-author`（workflow 引擎 + 6 個 template + 4 份 process 文件）
- 1 個 subagent：`spec-reader`（reader test 用的隔離讀者）
- 7 個 slash command
- 1 個範例：`wctmr`

---

## Plugin 結構

```
plugins/hw-spec-author/
├── .claude-plugin/
│   └── plugin.json
├── README.md, README.zh-TW.md
├── agents/
│   └── spec-reader.md
├── commands/
│   ├── spec-init.md
│   ├── spec-import.md
│   ├── spec-status.md
│   ├── spec-review.md
│   ├── spec-lint.md
│   ├── spec-gate.md
│   └── spec-help.md
├── examples/
│   └── wctmr/
│       ├── README.md
│       ├── READER_TEST_LOG.md
│       ├── doc/
│       └── dv/
└── skills/
    └── hw-spec-author/
        ├── SKILL.md
        └── references/
            ├── process/
            │   ├── stage_gates.md
            │   ├── reader_test.md
            │   ├── writing_principles.md
            │   └── rtl_extraction.md
            └── templates/
                ├── 01_summary.md
                ├── 02_theory_of_operation.md
                ├── 03_programmers_guide.md
                ├── 04_interfaces.md
                ├── 05_registers.md
                └── 06_dv_plan.md
```

---

## 授權

Apache 2.0。詳見 [LICENSE](../../LICENSE)。

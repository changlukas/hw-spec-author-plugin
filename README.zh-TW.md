# hw-spec-author-plugin

> **語言**：[English](./README.md) | [繁體中文](./README.zh-TW.md)

一個專為數位硬體設計工具打造的 Claude Code plugin marketplace。目前發布一個 plugin：**`hw-spec-author`**——以 [OpenTitan / lowRISC Comportability 框架](https://opentitan.org/book/doc/contributing/hw/comportability/index.html)為基礎，協助你撰寫**乾淨、完整、依讀者分層**的數位 IP 硬體設計規格書（hardware design specification）。

---

## 為什麼需要這個工具

硬體設計規格書（spec）會以可預測的方式腐化：

- **讀者混雜**：同一段同時對 HW 設計者、DV 工程師、SW driver 作者講話，結果三邊都看不順。
- **重複事實**：同一個 register 行為在三個檔案裡各寫一份，半年後三份悄悄不一致。
- **隱性假設**：「reset 把所有東西清掉」——但有清 RAM 嗎？scrambling state？scrubbing counter？作者腦袋裡知道，spec 沒寫。
- **「完成度」沒有紀律**：「spec 差不多寫好了」不是個能驗證的命題；review meeting 變成偏好投票。
- **晚期才浮現的歧義**：bugs 在 V1 verification 才現形，「spec 沒寫這個 case 該怎麼做」，每次代價是好幾週。

`hw-spec-author` 從根本強制四個性質，這四項合力消除上述漂移：

1. **依讀者分層（Audience-segregated）**：每個 section 標明主要讀者（HW / DV / SW / SoC integrator），並且**只**為那個讀者寫。
2. **單一事實源頭（Single source of truth）**：每個事實只活在一個檔案；其他檔案 cross-reference。
3. **階段閘 D0 → D3（Stage-gated）**：每個閘有明確 checklist 與穩定 anchor。「夠不夠進下一階段」是 yes/no，不是討論。
4. **可被讀者測試（Reader-testable）**：新 context 的 subagent 讀完 spec、回答具體 corner case 問題；缺漏是機械性事實，不是意見之爭。

這個框架已經在矽晶片上驗證過：OpenTitan 在 Google、ETH Zurich、Western Digital、Seagate、Nuvoton 都有實際使用。

---

## 此 marketplace 內含什麼

| Plugin | 描述 |
|---|---|
| [`hw-spec-author`](./plugins/hw-spec-author/) | 為數位 IP 撰寫硬體設計規格書。OpenTitan-Comportability 風格的 6-file 結構，含 D0→D3 stage gates 與 reader test。 |

---

## 安裝

### 在 Claude Code 內安裝（推薦）

```
/plugin marketplace add changlukas/hw-spec-author-plugin
/plugin install hw-spec-author@changlukas
```

重啟 Claude Code，然後驗證：

```
/spec-help
```

看到 workflow card 就代表裝好了。

### 透過 `settings.json` 持久安裝

如果要團隊共用或持久安裝，加進 `~/.claude/settings.json`（全域）或 `.claude/settings.json`（專案）：

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

---

## 快速上手

### Greenfield（從零開始，沒有舊材料）

```
/spec-init my_timer
```

會跑 Capture phase 訪談（問你 IP 的目的、bus interface、clock/reset domain、3–8 條 features），然後在 `./spec/my_timer/` 產生 6 個檔案的 OpenTitan 風格骨架。每個檔案會用 template 填入；你沒回答到的細節會留 `TODO(designer):`，**不會**被亂猜。

### Brownfield（已有舊 spec / RTL / hjson）

```
/spec-import path/to/old_spec_or_rtl/
```

讀任意組合：既有 markdown spec、SystemVerilog/Verilog RTL、OpenTitan 風格 hjson register 定義，重組成標準 6-file 配置。可機械性萃取的項目（ports、parameters、register reset values、FSM state names、SVA assertions）會被填入並加上行級 provenance 註解；其餘留 `TODO(designer):`。同時產出 `IMPORT_REPORT.md`，記錄萃取了什麼、信心多高、RTL vs doc 之間有哪些衝突需要 reconcile。

### 後續指令

| 指令 | 何時執行 | 做什麼 |
|---|---|---|
| `/spec-status` | 任何時候 | 報告目前 D-stage 與下一閘還缺什麼。唯讀。 |
| `/spec-lint` | Phase 2 任何時候 | 機械性 drift 檢查：壞掉的 cross-reference、未追蹤的 TODO、Feature ↔ testpoint 對應缺漏、register/port 大小寫不一致。唯讀。 |
| `/spec-review` | 宣告 D1 之前 | 透過 `spec-reader` subagent（fresh、隔離 context）跑 reader test。問 8–12 道 ambiguity 題；通過就引述 spec 句子，未通過就回報 gap。 |
| `/spec-gate D1` | 看起來 D1 ready 了 | 走完 D1 checklist；persistent waivers 寫在 `WAIVERS.md`。 |
| `/spec-gate D2` / `D3` | 後期 | 同樣協議，目標換成後面的 gate。 |
| `/spec-help` | 任何時候 | Workflow card + 根據當下狀態的下一步建議。 |

---

## `/spec-init` 會產生什麼

```
<ip_name>/
├── README.md                   # 摘要索引——Overview, Features, Description, Compatibility
├── doc/
│   ├── theory_of_operation.md  # 方塊圖、datapath、FSM、錯誤處理（HW + DV + SW）
│   ├── programmers_guide.md    # 初始化、use cases、錯誤處理、IRQ（SW + DV）
│   ├── interfaces.md           # Port table、parameters、IRQ（SoC integrator）
│   └── registers.md            # CSR map（SW + DV）
└── dv/
    └── plan.md                 # Testpoints、coverage model、sec_cm（DV）
```

小型 block 可以把 `theory_of_operation.md` 與 `programmers_guide.md` 併進 README；複雜 block 可把 `theory_of_operation.md` 拆成多個檔放在 `doc/` 下。預設使用上述標準 6-file 配置。

---

## 推薦的撰寫順序

Skill 強制以下順序，避免循環改寫：

1. **README** — Overview + Features（5 分鐘；強迫自己先寫出 elevator pitch）
2. **`interfaces.md`** — 釘住 port、參數、clock
3. **`registers.md`** — 釘住 programming model
4. **`theory_of_operation.md`** — 此時你已經有 signal + register 名字可以引用
5. **`programmers_guide.md`** — 此時你能用真實的 register 名字寫 usage flow
6. **`dv/plan.md`** — 此時你能 scope coverage

先寫 `theory_of_operation.md` 看似自然（畢竟它是「主文件」），但每次 port 或 register 名字變動，你就得重寫一次。

---

## 工作流程（D0 → D3）

### Phase 1 — Capture（目標：D0）

目標：產出含 `TODO(designer):` 標記的完整 6-file 骨架，凡是還沒釘住的細節都標 TODO。

- **Greenfield**：`/spec-init <ip_name> [output_dir]` 走訪談、用 template 產生骨架。**不會**自動腦補沒講到的 feature。
- **Brownfield**：`/spec-import <input_path> [output_dir]` 讀既有材料——Markdown spec、SystemVerilog/Verilog RTL、hjson register 定義，或任意組合——重組成標準 6-file 配置。RTL 是 ports/parameters/registers/FSM 的 canonical source；既有 prose 是敘事段落的 canonical source。衝突會被列入 `IMPORT_REPORT.md`，**不會**靜默自動選邊。

兩條路徑會匯流到相同的輸出結構，後續 Phase 2 流程一致。

### Phase 2 — Iterate（D0 → D1）

照推薦順序一個 section 一個 section 寫。寫每個檔案前，skill 會先讀對應的 template（在 `references/templates/` 下）以及 `wctmr` 範例。用 `/spec-status` 追蹤閘進度；用 `/spec-lint` 早早抓 drift。

### Phase 3 — Reader Test（D1 sign-off）

`/spec-review` 透過 Task tool 派發 `spec-reader` subagent。Subagent 在 fresh context、唯讀模式下讀 6 個 spec 檔，回答 8–12 道題目，題目來自：

- **Universal bank**：reset、CDC、bus、IRQ、errors、performance（每類至少一題）
- **Block-specific bank**：依 IP 類別挑選（DMA / FIFO / configurable / security-critical）

PASS 必須引述 spec 中的句子；NOT_ANSWERED 與 AMBIGUOUS 都是要補的 gap。

Subagent 的隔離很重要：spec 作者**沒辦法把已知的東西忘掉**（cannot un-know），會用設計腦袋去讀自己的 spec、不自覺填補空缺。Reader test 就是設計來戳破這種盲點。

### Phase 4 — Stage gating（D1 → D2 → D3）

`/spec-gate <D1|D2|D3>` 走目標 gate 的 checklist。每條 checklist item 都有穩定 anchor（例如 `(id: D1.too.fsm)`），所以 `WAIVERS.md` 裡的 waiver 跨 session、跨 checklist 重排都能對應到。

---

## Stage gate 定義

| 階段 | 定義 | 觸發時機 |
|---|---|---|
| **D0** | 概念階段；粗略形狀已知 | 訪談 / `/spec-init` 完成後 |
| **D1** | Spec 約 90% 完成；DV 可開始寫 testbench、RTL author 可開始寫 stub | RTL 實作開始前 |
| **D2** | RTL 可運作且符合 spec；spec 凍結，僅可澄清 | RTL 已存在、smoke test 通過 |
| **D3** | Sign-off，已可進 tape-out integration | 所有 review 完成、lint 乾淨、可適用時 FPV 已證明 |

每個階段的完整 checklist 與穩定 anchor 在 [`stage_gates.md`](./plugins/hw-spec-author/skills/hw-spec-author/references/process/stage_gates.md)。

**沒有「D1 minus」這種東西**。一個「D1 但少一個 FSM」的 spec 還在 D0；除非把缺的東西補齊，或在 `WAIVERS.md` 用一句話的理由顯式 waive。

---

## Subagent：`spec-reader`

Plugin 內含一個 subagent：[`agents/spec-reader.md`](./plugins/hw-spec-author/agents/spec-reader.md)。由 `/spec-review` 透過 Task tool 派發。

- **工具限制**：只能用 `Read`、`Grep`、`Glob`——不能 write、不能 shell、不能上網。
- **Fresh context**：完全沒有 authoring history。
- **嚴格隔離規則**：只能依 spec 文字作答；引述支撐句；如果 spec 沒寫，回報 `NOT_ANSWERED`，**不准**用常識或同類 IP 知識補答。

如果你發現自己在親自回答 reader-test 題目而不是派發 subagent，你已經把 reader test 想消除的偏誤又重新引入了。

---

## 範例：wctmr

[`plugins/hw-spec-author/examples/wctmr/`](./plugins/hw-spec-author/examples/wctmr/) 是一個完整的 D1 級別 spec：64-bit Wallclock Timer，雙 compare slot、可程式化 prescaler、APB slave。它示範了：

- 標準的 6-file 配置
- 依讀者分層的散文（ToO 給 HW/DV/SW；programmer's guide 給 SW；interfaces 給 SoC integrator）
- 單一事實源頭的紀律（registers 引用 ToO；ToO 引用 interfaces）
- 具體的 corner case 涵蓋：counter wrap、software write atomicity、clear-while-true 後 IRQ 重發、out-of-range prescale clamp、mid-APB-transaction reset
- 一份 [`READER_TEST_LOG.md`](./plugins/hw-spec-author/examples/wctmr/READER_TEST_LOG.md)，記錄實際跑過的 `/spec-review`：從 5 PASS / 5 gaps，補完之後 **10 / 10 PASS**——dogfood 證明 reader test 真的能抓出 author review 看不見的東西。

**寫新的 spec 之前先讀 wctmr 範例**。語氣、表格密度、讀者分層的拿捏，看具體範例比看 template 容易多了。

---

## 輸出語言

Spec 內容預設使用**英文**。理由：對齊產業慣例（OpenTitan、ARM AMBA、SiFive、RISC-V 規格都是英文），也符合下游用途（跨團隊 review、論文、專利）。Spec 寫作期間的對話可以用任何語言；只有產出的 spec 檔案預設英文。如果你明確要求其他語言，skill 會跟著調整。

---

## 自然語言觸發

Skill 也能在沒下 slash command 的情況下從自然語言觸發：

- "write a spec for ..."
- "draft a micro-arch spec"
- "design document for the ... module"
- 「幫我寫一個 ... 的 spec」

兩條路徑會走到同一個 workflow 邏輯。

---

## 設計哲學

Plugin 圍繞四個性質打造，每個 gate 都會檢查：

1. **依讀者分層**：每個 section 標明主要讀者並專為他們寫，不混雜。
2. **單一事實源頭**：每個事實活在一個地方，其他地方 cross-reference。
3. **Stage-gated**：D0 / D1 / D2 / D3 各有明確 checklist 與穩定 anchor，沒有「D1 minus」。
4. **可被讀者測試**：新 context 的讀者能單靠 spec 回答 corner case 問題。

完整理由請讀 [`SKILL.md`](./plugins/hw-spec-author/skills/hw-spec-author/SKILL.md) 與 [`references/process/`](./plugins/hw-spec-author/skills/hw-spec-author/references/process/) 下的文件。

---

## 出貨內容

- 1 個 skill：`hw-spec-author`（workflow 引擎 + 6 個 template + 4 份 process 文件）
- 1 個 subagent：`spec-reader`（reader test 用的隔離讀者）
- 7 個 slash command：`/spec-init`、`/spec-import`、`/spec-status`、`/spec-review`、`/spec-lint`、`/spec-gate`、`/spec-help`
- 1 個範例：`wctmr`（64-bit timer，雙 compare slot、APB slave、可程式化 prescaler）

---

## 倉儲結構

```
hw-spec-author-plugin/
├── .claude-plugin/
│   └── marketplace.json
├── plugins/
│   └── hw-spec-author/
│       ├── .claude-plugin/
│       │   └── plugin.json
│       ├── README.md
│       ├── agents/
│       │   └── spec-reader.md
│       ├── commands/
│       │   ├── spec-init.md
│       │   ├── spec-import.md
│       │   ├── spec-status.md
│       │   ├── spec-review.md
│       │   ├── spec-lint.md
│       │   ├── spec-gate.md
│       │   └── spec-help.md
│       ├── examples/
│       │   └── wctmr/
│       │       ├── README.md
│       │       ├── READER_TEST_LOG.md
│       │       ├── doc/
│       │       └── dv/
│       └── skills/
│           └── hw-spec-author/
│               ├── SKILL.md
│               └── references/
│                   ├── process/
│                   │   ├── stage_gates.md
│                   │   ├── reader_test.md
│                   │   ├── writing_principles.md
│                   │   └── rtl_extraction.md
│                   └── templates/
│                       ├── 01_summary.md
│                       ├── 02_theory_of_operation.md
│                       ├── 03_programmers_guide.md
│                       ├── 04_interfaces.md
│                       ├── 05_registers.md
│                       └── 06_dv_plan.md
├── LICENSE
└── README.md
```

---

## 授權

Apache 2.0。詳見 [LICENSE](./LICENSE)。

---

## 致謝

深受 [OpenTitan / lowRISC Comportability 框架](https://opentitan.org/book/doc/contributing/hw/comportability/index.html)啟發。Stage-gate 概念改編自 OpenTitan project signoff checklist。

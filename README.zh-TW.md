# hw-spec-author-plugin

> **語言**：[English](./README.md) | [繁體中文](./README.zh-TW.md)

一個專為數位硬體設計工具打造的 Claude Code plugin marketplace。目前發布一個 plugin：**`hw-spec-author`** —— 協助你撰寫**乾淨、完整、依讀者分層**的數位 IP 硬體設計規格書。支援兩種模式：

- **`behavioral-block`** mode（預設）—— [OpenTitan / lowRISC Comportability 框架](https://opentitan.org/book/doc/contributing/hw/comportability/index.html)的 layout，適用於有內部邏輯與軟體可見 register 的 IP block。
- **`protocol-bfm`** mode —— 適用於有嚴格 wire-level 協定契約的 block（AXI、AHB、APB、TileLink、NoC、自訂協定）。同時涵蓋純 verification BFM 與 RTL+C model 配對實作兩種使用場景。

---

## 為什麼需要這個工具

硬體設計規格書（spec）會以可預測的方式腐化：

- **讀者混雜**：同一段同時對 HW 設計者、DV 工程師、SW driver 作者講話，結果三邊都看不順。
- **重複事實**：同一個 register 行為在三個檔案裡各寫一份，半年後三份悄悄不一致。
- **隱性假設**：「reset 把所有東西清掉」—— 但有清 RAM 嗎？scrambling state？scrubbing counter？作者腦袋裡知道，spec 沒寫。
- **「完成度」沒有紀律**：「spec 差不多寫好了」不是個能驗證的命題；review meeting 變成偏好投票。
- **晚期才浮現的歧義**：bugs 在 V1 verification 才現形，「spec 沒寫這個 case 該怎麼做」，每次代價是好幾週。

`hw-spec-author` 從根本強制四個性質，這四項合力消除上述漂移：

1. **依讀者分層（Audience-segregated）**：每個 section 標明主要讀者（HW / DV / SW / SoC integrator / testbench 作者），並且**只**為那個讀者寫。
2. **單一事實源頭（Single source of truth）**：每個事實只活在一個檔案；其他檔案 cross-reference。
3. **階段閘 D0 → D3（Stage-gated）**：每個閘有明確 checklist 與穩定 anchor。「夠不夠進下一階段」是 yes/no，不是討論。
4. **可被讀者測試（Reader-testable）**：新 context 的 subagent 讀完 spec、回答具體 corner case 問題；缺漏是機械性事實，不是意見之爭。

`behavioral-block` mode 已在矽晶片上驗證過：OpenTitan 在 Google、ETH Zurich、Western Digital、Seagate、Nuvoton 都有實際使用。`protocol-bfm` mode 把同樣的四項性質擴展到協定嚴謹度的領域（cycle 級規則、channel handshake 依賴、pin 級 reset、transaction/channel API）。

---

## 此 marketplace 內含什麼（v0.5.0）

| Plugin | 描述 |
|---|---|
| [`hw-spec-author`](./plugins/hw-spec-author/) | 為數位 IP 撰寫硬體設計規格書。兩種 mode：**behavioral-block**（OpenTitan-Comportability 風格 6-file 結構）與 **protocol-bfm**（9–11 file 結構，含 pin 級嚴謹度、protocol rules、transaction/channel API）。兩種 mode 都有 D0→D3 stage gates 與 reader testing。 |

**v0.5.0** 新增第一批 dogfood 驅動的 lint 與報告工具，並把 v0.3.1 之後 ship、但當時未單獨 bump 版本的 `/spec-implementer-review` 命令一併納入版本。

v0.5.0 新增：

- **`/spec-stats`** —— mode-aware 聚合計數（rule FAIL/RECOMMEND 拆分、distinct testpoint count 與 max-ID、ABV count、register count、parameter defaults），作者原本要為簡報 / README / handoff 手動 grep 的那些數字。
- **LINT-010**（testpoint ID uniqueness）—— 重複 ID 報 FAIL、編號間隙報 INFO。查唯一性而非連號，依分類編號的 DV plan 不會被誤殺。
- **LINT-013**（register 欄位 bit-overlap / 寬度溢位）—— 對 register 欄位表的純結構解析，near-zero false-positive。
- 放寬 **LINT-002**（接受 em-dash TODO rationale）與 **LINT-BFM-001**（接受 grouped reset rows 與 channel-less interface），降低 false-positive 噪音。
- **`/spec-implementer-review`** —— 多 agent paradigm-paired review（protocol-bfm + has-rtl-counterpart），浮現 reader test 抓不到的 bit-equivalence 歧義。

每個 lint 變更都附帶對兩個內建 example 的 worked-example regression 實跑 —— 驗收門檻是「跑出來」而非「宣稱」。

先前版本：v0.3.1 對齊 `TODO(designer):` 格式（生成端與驗證端）；v0.3.0 新增 protocol-bfm mode。

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

看到 workflow card（最上方有 mode 行）就代表裝好了。

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

## 選擇 mode：`behavioral-block` vs `protocol-bfm`

Mode 在你執行 `/spec-init` 時選擇，並記錄在 spec root 的 `MODE.md`。所有後續 commands（`/spec-status`、`/spec-review`、`/spec-lint`、`/spec-gate`、`/spec-help`）都會讀它。

### 決策樹

依序回答這些問題：

1. **這個 block 對外的主要介面是 wire-level handshake 協定嗎？**（AXI4 / AXI4-Lite / AHB / APB / TileLink / CHI / 自訂 valid-ready）
   - **否**（是有 CSR 的 peripheral 但不需要 cycle 級嚴謹度；或是純內部 sub-block）→ **`behavioral-block`**。
   - **是** → 進到 Q2。

2. **你（或別人）會寫一個 cycle-accurate 的 C/C++/SystemC/UVM model**，**與 RTL 實作配對，兩者在 wire boundary 上行為等價嗎？**
   - **是** → **`protocol-bfm`**，`has-rtl-counterpart: yes`。Spec 服務兩種實作：共用 protocol 契約 + 各自獨立的內部架構章節（BFM-internal：driver/monitor/sequencer；RTL-internal：register file / decoder / pipeline）。
   - **否，只有 RTL 實作** → 目前先用 **`behavioral-block`**。（在有專屬 RTL-protocol 變體之前，先用 `behavioral-block` 的 `theory_of_operation.md`，比較貼近。）

3. **這是純 verification artifact 嗎？**（純 BFM/VIP，沒有 RTL counterpart，純為驗證別人的 RTL 而寫）
   - **是** → **`protocol-bfm`**，`has-rtl-counterpart: no`。Spec 描述 BFM 內部架構、testbench-API 介面、protocol 契約。

### 具體範例

| IP 類型 | Mode | `has-rtl-counterpart` | 備註 |
|---|---|---|---|
| Wallclock timer（CSR-driven、單 bus、~10 個 register） | `behavioral-block` | n/a | 見 `examples/wctmr/`。 |
| 一般 peripheral（UART、SPI、I2C controller）含 CSR | `behavioral-block` | n/a | Bus 介面以抽象形式處理；不聚焦 cycle 級嚴謹度。 |
| AXI4 master IP 設計 + 配對 C++ model 用於效能 / co-sim 驗證 | `protocol-bfm` | yes | Spec 一次涵蓋 wire-level AXI4 契約；RTL 與 C model 兩種實作都依此實作。內部架構分兩節記錄。 |
| AXI-Lite slave BFM（純 verification IP，無 RTL counterpart） | `protocol-bfm` | no | 見 `examples/axi_lite_slave_bfm/`。 |
| APB slave BFM | `protocol-bfm` | yes / no | 見 `examples/apb_slave_bfm/` 作為結構樣板。 |
| Network Interface 橋接 AXI4 ↔ NoC packet 介面 | `protocol-bfm` | yes | Bus-attached、雙協定、預期同時有 RTL 與 C model 兩種實作。 |

### 「BFM」—— 本 plugin 對這個詞的定義

業界用法：「BFM」/「VIP」通常 = **純 verification model**，沒有 RTL counterpart。

本 plugin 的 `protocol-bfm` mode：任何**外部介面是 wire-level 協定契約**、需要 cycle-accurate 文件的 block。這個 block 可能：

- 有 RTL 實作（例如你的 AXI master 設計，會做成矽晶片）
- 有 C/C++/SystemC/UVM 行為模型（例如你的 AXI master 的 reference C model 用於 co-sim）
- 兩者都有（最常見的工作流）
- 只有 BFM（業界標準的 verification IP）

Mode 名 `protocol-bfm` 反映**模板結構**（cycle 級協定嚴謹度、transaction/channel API 介面），而非**產物類別**（純驗證用）。當 `has-rtl-counterpart: yes` 時，spec 從單一事實源頭服務兩種實作，並明文記錄哪些功能（response delay、fault injection、active/passive mode）是 test-only BFM knob、哪些行為 RTL counterpart 也照樣實作。

---

## 快速開始

### Greenfield（無既有材料）—— `behavioral-block`

```
/spec-init my_timer
```

被問到時選 (A) Behavioral block。執行 Capture-phase 訪談（問 IP 用途、bus interface、clock/reset domain、3–8 個 features），寫 `MODE.md`，然後在 `./spec/my_timer/` 產生 6-file OpenTitan 風格 skeleton。

### Greenfield —— `protocol-bfm`

```
/spec-init my_axi_master
```

選 (B) Protocol BFM / VIP。額外問題：協定與版本、role（master/slave/both）、active/passive 支援、configuration knobs、**以及是否會有 RTL counterpart**。寫 `MODE.md`（含 `has-rtl-counterpart: yes/no`），然後在 `./spec/my_axi_master/` 產生 9–11 file skeleton。

### Brownfield（既有 spec / RTL / hjson）

```
/spec-import path/to/old_spec_or_rtl/
```

**v0.3.0 僅支援 behavioral-block mode**。讀既有 markdown / SystemVerilog / Verilog / hjson，重組為 6-file layout，含機械性抽取（ports、parameters、register reset values、FSM state names、SVA assertions）。衝突浮現於 `IMPORT_REPORT.md`。

BFM-mode brownfield（匯入既有 BFM 原始碼）：用 `/spec-init` greenfield 路徑加手動 port 填入。v2 方向見 [BFM mode design draft §11.Q6](./plan/BFM_MODE_DESIGN.md)。

### 後續 commands（mode-aware）

| Command | 何時跑 | 效果 |
|---|---|---|
| `/spec-status` | 任何時候 | 讀 `MODE.md`。報告當前 D-stage、對齊 mode-appropriate checklist。Mode-conditional 項目（例如 `D1.bfm.*`）只在適用時計分。Read-only。 |
| `/spec-lint` | Phase 2 任何時候 | 機械性 drift 檢查。LINT-001..007 永遠跑；LINT-BFM-001..005 在 `protocol-bfm` mode 額外跑。Read-only。 |
| `/spec-review` | 主張 D1 之前 | 透過 `spec-reader` subagent 跑 reader test。題庫依 mode 對應：`reader_test.md` 給 behavioral-block；`bfm_reader_test_bank.md` 給 protocol-bfm。 |
| `/spec-gate D1` | D1 看起來好了 | 走過所有 D1 checklist（mode-conditional N/A 項目跳過）。永久 waiver 在 `WAIVERS.md`。 |
| `/spec-gate D2` / `D3` | 後期 | 同樣的 protocol，給後面的 gate 用。 |
| `/spec-help` | 任何時候 | Workflow card 含 mode 行，依當前狀態建議下一步 command。 |

---

## `/spec-init` 產出什麼

### `behavioral-block` mode（6 個檔案）

```
<ip_name>/
├── MODE.md                     # mode: behavioral-block
├── README.md                   # 摘要索引——Overview、Features、Description、Compatibility
├── doc/
│   ├── theory_of_operation.md  # Block diagram、datapath、FSM、errors（HW + DV + SW）
│   ├── programmers_guide.md    # 初始化、use cases、error handling、IRQs（SW + DV）
│   ├── interfaces.md           # Port table、parameters、IRQs（SoC integrator）
│   └── registers.md            # CSR map（SW + DV）
└── dv/
    └── plan.md                 # Testpoints、coverage model、sec_cm（DV）
```

小 block 可以把 `theory_of_operation.md` 與 `programmers_guide.md` 收進 README；複雜 block 可以把 `theory_of_operation.md` 拆成多個 `doc/` 下的檔案。

### `protocol-bfm` mode（9–11 個檔案）

```
<ip_name>/
├── MODE.md                     # mode: protocol-bfm; has-rtl-counterpart: yes/no
├── README.md                   # 摘要索引——Overview、Features、Description、Compatibility
├── doc/
│   ├── theory_of_operation.md  # 兩節：## BFM internal architecture（永遠必填）+ ## RTL internal architecture（when has-rtl-counterpart: yes）
│   ├── signal_interface.md     # 每根 wire 完整指定（BFM mode 取代 interfaces.md）
│   ├── pin_level_reset.md      # 每根 wire 在 reset 期間/之後的值
│   ├── protocol_rules.md       # Cycle 級規則：per-channel + cross-channel + reset + config-knob
│   ├── channel_handshake.md    # Channel 依賴圖 + may/must 箭頭慣例
│   ├── transaction_api.md      # 高階 testbench API（~95% 測試只用這個）
│   ├── channel_api.md          # Phase 級 testbench API（5% corner-case 測試）
│   ├── active_passive_mode.md  # Active vs Passive capability + config knob
│   └── registers.md            # 選用——只在 block 有軟體可見 CSR 時生成
└── dv/
    └── plan.md                 # Testpoints + 對應每條 protocol-rule ID 的 coverage hooks
```

`registers.md` 在 BFM mode 是 optional（多數 BFM 沒有軟體可見 CSR）。有的話 standard register lints/gates 自動套用。

---

## 各檔案的職責與讀者

Spec 拆成多個檔案是因為每個檔案針對特定讀者。下表把每個檔案對到它解決的問題、讀者、與實作對應。

### `behavioral-block` mode

| 檔案 | 解決什麼 | 誰會讀 | 對應實作什麼 |
|---|---|---|---|
| `README.md` | 1 分鐘搞清楚「這是什麼 IP」+ Features 列表 | 任何第一次接觸的人 | 不直接對應；引導讀者進後續細節 |
| `doc/theory_of_operation.md` | Block 內部架構（datapath、FSMs、error handling、performance） | RTL 設計者 + DV 工程師 + 部分 SW 作者 | RTL submodule hierarchy + 行為演算法 |
| `doc/programmers_guide.md` | 軟體 driver 視角：初始化序列、use cases、error 回應、IRQ 處理 | Firmware/driver 作者 | C driver code 結構 |
| `doc/interfaces.md` | Port table、parameters、clocks、resets、IRQs、alerts | SoC integrator + RTL 設計者 | RTL module port list |
| `doc/registers.md` | CSR memory map + 每個 register 的 field 拆解 + reset 值 | SW 作者 + RTL 設計者 | RTL register file + driver register accesses |
| `dv/plan.md` | Verification scope + testpoints + coverage model + sec_cm | DV 工程師 | UVM testbench + assertions + covergroups |

### `protocol-bfm` mode

| 檔案 | 解決什麼 | 誰會讀 | 對應實作什麼 |
|---|---|---|---|
| `MODE.md` | Plugin metadata（mode + has-rtl-counterpart flag） | Plugin 工具 | 不對應實作 |
| `README.md` | 1 分鐘搞清楚「這是什麼 BFM」+ Features | 任何第一次接觸的人 | 不直接對應 |
| `doc/theory_of_operation.md` (§BFM internal) | BFM 內部結構：driver / monitor / sequencer / configuration store / sub-modules | C model 實作者 | C++ class 內部切分 |
| `doc/theory_of_operation.md` (§RTL internal) | 當 `has-rtl-counterpart: yes` 時——RTL block 結構、pipeline、register file、RTL-vs-BFM equivalence 表 | RTL 設計者 + C model 實作者（cross-check 用） | RTL submodule hierarchy |
| `doc/signal_interface.md` | 每根 wire 的契約（name、dir、width、sample edge、clock domain、reset linkage、optional-in-protocol、BFM-supports）。取代 `interfaces.md`。 | RTL DUT integrator + C model integrator + DPI-C bridge 作者 | RTL port list / C++ class member variables |
| `doc/pin_level_reset.md` | 每根 wire 在 reset 期間與剛 deassert 後的值，per reset signal（多 domain 處理） | 兩種 implementer | C model `reset()` function 每行 / RTL flop reset values |
| `doc/protocol_rules.md` | 所有 protocol invariants 列為可測試的 rule（`<PROTO>_<ROLE>_<CHANNEL>_<NAME>` IDs，含 severity FAIL/RECOMMEND + 選填 ARM SVA equivalent） | DV 工程師（→ SVA assertions）+ 實作者（→ if/else 條件） | RTL `assert property` / C model protocol-checker |
| `doc/channel_handshake.md` | Cross-channel 與跨協定依賴圖 + may/must 箭頭 + deadlock 避免說明 | 設計 FSM / 互鎖的實作者 | RTL FSM transition guards / C model sequencer ordering |
| `doc/transaction_api.md` | 高階 testbench API（95% 測試呼叫的部分）：`apply_*`、`expect_*`、`set_*`、`get_*`、`reset_state`。每個 method 含 Signature / Preconditions / Side effects / Return / Error modes + Equivalent channel API decomposition。 | 測試作者 | C++ public method 介面 |
| `doc/channel_api.md` | Phase 級細粒度 API（5% corner-case 測試）：per-channel `begin_phase`、`assert_valid`、`wait_for_ready`、`end_phase`。 | 進階測試作者 | C++ public method 介面（細粒度） |
| `doc/active_passive_mode.md` | Capability 表（active 驅動 wire；passive 只監聽）+ mode-switch knob + 常見 testbench 設定 | Testbench integrator | `bfm_mode` flag + driver enable/disable |
| `doc/registers.md`（選用） | CSR memory map（block 有軟體可見 CSR 時） | SW 作者 + 實作者 | 與 behavioral-block 同 |
| `dv/plan.md` | Testpoints + covergroups + ABV（每條 FAIL rule 對應一條 SVA）+ FPV strategy | DV 工程師 | UVM testbench + SVA library + covergroups |

### 各 implementer 的閱讀路徑

**RTL 實作者**（寫 synthesizable RTL）：
1. `signal_interface.md` → module port list
2. `pin_level_reset.md` → flop reset values
3. `protocol_rules.md` → SVA assertions + 內部邏輯條件
4. `channel_handshake.md` → cross-channel FSM 設計
5. `theory_of_operation.md §RTL internal` → submodule hierarchy
6. `registers.md`（如有） → register file 設計

**C/C++ model 實作者**（寫 BFM）：
1. 與上面 1–4 共用（共享契約）
2. `theory_of_operation.md §BFM internal` → driver/monitor/sequencer 切分
3. `transaction_api.md` + `channel_api.md` → public method 介面
4. `active_passive_mode.md` → mode-switch 邏輯
5. `registers.md`（如有） → CSR file 模擬後端

**Testbench / 測試作者**（寫測試）：
1. `README.md` → 知道有哪些 features 要測
2. `transaction_api.md` → 知道 BFM 暴露哪些 method
3. `channel_api.md` → 寫 corner-case 測試時用
4. `active_passive_mode.md` → 配置單一/多 BFM 拓樸
5. `dv/plan.md` → 知道要寫哪些 testpoints
6. `registers.md`（如有） → CSR-driven 配置測試

**DV 工程師**（寫 scoreboard / assertion）：
1. `protocol_rules.md` → 1:1 SVA assertion library
2. `dv/plan.md` → covergroup 設計
3. `channel_handshake.md` → cross-channel assertions

**SoC integrator / firmware 作者**：
1. `signal_interface.md`（或 behavioral-block 的 `interfaces.md`） → wire 連線
2. `pin_level_reset.md` → 上電假設
3. `registers.md` → driver code

---

## 建議的撰寫順序

Skill 強制的撰寫順序，避免循環改寫。

### `behavioral-block` mode

1. **README** —— Overview + Features（5 分鐘；強迫想清楚 elevator pitch）
2. **`interfaces.md`** —— 把 ports、parameters、clocks 釘下來
3. **`registers.md`** —— 把 programming model 釘下來
4. **`theory_of_operation.md`** —— 現在有 signals + registers 可以引用了
5. **`programmers_guide.md`** —— 現在可以寫 use case，引用真的 register
6. **`dv/plan.md`** —— 現在可以 scope coverage

先寫 `theory_of_operation.md` 感覺很自然（它是「主」doc），但每次 port 或 register 名稱變動就要回頭改一次。

### `protocol-bfm` mode

1. **README** —— Overview + protocol + role + active/passive
2. **`signal_interface.md`** —— 每根 wire 釘下來（這是其他檔案的 anchor；channel name 在這裡宣告，protocol-rule ID 引用它）
3. **`pin_level_reset.md`** —— 每根 wire 的 reset 值（連動 signal_interface）
4. **`protocol_rules.md`** —— 每個 channel 的 rule 表；每條 rule 會變成一條 SVA assertion
5. **`channel_handshake.md`** —— Cross-channel 依賴；抓 deadlock
6. **`transaction_api.md`** —— 高階 test API（~95% 測試）
7. **`channel_api.md`** —— Phase 級 API 給剩下 5%
8. **`active_passive_mode.md`** —— Capability 表 + mode knob
9. **`theory_of_operation.md`** —— 整合 BFM 內部架構與 RTL 內部架構（如有 RTL counterpart）
10. **`dv/plan.md`** —— 對應每條 protocol-rule ID 的 coverage hooks

不要在 `signal_interface.md` 之前寫 `protocol_rules.md` —— protocol-rule ID 引用 signal_interface 宣告的 channel name。

---

## 工作流程（D0 → D3）

### Phase 1 — Capture（target: D0）

目標：完整 skeleton（behavioral-block 6 檔案，protocol-bfm 9–11 檔案），所有沒釘下來的細節都標 `TODO(designer):`。

- **Greenfield path**：`/spec-init <ip_name> [output_dir]` **先問 IP 類別**（behavioral-block vs protocol-bfm），再問 mode-specific 後續問題。寫 `MODE.md`、依模板生 skeleton。不會自己編使用者沒講過的 features。
- **Brownfield path**：`/spec-import <input_path> [output_dir]`（v0.3.0 只支援 behavioral-block）讀既有材料、重組為 6-file layout。RTL 是 ports/parameters/registers/FSM 的 canonical；既有 prose 是 narrative 的 canonical。衝突浮現於 `IMPORT_REPORT.md`。BFM-mode brownfield：用 greenfield + 手動填入。

兩條路徑匯流到同樣的 Phase 2 / 3 / 4 工作流程。

### Phase 2 — Iterate（D0 → D1）

依推薦順序逐節寫。寫每個檔案前，skill 讀對應 template 與該 mode 的 worked example。用 `/spec-status` 追進度；用 `/spec-lint` 早期抓 drift。

### Phase 3 — Reader Test（D1 sign-off）

`/spec-review` 在新 context 下生 `spec-reader` subagent，唯讀存取 spec 檔案。題庫依 mode 對應：

- **`behavioral-block`** → 從 `reader_test.md` 抽 8–12 題（universal bank + block-specific bank：DMA / FIFO / configurable / security-critical）
- **`protocol-bfm`** → 從 `bfm_reader_test_bank.md` 抽 8–12 題（universal-BFM bank：protocol rules / handshake dependencies / pin-level reset / API completeness / mode coverage / outstanding tracking / ID rules / error injection + block-specific banks：master / slave / passive / ID-bearing / burst / cache-coherent）

PASS 需要 spec 中有可引用的句子；NOT_ANSWERED 與 AMBIGUOUS 是要補的洞。

Subagent 的隔離至關重要：spec 作者**不能 un-know** 自己寫的東西。Reader test 就是要把作者自己看不到的盲點挖出來。

### Phase 4 — Stage gating（D1 → D2 → D3）

`/spec-gate <D1|D2|D3>` 走目標 gate 的 checklist。項目有穩定 anchor（例如 `D1.too.fsm`、`D1.bfm.protocol_rules`），讓 `WAIVERS.md` 中的 waiver 跨 session、跨 checklist 重排都不失效。Mode-conditional 項目標 `**Applies when:**` 條件；不適用的項目計為 N/A，不影響 gate 完成度。

---

## Stage gate 定義

| Stage | 定義 | 何時觸發 |
|---|---|---|
| **D0** | 概念確立；大致形狀知道 | 訪談完 / `/spec-init` 後 |
| **D1** | Spec ~90% 完成；DV 可以開始寫 testbench、RTL 作者可以開始寫 stub、C model 作者可以開始實作 | RTL 或 BFM 實作開始之前 |
| **D2** | RTL/BFM 功能正常、與 spec 一致；spec 凍結（除澄清外） | 實作存在、smoke test 通過 |
| **D3** | Sign-off ready 進 tape-out（RTL）或廣泛部署（BFM） | 所有 review 完成、lint 乾淨、FPV 證明 (where applicable) |

完整每階段 checklist（含 mode-conditional `D1.bfm.*`、`D2.bfm.*`、`D3.bfm.*` 項目）見 [`stage_gates.md`](./plugins/hw-spec-author/skills/hw-spec-author/references/process/stage_gates.md)。

沒有「D1 minus」這種東西。「D1 但缺一個 FSM」就是 D0，直到那個缺漏的 artifact 被補上、或被 explicit waive（一句 rationale 寫在 `WAIVERS.md`）。

---

## Subagent：`spec-reader`

Plugin 內含一個 subagent 在 [`agents/spec-reader.md`](./plugins/hw-spec-author/agents/spec-reader.md)。`/spec-review` 透過 Task tool 呼叫它。

- **工具限縮為** `Read`、`Grep`、`Glob` —— 不能寫、不能 shell、不能上網。
- **新 context** —— 不接觸 authoring history。
- **嚴格隔離** —— 只能根據 spec 文字回答；引用支撐句子；不知道時回 `NOT_ANSWERED`，不從常識或同類 IP 知識補洞。

如果你發現自己在親自回答 reader test 的問題、而不是分派給 subagent，你就把 reader test 想消除的 bias 又拉回來了。

---

## Worked examples

### `behavioral-block` mode：`wctmr`

[`plugins/hw-spec-author/examples/wctmr/`](./plugins/hw-spec-author/examples/wctmr/) —— 64-bit Wallclock Timer 的 D1 級 spec（兩個 compare slot、可配置 prescaler、APB slave）。

展示：標準 6-file layout、依讀者分層的 prose（ToO 給 HW/DV/SW；programmer's guide 給 SW；interfaces 給 SoC integrator）、單一事實源頭紀律、具體 corner-case 涵蓋（counter wrap、software write 原子性、IRQ re-fire 在 clear-while-true、prescale 超範圍 clamp、APB-transaction 中途 reset）、以及 [`READER_TEST_LOG.md`](./plugins/hw-spec-author/examples/wctmr/READER_TEST_LOG.md) 紀錄真實 `/spec-review` 從 5 PASS / 5 gaps 到 **10 / 10 PASS** 的全過程。

### `protocol-bfm` mode：`axi_lite_slave_bfm`

[`plugins/hw-spec-author/examples/axi_lite_slave_bfm/`](./plugins/hw-spec-author/examples/axi_lite_slave_bfm/) —— AXI-Lite slave BFM 的 D1 級 spec，`has-rtl-counterpart: yes`。

展示：9-file BFM layout、ARM 風格 protocol-rule rows 含 severity + ARM SVA equivalent column、channel-handshake Mermaid 依賴圖、Transaction API + Channel API 切分、`theory_of_operation.md §BFM` 的 driver/monitor/sequencer 內部架構、`theory_of_operation.md §RTL` 的 RTL counterpart 架構與明文 RTL-vs-BFM 行為等價表。

### `protocol-bfm` mode：`apb_slave_bfm`

[`plugins/hw-spec-author/examples/apb_slave_bfm/`](./plugins/hw-spec-author/examples/apb_slave_bfm/) —— BFM template 對無 channel 協定的可攜性測試（APB 沒有 AXI 風格的 channel grouping；它是 SETUP→ACCESS 兩 phase 共用一組 signal）。

展示：rule ID 格式 `<PROTO>_<ROLE>_<NAME>`（無 channel 協定的 3-field 變體）、`channel_api.md` 中的 SETUP/ACCESS phase state machine（per-phase 變體、對應 AXI 的 per-channel state machine）、`CFG_WAIT_STATES_BOUND` rule 的 ASCII timing diagram。

---

寫新 spec 之前**先讀對應 mode 的 example**。Prose 語氣、表格密度、讀者分層，從具體 reference 模仿比從 template 推導容易。

---

## 輸出語言

Spec 內容預設 **English**。這對齊業界慣例（OpenTitan、ARM AMBA、SiFive、RISC-V references 都英文）與下游用途（cross-team review、paper、patent）。Spec 之外的對話可以用任何語言；只有產出的 spec 檔案預設英文。要改用其他語言，在對話中明說即可。

---

## 自動觸發

Skill 也會從自然語言觸發（不需要打 slash command）：

- "write a spec for ..."
- "draft a micro-arch spec"
- "design document for the ... module"
- "write a BFM spec for ..."
- "幫我寫一個 ... 的 spec"

兩條路徑都接到相同 workflow（並早期問 IP 類別問題）。

---

## 設計哲學

Plugin 圍繞四個性質，每個 gate 都檢：

1. **依讀者分層**：每個 section 標明主要讀者、為他寫。沒有混雜讀者的段落。
2. **單一事實源頭**：每個事實只活在一處；其他地方 cross-reference。
3. **階段閘**：D0 / D1 / D2 / D3 有明確 checklist 與穩定 anchor。沒有「D1 minus」。
4. **可被讀者測試**：新 context 的讀者能單從 spec 文字回答具體 corner-case 問題。

完整 rationale 讀 [`SKILL.md`](./plugins/hw-spec-author/skills/hw-spec-author/SKILL.md) 與 [`references/process/`](./plugins/hw-spec-author/skills/hw-spec-author/references/process/) 下的文件。BFM-mode design rationale 特別在 [`plan/BFM_MODE_DESIGN.md`](./plan/BFM_MODE_DESIGN.md)。

---

## 內含元件

- 1 個 skill：`hw-spec-author`（workflow engine + 13 個 templates + 7 個 process docs）
- 2 個 subagents：`spec-reader`（隔離 context 的 reader 給 reader test 用）、`implementer-reviewer`（paradigm-paired implementer reviewer 給 `/spec-implementer-review` 用）
- 8 個 slash commands：`/spec-init`、`/spec-import`、`/spec-status`、`/spec-review`、`/spec-implementer-review`、`/spec-lint`、`/spec-gate`、`/spec-help`
- 3 個 worked examples：`wctmr`（behavioral-block）、`axi_lite_slave_bfm`（protocol-bfm with RTL counterpart）、`apb_slave_bfm`（protocol-bfm 對無 channel 協定的可攜性測試）

---

## Repository layout

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
│       │   ├── spec-reader.md
│       │   └── implementer-reviewer.md
│       ├── commands/
│       │   ├── spec-init.md
│       │   ├── spec-import.md
│       │   ├── spec-status.md
│       │   ├── spec-review.md
│       │   ├── spec-implementer-review.md
│       │   ├── spec-lint.md
│       │   ├── spec-gate.md
│       │   └── spec-help.md
│       ├── examples/
│       │   ├── wctmr/                          # behavioral-block worked example
│       │   ├── axi_lite_slave_bfm/             # protocol-bfm with RTL counterpart
│       │   └── apb_slave_bfm/                  # protocol-bfm portability test
│       └── skills/
│           └── hw-spec-author/
│               ├── SKILL.md
│               └── references/
│                   ├── process/
│                   │   ├── stage_gates.md
│                   │   ├── reader_test.md           # behavioral-block 題庫
│                   │   ├── bfm_reader_test_bank.md  # protocol-bfm 題庫
│                   │   ├── writing_principles.md
│                   │   ├── slide_style.md           # 簡報風格（選用）
│                   │   ├── implementer_review.md    # 多 agent paradigm-paired review（BFM + RTL）
│                   │   └── rtl_extraction.md
│                   └── templates/
│                       ├── 01_summary.md            # behavioral-block templates
│                       ├── 02_theory_of_operation.md
│                       ├── 03_programmers_guide.md
│                       ├── 04_interfaces.md
│                       ├── 05_registers.md
│                       ├── 06_dv_plan.md
│                       └── bfm/                     # protocol-bfm templates
│                           ├── 02_theory_of_operation.md
│                           ├── 02b_protocol_rules.md
│                           ├── 02c_channel_handshake.md
│                           ├── 02d_pin_level_reset.md
│                           ├── 03b_transaction_api.md
│                           ├── 03c_channel_api.md
│                           ├── 03d_active_passive_mode.md
│                           └── 04b_signal_interface.md
├── plan/
│   └── BFM_MODE_DESIGN.md                      # protocol-bfm mode design 歷程
├── LICENSE
└── README.md
```

---

## 授權

Apache 2.0。見 [LICENSE](./LICENSE)。

---

## 致謝

`behavioral-block` mode 大量參考 [OpenTitan / lowRISC Comportability 框架](https://opentitan.org/book/doc/contributing/hw/comportability/index.html)。Stage-gate 概念來自 OpenTitan 專案的 signoff checklist。`protocol-bfm` mode 借用 ARM AMBA Protocol Checker User Guides 的 rule-by-rule 結構、ARM AMBA AXI Specification §A2 的 channel handshake 依賴慣例、UVM agent architecture（driver/monitor/sequencer + active/passive）、與 OSVVM / Aldec / Cadence 三層 BFM API 慣例（signal interface → channel API → transaction API）。

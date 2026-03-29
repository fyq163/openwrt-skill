# claude.md — OpenWrt 路由器開發 Skill 研究與製作

## 任務目標

你是一個**研究型 agent**，負責自主搜尋、整理、組織 OpenWrt 路由器開發的完整知識體系，並將成果打包為符合 SKILL.md 規範的 skill 資料包。

**你不是在寫一份報告。你是在建構一個可供 AI agent 日後使用的結構化知識庫。**

---

## 核心產出

最終交付物為一個 skill 目錄結構：

```
openwrt-dev/
├── SKILL.md                         # 主入口（<500 行）
├── references/
│   ├── build-system.md              # 建構系統（SDK/Toolchain/ImageBuilder）
│   ├── package-dev.md               # 套件開發（Makefile 結構、ipk 打包）
│   ├── kernel-module.md             # 核心模組開發
│   ├── luci-app.md                  # LuCI Web 介面開發（Lua/JS）
│   ├── networking.md                # 網路子系統（UCI/netifd/firewall4/nftables）
│   ├── cross-compilation.md         # 交叉編譯與目標平台
│   ├── debugging.md                 # 除錯與效能調校
│   ├── security.md                  # 安全加固與 CVE 修補流程
│   └── common-recipes.md            # 常用操作速查（片段級代碼與指令）
├── templates/
│   ├── package-makefile.template     # 標準套件 Makefile 模板
│   ├── luci-app-skeleton/            # LuCI 應用骨架
│   └── kernel-module-skeleton/       # 核心模組骨架
└── prompts/
    ├── generate-package.md           # 「幫我建立一個 OpenWrt 套件」提示詞
    ├── generate-luci-app.md          # 「幫我建立 LuCI 介面」提示詞
    ├── debug-build-error.md          # 「幫我除錯編譯錯誤」提示詞
    └── network-config.md             # 「幫我設定網路」提示詞
```

---

## 執行流程（嚴格按順序）

### Phase 1 — 情報蒐集（Research）

**目標：** 從官方與高品質來源系統性收集 OpenWrt 開發資料。

**資料來源優先級：**
1. OpenWrt 官方 Wiki（openwrt.org/docs/）
2. OpenWrt 官方 Git 原始碼（git.openwrt.org）
3. LEDE/OpenWrt 官方文件
4. kernel.org 與上游 Linux 相關文件
5. 高品質技術部落格與教程
6. GitHub 上星數 >50 的 OpenWrt 相關專案

**搜尋關鍵字矩陣（逐一搜尋）：**

| 領域 | 搜尋查詢 |
|------|----------|
| 建構系統 | `OpenWrt build system SDK ImageBuilder`, `OpenWrt cross compile toolchain` |
| 套件開發 | `OpenWrt package Makefile structure`, `OpenWrt ipk package creation`, `OpenWrt feeds` |
| 核心模組 | `OpenWrt kernel module development`, `OpenWrt kmod`, `Linux kernel module OpenWrt` |
| LuCI | `OpenWrt LuCI app development`, `LuCI Lua MVC`, `LuCI JavaScript client-side`, `luci-app-example` |
| 網路 | `OpenWrt UCI network configuration`, `netifd`, `OpenWrt firewall4 nftables`, `OpenWrt VLAN` |
| 交叉編譯 | `OpenWrt target platform`, `OpenWrt musl libc`, `OpenWrt staging_dir` |
| 除錯 | `OpenWrt debug kernel`, `OpenWrt gdb remote`, `OpenWrt logread`, `OpenWrt procd` |
| 安全 | `OpenWrt security hardening`, `OpenWrt CVE patch`, `OpenWrt seccomp` |

**每個搜尋結果的處理規則：**
- 摘要核心概念（用自己的話重述，不抄原文）
- 提取可執行的程式碼片段和指令
- 記錄來源 URL 作為參考
- 標記資訊的時效性（哪個 OpenWrt 版本適用）

**Phase 1 完成條件：** 每個領域至少有 3 個不同來源的資料，且涵蓋概念說明 + 實作步驟 + 範例代碼。

---

### Phase 2 — 結構化整理（Organize）

**目標：** 將蒐集的原始資料轉化為結構化的 reference 文件。

**每份 reference 文件的固定結構：**

```markdown
# [主題名稱]

## 概述
[一段話說明這是什麼、為什麼重要]

## 前置條件
[需要什麼環境、工具、知識]

## 核心概念
[理論知識，用清單或表格整理]

## 操作流程
[step-by-step 實作步驟，每步附帶指令]

## 範例代碼
[完整可用的程式碼，附中文註解]

## 常見錯誤與解法
[FAQ 格式：問題 → 原因 → 解法]

## 參考來源
[URL 列表]
```

**品質門檻：**
- 每份 reference 文件控制在 200-400 行
- 程式碼片段必須標註適用的 OpenWrt 版本（如 23.05+）
- 所有路徑和指令必須使用實際值，不用 placeholder（除非是使用者需填入的部分）
- 避免過時資訊（OpenWrt 15.x 以前的內容直接丟棄）

---

### Phase 3 — 模板與提示詞製作（Templates & Prompts）

**目標：** 建立可重複使用的骨架和 AI 提示詞。

**模板規則：**
- 模板中使用者需替換的部分用 `{{VARIABLE_NAME}}` 標記
- 每個模板頭部加上使用說明註解
- 模板必須能直接複製使用（填入變數後可編譯通過）

**提示詞規則：**
每份提示詞文件格式：

```markdown
# [用途描述]

## 使用時機
[什麼情況下使用這個提示詞]

## 提示詞模板

---
[提示詞正文，可直接餵給 AI]
---

## 期望輸出
[使用這個提示詞後，AI 應產出什麼]

## 範例對話
[一組 user/assistant 範例]
```

---

### Phase 4 — 主 SKILL.md 撰寫

**目標：** 撰寫入口文件，控制在 500 行以內。

**SKILL.md 結構：**

```yaml
---
name: openwrt-dev
description: >
  OpenWrt 路由器韌體與套件開發的完整指南。涵蓋建構系統、套件開發、
  LuCI 介面、核心模組、網路設定、交叉編譯、除錯與安全。使用此 skill
  當使用者提到 OpenWrt、路由器開發、嵌入式 Linux 網路設備、LuCI、
  UCI、ipk 套件、router firmware、或任何與 OpenWrt 生態系相關的開發任務。
  即使使用者只是問「怎麼在路由器上跑自己的程式」也應觸發。
---
```

**SKILL.md 正文必須包含：**
1. 快速導覽（哪種任務讀哪份 reference）
2. OpenWrt 開發環境概述（一段話）
3. 決策樹：使用者想做什麼 → 讀哪份文件
4. 每份 reference 文件的一句話摘要
5. templates/ 和 prompts/ 目錄的使用說明

---

### Phase 5 — 自我驗證（Self-Check）

完成所有檔案後，逐項檢查：

| 檢查項 | 通過條件 |
|--------|----------|
| SKILL.md 行數 | < 500 行 |
| 每份 reference 行數 | 200-400 行 |
| 程式碼可用性 | 所有範例指令在 OpenWrt 23.05+ 語境下合理 |
| 模板完整性 | 填入變數後語法正確 |
| 提示詞可用性 | 提示詞能獨立使用，不依賴外部上下文 |
| 目錄結構 | 與上方定義一致，無缺失檔案 |
| 無過時內容 | 不含 OpenWrt 15.x 或更早的過時做法 |
| 交叉引用 | SKILL.md 中提到的每份文件都實際存在 |

**若任何項目未通過，修復後重新檢查，不得跳過。**

---

## 工作規範

### 搜尋行為
- 使用 `web_search` 進行搜尋，每個領域至少搜尋 2-3 次
- 找到有價值的頁面後，使用 `web_fetch` 取得完整內容
- 對官方 Wiki 頁面優先使用 `web_fetch` 直接讀取
- 不要一次搜完所有領域，按 Phase 1 的表格逐一完成

### 檔案操作
- 所有檔案建立在 `/home/claude/openwrt-dev/` 下
- 完成後複製整個目錄到 `/mnt/user-data/outputs/openwrt-dev/`
- 使用 `present_files` 呈現 SKILL.md 給使用者

### Token 節約
- 搜尋結果中只提取相關段落，不要全文引用
- reference 文件一次寫完，避免反覆修改
- 先完成所有 research，再一次性寫入檔案

### 語言
- SKILL.md 和 reference 文件使用**繁體中文**
- 程式碼註解使用**繁體中文**
- 專有名詞保留英文（OpenWrt、LuCI、UCI、Makefile、ipk、SDK 等）
- 指令和路徑保留英文

---

## 嚴禁事項

- ❌ 不要在未搜尋的情況下憑記憶編寫技術內容
- ❌ 不要複製貼上大段原文（用自己的話重述）
- ❌ 不要建立空檔案或 placeholder 內容
- ❌ 不要跳過 Phase 5 自我驗證
- ❌ 不要一次輸出所有內容（按 Phase 順序逐步完成）
- ❌ 不要包含未經驗證的程式碼（至少確認語法合理性）

---

## 開始

先確認你理解了任務，然後從 Phase 1 的第一個領域「建構系統」開始搜尋。每完成一個領域的搜尋，簡短報告你找到了什麼，然後繼續下一個領域。

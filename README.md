# OpenWrt 開發技能包 (openwrt-dev)

> 為 AI 開發助手打造的 OpenWrt 路由器開發結構化知識庫

[![OpenWrt](https://img.shields.io/badge/OpenWrt-23.05%2B-00B5E2?logo=openwrt&logoColor=white)](https://openwrt.org/)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Language](https://img.shields.io/badge/語言-繁體中文-orange.svg)](#)

---

## 📋 專案簡介

本專案是一個 **AI agent 可用的結構化知識庫**，系統性地整理了 OpenWrt 路由器韌體與套件開發的完整知識體系。所有內容以 OpenWrt 23.05+（核心 5.15+）為基準，涵蓋從建構系統到安全加固的各個面向。

**主要特色：**
- 9 份結構化參考文件，覆蓋所有核心開發領域
- 3 套即用即改的開發模板（套件、LuCI 應用、核心模組）
- 4 份 AI 提示詞模板，可直接餵給 AI 助手使用
- 繁體中文撰寫，專有名詞保留英文

---

## 📁 目錄結構

```
openwrt-dev/
├── SKILL.md                              # 主入口（導覽 + 決策樹）
├── references/
│   ├── build-system.md                   # 建構系統（Buildroot/SDK/ImageBuilder）
│   ├── package-dev.md                    # 套件開發（Makefile 結構、ipk 打包）
│   ├── kernel-module.md                  # 核心模組開發（kmod、Kbuild）
│   ├── luci-app.md                       # LuCI Web 介面開發（JavaScript API）
│   ├── networking.md                     # 網路子系統（UCI/netifd/firewall4）
│   ├── cross-compilation.md              # 交叉編譯與目標平台
│   ├── debugging.md                      # 除錯與效能調校
│   ├── security.md                       # 安全加固與 CVE 修補
│   └── common-recipes.md                 # 常用操作速查
├── templates/
│   ├── package-makefile.template         # 標準套件 Makefile 模板
│   ├── luci-app-skeleton/                # LuCI 應用骨架
│   │   ├── Makefile
│   │   ├── htdocs/luci-static/resources/view/app/settings.js
│   │   └── root/usr/share/{luci,rpcd}/...
│   └── kernel-module-skeleton/           # 核心模組骨架
│       ├── Makefile
│       └── src/{module.c, Kbuild}
└── prompts/
    ├── generate-package.md               # 「幫我建立 OpenWrt 套件」
    ├── generate-luci-app.md              # 「幫我建立 LuCI 介面」
    ├── debug-build-error.md              # 「幫我除錯編譯錯誤」
    └── network-config.md                 # 「幫我設定網路」
```

---

## 🚀 快速開始

### 作為 AI Skill 使用

將 `openwrt-dev/` 目錄放入你的 AI 助手 skill 目錄中：

```bash
# Claude Code
cp -r openwrt-dev/ ~/.claude/skills/

# Copilot
cp -r openwrt-dev/ ~/.copilot/skills/
```

AI 助手會在使用者提到 OpenWrt、路由器開發、LuCI、UCI 等關鍵字時自動載入此技能包。

### 直接閱讀

從 [SKILL.md](openwrt-dev/SKILL.md) 開始，按照決策樹找到你需要的參考文件。

### 使用模板

```bash
# 建立新套件
cp openwrt-dev/templates/package-makefile.template my-package/Makefile
# 將所有 {{VARIABLE}} 替換為實際值

# 建立 LuCI 應用
cp -r openwrt-dev/templates/luci-app-skeleton/ luci-app-myapp/
# 重命名並替換變數

# 建立核心模組
cp -r openwrt-dev/templates/kernel-module-skeleton/ kmod-mymod/
# 填入模組名稱和邏輯
```

---

## 📚 參考文件總覽

| 文件 | 內容 | 適用場景 |
|------|------|----------|
| [build-system.md](openwrt-dev/references/build-system.md) | Buildroot、SDK、ImageBuilder、feeds | 編譯韌體或套件 |
| [package-dev.md](openwrt-dev/references/package-dev.md) | Makefile 結構、PKG_* 變數、ipk 打包 | 開發新套件 |
| [kernel-module.md](openwrt-dev/references/kernel-module.md) | KernelPackage、Kbuild、Netfilter | 寫核心模組 |
| [luci-app.md](openwrt-dev/references/luci-app.md) | JS API、form.Map、menu.d、ACL | 做 Web 管理介面 |
| [networking.md](openwrt-dev/references/networking.md) | UCI、netifd、firewall4、DSA VLAN | 設定網路 |
| [cross-compilation.md](openwrt-dev/references/cross-compilation.md) | toolchain、staging_dir、musl | 理解交叉編譯 |
| [debugging.md](openwrt-dev/references/debugging.md) | GDB、strace、tcpdump、perf | 除錯與調校 |
| [security.md](openwrt-dev/references/security.md) | PIE/SSP/RELRO、CVE 修補 | 安全加固 |
| [common-recipes.md](openwrt-dev/references/common-recipes.md) | opkg、UCI 操作、部署腳本 | 日常速查 |

---

## 🔧 適用版本

- **OpenWrt**: 23.05 LTS 及更新版本
- **核心**: Linux 5.15+
- **LuCI**: JavaScript client-side API（非 Lua CBI）
- **防火牆**: firewall4（nftables 後端）
- **交換器**: DSA（非 swconfig）

---

## 📖 資料來源

所有內容基於以下高品質來源整理（以自己的話重述，非抄襲）：

- [OpenWrt 官方 Wiki](https://openwrt.org/docs/)
- [OpenWrt Git 原始碼](https://git.openwrt.org/)
- [OpenWrt packages feed](https://github.com/openwrt/packages)
- [OpenWrt LuCI 原始碼](https://github.com/openwrt/luci)
- [LuCI JavaScript API 文件](https://openwrt.github.io/luci/jsapi/)
- kernel.org 上游 Linux 文件

---

## 📄 授權

MIT License — 自由使用、修改、分發。

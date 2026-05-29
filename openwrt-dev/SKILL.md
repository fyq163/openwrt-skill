---
name: openwrt
description: >
  OpenWrt 路由器開發與配置的完整指南。涵蓋固件開發（建構系統、套件開發、LuCI 應用、核心模組、交叉編譯、除錯、安全）和路由器配置（SSH、DDNS、防火牆、網絡接口、VLAN、DHCP/DNS、WiFi、包管理）。使用此 skill 當使用者提到 OpenWrt、路由器開發、路由器配置、固件編譯、嵌入式 Linux 網路設備、LuCI、UCI、ipk 套件、或任何與 OpenWrt 生態系相關的任務，包括遠程路由器管理和網絡配置。
globs:
  - "**/openwrt/**"
  - "**/luci/**"
  - "**/package/Makefile"
  - "**/router/**"
  - "**/network/**"
---

# OpenWrt 開發與配置技能包

> 適用版本：OpenWrt 23.05+（核心 5.15+）
> 建構系統：Buildroot / SDK / ImageBuilder
> 網路子系統：UCI + netifd + firewall4 (nftables)
> 遠程管理：SSH + DDNS + 防火牆規則

---

## 快速導覽 — 你想做什麼？

### 🔧 固件開發

| 我想要... | 閱讀 |
|-----------|------|
| 編譯整個 OpenWrt 韌體 | [build-system.md](references/build-system.md) |
| 把自己的程式打包成 `.ipk` | [package-dev.md](references/package-dev.md) |
| 寫一個核心模組 (`.ko`) | [kernel-module.md](references/kernel-module.md) |
| 做一個 LuCI Web 管理頁面 | [luci-app.md](references/luci-app.md) |
| 理解交叉編譯工具鏈 | [cross-compilation.md](references/cross-compilation.md) |
| 除錯程式或效能調校 | [debugging.md](references/debugging.md) |
| 安全加固與 CVE 修補 | [security.md](references/security.md) |
| 日常開發操作速查 | [common-recipes.md](references/common-recipes.md) |

### ⚙️ 路由器配置

| 我想要... | 閱讀 |
|-----------|------|
| 遠程訪問路由器（SSH） | [ssh-access.md](references/ssh-access.md) |
| 配置 DDNS（Cloudflare） | [ddns-config.md](references/ddns-config.md) |
| 設置防火牆規則和端口轉發 | [firewall-rules.md](references/firewall-rules.md) |
| 配置網絡接口和 VLAN | [network-interfaces.md](references/network-interfaces.md) |
| 設置 DHCP 和 DNS | [dhcp-dns.md](references/dhcp-dns.md) |
| 配置 WiFi 和 SSID | [wireless-config.md](references/wireless-config.md) |
| 包管理和系統設置 | [package-management.md](references/package-management.md) |

---

## OpenWrt 開發與配置環境概述

OpenWrt 是基於 Linux 的嵌入式作業系統，專為網路設備（路由器、AP、閘道器）設計，具有雙重用途：

**固件開發端：** 其建構系統（Buildroot）可交叉編譯整個作業系統及使用者空間套件，產生 sysupgrade 映像或單獨的 `.ipk` 套件。開發流程以 feeds（套件來源倉庫）和 Makefile 為核心。LuCI 是 Web 管理介面框架，現代開發使用 JavaScript client-side API。

**配置管理端：** OpenWrt 使用 UCI（Unified Configuration Interface）統一管理所有設定。netifd 管理網路介面，firewall4（nftables 後端）管理防火牆，dnsmasq 提供 DHCP/DNS 服務。管理員可透過 SSH 遠程訪問、配置路由器的所有方面。

---

## 決策樹

```
使用者的問題是什麼？
│
├─ 🔧 固件開發相關
│  │
│  ├─ 建構/編譯
│  │  ├─ 要編譯完整韌體映像 → build-system.md
│  │  ├─ 要編譯單一套件 (SDK) → build-system.md § SDK 模式
│  │  ├─ 只想打包自訂映像 → build-system.md § ImageBuilder
│  │  └─ 編譯失敗/錯誤 → debugging.md + prompts/debug-build-error.md
│  │
│  ├─ 套件開發
│  │  ├─ 建立新套件 → package-dev.md + templates/package-makefile.template
│  │  ├─ 套件 Makefile 語法 → package-dev.md § 核心變數
│  │  ├─ 多套件/子套件 → package-dev.md § 多套件分割
│  │  └─ Go/Python/Rust 套件 → package-dev.md § 語言特定
│  │
│  ├─ LuCI Web 介面
│  │  ├─ 建立新 LuCI 應用 → luci-app.md + templates/luci-app-skeleton/
│  │  ├─ JavaScript API 用法 → luci-app.md § form.Map 完整範例
│  │  ├─ 選單和權限設定 → luci-app.md § 選單與 ACL
│  │  └─ 狀態/監控頁面 → luci-app.md § 自訂視圖
│  │
│  ├─ 核心模組
│  │  ├─ 寫核心模組 → kernel-module.md + templates/kernel-module-skeleton/
│  │  ├─ Netfilter hook → kernel-module.md § Netfilter
│  │  └─ 驅動程式 → kernel-module.md § 裝置驅動
│  │
│  ├─ 交叉編譯
│  │  ├─ 工具鏈結構 → cross-compilation.md
│  │  ├─ staging_dir 用途 → cross-compilation.md § 目錄結構
│  │  └─ CMake/autotools 移植 → cross-compilation.md § 建構系統整合
│  │
│  ├─ 除錯
│  │  ├─ 線上除錯 → debugging.md § 遠端 GDB
│  │  ├─ 日誌分析 → debugging.md § logread/procd
│  │  ├─ 網路封包 → debugging.md § tcpdump
│  │  └─ 效能分析 → debugging.md § perf
│  │
│  └─ 安全
│     ├─ 安全加固 → security.md § 編譯時防護
│     ├─ CVE 修補 → security.md § CVE 修補流程
│     └─ SSH/防火牆硬化 → security.md § 運行時防護
│
└─ ⚙️ 路由器配置相關
   │
   ├─ 遠程管理
   │  ├─ SSH 訪問和密鑰認證 → ssh-access.md
   │  ├─ 檔案傳輸和遠程執行 → ssh-access.md § 操作流程
   │  └─ SSH 服務配置 → ssh-access.md § SSH 服務配置
   │
   ├─ 外網服務
   │  ├─ 設置 DDNS（Cloudflare） → ddns-config.md
   │  ├─ 配置 DDNS API Token → ddns-config.md § Cloudflare 設置
   │  └─ 監控 DDNS 狀態 → ddns-config.md § 監控腳本
   │
   ├─ 安全與訪問
   │  ├─ 防火牆規則管理 → firewall-rules.md
   │  ├─ 端口轉發（NAT） → firewall-rules.md § 端口轉發
   │  ├─ DMZ 和 UPnP → firewall-rules.md § DMZ 設置
   │  └─ IP 黑/白名單 → firewall-rules.md § 限制來源 IP
   │
   ├─ 網絡設置
   │  ├─ WAN/LAN 接口配置 → network-interfaces.md
   │  ├─ VLAN 創建和管理 → network-interfaces.md § 創建新 VLAN
   │  ├─ IPv6 配置 → network-interfaces.md § IPv6 支援
   │  └─ 靜態路由 → network-interfaces.md § 靜態路由
   │
   ├─ DHCP 和 DNS
   │  ├─ DHCP 池和租賃配置 → dhcp-dns.md
   │  ├─ 靜態 DHCP 租賃 → dhcp-dns.md § 靜態 DHCP 租賃
   │  ├─ 本地域名解析 → dhcp-dns.md § 本地域名解析
   │  └─ DNS 過濾和廣告攔截 → dhcp-dns.md § DNS 過濾
   │
   ├─ 無線網絡
   │  ├─ WiFi SSID 和加密配置 → wireless-config.md
   │  ├─ WPA2/WPA3 設置 → wireless-config.md § WPA2/WPA3
   │  ├─ 訪客網絡隔離 → wireless-config.md § 訪客網絡
   │  └─ 信道和功率優化 → wireless-config.md § 信道優化
   │
   └─ 系統維護
      ├─ 包安裝和升級 → package-management.md
      ├─ 磁盤空間管理 → package-management.md § 管理存儲空間
      ├─ 時間同步（NTP） → package-management.md § 時間同步
      └─ 系統日誌配置 → package-management.md § 系統日誌配置
```

---

## 參考文件摘要

### 🔧 固件開發文件

| 文件 | 說明 |
|------|------|
| **build-system.md** | Buildroot 全量建構、SDK 套件編譯、ImageBuilder 映像客製、feeds 管理、menuconfig 操作 |
| **package-dev.md** | Makefile 結構與 PKG_* 變數、Build/Configure/Compile/Install 階段、init 腳本與 UCI 設定、多套件分割 |
| **kernel-module.md** | KernelPackage 定義、Kbuild 整合、Netfilter hook、裝置樹 overlay、AutoLoad 設定 |
| **luci-app.md** | JavaScript client-side API（form.Map/Section/Option）、menu.d JSON、ACL 權限、rpcd 後端整合 |
| **cross-compilation.md** | toolchain 結構、staging_dir 四層目錄、TARGET_* 環境變數、autotools/CMake/Meson 整合、musl 注意事項 |
| **debugging.md** | logread/procd 日誌、遠端 GDB、strace/ltrace、tcpdump 封包分析、perf 效能剖析、kernel oops 解讀 |
| **security.md** | ASLR/PIE/SSP/FORTIFY_SOURCE/RELRO 編譯防護、SECCOMP、SSH 加固、CVE 修補流程、最小權限原則 |
| **common-recipes.md** | 系統管理速查、套件管理 opkg 指令、UCI 操作、Wi-Fi 快速設定、建構指令集、部署腳本範例、ubus 呼叫 |

### ⚙️ 路由器配置文件

| 文件 | 說明 |
|------|------|
| **ssh-access.md** | SSH 訪問方式、密鑰認證和密碼認證、檔案傳輸（scp/sftp）、遠程執行命令、SSH 服務配置、主機密鑰管理 |
| **ddns-config.md** | DDNS 原理、Cloudflare API Token 配置、UCI DDNS 設置、IPv4/IPv6 支援、DDNS 監控和故障排除 |
| **firewall-rules.md** | Zone 配置、入站/出站規則、端口轉發（NAT）、DMZ 設置、IP 黑/白名單、UPnP 自動映射、nftables 規則 |
| **network-interfaces.md** | 物理接口和邏輯接口配置、VLAN 創建和管理、靜態 IP 和 DHCP 方式、IPv6 配置、DSA VLAN、靜態路由 |
| **dhcp-dns.md** | DHCP 池配置、靜態 DHCP 租賃（MAC→IP）、上游 DNS 設置、本地域名解析、DNS 過濾和廣告攔截 |
| **wireless-config.md** | WiFi SSID 配置、WPA2/WPA3 加密、訪客網絡隔離、信道和功率優化、客戶端隔離、5GHz 配置 |
| **package-management.md** | opkg 包管理（安裝/升級/卸載）、倉庫配置、主機名和時區設置、NTP 時間同步、系統日誌、存儲空間管理 |
| **networking.md** | UCI 網路設定、netifd 運作原理、firewall4 nftables 規則、無線設定、ubus 查詢（開發和配置的交集文件） |

---

## 模板目錄 (templates/)

| 路徑 | 說明 | 使用方式 |
|------|------|----------|
| `templates/package-makefile.template` | 標準套件 Makefile 模板 | 複製後替換 `{{VARIABLE}}` 變數 |
| `templates/luci-app-skeleton/` | LuCI 應用骨架（JS view + 選單 + ACL + Makefile） | 複製整個目錄，重命名並替換變數 |
| `templates/kernel-module-skeleton/` | 核心模組骨架（Makefile + Kbuild + C 原始碼） | 複製整個目錄，填入模組名稱和邏輯 |

**模板變數慣例：** 使用 `{{VARIABLE_NAME}}` 格式標記需要使用者替換的位置。

### 使用步驟
1. 從 `templates/` 複製對應骨架
2. 全域搜尋 `{{` 找到所有需替換的變數
3. 用實際值替換每個 `{{VARIABLE}}`
4. 根據需求增刪功能
5. 編譯測試：`make package/你的套件/compile V=s`

---

## 提示詞目錄 (prompts/)

| 路徑 | 用途 | 觸發情境 |
|------|------|----------|
| `prompts/generate-package.md` | 產生 OpenWrt 套件 | 「幫我把這個程式打包成 ipk」 |
| `prompts/generate-luci-app.md` | 產生 LuCI Web 介面 | 「幫我做管理頁面 / Web UI」 |
| `prompts/debug-build-error.md` | 診斷編譯錯誤 | 「為什麼 make 失敗了」 |
| `prompts/network-config.md` | 產生網路設定 | 「幫我設定 VLAN / 防火牆 / Wi-Fi」 |

每份提示詞都包含：使用時機、提示詞模板（可直接使用）、期望輸出、範例對話。

---

## 版本相容性說明

| 功能 | 最低版本 | 說明 |
|------|----------|------|
| firewall4 (nftables) | 22.03+ | 取代 firewall3 (iptables) |
| DSA switch driver | 21.02+ | 取代 swconfig |
| LuCI JavaScript API | 21.02+ | 取代 Lua CBI |
| procd init system | 15.05+ | 標準 init 系統 |
| musl libc (預設) | 15.05+ | 取代 uClibc |

> **此技能包所有內容均以 OpenWrt 23.05+ 為準。** 若使用者環境為更舊版本，需注意防火牆（iptables vs nftables）、交換器（swconfig vs DSA）、LuCI（Lua vs JS）的差異。

---

## 常用指令速查

```bash
# 完整建構流程
git clone https://git.openwrt.org/openwrt/openwrt.git
cd openwrt
./scripts/feeds update -a && ./scripts/feeds install -a
make menuconfig    # 選擇 target 和套件
make -j$(nproc)    # 並行編譯

# 單一套件編譯 (SDK)
make package/myapp/compile V=s

# 安裝套件到設備
scp bin/packages/*/base/myapp_1.0-1_*.ipk root@192.168.1.1:/tmp/
ssh root@192.168.1.1 'opkg install /tmp/myapp_1.0-1_*.ipk'

# UCI 操作
uci show network.lan
uci set network.lan.ipaddr='192.168.10.1'
uci commit network && /etc/init.d/network restart
```

---

## 外部資源

- [OpenWrt 官方文件](https://openwrt.org/docs/)
- [OpenWrt Git 倉庫](https://git.openwrt.org/)
- [LuCI JavaScript API 文件](https://openwrt.github.io/luci/jsapi/)
- [OpenWrt packages feed](https://github.com/openwrt/packages)
- [OpenWrt 硬體支援清單](https://openwrt.org/toh/start)

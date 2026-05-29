# 無線網絡與 SSID 配置

## 概述

OpenWrt 支援多個無線接口，可配置多個 SSID（Service Set Identifier）、頻段（2.4GHz/5GHz）、安全認證方式（WPA2/WPA3）等。透過 UCI 配置，可以實現訪客網絡、負載均衡、信號優化等功能。無線配置直接影響用戶體驗和安全性。

## 前置條件

- OpenWrt 23.05+ 已安裝
- 路由器擁有無線模組（WiFi 適配器）
- SSH 訪問權限
- 基本 WiFi 和密碼學知識
- hostapd（無線接入點） 和 wpa_supplicant（客戶端）已安裝

## 核心概念

### 無線架構

```
無線模組（WiFi 晶片）
├─ PHY（物理層，如 phy0）
│  ├─ 2.4GHz 頻段（b/g/n/ax）
│  └─ 5GHz 頻段（a/n/ac/ax）
│
邏輯接口（UAP，Unmanaged Access Point）
├─ wlan0（2.4GHz AP）
│  └─ SSID：MyWiFi
├─ wlan1（5GHz AP）
│  └─ SSID：MyWiFi-5G
└─ wlan2（訪客網絡）
   └─ SSID：MyWiFi-Guest

UCI 配置
└─ /etc/config/wireless
   ├─ 裝置設定（頻段、功率、信道）
   └─ 接口設定（SSID、加密、認證）
```

### 安全認證方式

| 方式 | 加密 | 適用場景 | 安全性 |
|------|------|----------|--------|
| **Open** | 無 | 訪客網絡（臨時） | ⚠️ 無 |
| **WEP** | RC4 | ❌ 已過時 | ❌ 已破解 |
| **WPA2-PSK** | CCMP | 一般家用 | ✓ 安全 |
| **WPA3-PSK** | CCMP/GCMP | 現代設備 | ✓✓ 最安全 |
| **WPA2/WPA3 Mixed** | CCMP | 兼容性 | ✓ 推薦 |

### UCI 無線配置

```bash
# 無線裝置設定
config wifi-device 'radio0'
    option type 'mac80211'
    option channel '11'            # 2.4GHz 信道
    option hwmode '11g'            # 802.11g（2.4GHz）
    option htmode 'HT20'           # 帶寬 20MHz
    option txpower '20'            # 發射功率（dBm）
    option disabled '0'            # 是否禁用

# 無線接口設定
config wifi-iface 'wifinet0'
    option device 'radio0'
    option network 'lan'           # 綁定邏輯接口
    option mode 'ap'               # 接入點模式
    option ssid 'MyWiFi'           # SSID（顯示名稱）
    option encryption 'wpa2'       # WPA2 加密
    option key 'your_password_here' # WiFi 密碼
```

## 操作流程

### 1. 查看無線設備

```bash
# 查看無線裝置列表
iw dev

# 查看無線裝置詳細信息
iw phy phy0 info

# 查看當前 SSID 和連接狀態
iw dev wlan0 link

# 查看 UCI 無線配置
uci show wireless

# 查看 hostapd 日誌
logread -e hostapd

# 查看客戶端連接列表
iw dev wlan0 station dump
```

### 2. 配置主 WiFi（SSID）

#### WPA2-PSK 方式（推薦家用）

```bash
# 查看當前配置
uci show wireless.@wifi-iface[0]

# 設置 SSID 和密碼
uci set wireless.@wifi-iface[0].ssid='MyWiFi'
uci set wireless.@wifi-iface[0].encryption='wpa2'
uci set wireless.@wifi-iface[0].key='MySecurePassword123'  # 至少 8 字符

# 設置綁定的網絡接口
uci set wireless.@wifi-iface[0].network='lan'

# 設置 802.11 模式（ap = 接入點）
uci set wireless.@wifi-iface[0].mode='ap'

uci commit wireless
/etc/init.d/hostapd restart
```

#### WPA3-PSK 方式（最新，需新硬件）

```bash
# 使用 WPA3（需確保硬件支援）
uci set wireless.@wifi-iface[0].ssid='MyWiFi'
uci set wireless.@wifi-iface[0].encryption='wpa3'
uci set wireless.@wifi-iface[0].key='MySecurePassword123'

uci commit wireless
/etc/init.d/hostapd restart
```

#### WPA2/WPA3 Mixed（向後兼容）

```bash
# 支援舊設備也支援新設備
uci set wireless.@wifi-iface[0].encryption='wpa2+wpa3'
uci set wireless.@wifi-iface[0].key='MySecurePassword123'

uci commit wireless
/etc/init.d/hostapd restart
```

### 3. 配置 5GHz WiFi（如存在）

```bash
# 查看 5GHz 無線設備
uci show wireless.radio1  # 通常 5GHz 為 radio1

# 設置 5GHz 信道和頻寬
uci set wireless.radio1.channel='36'      # 5GHz 信道 36（起始）
uci set wireless.radio1.hwmode='11a'      # 802.11a（5GHz）
uci set wireless.radio1.htmode='HT80'     # 80MHz 寬帶

# 創建 5GHz SSID
uci add wireless wifi-iface
uci set wireless.@wifi-iface[-1].device='radio1'
uci set wireless.@wifi-iface[-1].network='lan'
uci set wireless.@wifi-iface[-1].mode='ap'
uci set wireless.@wifi-iface[-1].ssid='MyWiFi-5G'
uci set wireless.@wifi-iface[-1].encryption='wpa2'
uci set wireless.@wifi-iface[-1].key='MySecurePassword123'

uci commit wireless
/etc/init.d/hostapd restart
```

### 4. 設置訪客網絡

訪客網絡隔離於主網絡，無法訪問 LAN 設備：

```bash
# 第 1 步：創建訪客 VLAN（參考網絡接口文件）
# 假設已創建 guest VLAN，VLAN ID = 3

# 第 2 步：在現有 WiFi 上添加訪客 SSID
uci add wireless wifi-iface
uci set wireless.@wifi-iface[-1].device='radio0'
uci set wireless.@wifi-iface[-1].network='guest'    # 綁定 guest 接口
uci set wireless.@wifi-iface[-1].mode='ap'
uci set wireless.@wifi-iface[-1].ssid='MyWiFi-Guest'
uci set wireless.@wifi-iface[-1].encryption='wpa2'
uci set wireless.@wifi-iface[-1].key='GuestPassword123'
uci set wireless.@wifi-iface[-1].isolate='1'       # 客戶端隔離

uci commit wireless
/etc/init.d/hostapd restart
```

### 5. WiFi 信道和功率優化

```bash
# 查看當前信道和功率
uci show wireless.radio0

# 2.4GHz 信道選擇（1, 6, 11 不重疊）
# 根據地區選擇：
# 美國/日本：1-11
# 歐洲：1-13
# 中國：1-13

# 設置 2.4GHz 信道
uci set wireless.radio0.channel='6'        # 選擇不重疊信道

# 設置發射功率（0-30 dBm，數值越大功率越強）
uci set wireless.radio0.txpower='20'       # 標準功率

# 設置帶寬（HT20, HT40, HT80 等）
uci set wireless.radio0.htmode='HT20'      # 穩定性最佳

uci commit wireless
/etc/init.d/hostapd restart
```

### 6. 禁用無線功能（節電）

```bash
# 禁用 2.4GHz
uci set wireless.radio0.disabled='1'

# 禁用 5GHz
uci set wireless.radio1.disabled='1'

uci commit wireless
/etc/init.d/hostapd restart
```

## 範例代碼

### WiFi 配置腳本

```bash
#!/bin/bash
# setup-wifi.sh - 配置 WiFi SSID 和密碼

SSID="${1:-MyWiFi}"
******
BAND="${3:-both}"  # both, 2.4, 5

if [ ${#PASSWORD} -lt 8 ]; then
    echo "錯誤：密碼必須至少 8 個字符"
    exit 1
fi

echo "正在配置 WiFi..."

case "$BAND" in
    2.4|both)
        # 配置 2.4GHz
        uci set wireless.@wifi-iface[0].ssid="$SSID"
        uci set wireless.@wifi-iface[0].encryption='wpa2'
        uci set wireless.@wifi-iface[0].key="$PASSWORD"
        uci set wireless.radio0.disabled='0'
        ;;
esac

case "$BAND" in
    5|both)
        # 配置 5GHz
        uci set wireless.@wifi-iface[1].ssid="${SSID}-5G"
        uci set wireless.@wifi-iface[1].encryption='wpa2'
        uci set wireless.@wifi-iface[1].key="$PASSWORD"
        uci set wireless.radio1.disabled='0'
        ;;
esac

uci commit wireless
/etc/init.d/hostapd restart

echo "✓ WiFi 配置完成"
echo "SSID：$SSID"
echo "加密：WPA2"
```

### WiFi 監控腳本

```bash
#!/bin/bash
# monitor-wifi.sh - 監控 WiFi 狀態和客戶端

echo "=== WiFi 狀態 ==="
iw dev wlan0 link

echo ""
echo "=== 連接客戶端 ==="
iw dev wlan0 station dump | grep -E "^Station|signal:|inactive time:"

echo ""
echo "=== 信道使用情況 ==="
iw scan | grep "frequency\|signal\|SSID" | head -20
```

## 常見錯誤與解法

### Q：WiFi 無法啟動或客戶端無法連接
**原因：** hostapd 配置錯誤或無線模組未加載  
**解法：**
```bash
# 檢查無線模組是否加載
lsmod | grep mac80211

# 檢查無線設備是否存在
iw dev

# 查看 hostapd 錯誤日誌
logread -e hostapd | tail -30

# 檢查 UCI 配置
uci show wireless.@wifi-iface[0]

# 重新加載無線配置
uci commit wireless
wifi reload
```

### Q：WiFi 連接掉線或信號弱
**原因：** 信道幹擾、功率不足或天線問題  
**解法：**
```bash
# 檢查信道是否擁堵（掃描附近 SSID）
iw scan | grep "SSID\|signal" | head -20

# 更改信道（2.4GHz 推薦 1, 6, 11）
uci set wireless.radio0.channel='1'  # 或 6, 11

# 增加發射功率
uci set wireless.radio0.txpower='30'

# 檢查天線連接
iwconfig

uci commit wireless
/etc/init.d/hostapd restart
```

### Q：客戶端間無法通信（訪客網絡隔離）
**原因：** AP 隔離（isolation）設置錯誤  
**解法：**
```bash
# 啟用客戶端隔離（訪客網絡）
uci set wireless.@wifi-iface[2].isolate='1'

# 禁用客戶端隔離（允許通信）
uci set wireless.@wifi-iface[0].isolate='0'

uci commit wireless
/etc/init.d/hostapd restart
```

### Q：WPA3 不可用或不支援
**原因：** 硬件或驅動不支援 WPA3，或 hostapd 版本過舊  
**解法：**
```bash
# 檢查 hostapd 版本
hostapd -v

# 檢查硬件支援
iw phy phy0 info | grep -i wpa

# 降級到 WPA2
uci set wireless.@wifi-iface[0].encryption='wpa2'
uci commit wireless
/etc/init.d/hostapd restart
```

## 參考來源

- [OpenWrt 無線配置官方文件](https://openwrt.org/docs/guide/network/wifi/start)
- [hostapd 官方文檔](https://w1.fi/hostapd/)
- [802.11 標準信息](https://en.wikipedia.org/wiki/IEEE_802.11)
- [WiFi 安全最佳實踐](https://openwrt.org/docs/guide/security/wifi_security)
- [UCI 無線配置](https://openwrt.org/docs/uci/wireless)

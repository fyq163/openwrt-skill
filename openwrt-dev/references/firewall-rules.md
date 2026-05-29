# 防火牆規則與端口轉發

## 概述

OpenWrt 的防火牆（firewall4）使用 nftables 核心規則引擎管理流量。透過防火牆，可以實現：入站/出站流量控制、端口轉發（NAT）、DMZ 設置、IP 白名單/黑名單、連接追蹤等。理解防火牆規則對於安全的路由器設置至關重要。

## 前置條件

- OpenWrt 23.05+ 已安裝
- SSH 訪問權限
- 基本 TCP/IP 和防火牆概念
- firewall4 套件已安裝（通常預設包含）

## 核心概念

### 防火牆架構

```
外網流量 (WAN)
       │
       ▼
firewall4 (nftables 規則)
       ├─ Forward Rules（轉發規則）
       ├─ Input Rules（入站規則）
       ├─ Output Rules（出站規則）
       ├─ NAT Rules（網址轉換）
       └─ Mangle Rules（標記操作）
       │
       ▼
內網設備 (LAN)
```

### Zone（區域）概念

防火牆使用 Zone 組織接口，限定規則作用範圍：

| Zone | 接口 | 說明 |
|------|------|------|
| **lan** | eth0.1（交換機 LAN 埠）| 內網設備區域 |
| **wan** | eth0.2（交換機 WAN 埠）| 外網區域 |
| **guest** | wlan1（WiFi）| 訪客網絡（可選） |

### UCI 配置結構

防火牆配置位於 `/etc/config/firewall`：

```bash
# Zone 定義
config zone 'lan'
    option name 'lan'
    option input 'ACCEPT'        # 入站預設行為
    option output 'ACCEPT'       # 出站預設行為
    option forward 'ACCEPT'      # 轉發預設行為
    list network 'lan'           # 綁定的網絡接口

config zone 'wan'
    option name 'wan'
    option input 'REJECT'        # 拒絕外網入站
    option output 'ACCEPT'
    option forward 'REJECT'
    list network 'wan'

# Zone 間轉發規則
config forwarding 'wan2lan'
    option src 'wan'
    option dest 'lan'
    option enabled '0'           # 禁用 WAN→LAN 直接轉發

config forwarding 'lan2wan'
    option src 'lan'
    option dest 'wan'
    option enabled '1'           # 允許 LAN 訪問 WAN
```

## 操作流程

### 1. 查看當前防火牆配置

```bash
# 查看所有防火牆設定
uci show firewall

# 查看 Zone 配置
uci show firewall.@zone

# 查看轉發規則
uci show firewall.@forwarding

# 查看已啟用的規則
uci show firewall.@rule | grep enabled

# 查看 NAT 規則
uci show firewall.@redirect

# 查看當前 nftables 規則（需要 nftables 套件）
nft list ruleset
```

### 2. 打開特定端口（入站規則）

#### 允許外網訪問某個服務（如 SSH、Web）

```bash
# 例：允許外網 SSH 訪問（埠 22）
uci add firewall rule
uci set firewall.@rule[-1].name='Allow SSH'
uci set firewall.@rule[-1].src='wan'
uci set firewall.@rule[-1].dest_port='22'
uci set firewall.@rule[-1].proto='tcp'
uci set firewall.@rule[-1].target='ACCEPT'
uci commit firewall

# 例：允許外網 HTTP/HTTPS（埠 80, 443）
uci add firewall rule
uci set firewall.@rule[-1].name='Allow HTTP'
uci set firewall.@rule[-1].src='wan'
uci set firewall.@rule[-1].dest_port='80'
uci set firewall.@rule[-1].proto='tcp'
uci set firewall.@rule[-1].target='ACCEPT'
uci commit firewall

uci add firewall rule
uci set firewall.@rule[-1].name='Allow HTTPS'
uci set firewall.@rule[-1].src='wan'
uci set firewall.@rule[-1].dest_port='443'
uci set firewall.@rule[-1].proto='tcp'
uci set firewall.@rule[-1].target='ACCEPT'
uci commit firewall

# 重啟防火牆以應用規則
/etc/init.d/firewall restart
```

### 3. 端口轉發（NAT）

#### 將外網流量轉發到內網設備

```bash
# 例：將外網埠 8080 轉發到內網設備 192.168.1.100 的埠 80
uci add firewall redirect
uci set firewall.@redirect[-1].name='Port Forward 8080→80'
uci set firewall.@redirect[-1].src='wan'
uci set firewall.@redirect[-1].src_dport='8080'
uci set firewall.@redirect[-1].dest='lan'
uci set firewall.@redirect[-1].dest_ip='192.168.1.100'
uci set firewall.@redirect[-1].dest_port='80'
uci set firewall.@redirect[-1].proto='tcp'
uci set firewall.@redirect[-1].target='DNAT'
uci commit firewall

# 例：轉發 VPN（UDP 1194）到內網 192.168.1.50
uci add firewall redirect
uci set firewall.@redirect[-1].name='Port Forward VPN'
uci set firewall.@redirect[-1].src='wan'
uci set firewall.@redirect[-1].src_dport='1194'
uci set firewall.@redirect[-1].dest='lan'
uci set firewall.@redirect[-1].dest_ip='192.168.1.50'
uci set firewall.@redirect[-1].dest_port='1194'
uci set firewall.@redirect[-1].proto='udp'
uci set firewall.@redirect[-1].target='DNAT'
uci commit firewall

# 重啟防火牆
/etc/init.d/firewall restart

# 驗證 NAT 規則是否正確
iptables -t nat -L -n -v  # iptables 方式
nft list chain inet fw4 dstnat  # nftables 方式
```

### 4. DMZ 設置（非軍事區）

允許外網直接訪問特定內網設備：

```bash
# 將 192.168.1.100 設為 DMZ
uci set firewall.@zone[1].masq='0'  # 禁用 WAN Zone 的 NAT

# 或使用 DMZ 規則
uci add firewall redirect
uci set firewall.@redirect[-1].name='DMZ'
uci set firewall.@redirect[-1].src='wan'
uci set firewall.@redirect[-1].dest='lan'
uci set firewall.@redirect[-1].dest_ip='192.168.1.100'
uci set firewall.@redirect[-1].target='DNAT'
uci commit firewall

/etc/init.d/firewall restart
```

### 5. 限制來源 IP（黑/白名單）

```bash
# 允許特定 IP 訪問（白名單）
uci add firewall rule
uci set firewall.@rule[-1].name='Allow Specific IP'
uci set firewall.@rule[-1].src='wan'
uci set firewall.@rule[-1].src_ip='203.0.113.5'  # 允許的 IP
uci set firewall.@rule[-1].dest_port='22'
uci set firewall.@rule[-1].proto='tcp'
uci set firewall.@rule[-1].target='ACCEPT'
uci commit firewall

# 拒絕特定 IP 訪問（黑名單）
uci add firewall rule
uci set firewall.@rule[-1].name='Block Specific IP'
uci set firewall.@rule[-1].src='wan'
uci set firewall.@rule[-1].src_ip='192.0.2.100'  # 拒絕的 IP
uci set firewall.@rule[-1].target='REJECT'
uci commit firewall

/etc/init.d/firewall restart
```

### 6. UPnP 端口映射（自動端口轉發）

某些應用（如 Torrent、遊戲）支援 UPnP 自動開放端口：

```bash
# 安裝 UPnP 服務
opkg install miniupnpd

# 啟用 UPnP
uci set upnpd.config.enabled='1'
uci commit upnpd

# 啟動 UPnP 服務
/etc/init.d/miniupnpd start

# 確保防火牆允許 UPnP（通常自動配置）
# 查看 UPnP 映射
upnpc -l
```

## 範例代碼

### 防火牆配置腳本

```bash
#!/bin/bash
# setup-firewall.sh - 基本防火牆配置

set -e

echo "正在配置防火牆..."

# 設置 LAN Zone
uci set firewall.@zone[0].name='lan'
uci set firewall.@zone[0].input='ACCEPT'
uci set firewall.@zone[0].output='ACCEPT'
uci set firewall.@zone[0].forward='ACCEPT'

# 設置 WAN Zone（嚴格模式）
uci set firewall.@zone[1].name='wan'
uci set firewall.@zone[1].input='REJECT'   # 拒絕外網入站
uci set firewall.@zone[1].output='ACCEPT'
uci set firewall.@zone[1].forward='REJECT' # 拒絕外網轉發

# 允許 LAN→WAN
uci set firewall.@forwarding[0].src='lan'
uci set firewall.@forwarding[0].dest='wan'
uci set firewall.@forwarding[0].enabled='1'

# 禁用 WAN→LAN（默認安全）
uci set firewall.@forwarding[1].src='wan'
uci set firewall.@forwarding[1].dest='lan'
uci set firewall.@forwarding[1].enabled='0'

# 允許外網訪問 SSH（可選，改為安全的埠）
uci add firewall rule
uci set firewall.@rule[-1].name='Allow SSH'
uci set firewall.@rule[-1].src='wan'
uci set firewall.@rule[-1].dest_port='22'
uci set firewall.@rule[-1].proto='tcp'
uci set firewall.@rule[-1].target='ACCEPT'

# 允許外網 Ping（ICMP）
uci add firewall rule
uci set firewall.@rule[-1].name='Allow Ping'
uci set firewall.@rule[-1].src='wan'
uci set firewall.@rule[-1].proto='icmp'
uci set firewall.@rule[-1].target='ACCEPT'

uci commit firewall
/etc/init.d/firewall restart

echo "✓ 防火牆配置完成"
```

### 端口轉發管理腳本

```bash
#!/bin/bash
# manage-portforward.sh - 端口轉發管理

ACTION="${1:-list}"
EXTERNAL_PORT="${2}"
INTERNAL_IP="${3}"
INTERNAL_PORT="${4}"
PROTO="${5:-tcp}"

case "$ACTION" in
    list)
        echo "當前端口轉發規則："
        uci show firewall.@redirect
        ;;
    add)
        if [ -z "$EXTERNAL_PORT" ] || [ -z "$INTERNAL_IP" ] || [ -z "$INTERNAL_PORT" ]; then
            echo "用法：$0 add <external_port> <internal_ip> <internal_port> [protocol]"
            exit 1
        fi
        
        uci add firewall redirect
        uci set firewall.@redirect[-1].name="Port_${EXTERNAL_PORT}_to_${INTERNAL_IP}:${INTERNAL_PORT}"
        uci set firewall.@redirect[-1].src='wan'
        uci set firewall.@redirect[-1].src_dport="$EXTERNAL_PORT"
        uci set firewall.@redirect[-1].dest='lan'
        uci set firewall.@redirect[-1].dest_ip="$INTERNAL_IP"
        uci set firewall.@redirect[-1].dest_port="$INTERNAL_PORT"
        uci set firewall.@redirect[-1].proto="$PROTO"
        uci set firewall.@redirect[-1].target='DNAT'
        uci commit firewall
        /etc/init.d/firewall restart
        
        echo "✓ 已添加端口轉發：$EXTERNAL_PORT→$INTERNAL_IP:$INTERNAL_PORT"
        ;;
    remove)
        if [ -z "$EXTERNAL_PORT" ]; then
            echo "用法：$0 remove <external_port>"
            exit 1
        fi
        
        # 移除指定埠的規則
        uci show firewall.@redirect | grep "src_dport='$EXTERNAL_PORT'" | cut -d. -f2 | while read rule; do
            uci delete firewall."$rule"
        done
        uci commit firewall
        /etc/init.d/firewall restart
        
        echo "✓ 已移除埠 $EXTERNAL_PORT 的轉發規則"
        ;;
    *)
        echo "用法："
        echo "  $0 list                    # 列出所有轉發規則"
        echo "  $0 add <ext_port> <ip> <int_port> [proto]  # 添加轉發"
        echo "  $0 remove <ext_port>       # 移除轉發"
        ;;
esac
```

## 常見錯誤與解法

### Q：外網無法訪問轉發的端口
**原因：** 防火牆規則未正確應用或 WAN Zone 配置錯誤  
**解法：**
```bash
# 檢查防火牆規則是否正確
uci show firewall.@redirect | grep -A 5 'src_dport'

# 查看 nftables 規則是否加載
nft list chain inet fw4 dstnat

# 檢查防火牆服務是否運行
/etc/init.d/firewall status

# 重新啟動防火牆
/etc/init.d/firewall restart

# 測試連接
telnet yourip 8080  # 測試外網埠
```

### Q：LAN 設備無法訪問外網
**原因：** 轉發規則被禁用或 NAT 配置錯誤  
**解法：**
```bash
# 檢查 LAN→WAN 轉發是否啟用
uci show firewall.@forwarding | grep 'src.*lan' | grep 'dest.*wan'

# 啟用 LAN→WAN 轉發
uci set firewall.@forwarding[0].enabled='1'
uci commit firewall
/etc/init.d/firewall restart
```

### Q：防火牆規則編輯後未生效
**原因：** 未提交配置或防火牆未重啟  
**解法：**
```bash
# 確保已提交配置
uci commit firewall

# 重啟防火牆
/etc/init.d/firewall restart

# 驗證規則
uci show firewall.@rule
```

## 參考來源

- [OpenWrt 防火牆官方文件](https://openwrt.org/docs/guide/firewall/)
- [nftables 官方文件](https://netfilter.org/projects/nftables/)
- [firewall4 (nftables) 配置指南](https://github.com/openwrt/packages/tree/master/net/firewall4)
- [UCI 防火牆配置](https://openwrt.org/docs/uci/firewall)

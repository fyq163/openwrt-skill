# 網絡接口與 VLAN 配置

## 概述

OpenWrt 的網絡配置透過 UCI 系統管理，涉及物理接口、邏輯接口、IP 地址分配、路由表等。VLAN（Virtual LAN）允許在單一物理交換機上劃分多個隔離的網絡。理解接口配置是設置複雜網絡拓撲的基礎。

## 前置條件

- OpenWrt 23.05+ 已安裝
- SSH 訪問權限
- 基本 TCP/IP 和 VLAN 知識
- netifd（網絡接口守護程式）已安裝（預設）

## 核心概念

### 網絡分層結構

```
物理層（Hardware）
├─ 交換機（DSA - Distributed Switch Architecture）
│  ├─ CPU 埠（eth0）→ Linux
│  ├─ LAN 埠 1-4（VLAN 1）
│  └─ WAN 埠（VLAN 2）
│
邏輯層（UCI）
├─ 裝置層（device）
│  ├─ eth0.1（VLAN 1 - LAN）
│  └─ eth0.2（VLAN 2 - WAN）
│
接口層（interface）
├─ lan（LAN 邏輯接口）
│  ├─ IPv4：DHCP server 或 Static
│  └─ IPv6：ULA 或 DHCPv6
└─ wan（WAN 邏輯接口）
   ├─ IPv4：DHCP client 或 Static
   └─ IPv6：DHCPv6 client
```

### DSA VLAN 配置

現代 OpenWrt（21.02+）使用 DSA 替代舊的 swconfig：

```bash
# DSA VLAN 配置位置
/etc/config/network

# 典型結構
config device 'switch'
    option name 'eth0'
    option ports '0 1 2 3 4 6'   # CPU 埠 0, LAN 埠 1-4, 埠 6

# VLAN 1（LAN）
config device 'eth0.1'
    option type 'bridge'
    option ifname 'eth0.1'
    option ports '1 2 3 4 0t'    # LAN 埠 + CPU 埠 tagged

# VLAN 2（WAN）
config device 'eth0.2'
    option ifname 'eth0.2'
    option ports '0t 5'          # CPU 埠 + WAN 埠
```

### UCI 接口配置

```bash
# LAN 邏輯接口
config interface 'lan'
    option type 'bridge'
    option ifname 'eth0.1'       # 綁定 VLAN 1 裝置
    option proto 'static'
    option ipaddr '192.168.1.1'
    option netmask '255.255.255.0'
    option ip6assign '60'        # IPv6 /60 子網前綴

# WAN 邏輯接口
config interface 'wan'
    option ifname 'eth0.2'
    option proto 'dhcp'          # DHCP 動態獲取

# WAN IPv6 接口
config interface 'wan6'
    option ifname 'eth0.2'
    option proto 'dhcpv6'
```

## 操作流程

### 1. 查看當前網絡配置

```bash
# 查看所有接口
ip link show

# 查看 VLAN 設備
ip -br link show type vlan

# 查看 IP 地址
ip -br addr show

# 查看路由表
ip route show

# 查看 UCI 配置
uci show network

# 查看網絡接口狀態
ifstatus lan
ifstatus wan

# 查看 DSA 交換機信息
cat /sys/class/net/eth0/statistics/*
```

### 2. 配置 LAN 接口

#### 靜態 IP

```bash
# 設置 LAN 為靜態 IP
uci set network.lan.proto='static'
uci set network.lan.ipaddr='192.168.1.1'
uci set network.lan.netmask='255.255.255.0'
uci set network.lan.gateway='192.168.0.1'    # 可選：指定默認網關
uci set network.lan.dns='8.8.8.8 1.1.1.1'   # 可選：指定 DNS

# 提交並重啟網絡
uci commit network
/etc/init.d/network restart
```

#### IPv6 支援

```bash
# 啟用 IPv6 ULA（Unique Local Address）
uci set network.lan.ip6assign='60'

# 或啟用 DHCPv6 server
uci set network.lan.ip6class='wan6'

# 設置靜態 IPv6
uci set network.lan.ip6addr='fd00::1/60'

uci commit network
/etc/init.d/network restart
```

### 3. 配置 WAN 接口

#### DHCP 方式（動態獲取）

```bash
# 設置 WAN 為 DHCP
uci set network.wan.proto='dhcp'
uci set network.wan.ifname='eth0.2'

# WAN IPv6（DHCPv6）
uci set network.wan6.proto='dhcpv6'
uci set network.wan6.ifname='eth0.2'

uci commit network
/etc/init.d/network restart
```

#### 靜態 IP 方式

```bash
# 設置 WAN 為靜態 IP
uci set network.wan.proto='static'
uci set network.wan.ifname='eth0.2'
uci set network.wan.ipaddr='203.0.113.10'    # 公網 IP
uci set network.wan.netmask='255.255.255.0'
uci set network.wan.gateway='203.0.113.1'    # ISP 網關
uci set network.wan.dns='203.0.113.1'        # ISP DNS

uci commit network
/etc/init.d/network restart
```

### 4. 創建新 VLAN

#### 創建訪客網絡 (Guest VLAN)

假設想在 LAN 埠 1 上創建訪客網絡，隔離於主 LAN：

```bash
# 1. 創建新 VLAN 裝置（VLAN 3）
uci add network device
uci set network.@device[-1].name='eth0.3'
uci set network.@device[-1].type='bridge'
uci set network.@device[-1].ports='1 0t'    # LAN 埠 1 + CPU 埠（tagged）

# 2. 創建邏輯接口
uci add network interface
uci set network.@interface[-1].name='guest'
uci set network.@interface[-1].ifname='eth0.3'
uci set network.@interface[-1].proto='static'
uci set network.@interface[-1].ipaddr='192.168.2.1'
uci set network.@interface[-1].netmask='255.255.255.0'
uci set network.@interface[-1].ip6assign='60'

# 3. 創建防火牆規則（隔離 guest 網絡）
uci add firewall zone
uci set firewall.@zone[-1].name='guest'
uci set firewall.@zone[-1].input='REJECT'
uci set firewall.@zone[-1].output='ACCEPT'
uci set firewall.@zone[-1].forward='REJECT'
uci add_list firewall.@zone[-1].network='guest'

# 4. 允許 guest→WAN（讓訪客訪問外網）
uci add firewall forwarding
uci set firewall.@forwarding[-1].src='guest'
uci set firewall.@forwarding[-1].dest='wan'
uci set firewall.@forwarding[-1].enabled='1'

uci commit network firewall
/etc/init.d/network restart
/etc/init.d/firewall restart
```

### 5. 修改 VLAN 埠分配

#### 調整 LAN 埠與 WAN 埠

假設默認為「LAN 埠 1-4 + WAN 埠 5」，要改為「LAN 埠 1-3 + WAN 埠 4-5」：

```bash
# 查看當前交換機配置
uci show network | grep device

# 修改 LAN VLAN 1（移除埠 4）
uci set network.eth0_1.ports='1 2 3 0t'

# 修改 WAN VLAN 2（添加埠 4）
uci set network.eth0_2.ports='4 5 0t'

uci commit network
/etc/init.d/network restart
```

### 6. 靜態路由

某些場景需要自訂路由規則：

```bash
# 添加靜態路由：訪問 10.0.0.0/8 透過 192.168.1.100
uci add network route
uci set network.@route[-1].target='10.0.0.0'
uci set network.@route[-1].netmask='255.0.0.0'
uci set network.@route[-1].gateway='192.168.1.100'
uci set network.@route[-1].interface='lan'

uci commit network
/etc/init.d/network restart

# 驗證路由
ip route show
```

## 範例代碼

### 網絡配置腳本

```bash
#!/bin/bash
# setup-network.sh - 標準網絡配置

set -e

LAN_IP="${1:-192.168.1.1}"
LAN_NETMASK="${2:-255.255.255.0}"
WAN_PROTO="${3:-dhcp}"

echo "正在配置網絡..."

# 配置 LAN 接口
uci set network.lan.proto='static'
uci set network.lan.ipaddr="$LAN_IP"
uci set network.lan.netmask="$LAN_NETMASK"
uci set network.lan.ip6assign='60'

# 配置 WAN 接口
uci set network.wan.proto="$WAN_PROTO"
uci set network.wan6.proto='dhcpv6'

# 設置 DNS
uci set network.lan.dns='8.8.8.8 1.1.1.1'

uci commit network
/etc/init.d/network restart

echo "✓ 網絡配置完成"
echo "LAN IP：$LAN_IP"
echo "WAN 模式：$WAN_PROTO"
```

### VLAN 創建腳本

```bash
#!/bin/bash
# create-vlan.sh - 創建新 VLAN

VLAN_ID="${1}"
VLAN_PORTS="${2}"
VLAN_IP="${3}"
VLAN_NETMASK="${4:-255.255.255.0}"
VLAN_NAME="${5:-vlan${VLAN_ID}}"

if [ -z "$VLAN_ID" ] || [ -z "$VLAN_PORTS" ] || [ -z "$VLAN_IP" ]; then
    echo "用法：$0 <vlan_id> <ports> <ip> [netmask] [name]"
    echo "例如：$0 3 '1 0t' '192.168.2.1' '255.255.255.0' 'guest'"
    exit 1
fi

echo "正在創建 VLAN $VLAN_ID..."

# 創建 VLAN 裝置
uci add network device
uci set network.@device[-1].name="eth0.${VLAN_ID}"
uci set network.@device[-1].type='bridge'
uci set network.@device[-1].ports="$VLAN_PORTS"

# 創建邏輯接口
uci add network interface
uci set network.@interface[-1].name="$VLAN_NAME"
uci set network.@interface[-1].ifname="eth0.${VLAN_ID}"
uci set network.@interface[-1].proto='static'
uci set network.@interface[-1].ipaddr="$VLAN_IP"
uci set network.@interface[-1].netmask="$VLAN_NETMASK"
uci set network.@interface[-1].ip6assign='60'

uci commit network
/etc/init.d/network restart

echo "✓ VLAN $VLAN_ID 已創建"
echo "名稱：$VLAN_NAME，IP：$VLAN_IP"
```

## 常見錯誤與解法

### Q：修改 VLAN 配置後網絡斷開
**原因：** CPU 埠標籤設置錯誤或 VLAN 埠配置衝突  
**解法：**
```bash
# 檢查當前 VLAN 配置
uci show network.eth0_1
uci show network.eth0_2

# 確保 CPU 埠（0 或 6）有 'tagged' 標記（'0t' 或 '6t'）
# 例如：ports='1 2 3 4 0t'  # 埠 0 需要 't' 標記

# 重置為默認配置
uci revert network
/etc/init.d/network restart
```

### Q：LAN 設備無 IP 地址
**原因：** 接口未正確綁定到 VLAN 裝置  
**解法：**
```bash
# 檢查接口綁定
uci show network.lan

# 檢查是否綁定了正確的裝置
uci show network.lan.ifname
# 應該為 eth0.1

# 檢查物理層連接
ip link show
```

### Q：IPv6 無法正常獲取
**原因：** IPv6 路由或 DHCPv6 client 配置問題  
**解法：**
```bash
# 檢查 IPv6 配置
uci show network.wan6

# 確保 wan6 接口啟用且設為 dhcpv6
uci set network.wan6.proto='dhcpv6'
uci set network.wan6.ifname='eth0.2'

# 檢查 dhcpv6 client 狀態
ps aux | grep dhcpv6
```

## 參考來源

- [OpenWrt 網絡配置官方文件](https://openwrt.org/docs/guide/network/routing/start)
- [UCI 網絡配置](https://openwrt.org/docs/uci/network)
- [DSA (Distributed Switch Architecture) 指南](https://openwrt.org/docs/guide/network/dsa/start)
- [IPv6 在 OpenWrt 中的配置](https://openwrt.org/docs/guide/network/ipv6/start)

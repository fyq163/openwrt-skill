# DHCP 與 DNS 配置

## 概述

DHCP（Dynamic Host Configuration Protocol）自動為內網設備分配 IP 地址、網關和 DNS 伺服器。DNS（Domain Name System）負責域名解析。OpenWrt 使用 dnsmasq 作為 DHCP server 和 DNS forwarder，提供輕量級且功能全面的解決方案。透過適當配置，可以實現：靜態 DHCP 租賃、DNS 過濾、本地域名解析等。

## 前置條件

- OpenWrt 23.05+ 已安裝
- SSH 訪問權限
- 基本 DHCP/DNS 知識
- dnsmasq 已安裝（通常預設）

## 核心概念

### DHCP 與 DNS 架構

```
LAN 設備（客戶端）
       │
       ▼
DHCP discover/request（廣播）
       │
       ▼
dnsmasq (DHCP server)
       ├─ IP 地址分配（DHCP lease）
       ├─ DNS 伺服器推送
       └─ 靜態租賃（MAC→IP 映射）
       │
       ▼
LAN 設備獲得 IP、網關、DNS
       │
       ▼ (設備進行 DNS 查詢)
       │
       ▼
dnsmasq (DNS resolver)
       ├─ 本地 hosts 記錄
       ├─ 上游 DNS 轉發
       └─ DNS 快取
       │
       ▼
域名解析結果返回
```

### UCI DHCP 配置

DHCP 和 DNS 配置位於 `/etc/config/dhcp`：

```bash
# DHCP server 設置
config dnsmasq
    option domainneeded '1'        # 要求有效域名
    option boguspriv '1'           # 忽略私有 IP 反向查詢
    option filterwin2k '0'         # 過濾 Windows 查詢（可選）
    option cachesize '1000'        # DNS 快取大小
    option authoritative '1'       # 權威模式
    option localise_queries '1'    # 本地化查詢

# DHCP 池設置
config dhcp 'lan'
    option interface 'lan'
    option start '100'             # DHCP 範圍起始（如 192.168.1.100）
    option limit '150'             # DHCP 範圍大小
    option leasetime '12h'         # DHCP 租賃時間

# WAN 的 DNS 伺服器
config dns
    list server '8.8.8.8'
    list server '1.1.1.1'
```

## 操作流程

### 1. 查看當前 DHCP/DNS 配置

```bash
# 查看 DHCP 配置
uci show dhcp

# 查看 dnsmasq 進程
ps aux | grep dnsmasq

# 查看 DNS 配置（/etc/resolv.conf）
cat /etc/resolv.conf

# 查看 DHCP 租賃
cat /tmp/dhcp.leases

# 查看 dnsmasq 統計
echo "stats" | nc localhost 33333  # dnsmasq 統計埠

# 查看本地 hosts 記錄
cat /etc/hosts
```

### 2. 配置 DHCP 伺服器

#### 調整 DHCP 池

```bash
# 查看當前 DHCP 配置
uci show dhcp.lan

# 設置 DHCP IP 範圍（192.168.1.100-249）
uci set dhcp.lan.start='100'
uci set dhcp.lan.limit='150'

# 設置 DHCP 租賃時間（12 小時）
uci set dhcp.lan.leasetime='12h'

# 設置客戶端主機名解析
uci set dhcp.lan.hostname='OpenWrt'

uci commit dhcp
/etc/init.d/dnsmasq restart
```

#### 靜態 DHCP 租賃（MAC→IP 映射）

為特定設備指派固定 IP 地址：

```bash
# 方式 1：UCI 配置
uci add dhcp host
uci set dhcp.@host[-1].name='printer'        # 設備名稱
uci set dhcp.@host[-1].mac='aa:bb:cc:dd:ee:ff'  # 設備 MAC
uci set dhcp.@host[-1].ip='192.168.1.110'   # 分配 IP

uci add dhcp host
uci set dhcp.@host[-1].name='camera'
uci set dhcp.@host[-1].mac='11:22:33:44:55:66'
uci set dhcp.@host[-1].ip='192.168.1.111'

uci commit dhcp
/etc/init.d/dnsmasq restart

# 驗證靜態租賃
cat /tmp/dhcp.leases | grep 'printer\|camera'
```

### 3. 配置 DNS

#### 設置上游 DNS 伺服器

```bash
# 查看當前 DNS
uci show dhcp.@dnsmasq

# 設置上游 DNS（Google, Cloudflare）
uci set dhcp.@dnsmasq[0].server='8.8.8.8'
uci add_list dhcp.@dnsmasq[0].server='8.8.4.4'
uci add_list dhcp.@dnsmasq[0].server='1.1.1.1'
uci add_list dhcp.@dnsmasq[0].server='1.0.0.1'

# 設置 DNS 轉發埠（如非標準 53）
uci set dhcp.@dnsmasq[0].port='53'

# 啟用 DNS 快取
uci set dhcp.@dnsmasq[0].cachesize='1000'

uci commit dhcp
/etc/init.d/dnsmasq restart
```

#### 本地域名解析（Hosts）

```bash
# 添加本地域名映射
echo "192.168.1.100 gateway.local" >> /etc/hosts
echo "192.168.1.110 printer.local" >> /etc/hosts
echo "192.168.1.111 camera.local" >> /etc/hosts

# 或使用 UCI
uci add dhcp domain
uci set dhcp.@domain[-1].name='gateway.local'
uci set dhcp.@domain[-1].ip='192.168.1.1'

uci commit dhcp
/etc/init.d/dnsmasq restart

# 驗證本地域名
nslookup gateway.local
dig printer.local
```

#### DNS 過濾（廣告攔截）

dnsmasq 可阻止特定域名：

```bash
# 創建 DNS 黑名單文件
cat > /etc/dnsmasq.d/adblock.conf <<'EOF'
# 阻止廣告域名
address=/ads.google.com/127.0.0.1
address=/facebook.com/127.0.0.1
address=/doubleclick.net/127.0.0.1

# 或使用 address 指定多個域名
address=/example.com/example.net/another.com/10.0.0.1
EOF

# 重啟 dnsmasq
/etc/init.d/dnsmasq restart

# 驗證過濾
nslookup ads.google.com  # 應返回 127.0.0.1
```

### 4. DNS 上游配置（Advanced）

#### 使用 DoH（DNS over HTTPS）或 DoT（DNS over TLS）

某些應用場景需要加密 DNS：

```bash
# 安裝 DNS 加密工具
opkg install https-dns-proxy

# 配置 HTTPS DNS Proxy
uci add https-dns-proxy main
uci set https-dns-proxy.@main[-1].enabled='1'
uci set https-dns-proxy.@main[-1].provider='https://dns.google/dns-query'

uci commit https-dns-proxy
/etc/init.d/https-dns-proxy start
```

### 5. DHCP 和 DNS 故障排除

#### 檢查日誌

```bash
# 查看 dnsmasq 日誌
logread -e dnsmasq

# 或啟用詳細日誌
uci set dhcp.@dnsmasq[0].logdhcp='1'
uci set dhcp.@dnsmasq[0].logqueries='1'
uci commit dhcp
/etc/init.d/dnsmasq restart

# 實時監控日誌
logread -f | grep dnsmasq
```

## 範例代碼

### DHCP/DNS 配置腳本

```bash
#!/bin/bash
# setup-dhcp-dns.sh - 配置 DHCP 和 DNS

set -e

LAN_IP="${1:-192.168.1.1}"
DHCP_START="${2:-100}"
DHCP_LIMIT="${3:-150}"
UPSTREAM_DNS="${4:-8.8.8.8 1.1.1.1}"

echo "正在配置 DHCP 和 DNS..."

# DHCP 設置
uci set dhcp.lan.start="$DHCP_START"
uci set dhcp.lan.limit="$DHCP_LIMIT"
uci set dhcp.lan.leasetime='12h'

# DNS 設置
uci set dhcp.@dnsmasq[0].cachesize='1000'
uci set dhcp.@dnsmasq[0].filterwin2k='0'

# 清空現有 DNS 伺服器，添加新的
uci delete dhcp.@dnsmasq[0].server
for dns in $UPSTREAM_DNS; do
    uci add_list dhcp.@dnsmasq[0].server="$dns"
done

uci commit dhcp
/etc/init.d/dnsmasq restart

echo "✓ DHCP/DNS 配置完成"
echo "DHCP 池：$DHCP_START-$((DHCP_START + DHCP_LIMIT - 1))"
echo "上游 DNS：$UPSTREAM_DNS"
```

### 靜態 DHCP 租賃管理腳本

```bash
#!/bin/bash
# manage-static-dhcp.sh - 管理靜態 DHCP 租賃

ACTION="${1:-list}"
HOSTNAME="${2}"
MAC="${3}"
IP="${4}"

case "$ACTION" in
    list)
        echo "當前靜態 DHCP 租賃："
        uci show dhcp.@host 2>/dev/null || echo "無靜態租賃"
        ;;
    add)
        if [ -z "$HOSTNAME" ] || [ -z "$MAC" ] || [ -z "$IP" ]; then
            echo "用法：$0 add <hostname> <mac> <ip>"
            exit 1
        fi
        
        uci add dhcp host
        uci set dhcp.@host[-1].name="$HOSTNAME"
        uci set dhcp.@host[-1].mac="$MAC"
        uci set dhcp.@host[-1].ip="$IP"
        uci commit dhcp
        /etc/init.d/dnsmasq restart
        
        echo "✓ 已添加靜態租賃：$HOSTNAME ($MAC) → $IP"
        ;;
    remove)
        if [ -z "$HOSTNAME" ]; then
            echo "用法：$0 remove <hostname>"
            exit 1
        fi
        
        # 查找並刪除
        uci show dhcp.@host | grep "name='$HOSTNAME'" | cut -d. -f2 | while read host; do
            uci delete dhcp."$host"
        done
        uci commit dhcp
        /etc/init.d/dnsmasq restart
        
        echo "✓ 已移除靜態租賃：$HOSTNAME"
        ;;
    *)
        echo "用法："
        echo "  $0 list              # 列出所有靜態租賃"
        echo "  $0 add <host> <mac> <ip>  # 添加靜態租賃"
        echo "  $0 remove <host>     # 移除靜態租賃"
        ;;
esac
```

## 常見錯誤與解法

### Q：LAN 設備無法獲得 IP 地址
**原因：** DHCP 服務未運行或 DHCP 池已滿  
**解法：**
```bash
# 檢查 dnsmasq 是否運行
/etc/init.d/dnsmasq status

# 查看 DHCP 池配置
uci show dhcp.lan

# 檢查 DHCP 租賃
cat /tmp/dhcp.leases | wc -l

# 如池已滿，擴大范圍
uci set dhcp.lan.limit='200'
uci commit dhcp
/etc/init.d/dnsmasq restart
```

### Q：DNS 解析失敗（Unknown host）
**原因：** 上游 DNS 配置錯誤或網絡不通  
**解法：**
```bash
# 檢查上游 DNS
uci show dhcp.@dnsmasq[0].server

# 測試外部 DNS
nslookup google.com 8.8.8.8

# 檢查 dnsmasq 是否正運行
ps aux | grep dnsmasq

# 查看日誌
logread -e dnsmasq | tail -20

# 測試本地查詢
nslookup localhost
```

### Q：靜態 DHCP 租賃未生效
**原因：** MAC 地址錯誤或 dnsmasq 未重啟  
**解法：**
```bash
# 檢查靜態租賃配置
uci show dhcp.@host

# 確認 MAC 地址正確
# 在設備上運行：ipconfig /all (Windows) 或 ip link (Linux)

# 重啟 dnsmasq
/etc/init.d/dnsmasq restart

# 檢查 DHCP 租賃日誌
logread -e dnsmasq | grep -i "static"
```

## 參考來源

- [OpenWrt DHCP/DNS 官方文件](https://openwrt.org/docs/guide/network/dns/dnsmasq_configuration)
- [dnsmasq 官方文件](http://www.thekelleys.org.uk/dnsmasq/docs/dnsmasq-man.html)
- [UCI DHCP 配置](https://openwrt.org/docs/uci/dhcp)
- [DHCP RFC 2131](https://tools.ietf.org/html/rfc2131)
- [DNS RFC 1035](https://tools.ietf.org/html/rfc1035)

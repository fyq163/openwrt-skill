# DDNS 動態 DNS 配置

## 概述

DDNS（Dynamic Domain Name System）允許使用動態公網 IP 的路由器自動更新 DNS 記錄，使域名始終指向當前的 IP 地址。OpenWrt 透過 ddns-scripts 套件支援多個 DDNS 提供者，包括 Cloudflare、DNSPod、DigitalOcean 等。本文重點涵蓋 Cloudflare DDNS 配置。

## 前置條件

- OpenWrt 已安裝並可訪問
- SSH 連接到路由器
- 擁有一個域名並註冊於 DDNS 提供者（如 Cloudflare）
- 獲得 API 令牌或認證憑證
- ddns-scripts 或 ddns-scripts-cloudflare 套件已安裝

## 核心概念

### DDNS 運作流程

```
用戶端 (OpenWrt 路由器)
       │
       ├─ 偵測 WAN IP 變化
       ├─ 向 DDNS 提供者發送更新請求
       │   ├─ 使用 API 呼叫（較新方式）
       │   └─ HTTP 更新協議（較舊方式）
       │
       ▼
DDNS 提供者 (Cloudflare/DNSPod/etc)
       │
       ├─ 驗證認證資訊
       ├─ 更新 DNS A/AAAA 記錄
       │
       ▼
全球 DNS 伺服器
       │
       └─ 將 yourdomain.com 指向新 IP
```

### Cloudflare DDNS

Cloudflare 提供兩種更新方式：

| 方式 | 認證方式 | 優點 | 缺點 |
|------|---------|------|------|
| **API Token** | ****** | 精確控制權限範圍 | 需要生成 token |
| **Global API Key + Email** | 郵件 + 全局密鑰 | 簡單 | 權限過廣，安全性較低 |

### UCI 配置結構

DDNS 配置位於 `/etc/config/ddns`：

```bash
config service 'myddns'
    option enabled '1'             # 是否啟用
    option service_name 'cloudflare.com-v4'  # 服務提供者
    option domain 'example.com'    # 要更新的域名
    option username 'your@email.com'        # Cloudflare 郵件
    option password 'your_api_token'        # API Token 或全局密鑰
    option interface 'wan'         # 監控的網絡接口
    option check_interval '300'    # 檢查 IP 變化的間隔（秒）
    option force_interval '3600'   # 強制更新的間隔（秒）
```

## 操作流程

### 1. 安裝 DDNS 套件

```bash
# 連接到路由器
ssh root@192.168.1.1

# 更新套件列表
opkg update

# 安裝 DDNS 指令碼
opkg install ddns-scripts ddns-scripts-cloudflare

# 驗證安裝
opkg list-installed | grep ddns
```

### 2. 在 Cloudflare 生成 API Token

**步驟：**

1. 登入 [Cloudflare Dashboard](https://dash.cloudflare.com/)
2. 右上角帳號 → 「My Profile」→ 「API Tokens」
3. 點擊「Create Token」
4. 選擇「Edit zone DNS」模板（或自訂權限）
5. 設置權限：
   - **Permissions**: Zone → DNS → Edit
   - **Zone Resources**: Include → Specific zone → yourdomain.com
   - **IP Restrictions**: 可選（限制 IP 白名單）
6. 點擊「Continue to summary」→ 「Create Token」
7. 複製 token 值（僅顯示一次）

### 3. 配置 DDNS（使用 UCI）

#### 方式 A：API Token（推薦）

```bash
# SSH 連接到路由器
ssh root@192.168.1.1

# 添加 DDNS 服務
uci add ddns service
uci set ddns.@service[-1].enabled='1'
uci set ddns.@service[-1].service_name='cloudflare.com-v4'
uci set ddns.@service[-1].domain='yourdomain.com'
uci set ddns.@service[-1].username='your@email.com'
uci set ddns.@service[-1].password='your_api_token'
uci set ddns.@service[-1].interface='wan'
uci set ddns.@service[-1].check_interval='300'
uci set ddns.@service[-1].force_interval='86400'  # 每天強制更新一次

# 提交配置
uci commit ddns

# 查看配置
uci show ddns
```

#### 方式 B：編輯設定檔

```bash
# 直接編輯 /etc/config/ddns
ssh root@192.168.1.1 'cat > /etc/config/ddns' <<'EOF'
config service 'myddns'
    option enabled '1'
    option service_name 'cloudflare.com-v4'
    option domain 'yourdomain.com'
    option username 'your@email.com'
    option password 'your_api_token'
    option interface 'wan'
    option check_interval '300'
    option force_interval '86400'
    option use_https '1'
    option cacert '/etc/ssl/certs/ca-certificates.crt'
EOF
```

### 4. 啟動並測試 DDNS 服務

```bash
# 啟動 DDNS 服務
/etc/init.d/ddns start

# 檢查 DDNS 服務狀態
/etc/init.d/ddns status

# 查看 DDNS 日誌
logread -e ddns

# 或實時監控日誌
logread -f | grep ddns

# 手動觸發 DDNS 更新（用於測試）
# 方式：改變 WAN IP 或強制更新
ddns-scripts-cloudflare /etc/config/ddns myddns

# 檢查 DNS 解析是否正確
nslookup yourdomain.com
# 或使用 dig
dig yourdomain.com
```

### 5. 配置開機自啟

```bash
# 使 DDNS 在開機時自動啟動
uci set ddns.@service[0].enabled='1'
uci commit ddns

# 添加到開機啟動列表
/etc/init.d/ddns enable

# 驗證開機啟動已設置
ls -la /etc/rc.d/S*ddns
```

### 6. IPv6 支援（可選）

如果路由器有 IPv6 地址，可配置同時更新 AAAA 記錄：

```bash
# 添加 IPv6 DDNS 服務
uci add ddns service
uci set ddns.@service[-1].enabled='1'
uci set ddns.@service[-1].service_name='cloudflare.com-v6'
uci set ddns.@service[-1].domain='yourdomain.com'
uci set ddns.@service[-1].username='your@email.com'
uci set ddns.@service[-1].password='your_api_token'
uci set ddns.@service[-1].interface='wan6'  # IPv6 接口
uci set ddns.@service[-1].check_interval='300'
uci commit ddns

# 重啟 DDNS 服務
/etc/init.d/ddns restart
```

## 範例代碼

### DDNS 配置腳本

```bash
#!/bin/bash
# setup-ddns-cloudflare.sh - Cloudflare DDNS 自動設置

set -e

DOMAIN="${1:-yourdomain.com}"
EMAIL="${2:-your@email.com}"
API_TOKEN="${3}"
WAN_INTERFACE="${4:-wan}"

if [ -z "$API_TOKEN" ]; then
    echo "用法：$0 <domain> <email> <api_token> [interface]"
    echo "例如：$0 example.com user@email.com z9x8c7v6b5a4s3d2f1g0h9 wan"
    exit 1
fi

echo "正在配置 Cloudflare DDNS..."

# 安裝套件
opkg update
opkg install -y ddns-scripts ddns-scripts-cloudflare

# 配置 DDNS
uci set ddns.@service[0].enabled='1'
uci set ddns.@service[0].service_name='cloudflare.com-v4'
uci set ddns.@service[0].domain="$DOMAIN"
uci set ddns.@service[0].username="$EMAIL"
uci set ddns.@service[0].******
uci set ddns.@service[0].interface="$WAN_INTERFACE"
uci set ddns.@service[0].check_interval='300'
uci set ddns.@service[0].force_interval='86400'
uci commit ddns

# 啟動服務
/etc/init.d/ddns enable
/etc/init.d/ddns start

echo "DDNS 配置完成！"
echo "域名：$DOMAIN"
echo "檢查日誌：logread | grep ddns"
```

### DDNS 監控腳本

```bash
#!/bin/bash
# monitor-ddns.sh - 監控 DDNS 更新狀態

DOMAIN="${1:-yourdomain.com}"

while true; do
    echo "=== $(date) ==="
    
    # 獲取當前 WAN IP
    WAN_IP=$(curl -s http://api.ipify.org)
    echo "當前 WAN IP：$WAN_IP"
    
    # DNS 查詢
    DNS_IP=$(dig +short $DOMAIN @8.8.8.8)
    echo "DNS 解析 IP：$DNS_IP"
    
    # 比較
    if [ "$WAN_IP" = "$DNS_IP" ]; then
        echo "✓ DDNS 已同步"
    else
        echo "✗ DDNS 未同步（可能更新中）"
    fi
    
    echo "DDNS 狀態："
    /etc/init.d/ddns status || echo "服務未運行"
    
    echo ""
    sleep 60
done
```

## 常見錯誤與解法

### Q：DDNS 顯示「Auth failed」（認證失敗）
**原因：** API Token 或郵件/密碼錯誤  
**解法：**
```bash
# 檢查配置
uci show ddns

# 重新設置認證資訊
uci set ddns.@service[0].username='correct@email.com'
uci set ddns.@service[0].password='correct_api_token'
uci commit ddns

# 重啟 DDNS
/etc/init.d/ddns restart
```

### Q：DNS 未更新（仍指向舊 IP）
**原因：** DDNS 服務未運行或更新失敗  
**解法：**
```bash
# 檢查服務是否運行
/etc/init.d/ddns status

# 查看詳細日誌
logread -e ddns | tail -20

# 手動測試更新
ddns-scripts-cloudflare /etc/config/ddns myddns

# 檢查 WAN 連接
ifconfig wan
```

### Q：無法解析 Cloudflare API（DNS 錯誤）
**原因：** WAN DNS 配置不正確或網絡連接問題  
**解法：**
```bash
# 測試 DNS 解析
nslookup api.cloudflare.com
ping api.cloudflare.com

# 檢查 WAN 的 DNS 設置
uci show network.wan

# 設置手動 DNS（Google Public DNS）
uci set network.wan.peerdns='0'
uci set network.wan.dns='8.8.8.8 8.8.4.4'
uci commit network
/etc/init.d/network restart
```

## 參考來源

- [OpenWrt DDNS 官方文件](https://openwrt.org/docs/guide/services/ddns/client)
- [Cloudflare API 文件](https://developers.cloudflare.com/api/)
- [ddns-scripts GitHub](https://github.com/openwrt/packages/tree/master/net/ddns-scripts)
- [Cloudflare DNS 查詢工具](https://mxtoolbox.com/)

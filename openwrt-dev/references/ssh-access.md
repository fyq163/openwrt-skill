# SSH 訪問與遠程管理

## 概述

SSH 是訪問和管理 OpenWrt 路由器的基本方式。通過 SSH，可以遠程執行指令、傳輸檔案、進行配置修改和故障排除。OpenWrt 預設使用 dropbear SSH 伺服器（輕量級）或 OpenSSH，支援密鑰認證和密碼認證。

## 前置條件

- OpenWrt 已安裝在路由器上
- 網絡連接（通常為 LAN 或 WiFi）
- SSH 客戶端工具（ssh、scp、sftp 等）
- 路由器的 IP 地址（通常為 192.168.1.1）
- root 密碼或 SSH 公鑰

## 核心概念

### SSH 服務架構

```
用戶端 (ssh/scp/sftp)
       │
       ▼
SSH 伺服器 (dropbear/openssh)
       │
       ├─ 密鑰認證 (/root/.ssh/authorized_keys)
       ├─ 密碼認證 (root 密碼)
       └─ 連接埠 (預設 22)
       │
       ▼
Linux shell (ash/bash)
```

### SSH 服務配置

SSH 服務由 `/etc/config/dropbear` 或 `/etc/config/openssh` 配置：

```bash
# 查看 SSH 服務狀態
service dropbear status
service sshd status

# SSH 監聽埠
uci show dropbear.@dropbear[0].Port
# 典型值：22

# SSH 允許 root 登入
uci show dropbear.@dropbear[0].RootLogin
# 典型值：on
```

### 認證方式

| 認證方式 | 優點 | 缺點 | 適用場景 |
|---------|------|------|----------|
| **密碼** | 簡單、快速 | 容易被暴力破解 | 內網、臨時訪問 |
| **公鑰** | 安全、無密碼 | 需要管理密鑰文件 | 生產環境、自動化 |

## 操作流程

### 1. 基本 SSH 訪問

#### 密碼認證

```bash
# 連接到路由器（使用密碼）
ssh root@192.168.1.1
# 輸入 root 密碼後進入 shell

# 一次性執行指令
ssh root@192.168.1.1 'uci show network'

# 帶 verbose 輸出（用於除錯）
ssh -v root@192.168.1.1
```

#### 公鑰認證

```bash
# 在客戶端生成 SSH 密鑰對（如未擁有）
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""

# 複製公鑰到路由器
ssh-copy-id -i ~/.ssh/id_rsa.pub root@192.168.1.1
# 或手動複製：
cat ~/.ssh/id_rsa.pub | ssh root@192.168.1.1 'cat >> /root/.ssh/authorized_keys'

# 使用公鑰連接（無需密碼）
ssh -i ~/.ssh/id_rsa root@192.168.1.1

# 配置 SSH config 簡化訪問
# 編輯 ~/.ssh/config：
cat >> ~/.ssh/config <<EOF
Host openwrt
    HostName 192.168.1.1
    User root
    IdentityFile ~/.ssh/id_rsa
    Port 22
EOF
ssh openwrt  # 之後可直接使用此別名
```

### 2. 檔案傳輸

```bash
# 從路由器複製檔案到本地
scp root@192.168.1.1:/tmp/config.backup ~/config.backup

# 複製本地檔案到路由器
scp ~/my-package_1.0-1_ar71xx.ipk root@192.168.1.1:/tmp/

# 複製整個目錄（遞歸）
scp -r root@192.168.1.1:/etc/config ~/router-config-backup/

# 使用 SFTP 互動式傳輸
sftp root@192.168.1.1
> ls /etc/config/
> get /etc/config/network
> put ~/modified-network
> quit
```

### 3. SSH 服務配置

#### 啟用/禁用 SSH 服務

```bash
# 查看 SSH 服務狀態
/etc/init.d/dropbear status

# 啟動 SSH 服務
/etc/init.d/dropbear start

# 停止 SSH 服務
/etc/init.d/dropbear stop

# 重啟 SSH 服務
/etc/init.d/dropbear restart

# 設置開機自啟
uci set dropbear.@dropbear[0].enable='1'
uci commit dropbear
```

#### 修改 SSH 監聽埠

```bash
# 改為監聽 2222 埠（避免外網直接攻擊 22 埠）
uci set dropbear.@dropbear[0].Port='2222'
uci commit dropbear
/etc/init.d/dropbear restart

# 現在需要指定埠連接
ssh -p 2222 root@192.168.1.1
```

#### 禁用 root 密碼登入（強制使用公鑰）

```bash
# 將 RootLogin 改為 off
uci set dropbear.@dropbear[0].RootLogin='off'
uci commit dropbear
/etc/init.d/dropbear restart

# 之後必須使用 SSH 公鑰，密碼登入會被拒絕
```

### 4. SSH 密鑰管理

```bash
# 在路由器上設置 authorized_keys
ssh root@192.168.1.1 'mkdir -p /root/.ssh'
ssh-copy-id -i ~/.ssh/id_rsa.pub root@192.168.1.1

# 檢查路由器上的授權密鑰
ssh root@192.168.1.1 'cat /root/.ssh/authorized_keys'

# 移除授權密鑰
ssh root@192.168.1.1 'echo "" > /root/.ssh/authorized_keys'

# 備份路由器的 SSH 主機密鑰（用於簽名識別）
scp root@192.168.1.1:/etc/dropbear/dropbear_rsa_host_key ~/dropbear_host_key.backup
```

### 5. 遠程批量執行

```bash
# 執行多個指令
ssh root@192.168.1.1 <<'EOF'
    uci show network
    uci show wireless
    ps aux | grep dropbear
EOF

# 在本地執行並傳送結果到路由器
echo "config interface 'guest'" | ssh root@192.168.1.1 'cat >> /etc/config/network'

# 結合 SSH 與本地腳本
ssh root@192.168.1.1 'source /dev/stdin' < ./router-setup.sh
```

## 範例代碼

### SSH 連接腳本（Bash）

```bash
#!/bin/bash
# router-connect.sh - 連接到 OpenWrt 路由器的簡化腳本

ROUTER_IP="${1:-192.168.1.1}"
ROUTER_USER="${2:-root}"
SSH_KEY="${3:-$HOME/.ssh/id_rsa}"

# 檢查 SSH 密鑰是否存在
if [ ! -f "$SSH_KEY" ]; then
    echo "SSH 密鑰不存在：$SSH_KEY"
    echo "使用密碼認證（較不安全）"
    ssh "$ROUTER_USER@$ROUTER_IP"
else
    echo "使用公鑰認證連接 $ROUTER_IP"
    ssh -i "$SSH_KEY" "$ROUTER_USER@$ROUTER_IP"
fi
```

### 配置備份腳本

```bash
#!/bin/bash
# backup-router-config.sh - 備份路由器配置

ROUTER_IP="192.168.1.1"
BACKUP_DIR="$HOME/router-backups/$(date +%Y%m%d_%H%M%S)"

mkdir -p "$BACKUP_DIR"

# 備份 /etc/config/ 目錄
scp -r root@$ROUTER_IP:/etc/config "$BACKUP_DIR/"

# 備份 SSH 主機密鑰
scp root@$ROUTER_IP:/etc/dropbear/dropbear_rsa_host_key "$BACKUP_DIR/"

# 備份包列表
ssh root@$ROUTER_IP 'opkg list-installed' > "$BACKUP_DIR/packages.txt"

echo "備份完成：$BACKUP_DIR"
```

### SSH 自動化登入（expect 腳本）

```bash
#!/usr/bin/expect -f
# auto-ssh.exp - 自動輸入 SSH 密碼

set ROUTER_IP "192.168.1.1"
set ROUTER_PASS "your_password_here"

spawn ssh root@$ROUTER_IP
expect "password:"
send "$ROUTER_PASS\r"
interact
```

## 常見錯誤與解法

### Q：連接被拒絕（Connection refused）
**原因：** SSH 服務未啟動或埠配置錯誤  
**解法：**
```bash
# 檢查 SSH 服務是否運行
ssh root@192.168.1.1 '/etc/init.d/dropbear status'

# 如未運行，啟動它
ssh root@192.168.1.1 '/etc/init.d/dropbear start'

# 檢查監聽埠
ssh root@192.168.1.1 'netstat -ltn | grep :22'
```

### Q：主機鑰匙識別失敗（Host key verification failed）
**原因：** 首次連接時未接受主機鑰匙  
**解法：**
```bash
# 清空已知主機列表（如重新安裝系統）
ssh-keygen -R 192.168.1.1

# 再次連接，接受新鑰匙
ssh root@192.168.1.1
```

### Q：公鑰認證失敗（Permission denied）
**原因：** 密鑰文件權限不正確或未正確複製  
**解法：**
```bash
# 修復本地密鑰文件權限
chmod 600 ~/.ssh/id_rsa
chmod 644 ~/.ssh/id_rsa.pub

# 檢查遠程 authorized_keys 權限
ssh root@192.168.1.1 'ls -la /root/.ssh/'
# 應為：-rw-r--r-- authorized_keys

# 修復遠程權限
ssh root@192.168.1.1 'chmod 700 /root/.ssh && chmod 600 /root/.ssh/authorized_keys'
```

### Q：密碼登入速度緩慢或超時
**原因：** SSH 解析反向 DNS 或使用密鑰交換演算法  
**解法：**
```bash
# 禁用 DNS 反向查詢（加快連接）
ssh -o UseDNS=no root@192.168.1.1

# 或在 ~/.ssh/config 中設置
cat >> ~/.ssh/config <<EOF
Host 192.168.1.1
    UseDNS no
EOF
```

## 參考來源

- [OpenWrt 官方 - SSH](https://openwrt.org/docs/guide/security/secure_shell)
- [Dropbear SSH - 官方文件](https://matt.ucc.asn.au/dropbear/dropbear.html)
- [OpenSSH 官方網站](https://www.openssh.com/)
- [SSH 最佳實踐 - Teleport](https://goteleport.com/blog/ssh-key-management/)

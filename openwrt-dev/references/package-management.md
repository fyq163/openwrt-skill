# 包管理與系統設置

## 概述

OpenWrt 使用 opkg（OpenWrt Package Manager）進行包管理。透過 opkg，可以安裝、升級、卸載軟件套件。系統設置涵蓋時區、主機名、時間同步、系統日誌、性能調優等。正確的包管理和系統配置是維護穩定路由器的基礎。

## 前置條件

- OpenWrt 23.05+ 已安裝
- SSH 訪問權限
- WAN 網絡連接（下載包）
- 基本 Linux 包管理知識

## 核心概念

### 包管理架構

```
opkg 命令
       │
       ├─ 本地包數據庫（/var/opkg-lists/）
       ├─ 遠程倉庫索引（feeds）
       │  ├─ base（基礎包）
       │  ├─ packages（社區包）
       │  └─ luci（LuCI 應用）
       │
       ▼
包下載和安裝
       │
       ├─ 下載到 /tmp
       ├─ 解包到 /usr, /etc, /root
       └─ 運行安裝腳本
       │
       ▼
系統運行
```

### 倉庫配置

opkg 倉庫在 `/etc/opkg/distfeeds.conf` 中定義：

```bash
src/gz openwrt_core https://downloads.openwrt.org/releases/23.05.0/targets/ar71xx/generic/packages
src/gz openwrt_base https://downloads.openwrt.org/releases/23.05.0/packages/mips_24kc/base
src/gz openwrt_luci https://downloads.openwrt.org/releases/23.05.0/packages/mips_24kc/luci
src/gz openwrt_packages https://downloads.openwrt.org/releases/23.05.0/packages/mips_24kc/packages
```

### UCI 系統配置

系統設置位於 `/etc/config/system`：

```bash
config system
    option hostname 'OpenWrt'
    option timezone 'UTC'
    option zonename 'UTC'
    option log_size '64'           # 系統日誌大小（KB）
    option log_ip '0.0.0.0'        # 遠程日誌伺服器（可選）
```

## 操作流程

### 1. 包管理基本操作

#### 查看已安裝的包

```bash
# 列出所有已安裝包
opkg list-installed

# 查看特定包的信息
opkg info nginx

# 查看包的大小
opkg list-installed | grep "nginx"

# 查看包的依賴
opkg depends nginx
```

#### 搜索包

```bash
# 更新包索引
opkg update

# 搜索包
opkg list | grep "vim"
opkg list | grep "ntp"

# 查看包詳細信息
opkg list-available | grep "^nginx"
```

#### 安裝和卸載包

```bash
# 安裝單個包
opkg install vim

# 安裝多個包
opkg install vim curl wget

# 安裝特定版本
opkg install 'libopenssl=1.1.1n-1'

# 卸載包
opkg remove vim

# 卸載並清理依賴
opkg autoremove

# 強制重新安裝
opkg install --force-reinstall vim

# 安裝本地包文件
opkg install /tmp/myapp_1.0-1_mips_24kc.ipk
```

#### 升級包

```bash
# 升級所有包
opkg upgrade

# 升級特定包
opkg upgrade libopenssl

# 查看可升級的包
opkg list-upgradable
```

### 2. 系統配置

#### 設置主機名和時區

```bash
# 設置主機名
uci set system.@system[0].hostname='MyRouter'

# 設置時區（查看可用時區）
ls /usr/share/zoneinfo/
# 設置時區為 Asia/Shanghai
uci set system.@system[0].timezone='CST-8'
uci set system.@system[0].zonename='Asia/Shanghai'

uci commit system
```

#### 時間同步（NTP）

```bash
# 安裝 NTP 伺服器
opkg install ntpd

# 或安裝 chrony（更輕量級）
opkg install chrony

# 查看 NTP 設置
uci show system.ntp

# 設置 NTP 伺服器
uci set system.ntp.server='0.openwrt.pool.ntp.org' '1.openwrt.pool.ntp.org'

# 啟動 NTP 服務
/etc/init.d/ntpd enable
/etc/init.d/ntpd start

# 查看時間同步狀態
date
ntpq -p  # 如已安裝 NTP
```

#### 系統日誌配置

```bash
# 查看日誌大小限制
uci show system.@system[0].log_size

# 修改日誌大小（64 KB）
uci set system.@system[0].log_size='64'

# 配置遠程日誌伺服器（可選）
uci set system.@system[0].log_server='192.168.1.100'
uci set system.@system[0].log_port='514'

uci commit system

# 查看系統日誌
logread

# 實時監控日誌
logread -f

# 清空日誌
> /var/log/messages
```

### 3. 常用工具包安裝

#### 網絡診斷工具

```bash
# ping, traceroute, dig（DNS）
opkg install iputils-ping iputils-traceroute bind-dig

# mtr（路由追蹤）
opkg install mtr

# iperf（性能測試）
opkg install iperf3

# tcpdump（流量捕獲）
opkg install tcpdump

# curl/wget（文件下載）
opkg install curl wget

# netcat（網絡連接）
opkg install netcat-openbsd

# nmap（網絡掃描）
opkg install nmap
```

#### 文本編輯和系統工具

```bash
# vim（高級編輯器）
opkg install vim

# nano（簡單編輯器）
opkg install nano

# htop（進程監控）
opkg install htop

# git（版本控制）
opkg install git

# screen（會話管理）
opkg install screen
```

### 4. 管理存儲空間

OpenWrt 通常儲存空間有限，需要仔細管理：

```bash
# 查看磁盤使用情況
df -h

# 查看目錄大小
du -sh /usr /etc /root /tmp

# 清理包管理器快取
opkg clean
rm -rf /var/opkg-lists

# 移除未使用的依賴
opkg autoremove

# 列出最大的包
opkg list-installed | sort -t'-' -k2 -rn | head -20

# 卸載不需要的包以節省空間
opkg remove openssh-client openssh-server  # 如不需要 SSH
```

### 5. 性能調優

#### 內存優化

```bash
# 查看內存使用
free -h

# 清理緩存
sync && echo 3 > /proc/sys/vm/drop_caches

# 查看 top 進程
top -n 1 | head -20
```

#### 網絡優化

```bash
# 調整 TCP 緩衝大小
sysctl -w net.ipv4.tcp_rmem='4096 87380 67108864'
sysctl -w net.ipv4.tcp_wmem='4096 65536 67108864'

# 啟用 TCP Fast Open
sysctl -w net.ipv4.tcp_fastopen=3

# 調整並發連接限制
sysctl -w net.ipv4.ip_conntrack_max=16777216
```

## 範例代碼

### 包管理腳本

```bash
#!/bin/bash
# manage-packages.sh - 包管理助手

ACTION="${1:-list}"
PACKAGE="${2}"

case "$ACTION" in
    update)
        echo "正在更新包索引..."
        opkg update
        ;;
    list)
        echo "已安裝包："
        opkg list-installed
        ;;
    search)
        if [ -z "$PACKAGE" ]; then
            echo "用法：$0 search <package_name>"
            exit 1
        fi
        echo "搜索：$PACKAGE"
        opkg list | grep "$PACKAGE"
        ;;
    install)
        if [ -z "$PACKAGE" ]; then
            echo "用法：$0 install <package_name>"
            exit 1
        fi
        echo "正在安裝：$PACKAGE"
        opkg install "$PACKAGE"
        ;;
    remove)
        if [ -z "$PACKAGE" ]; then
            echo "用法：$0 remove <package_name>"
            exit 1
        fi
        echo "正在移除：$PACKAGE"
        opkg remove "$PACKAGE"
        ;;
    upgrade)
        echo "正在升級所有包..."
        opkg upgrade
        ;;
    autoremove)
        echo "正在移除未使用的依賴..."
        opkg autoremove
        ;;
    clean)
        echo "正在清理包快取..."
        opkg clean
        rm -rf /var/opkg-lists
        ;;
    space)
        echo "磁盤使用情況："
        df -h
        echo ""
        echo "目錄大小："
        du -sh /usr /etc /root /tmp
        ;;
    *)
        echo "用法："
        echo "  $0 update              # 更新包索引"
        echo "  $0 list                # 列出已安裝包"
        echo "  $0 search <pkg>        # 搜索包"
        echo "  $0 install <pkg>       # 安裝包"
        echo "  $0 remove <pkg>        # 移除包"
        echo "  $0 upgrade             # 升級所有包"
        echo "  $0 autoremove          # 移除未使用包"
        echo "  $0 clean               # 清理快取"
        echo "  $0 space               # 查看磁盤使用"
        ;;
esac
```

### 系統配置初始化腳本

```bash
#!/bin/bash
# init-system.sh - 初始化系統設置

HOSTNAME="${1:-OpenWrt}"
TIMEZONE="${2:-UTC}"

echo "正在初始化系統..."

# 設置主機名
uci set system.@system[0].hostname="$HOSTNAME"

# 設置時區
uci set system.@system[0].timezone="$TIMEZONE"

# 設置日誌大小
uci set system.@system[0].log_size='64'

uci commit system

# 安裝基本包
opkg update
opkg install vim curl wget ntpd

# 配置 NTP
uci set system.ntp.server='0.openwrt.pool.ntp.org' '1.openwrt.pool.ntp.org'
uci commit system

# 啟動 NTP
/etc/init.d/ntpd enable
/etc/init.d/ntpd start

echo "✓ 系統初始化完成"
echo "主機名：$HOSTNAME"
echo "時區：$TIMEZONE"
```

## 常見錯誤與解法

### Q：無法下載包（opkg update 失敗）
**原因：** WAN 網絡無連接或倉庫鏡像不可用  
**解法：**
```bash
# 檢查 WAN 連接
ping 8.8.8.8

# 檢查 DNS 解析
nslookup downloads.openwrt.org

# 查看倉庫配置
cat /etc/opkg/distfeeds.conf

# 修改倉庫為鏡像（如官方倉庫慢）
# 編輯 /etc/opkg/distfeeds.conf
# 或使用國內鏡像，例如：
uci set opkg.openwrt_core.src_default='https://mirrors.aliyun.com/openwrt/releases/23.05.0/targets/ar71xx/generic/packages'
```

### Q：包安裝失敗（Segmentation fault）
**原因：** 依賴衝突或硬件限制  
**解法：**
```bash
# 清理包快取
opkg clean
rm -rf /var/opkg-lists

# 重新更新
opkg update

# 強制重新安裝
opkg install --force-reinstall <package>

# 檢查磁盤空間
df -h
```

### Q：系統時間不同步
**原因：** NTP 服務未運行或伺服器不可達  
**解法：**
```bash
# 檢查 NTP 服務
/etc/init.d/ntpd status

# 手動啟動
/etc/init.d/ntpd start

# 查看 NTP 日誌
logread -e ntp

# 測試 NTP 伺服器
ntpdate -q 0.openwrt.pool.ntp.org

# 手動設置時間
date -s "2024-05-29 10:30:00"
```

## 參考來源

- [OpenWrt 包管理官方文件](https://openwrt.org/docs/guide/software/opkg)
- [opkg 命令詳解](https://openwrt.org/docs/guide/software/opkg/opkg)
- [OpenWrt 包倉庫](https://downloads.openwrt.org/)
- [系統設置 UCI 配置](https://openwrt.org/docs/uci/system)
- [NTP 官方文件](https://www.ntp.org/)

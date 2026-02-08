# Proxy Server Setup Skill

## Skill Metadata

```yaml
name: proxy-server-setup
description: 在 Linux 服务器上快速部署 mihomo (Clash.Meta) 代理，支持订阅管理、WebUI 配置和自动测试
version: 1.0.0
author: OpenCode AI
tags: [proxy, vpn, clash, mihomo, linux]
```

## Input Parameters

```json
{
  "subscription_url": {
    "type": "string",
    "description": "Clash 订阅链接 (必需)",
    "required": true,
    "example": "https://example.com/api/v1/client/subscribe?token=xxx"
  },
  "external_port": {
    "type": "integer",
    "description": "代理端口",
    "required": false,
    "default": 7890
  },
  "webui_port": {
    "type": "integer",
    "description": "WebUI 控制面板端口",
    "required": false,
    "default": 9090
  },
  "dns_port": {
    "type": "integer",
    "description": "DNS 服务端口",
    "required": false,
    "default": 1053
  },
  "webui_type": {
    "type": "string",
    "description": "WebUI 类型",
    "required": false,
    "default": "yacd",
    "enum": ["yacd", "clara", "metaui"]
  },
  "enable_auth": {
    "type": "boolean",
    "description": "是否开启 WebUI 认证",
    "required": false,
    "default": false
  },
  "auth_username": {
    "type": "string",
    "description": "认证用户名 (enable_auth=true 时必需)",
    "required": false
  },
  "auth_password": {
    "type": "string",
    "description": "认证密码 (enable_auth=true 时必需)",
    "required": false
  },
  "open_ports": {
    "type": "array",
    "description": "需要开放的防火墙端口",
    "required": false,
    "default": [7890]
  },
  "enable_tun": {
    "type": "boolean",
    "description": "是否启用 TUN 模式",
    "required": false,
    "default": true
  },
  "test_url": {
    "type": "string",
    "description": "代理测试 URL",
    "required": false,
    "default": "https://www.google.com"
  },
  "auto_test": {
    "type": "boolean",
    "description": "部署后是否自动测试代理",
    "required": false,
    "default": true
  }
}
```

## Installation Script

### 0. Pre-check (前置检查)

```bash
#!/bin/bash
set -euo pipefail

echo "=== 环境前置检查 ==="

# 检查是否为 root
if [ "$(id -u)" -ne 0 ]; then
    echo "❌ 请使用 root 用户运行此脚本"
    exit 1
fi

# 检查必需工具
for cmd in curl wget git gunzip python3 systemctl; do
    if ! command -v "$cmd" &>/dev/null; then
        echo "❌ 缺少依赖: $cmd，请先安装"
        exit 1
    fi
    echo "✅ $cmd 已安装"
done

# 检查系统架构
ARCH=$(uname -m)
if [[ "$ARCH" != "x86_64" && "$ARCH" != "aarch64" ]]; then
    echo "❌ 不支持的架构: $ARCH (仅支持 x86_64 / aarch64)"
    exit 1
fi
echo "✅ 系统架构: $ARCH"

# 检查端口是否被占用
check_port() {
    if ss -tlnp | grep -q ":$1 "; then
        echo "⚠️  端口 $1 已被占用: $(ss -tlnp | grep ":$1 " | awk '{print $NF}')"
        return 1
    fi
    echo "✅ 端口 $1 可用"
    return 0
}

check_port ${EXTERNAL_PORT:-7890}
check_port ${WEBUI_PORT:-9090}
check_port ${DNS_PORT:-1053}

echo "=== 前置检查完成 ==="
```

### 1. Download mihomo

```bash
set -euo pipefail

# Detect system architecture
ARCH=$(uname -m)
if [ "$ARCH" = "x86_64" ]; then
    ARCH="amd64"
elif [ "$ARCH" = "aarch64" ]; then
    ARCH="arm64"
else
    echo "❌ 不支持的架构: $ARCH"; exit 1
fi

# Get latest version (tag_name 已含 v 前缀，如 v1.19.0)
LATEST_VERSION=$(curl -sf https://api.github.com/repos/MetaCubeX/mihomo/releases/latest \
    | grep '"tag_name"' | cut -d '"' -f 4)

if [ -z "$LATEST_VERSION" ]; then
    echo "❌ 无法获取版本号，请检查网络连接"; exit 1
fi
echo "📦 最新版本: $LATEST_VERSION"

# Download (tag_name 已含 v 前缀，文件名无需额外添加)
DOWNLOAD_URL="https://github.com/MetaCubeX/mihomo/releases/download/${LATEST_VERSION}/mihomo-linux-${ARCH}-${LATEST_VERSION}.gz"
if ! wget -O /tmp/mihomo.gz "$DOWNLOAD_URL"; then
    echo "❌ 下载失败，请检查网络连接或手动下载"; exit 1
fi

gunzip -f /tmp/mihomo.gz
chmod +x /tmp/mihomo
mv /tmp/mihomo /usr/local/bin/mihomo
echo "✅ mihomo 安装完成: $(mihomo -v)"
```

### 2. Install WebUI & GeoData

```bash
set -euo pipefail

# Create directories
mkdir -p /etc/clash/{webui,providers}
mkdir -p /var/log/clash

# Download WebUI (default: yacd)
WEBUI_TYPE="${WEBUI_TYPE:-yacd}"
echo "📦 安装 WebUI: $WEBUI_TYPE"

# 清理旧版 WebUI
rm -rf /etc/clash/webui && mkdir -p /etc/clash/webui

case "$WEBUI_TYPE" in
    yacd)
        REPO_URL="https://github.com/MetaCubeX/Yacd-meta.git" ;;
    clara)
        REPO_URL="https://github.com/MetaCubeX/Clara.git" ;;
    metaui)
        REPO_URL="https://github.com/MetaCubeX/MetaUI.git" ;;
    *)
        echo "❌ 未知的 WebUI 类型: $WEBUI_TYPE"; exit 1 ;;
esac

if ! git clone --depth 1 --branch gh-pages "$REPO_URL" /etc/clash/webui; then
    echo "❌ WebUI 下载失败，请检查网络或 Git 安装"; exit 1
fi
echo "✅ WebUI 安装完成"

# Download GeoIP / GeoSite 规则数据库 (GEOIP 分流规则依赖此文件)
echo "📦 下载 GeoIP & GeoSite 数据库..."
GEO_BASE="https://github.com/MetaCubeX/meta-rules-dat/releases/latest/download"
wget -O /etc/clash/GeoIP.dat  "${GEO_BASE}/geoip.dat"   || echo "⚠️  GeoIP 下载失败，GEOIP 规则将不可用"
wget -O /etc/clash/GeoSite.dat "${GEO_BASE}/geosite.dat" || echo "⚠️  GeoSite 下载失败"
echo "✅ GeoData 下载完成"
```

### 3. Generate Config

> **注意**: 以下 `${VARIABLE}` 均为需替换的变量占位符，部署时请替换为实际值。

```yaml
# /etc/clash/config.yaml
mixed-port: ${EXTERNAL_PORT}          # 默认 7890
allow-lan: true
bind-address: "*"
mode: rule
log-level: info
# ⚠️ 安全提示: 生产环境建议改为 127.0.0.1，仅本机访问
#    如需远程管理，请配合 SSH 隧道或设置 secret
external-controller: 0.0.0.0:${WEBUI_PORT}
external-ui: /etc/clash/webui
${ENABLE_AUTH:+"secret: ${AUTH_PASSWORD}"}

profile:
  store-selected: true
  store-fake-ip: true

tun:
  enable: ${ENABLE_TUN}
  stack: gvisor
  dns-hijack:
    - any:53

dns:
  enable: true
  prefer-h3: true
  listen: 0.0.0.0:${DNS_PORT}
  default-nameserver:
    - 223.5.5.5
    - 119.29.29.29
  proxy-server-nameserver:
    - 'https://doh.pub/dns-query'
  nameserver:
    - 'https://doh.pub/dns-query'
  fallback:
    - 'https://doh.pub/dns-query'
    - 'https://dns.alidns.com/dns-query'
  fallback-filter:
    geoip: true
    geoip-code: CN
    ipcidr:
      - 240.0.0.0/4

proxy-providers:
  MySubscription:
    type: http
    url: "${SUBSCRIPTION_URL}"
    path: ./providers/mysub.yaml
    interval: 3600
    health-check:
      enable: true
      url: https://www.google.com/generate_204
      interval: 300

proxy-groups:
  - name: Domestic
    type: select
    proxies:
      - DIRECT

  - name: Others
    type: select
    proxies:
      - DIRECT
      - Proxy

  - name: Proxy
    type: select
    use:
      - MySubscription
    proxies:
      - DIRECT

rules:
  - DOMAIN-SUFFIX,local,DIRECT
  - DOMAIN-SUFFIX,lan,DIRECT
  - IP-CIDR,10.0.0.0/8,DIRECT
  - IP-CIDR,172.16.0.0/12,DIRECT
  - IP-CIDR,192.168.0.0/16,DIRECT
  - IP-CIDR,127.0.0.0/8,DIRECT
  - IP-CIDR,100.64.0.0/10,DIRECT
  - DOMAIN-SUFFIX,cn,Domestic
  - DOMAIN-SUFFIX,com.cn,Domestic
  - DOMAIN-SUFFIX,net.cn,Domestic
  - DOMAIN-SUFFIX,org.cn,Domestic
  - DOMAIN-SUFFIX,gov.cn,Domestic
  - DOMAIN-SUFFIX,edu.cn,Domestic
  - DOMAIN-SUFFIX,taobao.com,Domestic
  - DOMAIN-SUFFIX,baidu.com,Domestic
  - DOMAIN-SUFFIX,qq.com,Domestic
  - DOMAIN-SUFFIX,weixin.qq.com,Domestic
  - DOMAIN-SUFFIX,aliyun.com,Domestic
  - DOMAIN-SUFFIX,jingdong.com,Domestic
  - DOMAIN-SUFFIX,163.com,Domestic
  - GEOIP,CN,Domestic
  - MATCH,Others
```

### 4. Setup Systemd Service

```ini
# /etc/systemd/system/mihomo.service
[Unit]
Description=mihomo daemon
After=network.target NetworkManager.service systemd-resolved.service
Wants=network-online.target

[Service]
Type=simple
# ⚠️ TUN 模式需要 root；如不使用 TUN，建议创建专用用户:
#    useradd -r -s /sbin/nologin mihomo
#    并将 User 改为 mihomo，同时调整 /etc/clash 文件权限
User=root
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
ExecStart=/usr/local/bin/mihomo -d /etc/clash
ExecReload=/bin/kill -HUP $MAINPID
StandardOutput=append:/var/log/clash/access.log
StandardError=append:/var/log/clash/error.log

[Install]
WantedBy=multi-user.target
```

### 5. Configure Firewall

```bash
set -euo pipefail

EXTERNAL_PORT=${EXTERNAL_PORT:-7890}
WEBUI_PORT=${WEBUI_PORT:-9090}

# 自动检测防火墙类型
if command -v ufw &>/dev/null; then
    echo "📦 使用 UFW 配置防火墙..."
    ufw allow ${EXTERNAL_PORT}/tcp comment "mihomo proxy"
    ufw allow ${WEBUI_PORT}/tcp comment "mihomo webui"
    ufw reload
    echo "✅ UFW 规则已添加"
elif command -v firewall-cmd &>/dev/null; then
    echo "📦 使用 firewalld 配置防火墙..."
    firewall-cmd --permanent --add-port=${EXTERNAL_PORT}/tcp
    firewall-cmd --permanent --add-port=${WEBUI_PORT}/tcp
    firewall-cmd --reload
    echo "✅ firewalld 规则已添加"
else
    echo "⚠️  未检测到防火墙工具 (ufw/firewalld)，请手动开放端口: ${EXTERNAL_PORT}, ${WEBUI_PORT}"
fi

# 云服务器安全组需在厂商控制台额外配置 (阿里云/腾讯云/AWS 等)
```

### 6. Start Service

```bash
systemctl daemon-reload
systemctl enable mihomo
systemctl start mihomo
systemctl status mihomo
```

## Testing Script

```bash
#!/bin/bash
set -uo pipefail  # 不用 -e，测试脚本需要继续执行后续用例

TEST_URL="${TEST_URL:-https://www.google.com}"
PROXY_PORT="${EXTERNAL_PORT:-7890}"
API_PORT="${WEBUI_PORT:-9090}"
PASS=0
FAIL=0

echo "=== Testing Proxy Server ==="
echo "Test URL:  $TEST_URL"
echo "Proxy:     127.0.0.1:$PROXY_PORT"
echo "API:       127.0.0.1:$API_PORT"
echo ""

# 检查服务是否在运行
if ! systemctl is-active --quiet mihomo; then
    echo "❌ mihomo 服务未运行，请先启动: systemctl start mihomo"
    exit 1
fi

# Test 1: SOCKS5
echo "[1] Testing SOCKS5..."
RESULT1=$(curl --connect-timeout 10 --socks5 127.0.0.1:$PROXY_PORT "$TEST_URL" -o /dev/null -s -w "%{http_code}" 2>/dev/null)
if [ "$RESULT1" = "200" ]; then
    echo "✅ SOCKS5 OK (HTTP $RESULT1)"; ((PASS++))
else
    echo "❌ SOCKS5 Failed (HTTP ${RESULT1:-timeout})"; ((FAIL++))
fi

# Test 2: HTTP
echo "[2] Testing HTTP..."
RESULT2=$(curl --connect-timeout 10 -x http://127.0.0.1:$PROXY_PORT "$TEST_URL" -o /dev/null -s -w "%{http_code}" 2>/dev/null)
if [ "$RESULT2" = "200" ]; then
    echo "✅ HTTP OK (HTTP $RESULT2)"; ((PASS++))
else
    echo "❌ HTTP Failed (HTTP ${RESULT2:-timeout})"; ((FAIL++))
fi

# Test 3: API
echo "[3] Testing API..."
RESULT3=$(curl --connect-timeout 5 -s http://127.0.0.1:${API_PORT}/proxies 2>/dev/null)
if echo "$RESULT3" | grep -q "proxies"; then
    echo "✅ API OK"; ((PASS++))
else
    echo "❌ API Failed"; ((FAIL++))
fi

# Test 4: Nodes
echo "[4] Checking nodes..."
NODES=$(echo "$RESULT3" | python3 -c "import sys,json; print(len(json.load(sys.stdin)['proxies']))" 2>/dev/null)
if [ -n "$NODES" ] && [ "$NODES" -gt 0 ]; then
    echo "✅ Nodes loaded: $NODES"; ((PASS++))
else
    echo "❌ No nodes found"; ((FAIL++))
fi

echo ""
echo "=== Test Complete: ✅ $PASS passed, ❌ $FAIL failed ==="
[ "$FAIL" -eq 0 ] && exit 0 || exit 1
```

## Usage Examples

### Basic Usage
```bash
# 使用默认配置部署
deploy_proxy subscription_url="https://example.com/link"
```

### Full Customization
```bash
# 完整自定义配置
deploy_proxy \
  subscription_url="https://example.com/link" \
  external_port=7890 \
  webui_port=9090 \
  dns_port=1053 \
  webui_type="yacd" \
  enable_auth=true \
  auth_username="admin" \
  auth_password="your_password" \
  open_ports=[7890,9090] \
  enable_tun=true \
  test_url="https://www.google.com" \
  auto_test=true
```

### Minimal with Testing
```bash
# 最小配置 + 自动测试
deploy_proxy \
  subscription_url="https://example.com/link" \
  auto_test=true
```

## Output

```json
{
  "status": "success|partial|failed",
  "message": "部署结果描述",
  "service_status": "running|stopped|error",
  "proxy_port": 7890,
  "webui_port": 9090,
  "nodes_loaded": 38,
  "test_results": {
    "socks5": "200",
    "http": "200",
    "api": "success"
  },
  "webui_url": "http://your-server:9090",
  "next_steps": [
    "在浏览器中访问 WebUI",
    "选择节点并测试连接",
    "配置浏览器代理"
  ]
}
```

## Error Handling

| Error Code | Description | Solution |
|------------|-------------|----------|
| E01 | 下载 mihomo 失败 | 检查网络连接，尝试手动下载 |
| E02 | Git 克隆失败 | 检查 Git 安装，确认仓库地址 |
| E03 | 配置文件错误 | 检查 YAML 语法，验证订阅链接 |
| E04 | 端口被占用 | 使用其他端口或停止占用进程 |
| E05 | 服务启动失败 | 查看日志 /var/log/clash/error.log |
| E06 | 订阅下载失败 | 确认订阅链接是否有效 |
| E07 | 节点为空 | 检查订阅是否包含节点配置 |
| E08 | 代理测试失败 | 尝试更换节点 |

## Dependencies

- Ubuntu 20.04+ / Debian 10+ / CentOS 7+
- wget 或 curl
- unzip 或 gunzip
- git
- python3 (用于测试)
- systemctl (systemd)
- ufw (可选，用于防火墙)

## File Locations

| Path | Purpose |
|------|---------|
| `/usr/local/bin/mihomo` | mihomo 二进制文件 |
| `/etc/clash/config.yaml` | 主配置文件 |
| `/etc/clash/webui/` | WebUI 文件 |
| `/etc/clash/providers/` | 订阅配置文件 |
| `/etc/systemd/system/mihomo.service` | systemd 服务文件 |
| `/var/log/clash/access.log` | 访问日志 |
| `/var/log/clash/error.log` | 错误日志 |

## Maintenance Commands

```bash
# 查看状态
systemctl status mihomo

# 重启服务
systemctl restart mihomo

# 查看日志
journalctl -u mihomo -f

# 更新订阅
curl -X POST http://127.0.0.1:9090/providers/MySubscription/refresh

# 查看节点
curl http://127.0.0.1:9090/proxies

# 测试代理
./test_proxy.sh

# 备份配置 (卸载前建议先备份)
cp -r /etc/clash /etc/clash.bak.$(date +%Y%m%d)

# 卸载 (完整清理)
systemctl stop mihomo
systemctl disable mihomo
rm -f /usr/local/bin/mihomo
rm -rf /etc/clash
rm -f /etc/systemd/system/mihomo.service
systemctl daemon-reload
rm -rf /var/log/clash
# 如需清理防火墙规则 (UFW):
# ufw delete allow 7890/tcp && ufw delete allow 9090/tcp
echo "✅ mihomo 已完全卸载"
```

## Log Rotation (日志轮转)

配置 logrotate 防止日志文件无限增长：

```
# /etc/logrotate.d/mihomo
/var/log/clash/*.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    create 0640 root root
    postrotate
        systemctl reload mihomo 2>/dev/null || true
    endscript
}
```

## Notes

1. **TUN 模式**需要 root 权限；某些 VPS (如 OpenVZ) 不支持，部署前用 `systemd-detect-virt` 确认虚拟化类型
2. **安全加固**: 生产环境建议将 `external-controller` 绑定 `127.0.0.1`，通过 SSH 隧道远程管理
3. **WebUI 认证**: 强烈建议设置 `secret` 字段，避免 API 未授权访问
4. 定期更新订阅: `curl -X POST http://127.0.0.1:9090/providers/MySubscription/refresh`
5. 监控日志: `journalctl -u mihomo -f` 或查看 `/var/log/clash/error.log`
6. **备份**: 更新前备份配置 `cp -r /etc/clash /etc/clash.bak.$(date +%Y%m%d)`
7. **GeoIP 更新**: 定期更新 GeoIP/GeoSite 数据库以保证分流规则准确性

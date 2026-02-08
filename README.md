# 🚀 Proxy Server Setup

> 一键在 Linux 服务器上部署 [mihomo (Clash.Meta)](https://github.com/MetaCubeX/mihomo) 代理服务，支持订阅管理、WebUI 控制面板和自动化测试。

## ✨ 功能特性

- **一键部署** — 自动下载 mihomo、配置系统服务、开放防火墙端口
- **多 WebUI 支持** — Yacd / Clara / MetaUI 任选
- **订阅管理** — 自动拉取 Clash 订阅并定时更新
- **TUN 模式** — 支持全局透明代理
- **自动测试** — 部署后自动验证 SOCKS5 / HTTP / API 连通性
- **安全加固** — 支持 WebUI 认证、端口检测、服务隔离建议
- **完善的错误处理** — 前置环境检查、下载校验、端口冲突检测

## 📋 前置要求

| 依赖 | 说明 |
|------|------|
| **OS** | Ubuntu 20.04+ / Debian 10+ / CentOS 7+ |
| **权限** | root 用户 |
| curl / wget | 下载工具 |
| git | 克隆 WebUI |
| gunzip | 解压 mihomo |
| python3 | 测试脚本 |
| systemctl | 服务管理 |
| ufw / firewalld | 防火墙 (可选) |

## 🚀 快速开始

### 1. 克隆仓库

```bash
git clone https://github.com/<your-username>/proxy-server-setup.git
cd proxy-server-setup
```

### 2. 参考文档部署

完整的部署流程和脚本在 [proxy-server-setup.md](proxy-server-setup.md) 中，包含：

1. **环境前置检查** — 依赖工具、系统架构、端口可用性
2. **下载安装 mihomo** — 自动检测架构，获取最新版本
3. **安装 WebUI & GeoData** — WebUI 面板 + GeoIP/GeoSite 规则库
4. **生成配置文件** — 代理端口、DNS、分流规则、订阅源
5. **Systemd 服务** — 开机自启、故障自动重启
6. **防火墙配置** — 自动检测 ufw/firewalld
7. **代理测试** — SOCKS5 / HTTP / API / 节点检查

### 3. 最小化部署示例

```bash
# 设置变量
export SUBSCRIPTION_URL="https://your-provider.com/api/v1/client/subscribe?token=xxx"

# 按照 proxy-server-setup.md 中的步骤依次执行
# Step 0: 前置检查
# Step 1: 下载 mihomo
# Step 2: 安装 WebUI & GeoData
# Step 3: 生成配置 (替换变量)
# Step 4: 创建 Systemd 服务
# Step 5: 配置防火墙
# Step 6: 启动服务
systemctl start mihomo
```

## 🔧 常用配置

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `EXTERNAL_PORT` | 7890 | 代理端口 (HTTP/SOCKS5) |
| `WEBUI_PORT` | 9090 | WebUI 控制面板端口 |
| `DNS_PORT` | 1053 | DNS 服务端口 |
| `WEBUI_TYPE` | yacd | WebUI 类型 (yacd/clara/metaui) |
| `ENABLE_TUN` | true | 是否启用 TUN 模式 |
| `ENABLE_AUTH` | false | 是否启用 WebUI 认证 |

## 📁 文件结构

部署后服务器上的文件布局：

```
/usr/local/bin/mihomo          # mihomo 二进制文件
/etc/clash/
├── config.yaml                # 主配置文件
├── GeoIP.dat                  # GeoIP 规则库
├── GeoSite.dat                # GeoSite 规则库
├── providers/                 # 订阅配置
│   └── mysub.yaml
└── webui/                     # WebUI 面板
/etc/systemd/system/mihomo.service  # 系统服务
/var/log/clash/                # 日志目录
├── access.log
└── error.log
```

## 🛠️ 维护命令

```bash
# 查看服务状态
systemctl status mihomo

# 重启服务
systemctl restart mihomo

# 实时查看日志
journalctl -u mihomo -f

# 更新订阅
curl -X POST http://127.0.0.1:9090/providers/MySubscription/refresh

# 查看节点列表
curl http://127.0.0.1:9090/proxies
```

## ⚠️ 安全建议

1. 生产环境将 `external-controller` 绑定 `127.0.0.1`，通过 SSH 隧道远程管理
2. 务必设置 `secret` 字段保护 WebUI API
3. 非 TUN 模式下建议使用专用用户运行服务，而非 root
4. 定期更新 GeoIP/GeoSite 数据库确保分流规则准确

## 📄 License

[MIT](LICENSE)

## 🤝 Contributing

欢迎提交 Issue 和 Pull Request！

# OpenClaw + sing-box 从零到一部署指南

> 部署环境：Raspberry Pi (aarch64, Ubuntu, Linux 6.17.0-1003-raspi)
> 用户：claw@192.168.1.57
> 日期：2026-02-08 ~ 2026-02-09

---

## 目录

- [一、环境准备](#一环境准备)
  - [1.1 安装基础依赖](#11-安装基础依赖)
  - [1.2 安装 Node.js 22](#12-安装-nodejs-22)
- [二、安装 OpenClaw](#二安装-openclaw)
  - [2.1 npm 全局安装](#21-npm-全局安装)
  - [2.2 运行 Onboard 向导](#22-运行-onboard-向导)
- [三、安装与配置 sing-box 代理](#三安装与配置-sing-box-代理)
  - [3.1 安装 sing-box](#31-安装-sing-box)
  - [3.2 普通代理模式配置（mixed）](#32-普通代理模式配置mixed)
  - [3.3 创建 systemd 服务](#33-创建-systemd-服务)
  - [3.4 TUN 透明代理模式配置](#34-tun-透明代理模式配置)
- [四、OpenClaw 配置详解](#四openclaw-配置详解)
  - [4.1 配置文件位置](#41-配置文件位置)
  - [4.2 模型配置](#42-模型配置)
  - [4.3 Telegram 频道配置](#43-telegram-频道配置)
  - [4.4 完整配置示例](#44-完整配置示例)
- [五、踩坑记录与排错](#五踩坑记录与排错)
- [六、常用运维命令](#六常用运维命令)

---

## 一、环境准备

### 1.1 安装基础依赖

树莓派默认最小化安装，缺少 `curl` 和 `git`，需要先装：

```bash
sudo apt update
sudo apt install -y curl git
```

> **踩坑 #1**：直接运行 NodeSource 安装脚本会报 `curl: command not found`，必须先装 curl。

### 1.2 安装 Node.js 22

OpenClaw 要求 **Node ≥ 22**。通过 NodeSource 安装：

```bash
# 添加 NodeSource 仓库
curl -fsSL https://deb.nodesource.com/setup_22.x -o /tmp/nodesource_setup.sh
sudo bash /tmp/nodesource_setup.sh

# 安装 Node.js
sudo apt install -y nodejs

# 验证
node --version   # v22.22.0
npm --version    # 10.9.4
```

---

## 二、安装 OpenClaw

### 2.1 npm 全局安装

```bash
# 国内网络建议先切换镜像源
sudo npm config set registry https://registry.npmmirror.com

# 全局安装
sudo npm install -g openclaw@latest

# 验证
openclaw --version   # 2026.2.6-3
```

> **踩坑 #2**：首次安装时未安装 `git`，npm 报错 `ENOENT: spawn git`。安装 git 后重试即可。
>
> **踩坑 #3**：树莓派网络较慢，npm 从官方源下载超时 (`ETIMEDOUT`)。切换淘宝镜像 `registry.npmmirror.com` 后约 1 分钟安装完成。

### 2.2 运行 Onboard 向导

```bash
openclaw onboard --install-daemon
```

**这是一个交互式向导**，必须在交互式终端中执行（直连或正常 SSH），不能在非交互式环境中运行。

向导引导内容：
1. 安全警告确认
2. 模型提供商配置（API 密钥、端点）
3. 频道设置（Telegram Bot Token 等）
4. 守护进程安装（systemd user service）

向导完成后会自动：
- 生成配置文件 `~/.openclaw/openclaw.json`
- 创建 systemd 用户服务 `openclaw-gateway.service`
- 启动 Gateway 守护进程

---

## 三、安装与配置 sing-box 代理

### 3.1 安装 sing-box

从 GitHub Releases 下载预编译的 ARM64 二进制：

```bash
# 下载并安装
curl -Lo /tmp/sing-box.tar.gz https://github.com/SagerNet/sing-box/releases/download/v1.11.4/sing-box-1.11.4-linux-arm64.tar.gz
cd /tmp && tar -xzf sing-box.tar.gz
sudo mv sing-box-1.11.4-linux-arm64/sing-box /usr/local/bin/
sudo chmod +x /usr/local/bin/sing-box

# 创建配置目录
sudo mkdir -p /etc/sing-box
```

### 3.2 普通代理模式配置（mixed）

`/etc/sing-box/config.json`：

```json
{
  "log": { "level": "info" },
  "inbounds": [
    {
      "type": "mixed",
      "tag": "mixed-in",
      "listen": "127.0.0.1",
      "listen_port": 7890
    }
  ],
  "outbounds": [
    {
      "type": "shadowsocks",
      "tag": "proxy",
      "server": "103.181.165.140",
      "server_port": 33084,
      "method": "2022-blake3-aes-128-gcm",
      "password": "NjExNWRjZDE3MTNhNTVlMQ==:NzkxMzFhZGQtMTJiYi00NA=="
    },
    { "type": "direct", "tag": "direct" }
  ],
  "route": { "final": "proxy" }
}
```

**工作方式**：在 `127.0.0.1:7890` 开启一个 HTTP/SOCKS5 混合代理端口，应用需要**主动设置代理**才能使用。

**测试代理**：

```bash
# 通过代理访问 Telegram
curl -x http://127.0.0.1:7890 -s https://api.telegram.org/

# 如果返回 HTML 内容则代理正常
```

### 3.3 创建 systemd 服务

`/etc/systemd/system/sing-box.service`：

```ini
[Unit]
Description=sing-box service
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/sing-box run -c /etc/sing-box/config.json
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now sing-box

# 查看状态
sudo systemctl status sing-box
```

> **踩坑 #4**：创建 systemd 服务文件时，通过管道 `echo "xxx" | sudo tee` 写入的文件内容为空（0 字节）。原因是 `echo | sudo tee` 的管道顺序导致 tee 先读到 sudo 的密码而不是文件内容。
>
> **解决方案**：使用 `sudo bash -c "cat > file << END ... END"` 方式写入。
>
> **踩坑 #5**：服务被 masked（`Unit is masked`），执行 `sudo systemctl unmask sing-box` 后该命令删除了服务文件。需要重新创建服务文件后再 enable。

### 3.4 TUN 透明代理模式配置

TUN 模式创建虚拟网卡接管系统路由，**所有应用无需任何配置**即可自动走代理。

`/etc/sing-box/config.json`：

```json
{
  "log": { "level": "info" },
  "dns": {
    "servers": [
      { "tag": "proxy-dns", "address": "https://1.1.1.1/dns-query", "detour": "proxy" },
      { "tag": "direct-dns", "address": "223.5.5.5", "detour": "direct" }
    ],
    "rules": [
      { "outbound": "any", "server": "direct-dns" }
    ],
    "final": "proxy-dns"
  },
  "inbounds": [
    {
      "type": "tun",
      "tag": "tun-in",
      "interface_name": "tun0",
      "address": ["172.19.0.1/30"],
      "mtu": 9000,
      "auto_route": true,
      "strict_route": false,
      "stack": "system",
      "sniff": true
    },
    {
      "type": "mixed",
      "tag": "mixed-in",
      "listen": "127.0.0.1",
      "listen_port": 7890
    }
  ],
  "outbounds": [
    {
      "type": "shadowsocks",
      "tag": "proxy",
      "server": "103.181.165.140",
      "server_port": 33084,
      "method": "2022-blake3-aes-128-gcm",
      "password": "NjExNWRjZDE3MTNhNTVlMQ==:NzkxMzFhZGQtMTJiYi00NA=="
    },
    { "type": "direct", "tag": "direct" },
    { "type": "dns", "tag": "dns-out" }
  ],
  "route": {
    "rules": [
      { "protocol": "dns", "outbound": "dns-out" },
      { "ip_is_private": true, "outbound": "direct" }
    ],
    "final": "proxy",
    "auto_detect_interface": true
  }
}
```

#### TUN 模式关键配置说明

| 字段 | 值 | 说明 |
|------|-----|------|
| `type: "tun"` | - | 创建 TUN 虚拟网卡 |
| `interface_name` | `"tun0"` | 虚拟网卡名称 |
| `address` | `"172.19.0.1/30"` | 虚拟网卡 IP，选择不与局域网冲突的私有网段 |
| `auto_route` | `true` | **自动修改系统路由表**，将流量导向 TUN |
| `strict_route` | **`false`** ⚠️ | **必须为 false**，否则 SSH 等局域网连接会断 |
| `sniff` | `true` | 嗅探协议，识别 HTTP/TLS 等流量 |
| `stack` | `"system"` | 使用系统网络栈 |
| `auto_detect_interface` | `true` | 自动检测出站网卡，避免代理服务器流量回环 |
| `ip_is_private` → `direct` | - | 私有 IP（局域网）走直连 |
| `dns-out` | - | DNS 请求单独出站处理 |

#### TUN 模式流量走向

```
应用层 (OpenClaw / curl / 任何程序)
              │
              ▼
    系统路由表 (被 auto_route 修改)
              │
     ┌────────┴────────┐
     │                 │
  tun0 虚拟网卡    wlan0 (局域网直连)
  172.19.0.1/30    192.168.x.x
     │
     ▼
  sing-box 处理
     │
     ├─ ip_is_private → direct (直连)
     ├─ protocol: dns → dns-out
     └─ 其他 → proxy (Shadowsocks)
              │
              ▼
     wlan0 → 103.181.165.140:33084 → 互联网
```

> **踩坑 #6（严重）**：首次 TUN 配置使用了 `strict_route: true`，导致所有流量（包括 SSH）被强制劫持到 TUN，树莓派完全失联。
>
> **修复方式**：只能通过直连树莓派（显示器+键盘 / 串口）执行：
> ```bash
> sudo systemctl stop sing-box
> sudo ip link delete tun0 2>/dev/null
> sudo ip route flush table 100 2>/dev/null
> ```
>
> **结论**：远程操作时，**永远不要用 `strict_route: true`**。

> **踩坑 #7**：TUN 模式下代理节点连接超时 (`dial tcp 103.181.165.140:27779: i/o timeout`)，日本节点（端口 27779）不可用。切换到阿联酋节点（端口 33084）后恢复。
>
> **教训**：配置文件中最好使用多个 outbound 节点做 fallback。

#### 普通代理 vs TUN 模式对比

| 特性 | 普通代理 (mixed) | TUN 透明代理 |
|------|------------------|--------------|
| 应用需要配置 | ✅ 需要设置代理地址 | ❌ 不需要 |
| 能代理的应用 | 仅支持代理设置的应用 | **所有应用** |
| UDP 支持 | 有限 | ✅ 完整 |
| 影响层级 | 应用层 | 网络层 |
| SSH 风险 | 无 | 有（需正确配置） |
| 复杂度 | 低 | 较高 |

---

## 四、OpenClaw 配置详解

### 4.1 配置文件位置

```
~/.openclaw/
├── openclaw.json              # 主配置文件
├── openclaw.json.bak          # 配置备份
├── agents/                    # Agent 配置
│   └── main/sessions/         # 会话数据
├── canvas/                    # Canvas 资源
├── credentials/               # 凭据存储
│   ├── telegram-allowFrom.json   # Telegram 白名单
│   └── telegram-pairing.json     # 配对请求
├── devices/                   # 设备配置
├── identity/                  # 身份信息
└── workspace/                 # 工作空间
```

### 4.2 模型配置

使用第三方 API 提供商（anyrouter）的配置示例：

```json
{
  "models": {
    "providers": {
      "anyrouter": {
        "baseUrl": "https://anyrouter.top",
        "apiKey": "sk-free",
        "api": "anthropic-messages",
        "models": [
          {
            "id": "claude-opus-4-5-20251101",
            "name": "Claude Opus 4.5",
            "api": "anthropic-messages",
            "reasoning": true,
            "input": ["text"],
            "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
            "contextWindow": 200000,
            "maxTokens": 8192
          }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "anyrouter/claude-opus-4-5-20251101",
        "fallbacks": []
      }
    }
  }
}
```

> **踩坑 #8**：最初使用 Dashscope（阿里云通义千问）配置 `api: "anthropic-messages"` + `baseUrl: "https://dashscope.aliyuncs.com/apps/anthropic"`，但 Dashscope 实际上**不支持 Anthropic API 格式**（仅支持 OpenAI 兼容格式），导致所有 AI 请求 `fetch failed`。
>
> **踩坑 #9**：模型配置中 `"input": ["text", "images"]` 报错 `Invalid input`。OpenClaw 仅接受 `"text"` 作为 input 类型，`"images"` 不是合法值。

### 4.3 Telegram 频道配置

```json
{
  "channels": {
    "telegram": {
      "enabled": true,
      "dmPolicy": "pairing",
      "botToken": "YOUR_BOT_TOKEN",
      "groupPolicy": "allowlist",
      "streamMode": "partial",
      "proxy": "http://127.0.0.1:7890"
    }
  }
}
```

| 字段 | 说明 |
|------|------|
| `dmPolicy` | `"pairing"` = 未授权用户需配对码；`"open"` = 所有人可访问 |
| `botToken` | Telegram Bot 的 Token（从 @BotFather 获取） |
| `proxy` | 代理地址（普通代理模式需要；TUN 模式可不设置） |
| `streamMode` | `"partial"` = 流式输出 |

**白名单文件** `~/.openclaw/credentials/telegram-allowFrom.json`：

```json
{
  "version": 1,
  "allowFrom": ["YOUR_TELEGRAM_USER_ID"]
}
```

管理配对：
```bash
openclaw pairing list telegram          # 查看待配对
openclaw pairing approve telegram CODE  # 批准配对码
```

### 4.4 完整配置示例

`~/.openclaw/openclaw.json`：

```json
{
  "meta": {
    "lastTouchedVersion": "2026.2.6-3",
    "lastTouchedAt": "2026-02-08T05:11:18.500Z"
  },
  "models": {
    "providers": {
      "anyrouter": {
        "baseUrl": "https://anyrouter.top",
        "apiKey": "sk-free",
        "api": "anthropic-messages",
        "models": [
          {
            "id": "claude-opus-4-5-20251101",
            "name": "Claude Opus 4.5",
            "api": "anthropic-messages",
            "reasoning": true,
            "input": ["text"],
            "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
            "contextWindow": 200000,
            "maxTokens": 8192
          }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "anyrouter/claude-opus-4-5-20251101",
        "fallbacks": []
      },
      "workspace": "/home/claw/clawd",
      "compaction": { "mode": "safeguard" },
      "maxConcurrent": 4,
      "subagents": { "maxConcurrent": 8 }
    }
  },
  "channels": {
    "telegram": {
      "enabled": true,
      "dmPolicy": "pairing",
      "botToken": "YOUR_BOT_TOKEN",
      "groupPolicy": "allowlist",
      "streamMode": "partial",
      "proxy": "http://127.0.0.1:7890"
    }
  },
  "gateway": {
    "port": 18789,
    "mode": "local",
    "bind": "loopback",
    "auth": {
      "mode": "token",
      "token": "YOUR_GATEWAY_TOKEN"
    }
  },
  "plugins": {
    "entries": {
      "telegram": { "enabled": true }
    }
  }
}
```

---

## 五、踩坑记录与排错

### 完整踩坑清单

| # | 问题 | 原因 | 解决方案 |
|---|------|------|----------|
| 1 | NodeSource 安装脚本报 `curl not found` | 树莓派最小安装无 curl | `sudo apt install -y curl` |
| 2 | npm install 报 `ENOENT: spawn git` | 未安装 git | `sudo apt install -y git` |
| 3 | npm install 超时 `ETIMEDOUT` | 官方 npm 源网络不通 | 切换镜像：`npm config set registry https://registry.npmmirror.com` |
| 4 | systemd 服务文件写入为空 | `echo \| sudo tee` 管道顺序问题 | 改用 `sudo bash -c "cat > file << END"` |
| 5 | systemd 报 `Unit is masked` | unmask 操作删除了服务文件 | 重新创建服务文件后再 enable |
| 6 | **TUN strict_route 导致 SSH 断连** | 所有流量被劫持，包括 SSH | **设置 `strict_route: false`** |
| 7 | 代理节点连接超时 `i/o timeout` | 特定节点不可用 | 切换到其他可用节点 |
| 8 | AI API `fetch failed` | Dashscope 不支持 Anthropic API 格式 | 更换为支持 anthropic-messages 的提供商 |
| 9 | 配置报 `Invalid input` | `"images"` 不是合法 input 类型 | 改为 `"input": ["text"]` |
| 10 | Telegram handler 报 `EACCES: permission denied, mkdir '/home/ros'` | workspace 路径错误 | 修改 `workspace` 为当前用户目录 `/home/claw/clawd` |
| 11 | Telegram `sendChatAction failed` | Node.js 进程未使用代理发送请求 | 使用 TUN 模式替代环境变量方案 |

### 日志排查方法

```bash
# OpenClaw Gateway 日志
journalctl --user -u openclaw-gateway -f
journalctl --user -u openclaw-gateway --since "5 minutes ago" --no-pager

# 详细日志文件
tail -f /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log

# sing-box 日志
sudo journalctl -u sing-box -f

# 健康检查
openclaw doctor
openclaw doctor --fix
```

### 关键日志标识

| 日志内容 | 含义 |
|----------|------|
| `[telegram] starting provider (@bot_name)` | Telegram 连接成功 |
| `Telegram: ok (@bot_name)` (doctor) | Telegram 状态正常 |
| `embedded run done: ... aborted=false` | 消息处理完成 |
| `sendChatAction failed` | 发送 Telegram 消息失败（网络问题） |
| `fetch failed` | HTTP 请求失败（代理或 API 不通） |
| `EACCES: permission denied` | 文件/目录权限问题 |
| `dial tcp ... i/o timeout` | 代理节点连接超时 |

---

## 六、常用运维命令

### OpenClaw

```bash
# 服务管理
systemctl --user status openclaw-gateway
systemctl --user restart openclaw-gateway
systemctl --user stop openclaw-gateway

# 诊断
openclaw doctor
openclaw doctor --fix

# 配对管理
openclaw pairing list telegram
openclaw pairing approve telegram <code>

# 安全审计
openclaw security audit --deep

# 发送消息测试
openclaw message send --to <id> --message "test"

# 更新
sudo npm install -g openclaw@latest
```

### sing-box

```bash
# 服务管理
sudo systemctl status sing-box
sudo systemctl restart sing-box
sudo systemctl stop sing-box

# 验证配置
sing-box check -c /etc/sing-box/config.json

# 查看 TUN 网卡
ip addr show tun0

# 查看路由
ip route show table all | grep tun0

# 测试代理
curl -x http://127.0.0.1:7890 -s https://api.telegram.org/
# TUN 模式直连测试
curl -s https://api.telegram.org/
```

### 可用 Shadowsocks 节点

| 节点 | 端口 | 密码 |
|------|------|------|
| 🇦🇪 AE（当前使用） | 33084 | `NjExNWRjZDE3MTNhNTVlMQ==:NzkxMzFhZGQtMTJiYi00NA==` |
| 🇦🇺 AU | 15026 | `NjI3MTI3YWIyMzM0OTllNQ==:NzkxMzFhZGQtMTJiYi00NA==` |
| 🇯🇵 JP Cyclops | 27779 | `MzdlZjdlMjljZmI4MjAzMw==:NzkxMzFhZGQtMTJiYi00NA==` |
| 🇯🇵 JP Yoga | 36691 | `MmUzYWY1ZTgxYzYzNzQ3Ng==:NzkxMzFhZGQtMTJiYi00NA==` |
| 🇯🇵 JP HoYoLAB | 14405 | `YzFiOGUzMGFjMDViZTg0ZA==:NzkxMzFhZGQtMTJiYi00NA==` |

所有节点共用：`server: 103.181.165.140`，`cipher: 2022-blake3-aes-128-gcm`

### 紧急恢复

如果 TUN 模式导致网络异常，在树莓派本地执行：

```bash
sudo systemctl stop sing-box
sudo ip link delete tun0 2>/dev/null
sudo ip route flush table 100 2>/dev/null

# 恢复为普通代理模式
sudo tee /etc/sing-box/config.json << 'EOF'
{
  "log": { "level": "info" },
  "inbounds": [
    { "type": "mixed", "listen": "127.0.0.1", "listen_port": 7890 }
  ],
  "outbounds": [
    {
      "type": "shadowsocks",
      "server": "103.181.165.140",
      "server_port": 33084,
      "method": "2022-blake3-aes-128-gcm",
      "password": "NjExNWRjZDE3MTNhNTVlMQ==:NzkxMzFhZGQtMTJiYi00NA=="
    },
    { "type": "direct", "tag": "direct" }
  ],
  "route": { "final": "proxy" }
}
EOF

sudo systemctl start sing-box
```

如果仍无法恢复：
```bash
sudo reboot
```

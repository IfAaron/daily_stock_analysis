# Cloudflare Tunnel 远程访问 WebUI 指南

本文记录如何把本地运行的 WebUI 通过 Cloudflare Tunnel 暴露为 HTTPS 域名访问，适合“家里电脑运行服务，手机或外部浏览器远程访问”的个人自托管场景。

该方案不需要公网 IP，不需要路由器端口转发，也不需要把 `8000` 端口直接暴露到公网。

## 适用场景

- 家里或内网电脑可以长期运行 `daily_stock_analysis`。
- 远程设备只需要浏览器访问 WebUI，例如 iPhone Safari。
- 不方便安装 Tailscale / ZeroTier / VPN 客户端。
- 希望通过 HTTPS 域名访问，例如 `https://stock.example.com`。
- 希望保留 WebUI 内置管理员密码，并可额外使用 Cloudflare Access 做邮箱登录保护。

不适合以下场景：

- 需要家里电脑关机后仍可访问。此时应部署到云服务器或 NAS。
- 不希望购买或持有域名。此时可考虑国内内网穿透服务，但安全和稳定性取决于服务商。
- 打算直接开放路由器端口。该方式风险更高，不建议用于包含 API Key、分析历史和配置管理的 WebUI。

## 总体架构

```text
远程浏览器 / iPhone Safari
        |
        | HTTPS
        v
Cloudflare Access（可选但推荐）
        |
        v
Cloudflare Tunnel
        |
        | 出站连接，无需路由器端口转发
        v
家里电脑 cloudflared
        |
        v
http://127.0.0.1:8000
        |
        v
daily_stock_analysis WebUI
```

推荐最终访问地址使用子域名，而不是根域名：

```text
https://stock.example.com
```

根域名 `example.com` 可以留给首页、文档或其他用途。

## 准备工作

### 1. 准备域名

在任意域名注册商购买域名，例如腾讯云、阿里云、Namecheap、Porkbun 或 Cloudflare Registrar。

建议：

- 个人使用只买一个主域名即可，例如 `example.com`。
- 不需要同时购买 `.cn`、`.com.cn` 等防御性域名，除非计划做公开品牌。
- 买前确认续费价格，不要只看首年价格。
- 不要购买中文域名或过于冷门的后缀，避免兼容和续费问题。

如果通过国内注册商购买，通常需要先完成域名信息模板实名认证。实名认证审核通过后，才能正常管理域名和修改 DNS 服务器。

### 2. 注册 Cloudflare

打开 Cloudflare 控制台并注册账号：

```text
https://dash.cloudflare.com/
```

Cloudflare Free 计划即可满足 Tunnel 和基础 DNS 需求。

## 接入域名到 Cloudflare

### 1. 添加域名

在 Cloudflare 控制台选择：

```text
Connect a domain
```

输入主域名：

```text
example.com
```

不要输入子域名：

```text
stock.example.com
```

套餐选择：

```text
Free
```

如果 Cloudflare 扫描到 `Records we found: 0`，对新域名来说是正常的。此时不要手动添加 `A`、`AAAA`、`CNAME`、`MX` 或 `www` 记录，后续 Tunnel 会自动创建需要的子域名记录。

### 2. 修改 Nameserver

Cloudflare 会分配两个 Nameserver，例如：

```text
heather.ns.cloudflare.com
razvan.ns.cloudflare.com
```

回到域名注册商控制台，找到：

```text
域名注册 -> 我的域名 -> 域名管理 -> DNS 服务器 / Nameserver
```

删除注册商默认 DNS 服务器，例如腾讯云 DNSPod 可能显示：

```text
onion.dnspod.net
parameter.dnspod.net
```

替换成 Cloudflare 分配的两个 Nameserver。

注意：这里修改的是“DNS 服务器 / Nameserver”，不是添加普通 DNS 解析记录。

### 3. 等待 Cloudflare 激活

回到 Cloudflare，等待域名状态变成：

```text
Active
```

Cloudflare 页面可能先显示：

```text
Waiting for your registrar to propagate your new nameservers
```

这表示正在等待注册商的 Nameserver 变更传播。通常几分钟到数小时，极端情况下可能需要 24 小时。

域名 Active 后，再继续配置 Tunnel。

## 创建 Cloudflare Tunnel

### 1. 进入 Zero Trust

打开：

```text
https://one.dash.cloudflare.com/
```

首次进入可能需要选择 Zero Trust 计划，选择：

```text
Zero Trust Free
```

如果需要创建 team name，可使用任意易识别名称，例如：

```text
equantstation
home
personal-lab
```

### 2. 创建 Tunnel / Connector

Cloudflare Zero Trust 新版菜单可能不再直接显示 `Tunnels`，而是放在：

```text
Networks -> Connectors
```

点击：

```text
Create a tunnel
```

Tunnel 类型选择：

```text
Cloudflared
```

不要选择 `Mesh` 或需要 WARP 全局配置的方案。

Tunnel 名称建议：

```text
home-dsa
daily-stock-analysis
```

### 3. 安装 cloudflared

操作系统选择：

```text
Windows
```

Cloudflare 会生成一条安装命令，类似：

```powershell
cloudflared.exe service install <token>
```

在运行 WebUI 的家里电脑上，以 PowerShell 执行 Cloudflare 页面给出的命令。

注意：

- `<token>` 是敏感信息，不要提交到仓库，也不要发到聊天、Issue 或 PR。
- 执行成功后，Cloudflare 页面中的 Connector 应显示 `Connected`。
- `cloudflared` 会作为本机服务运行，家里电脑重启后通常会自动恢复连接。

## 配置 Public Hostname

在 Tunnel / Connector 详情页中添加 Public Hostname：

```text
Subdomain: stock
Domain: example.com
Path: 留空
```

Service 配置：

```text
Type: HTTP
URL: 127.0.0.1:8000
```

如果页面要求完整 URL，填写：

```text
http://127.0.0.1:8000
```

保存后，Cloudflare 会创建类似下面的访问入口：

```text
https://stock.example.com -> http://127.0.0.1:8000
```

此时不要在 DNS 页面手动重复创建 `stock` 记录，避免和 Tunnel 自动记录冲突。

## 配置本地项目

Cloudflare Tunnel 访问的是家里电脑本机的 `127.0.0.1:8000`，因此 WebUI 不需要监听公网或局域网地址。

项目根目录 `.env` 推荐配置：

```env
WEBUI_HOST=127.0.0.1
WEBUI_PORT=8000
ADMIN_AUTH_ENABLED=true
```

说明：

- `WEBUI_HOST=127.0.0.1`：只允许本机访问，由 `cloudflared` 在本机转发，减少暴露面。
- `WEBUI_PORT=8000`：保持默认端口。
- `ADMIN_AUTH_ENABLED=true`：开启 WebUI 内置管理员密码。即使 Cloudflare Access 已启用，也建议保留这一层。

启动 WebUI：

```powershell
cd D:\projects\github\daily_stock_analysis
uv run python main.py --serve-only
```

或直接使用虚拟环境 Python：

```powershell
.\.venv\Scripts\python.exe main.py --serve-only
```

首次开启 `ADMIN_AUTH_ENABLED=true` 后，访问 WebUI 会进入管理员密码设置流程。设置完成后，本地接口应返回类似：

```json
{
  "authEnabled": true,
  "loggedIn": false,
  "passwordSet": true,
  "passwordChangeable": true,
  "setupState": "enabled"
}
```

可用以下命令检查：

```powershell
Invoke-WebRequest -UseBasicParsing http://127.0.0.1:8000/api/v1/auth/status
```

## 验证访问

### 1. 本地验证

先在家里电脑打开：

```text
http://127.0.0.1:8000
```

确认 WebUI 能正常显示登录页或首页。

也可以检查端口监听：

```powershell
netstat -ano | findstr :8000
```

Cloudflare Tunnel 场景下，推荐看到：

```text
127.0.0.1:8000 LISTENING
```

### 2. Cloudflare 域名验证

在外部浏览器或 iPhone Safari 打开：

```text
https://stock.example.com
```

如果一切正常，应看到 WebUI 登录页。

如果打开后出现 Cloudflare 错误码，参考下方排查。

## 添加 Cloudflare Access 保护

Tunnel 打通后，建议再启用 Cloudflare Access，形成双层保护：

```text
Cloudflare Access 邮箱登录
        +
WebUI ADMIN_AUTH_ENABLED 管理员密码
```

在 Zero Trust 控制台中进入：

```text
Access -> Applications
```

创建应用：

```text
Add an application -> Self-hosted
```

应用配置示例：

```text
Application name: daily-stock-analysis
Application domain: stock.example.com
```

访问策略建议：

```text
Action: Allow
Include: Emails
Value: your-email@example.com
```

如果只给自己使用，只允许自己的邮箱即可。

保存后，再访问：

```text
https://stock.example.com
```

应先经过 Cloudflare Access 邮箱验证，再进入 WebUI 登录页。

## 安全建议

- 不要把 `.env`、API Key、Tushare Token、OpenAI Key、Tunnel token 提交到 Git。
- 不要把家庭路由器的 `8000` 端口直接转发到公网。
- 不要把 RDP `3389` 端口直接暴露到公网。
- Cloudflare Tunnel 场景下保持 `WEBUI_HOST=127.0.0.1`。
- 保持 `ADMIN_AUTH_ENABLED=true`。
- 如启用 Cloudflare Access，只允许自己的邮箱或可信身份访问。
- 如果未来配置反向代理并信任 `X-Forwarded-For`，再评估是否开启 `TRUST_X_FORWARDED_FOR=true`。普通 Cloudflare Tunnel 场景下不需要默认开启。

## 常见问题

### Cloudflare 显示 `Records we found: 0`

新域名没有现有解析记录时，这是正常的。继续流程即可，不需要手动添加记录。

### Cloudflare 一直显示等待 Nameserver 更新

确认注册商处 Nameserver 已替换为 Cloudflare 提供的两个值，并删除旧 Nameserver。

如果刚保存，等待一段时间再点：

```text
Check nameservers
```

### Connector 未显示 Connected

检查家里电脑上 `cloudflared` 是否安装成功，网络是否能访问 Cloudflare。

Windows 可在“服务”中查看 `cloudflared` 服务状态，也可以重新执行 Cloudflare 页面提供的安装命令。

### 打开域名出现 502 / 1033

通常表示 Tunnel 已到 Cloudflare，但无法连接本机 WebUI。

检查：

```powershell
Invoke-WebRequest -UseBasicParsing http://127.0.0.1:8000
netstat -ano | findstr :8000
```

确认项目正在运行，并且 Public Hostname 的 Service URL 是：

```text
http://127.0.0.1:8000
```

### 域名能打开，但不是 WebUI

检查 Cloudflare Tunnel 的 Public Hostname 是否配置到了正确子域名，例如：

```text
stock.example.com
```

同时确认没有在 DNS 页面手动创建冲突的 `stock` 记录。

### 本地可以打开，远程打不开

按顺序确认：

1. Cloudflare 域名状态是 `Active`。
2. Connector 状态是 `Connected`。
3. Public Hostname 指向 `http://127.0.0.1:8000`。
4. 本地 `http://127.0.0.1:8000` 可访问。
5. WebUI 日志没有启动失败或端口占用错误。

### LLM 请求失败，与 Cloudflare Tunnel 有关吗

通常无关。Tunnel 只负责访问 WebUI。

如果分析时报：

```text
All LLM models failed
Empty or invalid response from LLM endpoint
```

优先检查 `.env` 中的模型网关配置，例如 OpenAI-compatible 网关是否需要 `/v1`：

```env
OPENAI_BASE_URL=https://api.example.com/v1
```

修改 `.env` 后需要重启 WebUI。

## 回滚方式

如需暂时关闭公网访问：

1. 在 Cloudflare Zero Trust 中禁用或删除 Public Hostname。
2. 停止家里电脑上的 `cloudflared` 服务。
3. 保留本地 WebUI，仅通过 `http://127.0.0.1:8000` 访问。

如需完全撤销 Cloudflare 接入：

1. 删除 Tunnel / Connector。
2. 删除 Access 应用。
3. 在域名注册商处把 Nameserver 改回原 DNS 服务商。

本地项目 `.env` 可保持：

```env
WEBUI_HOST=127.0.0.1
WEBUI_PORT=8000
ADMIN_AUTH_ENABLED=true
```

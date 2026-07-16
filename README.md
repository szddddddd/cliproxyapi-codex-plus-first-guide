# CLIProxyAPI 配置 Codex CLI：个人 ChatGPT Plus 优先，多中转自动回退

这是一份经过实际部署验证的配置记录，目标是让 Codex CLI 统一连接本机
[CLIProxyAPI](https://github.com/router-for-me/CLIProxyAPI)，并按以下顺序选择凭据：

1. 个人 ChatGPT Plus OAuth
2. `api.opentech.top` Codex Plus Pool
3. `api.opentech.top` Codex Pro Pool
4. `https://sorryios.ai/codex` 低优先级备用线路

本文以 Linux、root 用户和 CLIProxyAPI `v7.2.77` 为例。路径可按实际用户调整。
公开模板将本地客户端密钥与上游中转站密钥分离；这比直接复用已有中转密钥更安全，且不改变路由顺序。

> [!IMPORTANT]
> CLIProxyAPI 不能按“Plus 剩余 5%”主动切换。它按凭据优先级和运行状态选择线路；只有当前凭据出现额度耗尽、429、上游失败或进入冷却状态时，才会尝试下一条凭据。

> [!WARNING]
> 仅添加你本人拥有或获准使用的 ChatGPT 账号和中转服务。不要公开设备代码、OAuth JSON、API Key 或管理密钥。

## 架构与实际行为

```mermaid
flowchart LR
    A["Codex CLI"] --> B["CLIProxyAPI\n127.0.0.1:8317"]
    B --> C["个人 Plus OAuth\npriority 0"]
    C -. "失败、限流或冷却" .-> D["OpenTech Plus\npriority -10"]
    D -. "失败、限流或冷却" .-> E["OpenTech Pro\npriority -20"]
    E -. "失败、限流或冷却" .-> F["sorryios\npriority -30"]
```

关键行为：

- `routing.strategy: fill-first` 优先使用当前可用的最高优先级凭据。
- 在已验证版本中，没有显式设置优先级的 Codex OAuth 凭据默认优先级为 `0`。
- 将中转站优先级设为负数，即可让个人 OAuth 排在中转站之前。
- `request-retry` 和 `max-retry-credentials` 控制失败后的重试与候选凭据数量。
- 高优先级凭据从冷却状态恢复后，会重新具备被选择的资格。
- 这不是按余额百分比调度，也不是精确的成本优化器。

## 1. 安装官方 CLIProxyAPI

下面以 `v7.2.77`、Linux amd64 为例。其他架构请从
[Releases](https://github.com/router-for-me/CLIProxyAPI/releases) 选择对应文件。

```bash
VERSION=7.2.77
cd /tmp

curl -fLO "https://github.com/router-for-me/CLIProxyAPI/releases/download/v${VERSION}/CLIProxyAPI_${VERSION}_linux_amd64.tar.gz"
curl -fLO "https://github.com/router-for-me/CLIProxyAPI/releases/download/v${VERSION}/checksums.txt"

sha256sum -c checksums.txt --ignore-missing
tar -xzf "CLIProxyAPI_${VERSION}_linux_amd64.tar.gz"
install -m 755 cli-proxy-api /usr/local/bin/cliproxyapi
```

创建运行目录：

```bash
install -d -m 700 /root/.config/cliproxyapi
install -d -m 700 /root/.local/share/cliproxyapi/auths
install -d -m 700 /root/.local/bin
```

确认版本。CLIProxyAPI 会在启动日志开头输出版本、commit 和构建时间：

```bash
/usr/local/bin/cliproxyapi -help 2>&1 | head
```

## 2. 准备本地客户端密钥和管理密钥

建议将“Codex CLI 访问本地 CLIProxyAPI 的密钥”和“上游中转站 API Key”分开，不要复用。

```bash
umask 077
printf 'local-%s\n' "$(openssl rand -hex 32)" > /root/.config/cliproxyapi/client.key
openssl rand -hex 32 > /root/.config/cliproxyapi/management.key
chmod 600 /root/.config/cliproxyapi/client.key
chmod 600 /root/.config/cliproxyapi/management.key
```

后续配置中的以下占位符需要在部署机器上替换：

| 占位符 | 含义 |
|---|---|
| `<LOCAL_CLIENT_KEY>` | `/root/.config/cliproxyapi/client.key` 的内容 |
| `<MANAGEMENT_KEY>` | `/root/.config/cliproxyapi/management.key` 的内容 |
| `<OPENTECH_PLUS_KEY>` | OpenTech Codex Plus Pool API Key |
| `<OPENTECH_PRO_KEY>` | OpenTech Codex Pro Pool API Key |
| `<SORRYIOS_KEY>` | sorryios 备用 API Key，可选 |
| `<OUTBOUND_PROXY_URL>` | 集群访问外网所需的 HTTP/SOCKS 代理；不需要时填空字符串 |

不要把替换后的真实运行配置提交到 Git。

## 3. 创建 CLIProxyAPI 配置

创建 `/root/.config/cliproxyapi/config.yaml`：

```yaml
host: "127.0.0.1"
port: 8317

tls:
  enable: false

remote-management:
  allow-remote: false
  secret-key: "<MANAGEMENT_KEY>"
  disable-control-panel: false

auth-dir: "/root/.local/share/cliproxyapi/auths"

# Codex CLI 访问本地代理时使用的密钥，不是上游中转站密钥。
api-keys:
  - "<LOCAL_CLIENT_KEY>"

debug: false
logging-to-file: false
usage-statistics-enabled: true

# 不需要出站代理时改成空字符串：proxy-url: ""
proxy-url: "<OUTBOUND_PROXY_URL>"

request-retry: 3
max-retry-credentials: 4
max-retry-interval: 30
disable-cooling: false
save-cooldown-status: true

routing:
  strategy: "fill-first"
  session-affinity: false

codex-api-key:
  - api-key: "<OPENTECH_PLUS_KEY>"
    priority: -10
    base-url: "https://api.opentech.top"
    websockets: false

  - api-key: "<OPENTECH_PRO_KEY>"
    priority: -20
    base-url: "https://api.opentech.top"
    websockets: false

  # 可选的最低优先级备用线路。
  # 不使用 sorryios 时删除整个条目，不要保留空 key。
  - api-key: "<SORRYIOS_KEY>"
    priority: -30
    base-url: "https://sorryios.ai/codex"
    websockets: false
```

保护配置文件：

```bash
chmod 600 /root/.config/cliproxyapi/config.yaml
```

第一次正常启动时，CLIProxyAPI 会把 `remote-management.secret-key` 的明文值替换为 bcrypt 哈希。
因此必须另外保留权限为 `600` 的 `management.key`，供管理面板登录使用。

如果不需要管理面板，可以使用更小的攻击面：

```yaml
remote-management:
  allow-remote: false
  secret-key: ""
  disable-control-panel: true
```

但这样也会关闭 `/v0/management` 管理接口。

## 4. 登录个人 ChatGPT Plus

先在 [ChatGPT 安全设置](https://chatgpt.com/#settings/Security) 中启用 Codex 设备代码授权，然后运行：

```bash
/usr/local/bin/cliproxyapi \
  -config /root/.config/cliproxyapi/config.yaml \
  -codex-device-login
```

按照终端显示的设备代码完成授权。OAuth 凭据会写入：

```text
/root/.local/share/cliproxyapi/auths/
```

授权文件应保持 `600` 权限，不要复制到仓库、聊天记录或工单中。

如果网页提示“无法授权此设备”：

1. 确认设备代码授权已经在 ChatGPT 安全设置中启用。
2. 重新运行 `-codex-device-login`，生成新的设备代码。
3. 只输入当前终端显示的代码；旧代码和过期代码不能继续使用。
4. 不要分享设备代码。

启用安全设置本身不会让已经失败或过期的设备代码恢复有效，因此通常需要重新运行登录命令。

### 多个 ChatGPT 账号

每个账号分别执行一次设备代码登录，并确认认证目录中生成了独立 JSON 文件。多个 OAuth 凭据具有相同优先级时，具体选择还会受到调度策略和凭据状态影响。

## 5. 启动 CLIProxyAPI

前台验证：

```bash
/usr/local/bin/cliproxyapi \
  -config /root/.config/cliproxyapi/config.yaml
```

日志应显示：

- 服务监听 `127.0.0.1:8317`
- OAuth 和三条 Codex API Key 凭据均已载入
- Management API 已注册（仅在配置管理密钥时）

### systemd 环境

在 systemd 是 PID 1 的常规 Linux 机器上，可创建 `/etc/systemd/system/cliproxyapi.service`：

```ini
[Unit]
Description=CLIProxyAPI Plus-first Codex gateway
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=root
Group=root
WorkingDirectory=/root/.config/cliproxyapi
ExecStart=/usr/local/bin/cliproxyapi -config /root/.config/cliproxyapi/config.yaml
Restart=on-failure
RestartSec=3
UMask=0077

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable --now cliproxyapi
systemctl status cliproxyapi
```

### PID 1 不是 systemd 的集群或容器

如果 `ps -p 1 -o comm=` 显示 `sshd`、`bash` 或容器入口程序，systemd unit 不会真正接管进程。
可以先用 `nohup`：

```bash
nohup /usr/local/bin/cliproxyapi \
  -config /root/.config/cliproxyapi/config.yaml \
  > /root/.local/share/cliproxyapi/cliproxyapi.log 2>&1 </dev/null &

echo $! > /root/.local/share/cliproxyapi/cliproxyapi.pid
chmod 600 /root/.local/share/cliproxyapi/cliproxyapi.pid
```

也可以在 SSH shell 启动脚本中加入“进程不存在才启动”的检查：

```bash
if command -v cliproxyapi >/dev/null 2>&1 && ! pgrep -x cliproxyapi >/dev/null 2>&1; then
    env -u OPENAI_BASE_URL -u OPENAI_API_KEY -u CODEX_API_KEY \
        nohup /usr/local/bin/cliproxyapi \
        -config /root/.config/cliproxyapi/config.yaml \
        > /root/.local/share/cliproxyapi/cliproxyapi.log 2>&1 </dev/null &
    echo $! > /root/.local/share/cliproxyapi/cliproxyapi.pid
    chmod 600 /root/.local/share/cliproxyapi/cliproxyapi.pid
fi
```

这只是登录时自动补启动，不是完整的进程监督器。如果代理在没有新 SSH 登录时退出，它不会自动重启。

## 6. 让 Codex CLI 使用本地账号池

创建只读取本地客户端密钥的辅助命令 `/root/.local/bin/codex-cliproxy-token`：

```sh
#!/bin/sh
set -eu
exec cat /root/.config/cliproxyapi/client.key
```

```bash
chmod 700 /root/.local/bin/codex-cliproxy-token
```

在现有 `/root/.codex/config.toml` 中保留原来的模型、项目和功能配置，只添加本地 provider，并将它设为默认：

```toml
model_provider = "PlusFirst"

[model_providers.PlusFirst]
name = "Plus-first CLIProxyAPI"
base_url = "http://127.0.0.1:8317/v1"
wire_api = "responses"
supports_websockets = false

[model_providers.PlusFirst.auth]
command = "/root/.local/bin/codex-cliproxy-token"
timeout_ms = 5000
refresh_interval_ms = 0
```

修改前先备份：

```bash
cp -a /root/.codex/config.toml \
  "/root/.codex/config.toml.pre-cliproxy-$(date +%Y%m%d-%H%M%S)"
```

不要覆盖与代理无关的字段，例如当前模型、reasoning effort、项目 trust level 或其他功能开关。

### 保留原中转站手动回退

如果原 Codex 配置中已经存在直接访问 OpenTech 的 provider，请保留该 provider。需要绕过本地 CLIProxyAPI 时，临时把：

```toml
model_provider = "PlusFirst"
```

改回原 provider 名称即可。自动账号池回退发生在 CLIProxyAPI 内部；直接 provider 只用于 CLIProxyAPI 本身停止时的人工应急。

## 7. 验证主链路

先验证进程和模型列表：

```bash
pgrep -af '^/usr/local/bin/cliproxyapi '

LOCAL_KEY=$(cat /root/.config/cliproxyapi/client.key)
curl --http1.1 --noproxy '*' -fsS \
  -H "Authorization: Bearer ${LOCAL_KEY}" \
  http://127.0.0.1:8317/v1/models
```

查看日志：

```bash
tail -f /root/.local/share/cliproxyapi/cliproxyapi.log
```

推荐分别验证四种凭据，但不要把 API Key 写进命令历史或测试输出。验证内容至少包括：

- 个人 OAuth 能完成一次最小 Codex 请求
- OpenTech Plus Pool 能独立返回成功
- OpenTech Pro Pool 能独立返回成功
- sorryios 可用时能独立返回成功
- Plus Pool 故障或冷却时，请求能继续尝试 Pro Pool

`GET /v1/models` 只能证明本地服务和认证正常，不能完整证明每条上游生成链路都可用。

## 8. 查看 Plus 配额和中转使用情况

### 官方管理面板

CLIProxyAPI 的官方管理面板由
[Cli-Proxy-API-Management-Center](https://github.com/router-for-me/Cli-Proxy-API-Management-Center)
提供，当前配置中的地址是：

```text
http://127.0.0.1:8317/management.html
```

服务只监听服务器回环地址。需要从自己的电脑建立 SSH 隧道：

```bash
ssh -p <SSH_PORT> -N \
  -L 8317:127.0.0.1:8317 \
  <USER>@<CLUSTER_HOST>
```

然后在本地浏览器打开：

```text
http://127.0.0.1:8317/management.html
```

输入 `/root/.config/cliproxyapi/management.key` 中保存的管理密钥。不要将 `8317` 直接映射到公网。

管理面板不只是只读仪表盘，它也能修改配置和认证文件。只向可信用户提供管理密钥，使用完关闭 SSH 隧道。

### Management API

有管理密钥时，可查询：

```text
GET /v0/management/auth-files
GET /v0/management/api-key-usage
POST /v0/management/api-call
```

其中：

- `auth-files` 提供 OAuth 凭据状态、成功/失败次数、冷却状态和最近请求桶。
- `api-key-usage` 提供各 API Key 凭据的本机成功/失败次数和最近请求桶。
- `api-call` 可让管理面板代表某个 OAuth 凭据查询上游配额。

这些原始接口可能返回账号标识或包含 API Key 的组合键。不要直接把原始 JSON 粘贴到公开位置，展示前必须过滤和脱敏。

### 统计边界

- ChatGPT Plus 配额通常以 5 小时、7 天等窗口百分比表示，但上游可能只返回其中一个窗口。
- 如果 `secondary_window` 为 `null`，应显示“上游未返回”，不要自行估算。
- 配额查询不是稳定的公开 OpenAI API，字段和行为可能变化。
- `api-key-usage` 是 CLIProxyAPI 本机路由计数，不是 OpenTech 或 sorryios 的余额、账单或套餐剩余额度。
- 本机计数保存在内存中，CLIProxyAPI 重启后会归零。
- 当前官方 README 说明内置面板不提供持久化用量数据库。需要长期 token、费用和模型维度分析时，应部署独立统计服务，例如官方 README 提到的 CPA Usage Keeper 或 CPA-Manager-Plus。

## 9. 安全检查

公开或提交任何文档前，至少检查：

```bash
rg -n --hidden \
  'Bearer |api[_-]?key|access_token|refresh_token|management.key|client.key' \
  .
```

安全基线：

- CLIProxyAPI 绑定 `127.0.0.1`，不绑定 `0.0.0.0`。
- `remote-management.allow-remote` 保持 `false`。
- 使用 SSH 隧道访问面板，不公开代理端口。
- OAuth JSON、运行配置和密钥文件权限为 `600`。
- 认证目录和配置目录权限为 `700`。
- 上游 API Key、OAuth token、设备代码和管理密钥不进入 Git。
- 如果 API Key 曾出现在聊天记录、终端录屏或公开日志中，立即轮换。
- 不在进程命令行参数中放置 OAuth access token。

## 10. 常见问题

### 为什么个人 Plus 还没到剩余 5% 就切换了？

CLIProxyAPI 不理解“剩余 5%”这个业务阈值。只要个人 OAuth 被判定为不可用、限流、模型不可用或进入冷却，它就可能立即回退到中转站。

### 为什么剩余已经低于 5% 仍然没有切换？

因为调度器不读取该百分比作为选择条件。只要个人 OAuth 仍被认为可用，它仍保持最高优先级。

### 为什么看不到 5 小时窗口？

上游配额响应可能只提供 7 天窗口，另一个窗口可能为 `null`。这不代表监控脚本计算错误。

### 为什么中转站统计重启后变成 0？

原生成功/失败与最近请求统计是内存状态，不是持久化账本。

### OpenTech Plus Pool 不稳定怎么办？

保持 Plus Pool 为 `-10`、Pro Pool 为 `-20`，并让冷却和跨凭据重试生效。先直接验证 Pro Pool 的 key 和 base URL 可用，再判断是 Plus Pool 波动、集群出站代理问题还是 CLIProxyAPI 配置问题。

### systemctl 为什么无法启动服务？

先检查：

```bash
ps -p 1 -o pid,comm,args
```

PID 1 不是 systemd 时，unit 文件即使存在也不会提供真正的服务监督。应使用容器编排器、supervisord、s6、集群提供的进程管理器，或临时使用 `nohup`。

## 11. 回滚

停止 CLIProxyAPI：

```bash
pkill -TERM -x cliproxyapi
```

恢复 Codex 配置备份：

```bash
cp -a /root/.codex/config.toml.pre-cliproxy-<TIMESTAMP> \
  /root/.codex/config.toml
```

或者仅把 `model_provider` 改回原来的直接中转 provider。CLIProxyAPI 配置和 OAuth 文件可以暂时保留，不会在进程停止后继续接管 Codex 请求。

## 参考

- [router-for-me/CLIProxyAPI](https://github.com/router-for-me/CLIProxyAPI)
- [CLIProxyAPI Management API](https://help.router-for.me/cn/management/api)
- [Cli-Proxy-API-Management-Center](https://github.com/router-for-me/Cli-Proxy-API-Management-Center)
- [OpenAI Codex](https://github.com/openai/codex)

## 验证范围

本方法在以下组合中完成过实际验证：

- CLIProxyAPI `v7.2.77`
- Codex CLI 使用 Responses wire API
- 一个个人 ChatGPT Plus OAuth 凭据
- OpenTech Codex Plus Pool 与 Pro Pool
- 一个低优先级 sorryios 备用入口
- SSH 集群，PID 1 为 `sshd`，通过 `nohup` 维持代理进程
- 本地管理面板仅通过 SSH tunnel 访问

不同版本的 CLIProxyAPI、Codex CLI 和上游中转服务可能改变字段或行为。升级前应先备份配置，并在独立终端验证个人 OAuth、每条中转线路和回退顺序。

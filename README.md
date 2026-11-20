# Claude / Anthropic Mihomo Rule Set

> 适用于 **Mihomo (Clash Meta)** / **OpenClash** 的 GitHub 远程规则集（Rule Provider）。
>
> 更新日期：**2026-11-20**。本项目是第三方规则汇编，**非 Anthropic 官方维护**。

## 1. 仓库用途与重要说明

本仓库整理 **Anthropic 官方域名、Claude 相关第三方端点，以及官方公开的入站 IP 网段**。分流规则有不同证据等级：

- **A · 官方网络要求**：Anthropic 文档直接列出的基础域名；优先使用。
- **B · 官方条件性依赖**：仅在某些 Claude Code/桌面功能需要，或为可选遥测。
- **C · 网页 CSP 或客户端审计**：说明网页允许加载、或某版本应用调用过，**不等于所有客户端必需**。
- **D · 通用第三方域名**：Google、GitHub、npm、CDN 等也被大量其他应用使用，不应无条件加入全设备 Claude 代理。

特别注意：**IP 归属、服务商域名真实性和 Claude 专用性是三件不同的事**。配置远程规则集不意味着能够控制账号风控或绕过官方服务限制。

## 2. 仓库文件

```text
claude-mihomo-rules/
├── README.md
├── Claude-Core.list        # A: 仅 6 条 Anthropic 官方域名，首选
├── Claude-ThirdParty.list  # B/C: 精确第三方端点 + clau.de
├── Claude-IP.list          # A: 官方入站 IP，behavior: ipcidr
├── Claude-Shared.list      # B/D: 跨应用共享依赖，谨慎选择
├── Claude.list             # 合并：Core + ThirdParty + 官方入站 IP
└── Claude-Full.list        # 最大集合：Claude.list + Shared（不推荐默认启用）
```

`.list` 采用 Mihomo 规则集文本格式。`Claude-IP.list` **只有裸 CIDR**，需配 `behavior: ipcidr`；其他文件采用 `behavior: classical`。

### 选哪个文件？

| 方案 | 远程文件 | 适用场景 |
|---|---|---|
| **推荐：最小官方集** | `Claude-Core.list` | 仅分流 Anthropic 官方域名，不影响其他第三方服务 |
| **推荐：扩展集** | `Claude-Core.list` + `Claude-ThirdParty.list` | 希望同时覆盖客服/支付/可选遥测等精确第三方主机 |
| **按需增加 IP** | 再加 `Claude-IP.list` | 需要匹配直接访问目标 IP 的连接 |
| **方便的一体化文件** | `Claude.list` | 官方 + 第三方精确主机 + 入站 IP 一份文件搞定 |
| **仅实验使用** | `Claude-Full.list` | 附加 GitHub/npm/Google 等共享服务，可能改变其他应用路由 |

不要同时加载 `Claude.list` 与其组成的三个分文件，否则规则重复、徒增维护复杂度。

## 3. 快速部署：直接从 GitHub Raw 加载

1. 在 GitHub 创建一个公开仓库，如 `claude-mihomo-rules`。
2. 上传本包所有文件，文件名大小写保持一致，放在仓库根目录。
3. 将示例中的 `YOUR_USERNAME` 和 `YOUR_REPO` 换成自己的 GitHub 用户名和仓库名。
4. 确保 `Claude` 是你**已经存在**的策略组；如果不是，修改 `RULE-SET` 最后一段为现有策略组名称。
5. 将以下条目合并进 Mihomo 主配置的 `rule-providers:`、`rules:`。**不要把它们写进 `.list` 文件。**

### 方案 A（推荐）：只加载官方域名

```yaml
rule-providers:
  Claude-Core:
    type: http
    behavior: classical
    format: text
    url: "https://raw.githubusercontent.com/YOUR_USERNAME/YOUR_REPO/main/Claude-Core.list"
    path: ./ruleset/Claude-Core.list
    interval: 86400

rules:
  - RULE-SET,Claude-Core,Claude
  # 你的其他现有规则继续放在下面
```

### 方案 B：官方 + 精确第三方 + 官方入站 IP（模块化）

```yaml
rule-providers:
  Claude-Core:
    type: http
    behavior: classical
    format: text
    url: "https://raw.githubusercontent.com/YOUR_USERNAME/YOUR_REPO/main/Claude-Core.list"
    path: ./ruleset/Claude-Core.list
    interval: 86400

  Claude-ThirdParty:
    type: http
    behavior: classical
    format: text
    url: "https://raw.githubusercontent.com/YOUR_USERNAME/YOUR_REPO/main/Claude-ThirdParty.list"
    path: ./ruleset/Claude-ThirdParty.list
    interval: 86400

  Claude-IP:
    type: http
    behavior: ipcidr
    format: text
    url: "https://raw.githubusercontent.com/YOUR_USERNAME/YOUR_REPO/main/Claude-IP.list"
    path: ./ruleset/Claude-IP.list
    interval: 86400

rules:
  - RULE-SET,Claude-Core,Claude
  - RULE-SET,Claude-ThirdParty,Claude
  - RULE-SET,Claude-IP,Claude,no-resolve
  # 其他规则继续放在这里
```

`no-resolve` 放在 **IP 规则集引用行**，让该规则匹配已有目标 IP 时不主动触发域名解析。IP 规则不应取代域名规则。若不需要目标 IP 分流，可删除 `Claude-IP` 的 provider 和对应规则。

### 方案 C：只用一个远程文件（对应「这一整套规则」）

```yaml
rule-providers:
  Claude:
    type: http
    behavior: classical
    format: text
    url: "https://raw.githubusercontent.com/YOUR_USERNAME/YOUR_REPO/main/Claude.list"
    path: ./ruleset/Claude.list
    interval: 86400

rules:
  - RULE-SET,Claude,Claude,no-resolve
  # 其他规则继续放在这里
```

上面 provider 名称与代理组名称恰好都叫 `Claude`，在 Mihomo 中分别属于不同配置项；为便于区分，也可将 provider 改名 `Claude-Rules`，再将规则写成 `RULE-SET,Claude-Rules,Claude,no-resolve`。

如果希望使用**最宽覆盖**，把方案 C 两处文件名由 `Claude.list` 改成 `Claude-Full.list`；它还会将不少非 Claude 请求路由到 Claude 策略组，**不建议在路由器上默认启用**。

**规则排序：**将上述 `RULE-SET` 行放在通用 `GEOIP`、`GEOSITE`、`MATCH` 和宽泛域名规则之前，否则可能被先匹配的规则截获。`rule-providers` 与 `rules` 在主配置里各自只能有一个顶层同名键；与现有配置**合并**，不要复制出第二个 `rules:` 顶层键。

**OpenClash 提醒：**订阅生成或覆写操作可能重建运行配置。应将 provider 定义和规则引用保存在能够持久生效的自定义配置/覆写位置，避免仅修改运行中生成的临时 YAML 文件。

## 4. 为什么 `.list` 中没有策略组名称？

这是 **Mihomo 规则集（Rule Provider）**，不是 Clash 主配置的 `rules:` 列表。

合法的 classical `.list` 行示例：

```text
DOMAIN-SUFFIX,claude.ai
DOMAIN,api-iam.intercom.io
IP-CIDR,160.79.104.0/23
IP-CIDR6,2607:6bc0::/48
```

**不要**在 `.list` 每行末尾加 `,Claude`，也不要加入 `rules:`、`payload:`，或 YAML 列表的 `- `；策略组由 `RULE-SET,规则集名称,策略组名称` 指定。

`Claude-IP.list` 属于 `ipcidr`，内容格式不同：

```text
160.79.104.0/23
153.61.224.0/22
2607:6bc0::/48
```

## 5. 官方域名（A 类）

`Claude-Core.list` 仅收录下列六条后缀规则：

| 规则 | Anthropic 官方描述 |
|---|---|
| `DOMAIN-SUFFIX,anthropic.com` | API、官方静态资源、相关服务主机 |
| `DOMAIN-SUFFIX,claude.ai` | 网页、身份认证、桌面程序资源 |
| `DOMAIN-SUFFIX,claude.com` | 开发者平台、SSO 等 |
| `DOMAIN-SUFFIX,claude.app` | 动态应用实时预览 |
| `DOMAIN-SUFFIX,claudeusercontent.com` | 文件、Artifacts、bridge 主机 |
| `DOMAIN-SUFFIX,claudemcpcontent.com` | Claude MCP Apps 的动态内容主机 |

官方给出的具体主机例子（**均已由上面后缀规则覆盖，无须再逐一添加**）：

```text
api.anthropic.com
a-api.anthropic.com
a-cdn.anthropic.com
s-cdn.anthropic.com
assets-proxy.anthropic.com
mcp-proxy.anthropic.com
a.claude.ai
assets.claude.ai
downloads.claude.ai
platform.claude.com
sso.claude.com
bridge.claudeusercontent.com
*.livepreview.claude.ai
*.livepreview.claude.app
*.frame.claudeusercontent.com
```

`clau.de` 另外在客户端/社区证据中出现，收录在 `Claude-ThirdParty.list`、`Claude.list` 和 `Claude-Full.list`，**不冒充上面六条官方必须域名**。

## 6. 第三方精确端点（B/C 类）

| 服务 | 规则中列入的主机 | 核查强度与用途 |
|---|---|---|
| Intercom | `api-iam.intercom.io`、`nexus-websocket-a.intercom.io`、`nexus-websocket-b.intercom.io`、`api.intercom.io`、`api-ping.intercom.io`、`api-iam.eu.intercom.io`、`api-iam.au.intercom.io`、`nexus-europe-websocket.intercom.io`、`nexus-australia-websocket.intercom.io`、`js.intercomcdn.com`、`downloads.intercomcdn.com`、`uploads.intercomusercontent.com` | 见 Claude 网页 CSP；表示许可加载或连接，不等于每次实际调用 |
| Stripe | `r.stripe.com`、`js.stripe.com`、`m.stripe.network`、`b.stripecdn.com` | Claude 网页 CSP，支付相关嵌入资源 |
| Datadog | `http-intake.logs.us5.datadoghq.com`、`browser-intake-us5-datadoghq.com` | **Anthropic 官方 Claude Code 网络文档明确列出，但属于可选遥测** |
| Sentry | `o1158394.ingest.us.sentry.io` | 客户端审计/相关技术资料，可选错误上报 |
| Sift | `api3.siftscience.com` | 第三方 Claude Android 逆向审计识别的设备风控端点 |
| hCaptcha | `newassets.hcaptcha.com` | Claude 网页 CSP 中出现的验证资源，按流程触发 |

**特别纠正原截图：**

- `DOMAIN,us5.datadoghq.com` 不能覆盖 `http-intake.logs.us5.datadoghq.com`，更无法覆盖独立的 `browser-intake-us5-datadoghq.com`。
- `DOMAIN,us.sentry.io` 不能覆盖 `o1158394.ingest.us.sentry.io`。
- 上面两个截图中的泛化根主机没有放进可用规则集；README 留档供参考。
- `fcmtoken.googleapis.com` 属于 Google Firebase 通用服务；记录在 `Claude-Shared.list`，其作为 Claude 专用必要地址并未被 Anthropic 官方文档证实。

## 7. 通用第三方依赖（D 类）

`Claude-Shared.list` 包含下列跨应用共享主机。**只有想让这些服务一并固定走 Claude 出口时才启用。**

- Claude Code 插件/依赖：`storage.googleapis.com`、`registry.npmjs.org`、`github.com`、`raw.githubusercontent.com`、`formulae.brew.sh`。
- Claude Artifacts 依赖：`fonts.googleapis.com`、`fonts.gstatic.com`、`cdnjs.cloudflare.com`、`cdn.jsdelivr.net`、`cdn.tailwindcss.com`、`code.jquery.com`、`unpkg.com`。
- 网页 CSP/其他共享接口：`fcmtoken.googleapis.com`、`accounts.google.com`、`pay.google.com`、`maps.googleapis.com`、`maps.gstatic.com`。

这些域名为真实第三方服务，**不是专用 Claude 地址**。对家用路由器而言，特别是 `github.com`、`registry.npmjs.org`、`storage.googleapis.com`，会影响其他设备和软件的流量出口。因此默认组合 `Claude.list` **不包含**它们。

官方还提到特定 `*-review.googlesource.com` Gerrit 代码审查主机；由于它是动态子域名模式，本仓库没有用宽泛的 `DOMAIN-SUFFIX,googlesource.com` 代替，以免误代理无关站点。需要时按你的实际仓库填写精确 `DOMAIN` 规则。

## 8. IP 网段：入站、出站、停用和第三方

Anthropic [官方 IP 地址文档](https://platform.claude.com/docs/en/api/ip-addresses) 将入站（服务接收访问）与出站（Anthropic 向外部系统发起连接）区分开。不要混淆源 IP 和目标 IP。

### 8.1 官方入站地址：可选目标 IP 规则

| 网段 | 类型 | 在仓库中 |
|---|---|---|
| `160.79.104.0/23` | IPv4，入站 | `Claude-IP.list` 和合并版 |
| `153.61.224.0/22` | IPv4，入站 | `Claude-IP.list` 和合并版 |
| `2607:6bc0::/48` | IPv6，入站 | `Claude-IP.list` 和合并版 |

这些 CIDR 属于 Anthropic 官方所公布的服务入站范围，但**不代表 Claude 所有 CDN/代理目标 IP 都只能来自这些范围**，也不意味着必须依靠 IP 规则才能运行 Claude。

### 8.2 官方出站地址：不加入 Mihomo 目标地址规则

```text
160.79.104.0/21
153.61.208.0/20
```

它们用于识别**从 Anthropic 服务端发往外部服务器**的流量（例如远程 MCP）。典型场景是给你自己的服务器配置防火墙**来源 IP 允许列表**，而不是拿来给家用路由器匹配 Claude 目标地址。

### 8.3 Anthropic 官方公布的停用地址：不加入

```text
34.162.46.92/32
34.162.102.82/32
34.162.136.91/32
34.162.142.92/32
34.162.183.95/32
```

### 8.4 原截图中的 Google Cloud 共享网段：不加入

```text
34.8.0.0/16
```

这是云服务地址段，不是 Anthropic 官方公布的 Claude 专用 IP 范围；其覆盖 **65,536 个 IPv4 地址**，过宽，故仅在此留档，不包含在任何活动 `.list` 文件中。

## 9. 维护、验证与排障

1. 更新 GitHub 仓库文件后，等待 `interval: 86400`（秒）到期，或在 OpenClash/控制面板手动更新规则集。
2. 如果远程更新失败，检查 GitHub 仓库是否公开，是否使用 `raw.githubusercontent.com` 的**原始文件 URL**，而不是 GitHub 网页 `blob` 链接。
3. 验证文件下载的第一行不是 HTML，也不是 GitHub 登录页。`.list` 文件只含注释和纯文本规则。
4. 检查 provider 的 `behavior` / `format` 是否与文件一致：`classical + text` 对应 `Claude*.list`；**唯一例外**是 `Claude-IP.list`，要求 `ipcidr + text`。
5. 确认策略组 `Claude` 已定义；未定义将导致规则引用失败。
6. 确认 `RULE-SET` 在宽泛 `GEOIP` / `MATCH` 规则之前。
7. 当遇到第三方支付、推送或验证码异常时，先观察 Mihomo 连接日志，尽量只新增实际需要的精确域名。
8. 有条件时使用 `mihomo -t -f /path/to/config.yaml` 检查**整份主配置**；本仓库已完成离线格式、规则和 CIDR 校验，但不能替代在你实际 OpenClash 版本上的联网加载测试。
9. 如果局域网启用 IPv6，注意操作系统、Mihomo DNS/TUN、代理出口三者的 IPv6 状态可能不同；单纯添加 `IP-CIDR6` 不能保证 IPv6 代理可用。

### 两种格式不可混用

`format: text` 的 `classical` 内容形如 `DOMAIN-SUFFIX,claude.ai`，不是 `- DOMAIN-SUFFIX,claude.ai`；而 `format: yaml` 才要求 `payload:` 数组。本仓库统一以 GitHub 上易读、易审查的文本 `.list` 为主。

## 10. 来源与维护政策

**一级来源（官方文档）：**

- [Anthropic Claude Desktop：Network access requirements](https://code.claude.com/docs/en/desktop#network-access-requirements)
- [Anthropic Claude Code：Network access requirements](https://code.claude.com/docs/en/network-config#network-access-requirements)
- [Anthropic API：IP addresses](https://platform.claude.com/docs/en/api/ip-addresses)
- [Mihomo：rule-provider 配置](https://wiki.metacubex.one/en/config/rule-providers/)
- [Mihomo：rule-provider 文件内容](https://wiki.metacubex.one/en/config/rule-providers/content/)

**二级来源（网页扫描 / 社区客户端审计；不能代替官方要求）：**

- [urlscan.io：claude.ai 网页 CSP 样本](https://urlscan.io/result/019cafe1-ec34-743d-9e19-b1fa89df2fc4)
- [Claude 客户端遥测研究](https://github.com/B1anYu/claude-telemetry-audit)
- [Claude Android API 逆向审计](https://github.com/csr0403p-bit/claude-app-forensics/blob/main/API-SURFACE.md)

**更新原则：** 官方文档改动优先；仅凭第三方域名注册或一次 DNS 解析，不应将其提升为 Claude 专用地址；新增共享服务尽量放入 `Claude-Shared.list`；保留来源并注明适用客户端/版本。

---

> 本项目只处理路由分流，不解决订阅授权、地区准入、账号验证及服务可用性问题。

# Rules

Surge / Clash(Stash、OpenClash)/ Loon 共用的分流规则集。
**一份 `.list` 文件三家通吃，改一处全端同步。**

```
https://raw.githubusercontent.com/Rn1388/Config/refs/heads/main/rules/<文件名>
```

---

## 为什么一份文件能给三家用

| 客户端 | 所需格式 |
|---|---|
| Surge | `.list` 原生支持 |
| Clash / Stash / OpenClash | `behavior: classical` + `format: text` |
| Loon | `[Remote Rule]` 原生支持 |

三者的纯文本规则格式完全兼容。能共用的关键是——

> **策略名写在「引用行」，不写在文件里。**

所以同一个 `EU.list`，各端都指向自己的「欧洲」组，文件本身不含任何策略信息。

**前提：各端策略组名必须一致。** 本套以 Surge 端命名为准。

---

## 文件清单

| 文件 | 策略组 | 条数 | 内容 |
|---|---|---:|---|
| `IPcheck.list` | IP检查 | 68 | IP 查询 API / 设备指纹 / 反欺诈 SDK / 自测工具站 |
| `EU.list` | 欧洲 | 67 | KYC 认证、ether.fi、Plasma、MetaMask、MEXC、欧洲银行与运营商 |
| `Direct.list` | 直连 | 41 | 港澳银行、阿里系、IBKR 中国网关、Apple 国内 CDN |
| `Crypto.list` | 虚拟币 | 23 | Bybit 全家、WalletConnect / Reown |
| `Docker.list` | Docker更新 | 22 | 容器镜像仓库(Docker Hub / GHCR / Quay / GCR) |
| `US.list` | 美国 | 16 | IBKR 官网、Schwab、美国 MVNO |
| `Proxy.list` | 节点选择 | 9 | saltyfish、moomoo / 富途、IBKR 香港 |
| `Tailscale.list` | Tailscale / 直连 | 8 | MagicDNS + 家庭 LAN 网段 |
| `HK.list` | 香港 | 4 | fluidkey、CTM 澳门电讯 |
| `Douyin.list` | 抖音 | 4 | 抖音主域名 + keyword 兜底 |
| `Google.list` | 谷歌服务 | 2 | Google Play punycode 域名 |
| `ApplePush.list` | 苹果服务 | 1 | `push.apple.com` |
| `NG.list` | 尼日利亚 | 1 | `gov.ng` |
| `X.list` | X | 1 | Tapbots |
| `JP` `KR` `SG` `TW`.list | 日本 / 韩国 / 新加坡 / 台湾 | 占位 | 预留空文件 |

空文件里的 `DOMAIN,placeholder.invalid` 是 [RFC 2606](https://www.rfc-editor.org/rfc/rfc2606) 保留后缀，
永不解析、永不误伤，仅用于保证规则集非空（Clash 空 payload 会报错）。加入真实规则后可删。

---

## 引用方式

`<base>` = `https://raw.githubusercontent.com/Rn1388/Config/refs/heads/main/rules`

### Surge — `[Rule]` 段

```ini
RULE-SET,<base>/IPcheck.list,IP检查,extended-matching
RULE-SET,<base>/EU.list,欧洲,extended-matching
RULE-SET,<base>/Tailscale.list,Tailscale
```

### Clash / Stash / OpenClash

```yaml
rule-providers:
  EU-Custom:
    type: http
    behavior: classical      # 必须 classical
    format: text             # 必须 text
    url: <base>/EU.list
    path: ./ruleset/EU.list
    interval: 86400

rules:
  - RULE-SET,EU-Custom,欧洲
```

### Loon — `[Remote Rule]` 段

```
<base>/EU.list, policy=欧洲, tag=EU, enabled=true
```

---

## 三个坑

### 1. Clash 用错 `behavior` 会静默失效

`behavior: domain` 期望的是 `+.example.com` 这种纯域名，
本套是 `DOMAIN-SUFFIX,example.com` 的 classical 格式。

用 domain 去读会把整行当成一个域名字符串匹配 —— **永远不命中，且不报错**。

判断方法：看客户端面板里该 provider 的规则条数，对不上就是格式没配对，显示 0 通常是漏了 `format: text`。

### 2. GitHub 可达性是鸡生蛋问题

规则集本身要从 GitHub 拉取。如果「GitHub 走代理」这条规则只存在于远程规则集里，
首次加载或缓存失效时会拉取失败 —— **整个规则集静默不生效**，不报错。

主配置里硬编码一条，放在所有 `RULE-SET` 之前：

```ini
DOMAIN-SUFFIX,githubusercontent.com,节点选择
```

一条覆盖 `raw.` / `objects.` / `gist.` / `avatars.` 全部子域。

或者确保 `FINAL` 指向代理策略，靠兜底也能拉到——但那样可达性就押在 `FINAL` 上了。

### 3. `no-resolve` 写在文件里，不写在引用行

`EU.list` 和 `Tailscale.list` 含 IP-CIDR 规则，**参数已直接写在规则行内**：

```
IP-CIDR,87.194.0.0/16,no-resolve
```

这样三家通吃 —— Loon 的 `[Remote Rule]` 只认 `policy` / `tag` / `enabled`，
引用行没地方写参数，只能靠文件内的。

不加 `no-resolve` 的话，每个域名匹配前都要先做一次 DNS 解析去比对 IP 段，拖慢整条匹配链。

---

## 规则集之间无交集

16 个规则集经脚本交叉校验，**域名零重叠**（唯一重合是 `JP` / `KR` 的占位符）。

这意味着 **引用顺序不影响匹配结果** —— 想怎么排都行。
真正决定优先级的是它们相对于第三方规则集和 `GEOIP` / `FINAL` 的位置。

---

## Loon 的特殊性

Loon 的匹配优先级是 **`[Rule]` 本地规则 > 插件规则 > `[Remote Rule]` 订阅规则**，
和 Surge / Clash 的「按文件顺序」不同。带来两个后果：

**好处** — 本地覆盖特别方便。想让某个域名不跟规则集走，直接写进 `[Rule]` 即可，
自动压过 `[Remote Rule]`，不用考虑插入位置：

```ini
[Rule]
# 本地覆盖区
DOMAIN-SUFFIX,wise.com,节点选择
```

**坑** — `[Rule]` 里的 `GEOIP,CN` 会排在**所有**规则集之前。
凡是「域名解析为中国 IP、但需要走代理」的规则必须留在 `[Rule]` 本地，
否则会被 `GEOIP,CN` 提前命中走直连。典型是抖音和 Google Play 的 `.cn` 域名。

---

## 各端差异

`Tailscale.list` 是唯一各端指向不同的规则集：

| 端 | 指向 | 原因 |
|---|---|---|
| Surge | Tailscale 组 | 支持内置 Tailscale 策略 |
| Stash / Loon | 直连 | 无 Tailscale 策略组 |

另外 Surge 端 `[General]` 里如果设了 `tun-excluded-routes = 100.64.0.0/10`，
`Tailscale.list` 中该网段的规则不会生效——流量被排除在 TUN 之外，压根不进规则系统。
Mac 上这是必须的（让系统 Tailscale 客户端的 utun 独占该段路由），不是配置错误。

---

## 维护

### 只改本仓库即可（绝大多数情况）

- 增删改任何域名规则
- 把域名从一个策略组挪到另一个 → 从 A.list 剪到 B.list
- 收紧某条 keyword
- 给预留的地区文件补规则

push 后各端在 `interval` 到期时自动拉取（当前 86400 秒 = 24 小时）。
想立即生效就在客户端手动点一次「更新规则集」。

### 需要动主配置的情况

- 新增**策略组**（规则集只能引用已存在的组）
- 新增**规则集文件**（各端需各加一行引用）
- 改策略组的候选节点
- 改 DNS / MITM / 进程规则

### 日常流程

1. 平时在客户端面板一键加规则（写进本地配置）
2. 定期把新增的按策略归类，剪切进对应的 `rules/*.list`
3. push → 各端自动同步

---

## 关于 `IPcheck.list`

这个表值得单独说明。它收录的是**客户端会主动调用、用来查询自己出口 IP 和地区**的服务：

- **IP 回显 / 地理定位 API** — `ipinfo.io`、`ipify.org`、`ip-api.com`、`ipapi.co` 等
- **设备指纹 / 反欺诈 SDK** — Sardine、ThreatMetrix、Fingerprint、Sift、SEON、Forter 等
- **KYC 厂商** — Onfido、Persona、Veriff、Jumio
- **自测工具站** — whoer、ipleak、browserleaks

**为什么要单独成组**：这些域名如果和主业务走不同出口，服务端拿到的地区信息会自相矛盾，
直接触发风控。所以要么全部锁到和业务同一节点，要么全部 REJECT。

**一个前提**：实测 Uniswap / Aave / GMX / Hyperliquid 等站点，客户端一次 IP 查询都没有——
它们的地域判断在服务端 / CDN 边缘完成，设备上抓不到，分流规则管不了。
客户端能抓到的（如某些 App 调 `ipinfo.io`）是少数，但恰恰是这少数可以也应该控制好。

---

## License

个人自用配置，无授权限制，随意取用。

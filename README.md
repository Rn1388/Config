# Rn1388/Config —— Surge / Stash 共用规则集

一份 `.list` 文件,Surge 与 Clash/Stash 同时使用,改一处两端生效。

## 为什么能共用

Surge 的 `.list` 与 Clash 的 `behavior: classical` + `format: text` 格式完全兼容。
策略名写在**引用行**而不是文件内,所以同一个文件可以被两端指向各自的策略组。

前提:**两端策略组名必须一致**。本仓库以 Surge 端命名为准。

## 文件清单

| 文件 | 策略组 | 条数 | 说明 |
|---|---|---|---|
| `ipcheck.list` | IP检查 | 67 | 通用 IP 查询 / 设备指纹 / 风控 SDK |
| `eu.list` | 欧洲 | 59 | ether.fi、MetaMask、MEXC、n26、欧洲运营商、自用 KYC |
| `direct.list` | 直连 | 36 | 港澳银行、阿里系、IBKR 中国网关、Apple 国内 CDN |
| `crypto.list` | 虚拟币 | 19 | Bybit 全家(含混淆域名) |
| `us.list` | 美国 | 15 | IBKR 官网、Schwab、美国 MVNO |
| `proxy.list` | 节点选择 | 5 | IBKR TWS、deepseek、moomoo |
| `douyin.list` | 抖音 | 4 | |
| `ng.list` | 尼日利亚 | 3 | gov.ng、LemFi |
| `hk.list` | 香港 | 2 | fluidkey、CTM 澳门电讯 |
| `ai.list` | 人工智能 | 2 | Skywork |
| `google.list` | 谷歌服务 | 2 | Google Play punycode 域名 |
| `apple.list` | 苹果服务 | 1 | push.apple.com |
| `twitter.list` | T | 1 | tapbots |
| `eu-ip.list` | 欧洲 | 1 | IP 类规则,引用时必须带 `no-resolve` |
| `jp/kr/sg/tw.list` | 日本/韩国/新加坡/台湾 | 预留 | 空文件,直接追加即可 |

## 已废弃的旧规则集

以下文件的内容已全部并入上表,**可以从仓库删除**:

| 旧文件 | 并入 | 备注 |
|---|---|---|
| `EU.yaml` | `eu.list` + `eu-ip.list` | 其中 `IP-CIDR,87.194.0.0/16` 拆到 eu-ip.list |
| `US.yaml` | `us.list` | 三组归属调整见下 |
| `Direct.yaml` | `direct.list` | 新增 ikuai / asusgo;`ts.net` 未并入,见下 |
| `HK.yaml` | `hk.list` | |
| `NG.yaml` | `ng.list` | |
| `SG.yaml` | `proxy.list` | 其中 moomoo 原本就指向「节点选择」 |
| `TW.yaml` | `tw.list` | 原文件 payload 为空,Clash 加载会报错 |

**US.yaml 的三处归属调整**(默认候选第一位仍是「美国」,行为不变):

- `xn--ngstr-lra8j.com` / `services.googleapis.cn` → `google.list`(谷歌服务)
- `skywork.ai` / `skyworkcdn.com` → `ai.list`(人工智能)
- `push.apple.com` → `apple.list`(苹果服务)

**两处有意保留的两端差异**:

- `ts.net`:Surge 走 Tailscale 策略组,Stash 走直连。Tailscale 属设备级配置,不进共享规则集。
- `bankera`:归入 `direct.list`(按 Surge 主配置)。`eu.list` 里留有注释掉的对照版本,想改回欧洲时两处一起调。

空文件里的 `DOMAIN,placeholder.invalid` 是 RFC 2606 保留后缀,永不解析、永不误伤,
仅用于保证规则集非空(Clash 空 payload 会报错)。加入真实规则后可删。

## 引用方式

`<base>` = `https://raw.githubusercontent.com/Rn1388/Config/refs/heads/main/rules`

**Surge** —— `[Rule]` 段:

```
RULE-SET,<base>/eu.list,欧洲,extended-matching
```

**Clash / Stash** —— `rule-providers` + `rules`:

```yaml
rule-providers:
  EU-Custom:
    type: http
    behavior: classical      # 必须 classical
    format: text             # 必须 text,漏写会静默失效
    url: <base>/eu.list
    path: ./ruleset/eu.list
    interval: 86400
rules:
  - RULE-SET,EU-Custom,欧洲
```

**Loon** —— `[Remote Rule]` 段:

```
<base>/eu.list, policy=欧洲, tag=欧洲, enabled=true
```

## 两个必须知道的坑

**一、GitHub 可达性是鸡生蛋问题。**
规则集本身要从 GitHub 拉,而「GitHub 走代理」这条规则若只存在于远程规则集里,
首次加载或缓存失效时会拉取失败,**整个规则集静默不生效**,不报错。
所以两端主配置都必须硬编码一条,放在所有 RULE-SET 之前:

```
DOMAIN-SUFFIX,githubusercontent.com,节点选择
```

一条覆盖 `raw.` / `objects.` / `gist.` / `avatars.` 全部子域。

**二、Clash 用错 behavior 会静默失效。**
`behavior: domain` 期望的是 `+.example.com` 这种纯域名;
本仓库全部是 `DOMAIN-SUFFIX,example.com` 的 classical 格式。
用 domain 去读会把整行当成一个域名字符串匹配,**永远不命中且不报错**。

## 日常维护

1. 平时照旧在 Surge 面板一键加规则(写进本地 conf 底部)
2. 定期把新增的 `// Added for:` 按策略归类,剪切进对应的 `rules/*.list`
3. `git push` 后两端自动同步,主配置无需改动

## 验证

Surge:看规则集加载状态,无「加载失败」。
Clash/Stash:看面板各 provider 条数,应与上表一致。**显示 0 通常是 `format: text` 忘了写。**

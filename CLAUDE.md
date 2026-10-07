# CLAUDE.md

本文件为维护者与 AI 助手提供仓库的架构约定与操作规范。修改规则前请通读本文件。

## 仓库定位

Shadowrocket（iOS）代理分流配置。核心策略：国内域名/IP 直连，境外走代理，强化 DNS 防泄露与防污染。

> **v2rayN 端已停止维护**（2026-09-27 冻结），`v2rayn/` 目录保留可用。该端的配置说明、维护流程与排查记录归档至 [docs/v2rayn.md](docs/v2rayn.md)，本文件不再承载其约定。

## 目录结构

| 路径 | 作用 | 是否手改 |
|------|------|---------|
| `shadowrocket.conf` | Shadowrocket 主配置（DNS、Rule、URL Rewrite） | 手改 |
| `rules/ChinaDirect.list` | 国内域名直连补充 | **手改（唯一数据源）** |
| `rules/Reject.list` | 自定义广告/追踪拦截 | **手改（唯一数据源）** |
| `v2rayn/` | v2rayN 端配置（已停止维护） | 不再维护，见 [docs/v2rayn.md](docs/v2rayn.md) |

## 核心约定

### DRY：规则数据源唯一

`rules/ChinaDirect.list`（直连）与 `rules/Reject.list`（拦截）各是其规则的**唯一数据源**，通过远程 `RULE-SET` 直接拉取，订阅更新即生效，无需构建。

### 路由规则顺序绝不可乱改

代理分流按**首条匹配生效**（first-match-wins），规则顺序即优先级。

`shadowrocket.conf` 的 `[Rule]` 按 `Reject → ChinaDirect → 私有地址(meta) → ACL4SSR/meta(含 BanAD、category-ads-all) → 国内域名/IP → GEOIP,CN → FINAL` 自上而下匹配。关键约束：

- **自定义规则必须在广告拦截之前** —— 否则会被 `BanAD` 抢先拦掉，自定义直连形同虚设。
- **`FINAL` 必须置于末尾** —— 它匹配一切流量，一旦前移会吞掉后续所有规则，分流形同虚设。

### IPv6 保持彻底关闭

`shadowrocket.conf` 中 `ipv6 = false` + `prefer-ipv6 = false`，避免 v6 通道绕过 DNS 配置造成泄露。

### 新增直连域名前必须验证归属

新增任何直连域名前，**必须**先用 `.claude/skills/domain-verify` skill 核实域名真实存在、归属可信、非抢注。完整查询命令与归属判断规则见该 skill 文档，此处不复述。

红线（绝对不得添加）：

- 无 NS 记录（不存在 / 拼写错误）的域名**不得添加**
- 疑似抢注（TXT 含 afternic / sedo / dan.com 等挂售特征）的域名**不得添加**

黄线（可作特例收录，需约束）：

- 归属主体无法确认、且无抢注特征的域名，满足下列任一依据时可作特例收录：
  - **NS 在国内托管商**（阿里云 / 腾讯 DNSPod / 华为云等）——推定国内业务，按国内线路直连。
  - **站点排斥代理出口 IP**（解析落境外或不走国内线路）——依赖代理会触发站点风控，需域名级直连方可访问。
- 特例一律收录至 `ChinaDirect.list` 末尾「归属未确认的特例」分区，收录依据由**分区标题统一记录**，不逐条注释；待归属明确后迁移至上方对应分组或移除。核查过程中的 DNS 快照（TXT / SOA / 解析 IP 等）不入档——易过期，且复核时必然重查，详见「规则注释规范」。

目的：避免把抢注或拼写错误的域名误加进直连，造成流量错误放行。

### 优先依赖上游规则，不重复收录

`ChinaDirect.list` 只补 **ACL4SSR `ChinaDomain.list` 与 meta-rules-dat `geosite cn` 均未覆盖**的域名（IP 兜底带 `no-resolve`，不匹配域名请求，不视为域名兜底）。已被上游覆盖的不重复添加（DRY）：

- `.com.cn` / `.cn` 域名：由 ACL4SSR `ChinaDomain.list` / meta `cn.list`（含 `tld-cn`）兜底，通常无需手动添加。注意 Shadowrocket 的 IP 兜底带 `no-resolve`，**不会为域名触发解析**，纯域名请求跳过 IP 规则落 `FINAL`，不能指望它兜住域名
- 上游已收录的域名（ACL4SSR 或 meta `cn.list`）：如 `abchina.com`、`cmbchina.com`、`ecitic.com`

真正需要手动补的是 **`.com` 顶级域且不被 ACL4SSR `ChinaDomain.list` 与 meta `cn.list` 覆盖** 的国内业务域名。

### 上游规则集（ACL4SSR + meta-rules-dat）

两套上游并用，分工如下（覆盖率 2026-10-07 实测）：

| 用途 | 来源 | 说明 |
|---|---|---|
| 私有地址 | meta `geosite private` + `geoip private` | 取代 ACL4SSR `LocalAreaNetwork`，含 `100.64.0.0/10`（Tailscale） |
| 国内域名 | ACL4SSR `ChinaDomain/ChinaMedia` + meta `geosite cn` | ACL 的 623 条中 63 条（如 baidustatic.com、bootcss.com）meta 未收录，故并存；meta `cn.list` 约 11 万条/3MB，**iOS 隧道内存有风险，上手机后若 Shadowrocket 崩溃/卡顿，先移除该行** |
| 国内 IP | ACL4SSR `ChinaCompanyIp` + meta `geoip cn` | meta 列表无内联 `no-resolve`，已在 `RULE-SET` 行补 |
| 广告 | ACL4SSR `BanAD/BanProgramAD` + meta `category-ads-all` | meta 仅覆盖 ACL 的约 12%，取并集而非替换 |
| Apple / UnBan | ACL4SSR | meta `apple` 缺 `appstore.com`、`akadns.net`，UnBan 无对应物 |

**不要用** meta `geolocation-cn` 代替 `cn.list`：仅 5.4k 条，漏掉 163.com、1688.com、360buyimg.com 等；**不要加** `category-httpdns-cn`（见 [troubleshooting](docs/troubleshooting.md) BlockHttpDNS 一节）。

### DOMAIN-SUFFIX 优先

银行、大厂等多子域场景优先用 `DOMAIN-SUFFIX`（后缀匹配），一条覆盖全部子域（如 `example.com` 覆盖 `www.example.com` / `api.example.com`）。仅在需精确匹配单个域名时用 `DOMAIN`。

### 规则注释规范

`.list` 文件的分组注释统一为**单行** `# 中文名 (运营主体公司)`，例如：

```
# 猎聘 (同道精英（天津）信息技术有限公司)
DOMAIN-SUFFIX,liepin.com
DOMAIN-SUFFIX,lietou-static.com
```

**不写入注释的内容**——`domain-verify` 的核查过程属于决策依据，不属于规则本身，核查完成后即丢弃，不留档：

- 解析 IP、NS、SPF、MX、TXT、证书主体等 DNS 快照（易过期，制造「注释与现实不符」的维护负担）
- 「ACL4SSR 未收录」「GEOIP no-resolve 不匹配域名请求」「需域名级直连」等论证——这是**全表的收录前提**，已在 `ChinaDirect.list` 文件头声明一次，逐条复述违反 DRY

**必须保留的两类例外：**

| 类型 | 写法 | 原因 |
|---|---|---|
| 海外域名的收录理由 | `# iHerb 海淘保健品电商 (美国公司，非国内域名；直连避免代理触发风控拒单)` | 否则无法解释它为何在国内直连表 |
| 末尾特例区说明 | 分区标题统一记录收录依据（NS 在国内托管商 / 站点排斥代理出口 IP） | 见「新增直连域名前必须验证归属」的黄线要求 |

## 排查参考

Shadowrocket 端运行时排查见 [docs/troubleshooting.md](docs/troubleshooting.md)；v2rayN 端的排查记录见 [docs/v2rayn.md](docs/v2rayn.md)。遇到代理异常**先查对应文档**，避免误改分流规则——许多"看似分流问题"的症状实为系统层原因。

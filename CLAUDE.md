# CLAUDE.md

本文件为维护者与 AI 助手提供仓库的架构约定与操作规范。修改规则前请通读本文件。

## 仓库定位

Shadowrocket（iOS）代理分流配置。核心策略：国内域名/IP 直连，境外走代理，强化 DNS 防泄露与防污染。

> **v2rayN 端已停止维护**（2026-09-27 冻结），`v2rayn/` 目录保留可用。该端的配置说明、维护流程与排查记录归档至 [docs/v2rayn.md](docs/v2rayn.md)，本文件不再承载其约定。

## 目录结构

| 路径 | 作用 | 是否手改 |
|------|------|---------|
| `shadowrocket.conf` | Shadowrocket 主配置（DNS、Rule、URL Rewrite） | 手改 |
| `v2rayn/` | v2rayN 端配置（已停止维护） | 不再维护，见 [docs/v2rayn.md](docs/v2rayn.md) |

## 核心约定

### 只用上游规则集，不自建规则

`[Rule]` 只引用 meta-rules-dat 的现成列表，不在仓库里维护自建 `.list`（`rules/` 目录已于 2026-10-07 移除，历史见 git）。某域名分流不对时，不要加自建规则，而是换/调整上游列表或接受现状。

### 路由规则顺序绝不可乱改

代理分流按**首条匹配生效**（first-match-wins），规则顺序即优先级。

`shadowrocket.conf` 的 `[Rule]` 按 `私有地址 → Apple → 广告(category-ads-all) → 国内域名/IP → GEOIP,CN → FINAL`（均为 meta-rules-dat） 自上而下匹配。关键约束：

- **自定义规则必须在广告拦截之前** —— 否则会被 `BanAD` 抢先拦掉，自定义直连形同虚设。
- **`FINAL` 必须置于末尾** —— 它匹配一切流量，一旦前移会吞掉后续所有规则，分流形同虚设。

### IPv6 保持彻底关闭

`shadowrocket.conf` 中 `ipv6 = false` + `prefer-ipv6 = false`，避免 v6 通道绕过 DNS 配置造成泄露。

### 上游规则集（meta-rules-dat）

全部上游规则取自 [MetaCubeX/meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat) `meta` 分支的 classical 列表（已于 2026-10-07 全面取代 ACL4SSR）：

| 用途 | 列表 | 说明 |
|---|---|---|
| 私有地址 | `geosite private` + `geoip private` | 含 `100.64.0.0/10`（Tailscale） |
| Apple | `geosite apple` | 比原 ACL `Apple.list` 缺 `appstore.com`、`akadns.net` |
| 广告 | `geosite category-ads-all` | 仅覆盖原 ACL `BanAD/BanProgramAD` 的约 12%（取舍已接受） |
| 国内域名 | `geosite cn` | 约 11 万条/3MB，**iOS 隧道内存有风险，上手机后若 Shadowrocket 崩溃/卡顿，先移除该行** |
| 国内 IP | `geoip cn` | meta 列表无内联 `no-resolve`，已在 `RULE-SET` 行补 |

迁移时的损失：原 ACL `ChinaCompanyIp` 有 94 段（阿里/腾讯海外段等）不在 `geoip cn`，现走代理；原 `UnBan` 无对应物，已丢弃；原 `ChinaDomain/ChinaMedia` 有 63 条 meta 未收录，包括 `baidustatic.com`、`snssdk.com`、`bootcss.com` 等国内业务域名与境外游戏/下载域名（steam、epic、playstation、teamviewer 等），现均不再直连。自建 `ChinaDirect.list`/`Reject.list` 同时移除，影视站 jisu 域名等需自行评估（见 troubleshooting）。

**不要用** meta `geolocation-cn` 代替 `cn.list`：仅 5.4k 条，漏掉 163.com、1688.com、360buyimg.com 等；**不要加** `category-httpdns-cn`（见 [troubleshooting](docs/troubleshooting.md) BlockHttpDNS 一节）。

## 排查参考

Shadowrocket 端运行时排查见 [docs/troubleshooting.md](docs/troubleshooting.md)；v2rayN 端的排查记录见 [docs/v2rayn.md](docs/v2rayn.md)。遇到代理异常**先查对应文档**，避免误改分流规则——许多"看似分流问题"的症状实为系统层原因。

# Shadowrocket 防 DNS 泄露配置

基于 [meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat) 规则集的 Shadowrocket 配置文件，国内 UDP + 境外 DoH 混合策略，兼顾防泄露与 CDN 调度精准。

## 订阅链接

复制以下任一链接到 Shadowrocket 进行订阅。电脑端如使用 v2rayN，请见 [v2rayN 归档说明](docs/v2rayn.md)。

**raw.githubusercontent.com（权威源）：**

```
https://raw.githubusercontent.com/fqix/shadowrocket-config/main/shadowrocket.conf
```

**jsDelivr CDN（国内推荐，速度更快）：**

```
https://cdn.jsdelivr.net/gh/fqix/shadowrocket-config@main/shadowrocket.conf
```

## 使用方法

1. 打开 Shadowrocket → 底栏「配置」→ 右上角 `+`
2. 粘贴上述任一订阅链接 → 「下载」
3. 等待下载完成后，长按该配置 → 「使用配置」
4. 配置生效后，规则集会自动从远端拉取（首次加载需联网）

## 核心特性

### DNS 防泄露

- **国内 UDP + 境外 DoH 混合并发**：阿里/腾讯 UDP + Cloudflare/Google DoH 四路并发取最快
- **国内站点保最优 CDN**：UDP DNS 携带 ECS 客户端子网，CDN 调度精准到本地节点
- **境外站点防污染**：Cloudflare/Google DoH 加密兜底，自动剔除被污染的 IP
- **禁用 IPv6**：避免 v6 通道绕过 DNS 配置造成泄露
- **拒绝私有 IP 应答**：防 DNS rebinding 攻击
- **DNS 失败请求走代理重试**：`FINAL,PROXY,dns-failed`

### URL 重写

- **Google 域名重定向**：`g.cn` / `google.cn` 自动 302 跳转至 `google.com`，避免访问国内镜像

### 分流策略

| 序号 | 策略 | 规则 | 说明 |
|---|---|---|---|
| 1 | DIRECT | 局域网 | meta `geosite/geoip private`（含 Tailscale `100.64.0.0/10`） |
| 2 | DIRECT | Apple / App Store | meta `geosite apple`，**美区账号不清楚是否有风险** |
| 3 | REJECT | 广告域名 | meta `category-ads-all` |
| 4 | DIRECT | 国内域名 | meta `geosite cn` |
| 5 | DIRECT | 国内 IP | meta `geoip cn` |
| 6 | DIRECT | `GEOIP,CN` | 内置 GeoIP 兜底 |
| 7 | PROXY | 其他流量 | `FINAL,PROXY,dns-failed` 兜底 |


## 维护

### 规则集同步

所有 `RULE-SET` 均引用 [MetaCubeX/meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat) 的 `meta` 分支 classical 列表，上游更新后下次刷新订阅自动同步。`geosite cn` 约 11 万条，若 iOS 上 Shadowrocket 内存吃紧，移除该行即可。

### 修改配置

直接编辑 `shadowrocket.conf` 并推送到本仓库，Shadowrocket 下次刷新订阅即生效。

### 切换 CDN 源

如发现 `raw.githubusercontent.com` 在国内拉取失败，临时改用 jsDelivr 订阅链接即可。如需让配置文件内部的 `RULE-SET` 也走 jsDelivr，替换以下前缀：

**meta-rules-dat 规则：**

```
https://raw.githubusercontent.com/MetaCubeX/meta-rules-dat/meta/
```
→
```
https://cdn.jsdelivr.net/gh/MetaCubeX/meta-rules-dat@meta/
```

文件路径保持不变。

## v2rayN（已停止维护）

v2rayN 端已于 2026-09-27 停止维护，配置说明、维护流程与排查记录归档至 [docs/v2rayn.md](docs/v2rayn.md)。`v2rayn/` 目录与 `routing.json` 订阅链接继续可用，规则不再更新。

# 许可证

仅供个人使用。规则集版权归 [meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat) 所有。

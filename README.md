# Shadowrocket Config

适用于 macOS、iOS 和 iPadOS 的 Shadowrocket 覆盖模块，负责局域网直连和 DNS 防污染。
代理节点、订阅及远程规则集仍由主配置管理。

## 文件

| 文件 | 用途 |
| --- | --- |
| `LANDirect.sgmodule` | RFC 1918 私网直连及常见 LAN 旁路 |
| `NoDNSLeak.sgmodule` | 国内 DoH、代理远端解析和 DNS 回退 |
| `CorpDirect.example.sgmodule` | 私有公司 IP 模块的脱敏模板 |

仓库不包含节点、订阅链接、凭据、私钥或证书。

## 安装

在 `配置 > 模块 > 右上角 +` 添加：

```text
https://raw.githubusercontent.com/felixtensor/shadowrocket-config/main/LANDirect.sgmodule
https://raw.githubusercontent.com/felixtensor/shadowrocket-config/main/NoDNSLeak.sgmodule
```

启用模块后，对当前主配置执行一次“使用配置”，再断开并重新连接 Shadowrocket。

## 私有模块

`CorpDirect.example.sgmodule` 仅作为格式模板。复制为 `CorpDirect.sgmodule` 并替换成真实
CIDR 后在本地使用。

真实文件已由 `.gitignore` 排除，不会进入公开仓库，可通过 Shadowrocket iCloud 同步到
Mac、iPhone 和 iPad。

## 必要设置

- `设置 > 代理 > 代理类型 > None`：TUN Only。
- `设置 > 隧道 > 强制路由`：开启。
- `设置 > 隧道 > 包括所有网络`：开启。
- `设置 > 隧道 > 包括本地网络`：关闭。
- `设置 > 隧道 > 包括 APNs`：关闭。
- `设置 > 隧道 > 包括蜂窝服务`：关闭。
- `设置 > UDP > 启用转发`：节点支持 UDP 时开启。
- `设置 > UDP > 禁用 STUN`：开启。
- 全局路由：选择“配置”。

“包括所有网络”扩大 TUN 接管范围，但流量仍由当前配置决定 `DIRECT` 或 `PROXY`。
关闭“包括本地网络”以保留局域网兼容性，并继续启用 `LANDirect.sgmodule`，为进入
TUN 的私网流量提供明确的 CIDR 直连保障。

不要在其他启用的模块或主配置覆盖中重复声明相同的 `[General]` DNS 键。旧版
`bypass-system` 已被社区手册标记为弃用，不应加入本配置。

## 行为

路由：

- `10.0.0.0/8`：`DIRECT`，仍由 TUN 接管。
- `172.16.0.0/12`、`192.168.0.0/16`：`DIRECT`，同时旁路 TUN。
- 回环、链路本地、组播和广播地址旁路 TUN。
- 私有 `CorpDirect.sgmodule` 中的地址使用 `DIRECT,no-resolve`。

DNS：

```text
DIRECT  -> AliDNS / DNSPod DoH
PROXY   -> 代理服务器远端解析
Fallback -> Cloudflare / Google DoH，经当前代理
.local  -> Apple 系统解析器
```

公网查询不会主动回落到系统 DNS。应用自行使用的 DoH 属于普通 HTTPS 流量，不在
`hijack-dns` 的拦截范围内。

## 验证

在 `数据 > 代理 > DNS` 开启日志，确认：

- 公网 DIRECT 查询不使用 `system`。
- PROXY 域名由代理端解析。
- 私网和私有模块地址命中 `DIRECT`。
- DNS 泄露测试不显示本地运营商 DNS。

测试工具：

- <https://www.dnsleaktest.com/>
- <https://browserleaks.com/dns>
- <https://ipleak.net/>

## 参考

- [Shadowrocket 社区使用手册](https://github.com/LOWERTOP/Shadowrocket)
- [IANA IPv4 Special-Purpose Address Space](https://www.iana.org/assignments/iana-ipv4-special-registry/iana-ipv4-special-registry.xhtml)

本仓库不提供代理服务。LOWERTOP 手册由社区维护，并非 Shadowrocket 官方文档。

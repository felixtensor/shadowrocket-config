# Shadowrocket Config

一组适用于 Shadowrocket for macOS、iOS 和 iPadOS 的个人覆盖模块。公开仓库负责局域网
和 DNS 防污染，环境相关的公司 IP 模块使用脱敏模板管理；代理节点、订阅和远程规则集
仍由主配置负责。

本仓库不包含代理节点、订阅链接、账号密码、私钥或 MITM 证书。

## 模块

| 模块 | 用途 |
| --- | --- |
| `CorpDirect.example.sgmodule` | 公司私网与指定公网 IP 的脱敏模板，不应直接启用 |
| `LANDirect.sgmodule` | RFC 1918 私网直连，旁路常见 LAN、链路本地和组播路由 |
| `NoDNSLeak.sgmodule` | 国内 DoH、代理远端解析、代理 DNS 回退和明文 DNS 劫持 |

这些文件是覆盖模块，不是完整 Shadowrocket 配置。主配置必须提供可用代理节点，并以
`FINAL,PROXY` 作为兜底规则。

## 安装

在 Shadowrocket 打开 `配置 > 模块 > 右上角 +`，添加两个公开远程模块：

```text
https://raw.githubusercontent.com/felixtensor/shadowrocket-config/main/LANDirect.sgmodule
https://raw.githubusercontent.com/felixtensor/shadowrocket-config/main/NoDNSLeak.sgmodule
```

`CorpDirect.sgmodule` 是本地私有文件，由 `CorpDirect.example.sgmodule` 复制并替换为真实
CIDR 后使用。它被 `.gitignore` 排除，不会提交到公开仓库；可通过 Shadowrocket iCloud
同步到 Mac、iPhone 和 iPad。

启用模块后：

1. 停用其他包含重复 DNS、LAN 或公司 IP 直连设置的模块。
2. 确认 `Corp Direct`、`LAN Direct` 和 `No DNS Leak` 均已启用。
3. 对当前主配置执行一次“使用配置”，重新编译模块和远程资源。
4. 断开并重新连接 Shadowrocket。

模块规则优先于主配置，规则从上到下匹配。`Corp Direct` 建议排在 `LAN Direct`
前面，让精确 CIDR 优先出现在编译结果和日志中；两者都使用 `DIRECT`，顺序不会改变
实际出口。

## 更新

- 手动：`配置 > 模块`，更新远程模块后重新“使用配置”。
- 自动：在 Shadowrocket 自动更新设置中启用模块后台更新。
- 多设备：可同时启用 Shadowrocket iCloud 同步，让模块记录在 Mac、iPhone 和 iPad
  之间同步；公开模块内容仍以本仓库 Raw URL 为准，私有 `CorpDirect.sgmodule` 以
  iCloud 副本为准。

首次下载 Raw URL 需要能够访问 GitHub。Shadowrocket 会保留已下载的本地副本，网络
不可用时不会因为 GitHub 暂时无法访问而删除现有模块。

## 路由设计

### 私有公司模块

公开仓库只提供 `CorpDirect.example.sgmodule`，其中使用示例地址：

```text
10.20.30.0/24
10.40.50.0/24
10.60.70.0/24
203.0.113.10/32
```

实际 `CorpDirect.sgmodule` 保存在本地且不进入 Git。相关规则使用
`DIRECT,no-resolve`：

- 连接不经过代理节点。
- 流量仍由 Shadowrocket TUN 接管和记录。
- `no-resolve` 避免域名请求为了匹配 CIDR 而触发本地 DNS。

### LAN

`LANDirect` 将三个 RFC 1918 私网设为 `DIRECT`：

```text
10.0.0.0/8
172.16.0.0/12
192.168.0.0/16
```

其中 `172.16/12` 和 `192.168/16` 完全旁路 TUN，用于不同 Wi-Fi 下的路由器、打印机、
AirPlay、mDNS、SSDP 和其他本地 UDP/组播服务。`10/8` 不加入 TUN 旁路；它保持
`DIRECT`，但仍可由 Shadowrocket 观察。

其他旁路范围：

```text
127.0.0.0/8
169.254.0.0/16
224.0.0.0/4
255.255.255.255/32
fe80::/10
ff02::fb/128
```

不配置 `100.64.0.0/10`，因为该共享地址空间可能属于运营商 CGNAT 或 Tailscale，统一
设为 `DIRECT` 可能与实际用途冲突。文档测试网段和已废弃的特殊地址也不加入 TUN
旁路，因为它们不参与本地发现。

仓库只记录 RFC 1918 聚合网段和脱敏示例，不保存任何环境中的真实私网子网。位于
`10/8`、`172.16/12` 或 `192.168/16` 的内部服务仍会命中 `LANDirect` 并直连，但无法
从公开仓库反推出实际内部网络划分。

## DNS 设计

```text
DIRECT 域名
  -> AliDNS / DNSPod DoH 并行查询

PROXY 域名
  -> 代理服务器远端解析

代理节点域名
  -> 继承 AliDNS / DNSPod DoH

国内 DoH 超时或失败
  -> Cloudflare / Google DoH
  -> 当前默认代理

.local
  -> Apple 系统解析器
```

关键行为：

- AliDNS 与 DNSPod DoH 并行查询，采用最先返回的有效结果。
- 不设置 `proxy-dns-server`；节点域名按 Shadowrocket 规则继承 `dns-server`。
- `[Host]` 固定两个国内 DoH 入口的 IPv4，启动过程不依赖系统 DNS。
- `fallback-dns-server` 显式使用 `#proxy`，不留空，避免回落到 `system`。
- `dns-direct-system = false` 禁止公网 DIRECT 域名使用系统 DNS。
- `always-ip-address = false` 保留代理域名的远端解析。
- `dns-direct-fallback-proxy = true` 在国内 DoH 失败时优先保证可用性。
- `udp-policy-not-supported-behaviour = REJECT` 在节点不支持 UDP 时失败关闭。
- `block-quic = all-proxy` 只阻断代理流量的 QUIC，不影响 DIRECT 流量。
- `ipv6 = false` 与 `prefer-ipv6 = false` 禁用常规公网 IPv6 解析偏好。

`hijack-dns` 只覆盖模块中列出的常见明文 DNS 地址。应用自行使用的任意 DoH 属于普通
HTTPS 流量，不在该规则的拦截范围内。

## 必要设置

- `设置 > 代理 > 代理类型 > None`：TUN Only。
- `设置 > 隧道 > 强制路由`：开启。
- `设置 > 隧道 > 包括所有网络`：开启。
- `设置 > 隧道 > 包括本地网络`：开启。
- `设置 > UDP > 启用转发`：节点支持 UDP 时开启。
- `设置 > UDP > 禁用 STUN`：开启。
- 全局路由：选择“配置”。

不要在其他启用的模块或主配置覆盖中重复声明相同的 `[General]` DNS 键。旧版
`bypass-system` 已被社区手册标记为弃用，不应加入本配置。

## 验证

启用模块后，打开 `数据 > 代理 > DNS > 启用日志记录`，检查：

- 公网 DIRECT 查询由 AliDNS 或 DNSPod DoH 处理，不出现 `system`。
- PROXY 域名由代理端解析。
- 国内 DoH 失败时，回退请求通过代理访问 Cloudflare 或 Google DoH。
- RFC 1918 私网和私有 `CorpDirect.sgmodule` 中的地址命中 `DIRECT`。
- 网页测试不显示本地运营商 DNS 或本机公网 IPv6。

可使用：

- <https://www.dnsleaktest.com/> Extended test
- <https://browserleaks.com/dns>
- <https://ipleak.net/>

macOS 还可以检查物理接口是否存在非预期的明文 53 端口查询：

```sh
IFACE=$(route -n get default | awk '/interface:/{print $2}')
sudo tcpdump -ni "$IFACE" '(udp port 53 or tcp port 53)'
```

仅当 Shadowrocket DNS 日志、物理接口抓包和网页测试同时符合预期，才认为完成实机
闭环。`fallback-dns-server` 与 `#proxy` 的组合仍应以当前客户端日志确认。

## 维护

- 公司 CIDR 或指定公网服务 IP 变化：修改本地 `CorpDirect.sgmodule`，不要提交该文件。
- 公开模板结构变化：修改 `CorpDirect.example.sgmodule`。
- LAN 旁路需求变化：修改 `LANDirect.sgmodule`。
- DNS 上游或隐私策略变化：修改 `NoDNSLeak.sgmodule`。
- 机场节点和远程规则变化：更新主配置，不修改这些模块。

每次修改后应重新编译配置，并至少检查规则命中和 DNS 日志。

## 参考

- [Shadowrocket 社区使用手册](https://github.com/LOWERTOP/Shadowrocket)
- [IANA IPv4 Special-Purpose Address Space](https://www.iana.org/assignments/iana-ipv4-special-registry/iana-ipv4-special-registry.xhtml)
- [腾讯云 Public DNS 接入说明](https://cloud.tencent.com/document/product/302/110786)

LOWERTOP 手册由社区维护，并非 Shadowrocket 官方文档。本仓库不提供代理服务，使用者
需要自行确保配置符合所在地区的法律法规，并自行承担使用风险。

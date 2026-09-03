# Shadowrocket Config

适用于 macOS、iOS 和 iPadOS。主配置基于 LOWERTOP / Johnshall 的懒人配置（2026-08-07）
维护，默认国内直连、国外和未知域名代理，并针对 DNS 泄露做了加固。

## 文件

| 文件 | 用途 |
| --- | --- |
| `Main.conf` | 主配置，启用 IPv6 |
| `Main_no_ipv6.conf` | 同一份主配置，关闭 IPv6 |
| `NoDNSLeak.sgmodule` | DNS、代理 fallback、节点 bootstrap 和 53 端口劫持，两个主配置共用 |
| `CorpDirect.example.sgmodule` | 可选的公司网段与内网 DNS 模板 |

两个主配置只启用一个。默认用 `Main.conf`；IPv6 链路不稳时换 `Main_no_ipv6.conf`，典型
症状是微信一直显示连接中。

节点订阅在 Shadowrocket 首页单独管理，不要把订阅 URL、节点密码、UUID 或私钥提交到 Git。

## 安装

1. 在 `配置 > 右上角 + > 下载配置` 添加主配置，二选一：

   ```text
   https://raw.githubusercontent.com/felixtensor/shadowrocket-config/main/Main.conf
   https://raw.githubusercontent.com/felixtensor/shadowrocket-config/main/Main_no_ipv6.conf
   ```

2. 在 `配置 > 模块 > 右上角 +` 添加并启用 DNS 模块：

   ```text
   https://raw.githubusercontent.com/felixtensor/shadowrocket-config/main/NoDNSLeak.sgmodule
   ```

3. 在首页添加自己的节点订阅。

4. 如需公司网络，复制 `CorpDirect.example.sgmodule` 为本地 `CorpDirect.sgmodule`，替换
   真实 CIDR 和域名后启用。该文件已被 `.gitignore` 排除。

5. 对主配置执行一次“使用配置”，断开并重新连接 Shadowrocket；全局路由选择“配置”。

内网域名需要同时配置规则和解析器：

```text
DOMAIN-SUFFIX,corp.example,DIRECT
*.corp.example = server:system
```

也可以把 `server:system` 换成明确的公司 DNS 地址。

## 必须设置

- `设置 > 代理 > 代理类型 > None`（TUN Only）。
- `设置 > 隧道 > 强制路由`：开启。
- `设置 > 隧道 > 包括所有网络`：开启。
- `设置 > 隧道 > 包括本地网络`：关闭。
- `设置 > UDP > 启用转发`：节点支持 UDP 时开启。
- `设置 > UDP > 禁用 STUN`：开启。
- HTTPS 解密：关闭。
- `配置文件 ⓘ > 通用 > 启用IPv6`：跟随主配置，不要在 UI 里手动改，切换请换配置文件。

不要启用其他会修改 `dns-server`、`direct-dns-server`、`fallback-dns-server`、
`proxy-dns-server`、`hijack-dns`、`[Host]` 或 `ipv6` 的模块。

## DNS 行为

```text
DIRECT 域名             -> AliDNS / DNSPod DoH，直连
DIRECT DNS 失败回退      -> Cloudflare DoH，经代理
PROXY 域名               -> 代理服务器远端解析
代理节点域名             -> AliDNS / DNSPod DoH，直连 bootstrap
普通 UDP/TCP 53          -> Shadowrocket 全量劫持
局域网和公司内网域名     -> system 或指定内网 DNS
```

国内域名先由 `RULE-SET` / `DOMAIN-SET` 命中 `DIRECT`，再使用国内 DoH 获取适合当前网络
的 CDN 地址；国外和未知域名命中代理策略后由代理服务器远端解析。`GEOIP,CN` 仅使用设备
中已安装的 Country MMDB 兜底直接 IP 连接，不触发域名预解析。

节点域名必须在代理建立前解析，因此国内 DoH bootstrap 不能依赖当前代理。浏览器或应用
自带 DoH/DoQ 不使用 53 端口，建议关闭浏览器“安全 DNS”，或确认其 DNS 服务域名走代理。

## IPv6 行为

两份主配置只差两处：`[General]` 的 `ipv6`、`prefer-ipv6`、`ipv6-only-if-no-ipv4-dns`，
以及 `Main.conf` 的 `tun-excluded-routes` 补齐了 `::1/128`、`fc00::/7`、`fe80::/10`、
`ff00::/8`。分流规则两份相同，`Lan.list` 已含 IPv6 局域网段。

`ipv6` 是 DNS 层开关，代理流量的出口 IP 族由节点决定。

已知限制：

- `[Host]` 的 DoH bootstrap 固定为 IPv4，纯 IPv6 网络下需自行改成 IPv6 地址。
- `hijack-dns = *:53` 是否覆盖 IPv6 目的地址，手册未说明，用 DNS 日志确认。
- 即使 `ipv6 = false`，节点域名能解析出 AAAA 时仍会走节点的 IPv6 地址。
- 内置 Country MMDB 是否含 IPv6 段未实测，不含时国内 IPv6 直连会落到 `FINAL,PROXY`。
- WebRTC 的 IPv6 泄露依赖 `设置 > UDP > 禁用 STUN`。

## 验证

在 `数据 > 代理 > DNS` 开启日志，并在 `配置 > 测试规则` 检查：

```text
www.xiaohongshu.com               -> DIRECT
sns-webpic-qc.xhscdn.com          -> DIRECT
tieba.baidu.com                   -> DIRECT
www.google.com                    -> 代理策略
随机字符串.dns4.browserleaks.net  -> FINAL / PROXY
223.5.5.5                         -> GEOIP CN / DIRECT
1.1.1.1                           -> FINAL / PROXY
240e::1                           -> GEOIP CN / DIRECT
2606:4700:4700::1111              -> FINAL / PROXY
```

重新连接后测试：

- <https://www.dnsleaktest.com/>
- <https://browserleaks.com/dns>
- <https://ipleak.net/>
- <https://test-ipv6.com/>

DNS 泄露测试生成的未知域名不应由本地运营商、AliDNS 或 DNSPod 解析。国内直连域名使用
AliDNS / DNSPod 属于预期行为；测试页显示代理出口侧的 Cloudflare、Google 或代理服务商
递归 DNS 也属于正常结果，DNS IP 不需要与代理出口 IP 相同。IPv6 测试页显示本地运营商
地址说明有流量绕过隧道。

## 参考上游：

- [LOWERTOP lazy_group.conf](https://lowertop.github.io/Shadowrocket/lazy_group.conf)
- [Johnshall Shadowrocket Rules](https://github.com/Johnshall/Shadowrocket-ADBlock-Rules-Forever)
- [Shadowrocket 社区手册](https://github.com/LOWERTOP/Shadowrocket)

更新上游时不要直接覆盖两个主配置。必须保留：DNS 仅由模块管理、IPv6 开关留在主配置、
所有 IP 型规则与 `RULE-SET` 使用 `no-resolve`、不引入公网 `server:system`/MITM、LAN 留在
主配置、公司路由留在本地模块、国内 `DIRECT` 域名使用国内 DoH、代理域名远端解析，以及
最后一条规则始终为 `FINAL,PROXY`。

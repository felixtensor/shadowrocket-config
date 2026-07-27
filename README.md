# Shadowrocket Config

适用于 macOS、iOS 和 iPadOS。`Main.conf` 基于 LOWERTOP / Johnshall 的懒人配置维护，
默认国内直连、国外和未知域名代理，并针对 DNS 泄露做了加固。

## 文件

| 文件 | 用途 |
| --- | --- |
| `Main.conf` | 策略组、国内外分流和 LAN 路由 |
| `NoDNSLeak.sgmodule` | DNS、代理 fallback、节点 bootstrap 和 53 端口劫持 |
| `CorpDirect.example.sgmodule` | 可选的公司网段与内网 DNS 模板 |

节点订阅在 Shadowrocket 首页单独管理，不要把订阅 URL、节点密码、UUID 或私钥提交到 Git。

## 安装

1. 在 `配置 > 右上角 + > 下载配置` 添加主配置：

   ```text
   https://raw.githubusercontent.com/felixtensor/shadowrocket-config/main/Main.conf
   ```

2. 在 `配置 > 模块 > 右上角 +` 添加并启用 DNS 模块：

   ```text
   https://raw.githubusercontent.com/felixtensor/shadowrocket-config/main/NoDNSLeak.sgmodule
   ```

3. 在首页添加自己的节点订阅。

4. 如需公司网络，复制 `CorpDirect.example.sgmodule` 为本地 `CorpDirect.sgmodule`，替换
   真实 CIDR 和域名后启用。该文件已被 `.gitignore` 排除。

5. 对 `Main.conf` 执行一次“使用配置”，断开并重新连接 Shadowrocket；全局路由选择
   “配置”。

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

不要启用其他会修改 `dns-server`、`fallback-dns-server`、`proxy-dns-server`、
`hijack-dns` 或 `[Host]` 的模块。

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
```

重新连接后测试：

- <https://www.dnsleaktest.com/>
- <https://browserleaks.com/dns>
- <https://ipleak.net/>

DNS 泄露测试生成的未知域名不应由本地运营商、AliDNS 或 DNSPod 解析。国内直连域名使用
AliDNS / DNSPod 属于预期行为；测试页显示代理出口侧的 Cloudflare、Google 或代理服务商
递归 DNS 也属于正常结果，DNS IP 不需要与代理出口 IP 相同。

## 维护约束

参考上游：

- [LOWERTOP lazy_group.conf](https://lowertop.github.io/Shadowrocket/lazy_group.conf)
- [Johnshall Shadowrocket Rules](https://github.com/Johnshall/Shadowrocket-ADBlock-Rules-Forever)
- [Shadowrocket 社区手册](https://github.com/LOWERTOP/Shadowrocket)

更新上游时不要直接覆盖 `Main.conf`。必须保留：DNS 仅由模块管理、所有 IP 型规则与
`RULE-SET` 使用 `no-resolve`、不引入公网 `server:system`/MITM、LAN 留在主配置、公司
路由留在本地模块、国内 `DIRECT` 域名使用国内 DoH、代理域名远端解析，以及最后一条规则
始终为 `FINAL,PROXY`。

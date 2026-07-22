# Shadowrocket Config

适用于 macOS、iOS 和 iPadOS 的 Shadowrocket 覆盖模块：局域网直连、国内域名使用国内
DoH，国外和未知域名由代理端解析。节点、订阅和主分流规则仍由主配置管理。

## 文件

| 文件 | 用途 |
| --- | --- |
| `LANDirect.sgmodule` | 私网直连及常见 LAN 旁路 |
| `NoDNSLeak.sgmodule` | DNS 分流、代理回退和 53 端口劫持 |
| `CorpDirect.example.sgmodule` | 公司网段与内网 DNS 模板 |

仓库不包含节点、订阅链接、凭据、私钥或证书。

## 安装

在 `配置 > 模块 > 右上角 +` 添加：

```text
https://raw.githubusercontent.com/felixtensor/shadowrocket-config/main/LANDirect.sgmodule
https://raw.githubusercontent.com/felixtensor/shadowrocket-config/main/NoDNSLeak.sgmodule
```

启用后对当前主配置执行一次“使用配置”，再断开并重新连接 Shadowrocket。

## Shadowrocket 设置

- `设置 > 代理 > 代理类型 > None`（TUN Only）。
- `设置 > 隧道 > 强制路由`：开启。
- `设置 > 隧道 > 包括所有网络`：开启。
- `设置 > 隧道 > 包括本地网络`：关闭。
- `设置 > 隧道 > 包括 APNs`：关闭。
- `设置 > 隧道 > 包括蜂窝服务`：关闭。
- `设置 > UDP > 启用转发`：节点支持 UDP 时开启。
- `设置 > UDP > 禁用 STUN`：开启。
- 全局路由：选择“配置”。

需要严格接管 APNs 或蜂窝系统服务时可分别开启；若推送、Wi-Fi Calling、MMS 或语音
信箱异常，应恢复关闭。

不要添加已弃用的 `bypass-system`。其他模块和主配置不要重复设置 `dns-server`、
`fallback-dns-server`、`proxy-dns-server` 或 `hijack-dns`。

## 主配置必须修改

模块不能修正主配置的规则顺序。主配置需要同时满足：

1. 中国域名规则在前并使用 `DIRECT`。
2. 所有 `IP-CIDR`、`IP-ASN`、`GEOIP` 和 IP 规则集使用 `no-resolve`。
3. 未知域名最终使用代理策略。

末尾可参考：

```text
# 已有可靠的中国域名集时，不要重复添加
DOMAIN-SET,https://raw.githubusercontent.com/blackmatrix7/ios_rule_script/master/rule/Shadowrocket/ChinaMax/ChinaMax_Domain.list,DIRECT

GEOIP,CN,DIRECT,no-resolve
FINAL,PROXY
```

若代理策略不叫 `PROXY`，替换为实际策略名。把现有 IP 类规则也改为：

```text
IP-CIDR,91.108.4.0/22,PROXY,no-resolve
RULE-SET,<中国 IP 规则集 URL>,DIRECT,no-resolve
```

`no-resolve` 使域名跳过 IP 规则，避免未知国外域名先经国内 DNS 解析、再落入代理，
这正是检测时同时出现国内 DNS 和代理出口 DNS 的常见原因。直接访问 IP 仍可匹配。

第三方远程主配置应先复制或 fork 后修改，否则更新订阅会覆盖改动。

## 公司内网

复制 `CorpDirect.example.sgmodule` 为 `CorpDirect.sgmodule`，替换真实 CIDR 和域名后在本地
使用；该文件已被 `.gitignore` 排除。

内网域名需要同时配置 `DOMAIN-SUFFIX,...,DIRECT` 和 `[Host]` 解析器。模板提供
`server:system` 与指定公司 DNS 两种写法，只选一种。

## DNS 行为

```text
国内域名     -> AliDNS DoH
国外/未知域名 -> 代理服务器远端解析
代理节点域名 -> AliDNS DoH（建立代理前）
DIRECT 回退  -> Cloudflare/Google DoH，经代理
局域网域名   -> system，仅限 .local、.lan、.home.arpa 和自定义公司后缀
```

浏览器或应用自带 DoH 不受 `hijack-dns` 拦截。建议关闭浏览器“安全 DNS”，或确保其 DoH
服务域名命中代理规则。

## 验证

在 `数据 > 代理 > DNS` 开启日志，并在 `配置 > 测试规则` 检查：

```text
www.baidu.com                     -> 中国域名规则 / DIRECT
www.google.com                    -> 国外域名规则 / PROXY
随机字符串.dns4.browserleaks.net -> FINAL / PROXY
```

重新测试后，不应出现本地运营商、AliDNS 或 DNSPod；出现代理出口地区的 Cloudflare、
Google 或代理服务端解析器属于正常结果。Cloudflare 显示多个 IPv4/IPv6 递归出口也正常。

- <https://www.dnsleaktest.com/>
- <https://browserleaks.com/dns>
- <https://ipleak.net/>

## 参考

- [Shadowrocket 社区使用手册](https://lowertop.github.io/Shadowrocket/)
- [Shadowrocket-First Wiki](https://github.com/LOWERTOP/Shadowrocket-First/wiki)
- [Shadowrocket-ADBlock-Rules-Forever](https://johnshall.github.io/Shadowrocket-ADBlock-Rules-Forever/)
- [Blackmatrix7 ChinaMax](https://github.com/blackmatrix7/ios_rule_script/tree/master/rule/Shadowrocket/ChinaMax)

本仓库不提供代理服务；上述 Shadowrocket 手册由社区维护，并非官方文档。

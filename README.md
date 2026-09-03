# Shadowrocket Config

适用于 macOS、iOS 和 iPadOS。基于 LOWERTOP / Johnshall 的懒人配置维护，默认国内直连、
国外和未知域名代理，并针对 DNS 泄露做了加固。

## 主配置

二选一，只启用一个。默认 `Main.conf`；IPv6 链路不稳时换 `Main_no_ipv6.conf`，典型症状是
微信一直显示连接中。

| 文件 | 说明 | 订阅链接 | 文档 |
| --- | --- | --- | --- |
| `Main.conf` | 启用 IPv6 | [link](https://raw.githubusercontent.com/felixtensor/shadowrocket-config/main/Main.conf) | [说明](docs/ipv6.md) |
| `Main_no_ipv6.conf` | 关闭 IPv6 | [link](https://raw.githubusercontent.com/felixtensor/shadowrocket-config/main/Main_no_ipv6.conf) | [说明](docs/ipv6.md) |

## 模块

| 模块 | 用途 | 必选 | 订阅链接 | 文档 |
| --- | --- | --- | --- | --- |
| `NoDNSLeak.sgmodule` | DNS、代理 fallback、节点 bootstrap、53 端口劫持 | 是 | [link](https://raw.githubusercontent.com/felixtensor/shadowrocket-config/main/modules/core/NoDNSLeak.sgmodule) | [说明](docs/nodnsleak.md) |
| `YouTube.sgmodule` | iOS / iPadOS 客户端去广告、拦遥测、画中画、后台播放 | 否 | [link](https://raw.githubusercontent.com/felixtensor/shadowrocket-config/main/modules/apps/YouTube.sgmodule) | [说明](docs/youtube.md) |
| `CorpDirect.example.sgmodule` | 公司网段与内网 DNS 模板 | 否 | 复制为本地文件 | [说明](docs/corpdirect.md) |

`modules/core/` 放改路由和 DNS 行为的模块，`modules/apps/` 放需要 HTTPS 解密的应用增强
模块。每个模块在 `docs/` 下有一份同名文档。

## 安装

1. `配置 > 右上角 + > 下载配置` 添加一份主配置。
2. `配置 > 模块 > 右上角 +` 添加并启用 `NoDNSLeak.sgmodule`，按需添加其他模块。
3. 首页添加自己的节点订阅。订阅 URL、节点密码、UUID、私钥都不要提交到 Git。
4. 对主配置执行一次“使用配置”，断开并重新连接；全局路由选择“配置”。

## 必须设置

- `设置 > 代理 > 代理类型 > None`（TUN Only）。
- `设置 > 隧道 > 强制路由`：开启。
- `设置 > 隧道 > 包括所有网络`：开启。
- `设置 > 隧道 > 包括本地网络`：关闭。
- `设置 > UDP > 启用转发`：节点支持 UDP 时开启。
- `设置 > UDP > 禁用 STUN`：开启。
- HTTPS 解密：默认关闭，仅启用 `YouTube.sgmodule` 时开启。
- `配置文件 ⓘ > 通用 > 启用IPv6`：跟随主配置，不要手动改，切换请换配置文件。

## 验证

`数据 > 代理 > DNS` 开启日志，`配置 > 测试规则` 检查：

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

DNS 泄露测试见 [NoDNSLeak 说明](docs/nodnsleak.md)。

## 参考

- [LOWERTOP](https://lowertop.github.io/Shadowrocket/lazy_group.conf)
- [Johnshall](https://github.com/Johnshall/Shadowrocket-ADBlock-Rules-Forever)
- [社区手册](https://github.com/LOWERTOP/Shadowrocket)

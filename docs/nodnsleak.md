# NoDNSLeak.sgmodule

DNS、代理 fallback、节点 bootstrap 和 53 端口劫持。两份主配置共用，必选。

## 解析路径

```text
DIRECT 域名             -> AliDNS / DNSPod DoH，直连
DIRECT DNS 失败回退      -> Cloudflare DoH，经代理
PROXY 域名               -> 代理服务器远端解析
代理节点域名             -> AliDNS / DNSPod DoH，直连 bootstrap
普通 UDP/TCP 53          -> Shadowrocket 全量劫持
局域网和公司内网域名     -> system 或指定内网 DNS
```

## 冲突

不要启用其他会修改这些项的模块：`dns-server`、`direct-dns-server`、`fallback-dns-server`、
`proxy-dns-server`、`hijack-dns`、`[Host]`、`ipv6`。

浏览器自带的 DoH/DoQ 不走 53 端口，劫持不到。关掉浏览器的“安全 DNS”，或确认它的 DNS 服务
域名走代理。

## 读测试结果

跑 [dnsleaktest](https://www.dnsleaktest.com/)、[browserleaks](https://browserleaks.com/dns)、
[ipleak](https://ipleak.net/)、[test-ipv6](https://test-ipv6.com/)。

| 看到 | 结论 |
| --- | --- |
| 测试生成的未知域名由本地运营商、AliDNS 或 DNSPod 解析 | 有泄露 |
| 国内直连域名用 AliDNS / DNSPod | 正常，设计如此 |
| 测试页显示代理出口侧的 Cloudflare、Google 或代理商递归 DNS | 正常，DNS IP 不需要和出口 IP 相同 |
| IPv6 测试页显示本地运营商地址 | 有流量绕过隧道 |

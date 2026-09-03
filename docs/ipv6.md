# IPv6

`Main.conf` 和 `Main_no_ipv6.conf` 的差异与已知限制。

## 两份主配置

只差两处：

- `[General]` 的 `ipv6`、`prefer-ipv6`、`ipv6-only-if-no-ipv4-dns`
- `Main.conf` 的 `tun-excluded-routes` 补齐了 `::1/128`、`fc00::/7`、`fe80::/10`、`ff00::/8`

分流规则相同，`Lan.list` 已含 IPv6 局域网段。

`ipv6` 只是 DNS 层开关，管的是查不查 AAAA，代理流量的出口 IP 族由节点决定。切换请换配置
文件，不要在 `配置文件 ⓘ > 通用 > 启用IPv6` 里手动改。

## 已知限制

- `[Host]` 里的 DoH bootstrap 固定为 IPv4，纯 IPv6 网络下要自己改成 IPv6 地址。
- 即使 `ipv6 = false`，节点域名能解析出 AAAA 时仍会走节点的 IPv6 地址。
- 内置 Country MMDB 是否含 IPv6 段未实测，不含时国内 IPv6 直连会落到 `FINAL,PROXY`。
- WebRTC 的 IPv6 泄露靠 `设置 > UDP > 禁用 STUN` 兜，这项必须开。

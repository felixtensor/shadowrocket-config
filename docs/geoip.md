# GeoIP

`GEOIP,CN` 依赖的 GeoLite2 数据库怎么配、怎么更新。

## 为什么要管

`GEOIP,CN` 是主配置里 `FINAL` 之前的最后一条规则，判断依据是设备上的 GeoLite2 数据库。
配置文件无法指定或锁定它，换设备要重新配。它带 `no-resolve`，所以只影响裸 IP 连接，域名
由前面的规则集决定。

内置库只随 App 版本走，不填数据源就没有东西可以更新，点「重置」也只是退回随包那一份。

## 配置

`设置 > GeoLite2 数据库` 填入：

```text
国家  https://github.com/P3TERX/GeoLite.mmdb/raw/download/GeoLite2-Country.mmdb
ASN   https://github.com/P3TERX/GeoLite.mmdb/raw/download/GeoLite2-ASN.mmdb
```

点「更新」，并打开「自动后台更新」，间隔 1-7 天。iOS / iPadOS 还需在
`系统设置 > 通用 > 后台App刷新` 里允许 Shadowrocket。后台任务在重启设备或杀掉 App 后不会
恢复，偶尔看一眼「更新」一行的时间戳。

[P3TERX](https://github.com/P3TERX/GeoLite.mmdb) 每日镜像 MaxMind 官方 GeoLite2。
2026-09-21 实测国家库构建于 2026-09-18，CN 的 IPv4 与 IPv6 均覆盖。

## 库太旧的症状

国内新划分的网段（尤其是 IPv6）裸 IP 直连会落到 `FINAL,PROXY`，表现为国内服务绕一圈代理。
`配置 > 测试规则` 里 `240e::1` 不判 `DIRECT` 就是这种情况。

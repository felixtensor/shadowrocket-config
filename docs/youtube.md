# YouTube.sgmodule

YouTube / YouTube Music 客户端去广告、拦遥测、画中画、后台播放。只对 iOS / iPadOS 生效，
可选。

## 前提

- 开启 HTTPS 解密并信任 CA 证书，范围见[解密范围](#解密范围)。不装证书就只能放弃站内去广告。
- 全局路由选「配置」，否则 URL Rewrite 的 REJECT 不执行，后台播放静默失效。

## 使用

后台播放的开关要手动开一次，脚本只负责把它注入设置页：`YouTube > 设置 > 后台播放`，在列表
最末尾。改完强制退出 YouTube 重开。

验证时换一个没看过的视频，客户端按 videoId 缓存 player 响应。

## 参数

`argument="{}"` 表示全用默认值，要改就写成 `{"captionLang":"zh-Hans","blockShorts":true}`。

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `captionLang` | `off` | 字幕翻译目标语言，填 Google Translate 语言代码 |
| `blockUpload` | `true` | 隐藏上传按钮 |
| `blockImmersive` | `true` | 隐藏选段按钮 |
| `blockShorts` | `false` | 隐藏 Shorts 按钮 |
| `debug` | `false` | 输出更多日志 |

## 解密范围

| 主机 | 用途 |
| --- | --- |
| `youtubei.googleapis.com` | 改写 player、设置页和列表接口 |
| `www.youtube.com`、`s.youtube.com` | 拦遥测和独立广告请求 |

视频域名 `*.googlevideo.com` 不解密。

## 限制

画中画和后台播放靠改写 `/youtubei/v1/player` 的 `playabilityStatus`，去广告靠删除同一份
响应里的 `adPlacements`、`adSlots`。客户端在部分区域把 player 响应塞进 initplayback 的
加密 onesie 流，脚本够不到，这些区域只剩部分去广告。

上游 [Maasea/sgmodule](https://github.com/Maasea/sgmodule) 的完整方案会解密
`*.googlevideo.com`，由 `youtube.request.js` 把 initplayback 重定向到作者的 Cloudflare
Worker 解 onesie。Worker 只承担 onesie 的加解密，TLS 仍在手机上终结，视频流量和 QUIC
回退的开销都在本机。

## 排查

| 症状 | 原因 |
| --- | --- |
| 广告、画中画、后台都不稳定 | 客户端走了 onesie，见[限制](#限制) |
| 画中画有了，后台没有 | 设置页开关没打开，或视频有缓存 |
| 全都没有 | HTTPS 解密没开，或全局路由不是「配置」 |

## 升级

脚本来自 [Maasea/sgmodule](https://github.com/Maasea/sgmodule)，固定在 commit
`65075cdb388fc5e3094afd7e7314c67b243f3525`，不跟随 `master`。YouTube 改协议时手动改模块里的
commit 号。

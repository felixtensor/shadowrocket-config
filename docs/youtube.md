# YouTube.sgmodule

去广告、拦截遥测、画中画、后台播放。只对 iOS / iPadOS 的 YouTube 和 YouTube Music 客户端
生效，可选。

## 前提

- 开启 HTTPS 解密并信任 CA 证书。这是仓库里唯一需要解密的模块。
- 全局路由选「配置」。其他模式下 URL Rewrite 的 REJECT 不执行，后台播放会静默失效。

## 使用

后台播放的开关要手动开一次，脚本只负责把它注入设置页：`YouTube > 设置 > 后台播放`，在列表
最末尾。改完模块强制退出 YouTube 重开。

验证时换一个没看过的视频，客户端按 videoId 缓存 player 响应。

## 排查

| 症状 | 原因 |
| --- | --- |
| 广告没了，画中画和后台都没有 | 该节点走了 onesie，检查 initplayback 那条 reject 是否命中 |
| 画中画有了，后台没有 | 设置页开关没打开，或视频有缓存 |
| 全都没有 | HTTPS 解密没开，或全局路由不是「配置」 |

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
| `*.googlevideo.com` | 打空 initplayback，以及两条去广告规则 |
| `www.youtube.com`、`s.youtube.com` | 拦遥测 |

## 实现

后台播放和画中画靠改写 `/youtubei/v1/player` 里的 `playabilityStatus`。新版客户端在部分
区域把 player 响应塞进 `initplayback` 的加密 onesie 流，脚本够不到，所以模块把 initplayback
打空，让客户端退回明文 `v1/player`。代价是起播多一次往返。

上游的做法是重定向到作者的 Cloudflare Worker 解 onesie，起播更快，但会把解密密钥和完整
URL 发给第三方。要换就删掉 initplayback 那条 reject，`[Script]` 整节替换成：

```text
youtube_response=type=http-response,pattern=^https://youtubei\.googleapis\.com/youtubei/v1/(?:browse|next|player|search|reel/reel_watch_sequence|guide|account/get_setting|get_watch|log_event|config)(?:[/?#]|$),requires-body=1,max-size=-1,binary-body-mode=1,script-path=https://raw.githubusercontent.com/Maasea/sgmodule/65075cdb388fc5e3094afd7e7314c67b243f3525/Script/Youtube/youtube.response.js,argument="{}"
youtube_request_init=type=http-request,pattern=^https?://[\w-]+\.googlevideo\.com/initplayback.+&ack.*,requires-body=1,max-size=-1,binary-body-mode=1,script-path=https://raw.githubusercontent.com/Maasea/sgmodule/65075cdb388fc5e3094afd7e7314c67b243f3525/Script/Youtube/youtube.request.js,argument="{}"
youtube_request_log=type=http-request,pattern=^https://youtubei\.googleapis\.com/youtubei/v1/log_event,requires-body=1,max-size=-1,binary-body-mode=1,script-path=https://raw.githubusercontent.com/Maasea/sgmodule/65075cdb388fc5e3094afd7e7314c67b243f3525/Script/Youtube/youtube.request.js
```

initplayback 是 POST，两家手册都没说 URL Rewrite 的 REJECT 对 POST 是否生效。真不命中就只
加上面的 `youtube_request_init` 一行，别给响应脚本加 `log_event|config`——密钥抓不到，脚本
就恒走返回空响应的分支，不碰 Worker。

## 升级

脚本来自 [Maasea/sgmodule](https://github.com/Maasea/sgmodule)，固定在 commit
`65075cdb388fc5e3094afd7e7314c67b243f3525`，不跟随 `master`。YouTube 改协议时手动改模块里的
commit 号。

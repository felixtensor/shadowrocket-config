# YouTube.sgmodule

去广告、拦截遥测、画中画、后台播放。只对 iOS / iPadOS 的 YouTube 和 YouTube Music 客户端
生效，可选。

## 前提

需要开启 HTTPS 解密并信任 CA 证书。这是仓库里唯一需要解密的模块，不用就别启用。

全局路由必须选「配置」。主配置里 `always-reject-url-rewrite = false`，Shadowrocket 在这个
设置下只有配置模式才执行 URL Rewrite 的 REJECT，切到其他模式后台播放会静默失效。

## 解密范围

| 主机 | 用途 |
| --- | --- |
| `youtubei.googleapis.com` | 响应脚本改写 player、设置页和列表接口 |
| `*.googlevideo.com` | 打空 initplayback，以及 `&ctier`、`&oad` 两条去广告规则 |
| `www.youtube.com`、`s.youtube.com` | 拦 `stats/ads`、`pagead`、`ptracking`、`qoe?adcontext` 遥测 |

`-redirector*.googlevideo.com` 排除重定向主机，它不承载上面任何一类请求。

后两个 host 上游没有，是本仓库为遥测拦截加的。不要为了缩小解密面删掉——遥测拦截是这个
模块的功能之一，和去广告相互独立：广告是响应脚本删 `adPlacements`、`adSlots` 去的。

## 原理

后台播放和画中画靠改写 `/youtubei/v1/player` 响应里的 `playabilityStatus`，脚本往里注入
`backgroundPlayerRender` 和 `pictureInPictureRender`。

新版客户端会把 player 响应塞进发往 `rr*.googlevideo.com/initplayback` 的加密 onesie 流，
明文接口不再被请求，脚本就改不到。症状是广告照去、后台播放和画中画失效。这套机制按区域
灰度，实测同一台设备换节点表现就不同（美区不生效、港区生效），但具体覆盖范围会变。

模块把 initplayback 打空，客户端退回明文 `v1/player`。上游脚本在密钥对不上时也是返回空
响应退回 `v1/player`，这里等价于让它无条件发生，代价是起播多一次往返。

## 使用

脚本只把开关注入设置页，不会替你打开。去 `YouTube > 设置 > 后台播放` 手动开一次，它在设置
列表最末尾。改完模块要强制退出 YouTube 重开，设置页才会重新拉取。

验证换一个没看过的视频。客户端按 videoId 缓存 player 响应，老视频重开 app 也不会重拉。

## 排查

| 症状 | 原因 |
| --- | --- |
| 广告没了，画中画和后台都没有 | player 响应没被改到，检查 initplayback 那条 reject 是否命中 |
| 画中画有了，后台没有 | 设置页开关没打开，或视频有缓存 |
| 全都没有 | HTTPS 解密没开，或 CA 证书没安装、未信任 |

## 参数

模块里的 `argument="{}"` 表示全用脚本默认值：

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `captionLang` | `off` | 字幕翻译目标语言，填 Google Translate 语言代码 |
| `blockUpload` | `true` | 隐藏上传按钮 |
| `blockImmersive` | `true` | 隐藏选段按钮 |
| `blockShorts` | `false` | 隐藏 Shorts 按钮 |
| `debug` | `false` | 输出更多日志 |

改的话写成 `{"captionLang":"zh-Hans","blockShorts":true}` 这种形式。

## 备选方案

上游原版不打空 initplayback，而是重定向到作者的 Cloudflare Worker 去解 onesie。

| | 模块默认 | 上游原版 |
| --- | --- | --- |
| 做法 | reject initplayback，退回明文 `v1/player` | 重定向到 Worker 解 onesie |
| 起播 | 多一次往返 | 保留 onesie 优化 |
| 依赖 | 无 | `init-stream.maasea.workers.dev` |
| 风险 | 依赖 URL Rewrite 对 POST 生效，见下 | 解密密钥和完整 URL 发给第三方，服务挂掉起播失败 |

要换：删掉 `[URL Rewrite]` 里 initplayback 那条，`[Script]` 整节替换成

```text
youtube_response=type=http-response,pattern=^https://youtubei\.googleapis\.com/youtubei/v1/(?:browse|next|player|search|reel/reel_watch_sequence|guide|account/get_setting|get_watch|log_event|config)(?:[/?#]|$),requires-body=1,max-size=-1,binary-body-mode=1,script-path=https://raw.githubusercontent.com/Maasea/sgmodule/65075cdb388fc5e3094afd7e7314c67b243f3525/Script/Youtube/youtube.response.js,argument="{}"
youtube_request_init=type=http-request,pattern=^https?://[\w-]+\.googlevideo\.com/initplayback.+&ack.*,requires-body=1,max-size=-1,binary-body-mode=1,script-path=https://raw.githubusercontent.com/Maasea/sgmodule/65075cdb388fc5e3094afd7e7314c67b243f3525/Script/Youtube/youtube.request.js,argument="{}"
youtube_request_log=type=http-request,pattern=^https://youtubei\.googleapis\.com/youtubei/v1/log_event,requires-body=1,max-size=-1,binary-body-mode=1,script-path=https://raw.githubusercontent.com/Maasea/sgmodule/65075cdb388fc5e3094afd7e7314c67b243f3525/Script/Youtube/youtube.request.js
```

响应脚本的 pattern 多了 `log_event|config`，onesie 密钥就在这两个路由里抓。

### URL Rewrite 打不空 initplayback 时

initplayback 是带 protobuf body 的 POST，Surge 和 Shadowrocket 都没有文档说明 URL Rewrite
的 REJECT 对 POST 是否生效。如果发现那条 reject 不命中，改用上游的请求脚本，但**不要**给
响应脚本加 `log_event|config`：

```text
youtube_request_init=type=http-request,pattern=^https?://[\w-]+\.googlevideo\.com/initplayback.+&ack.*,requires-body=1,max-size=-1,binary-body-mode=1,script-path=https://raw.githubusercontent.com/Maasea/sgmodule/65075cdb388fc5e3094afd7e7314c67b243f3525/Script/Youtube/youtube.request.js,argument="{}"
```

密钥只在 `log_event` 和 `config` 两个路由里抓，不加就永远抓不到，脚本每次都走返回空响应
的分支，不会碰 Worker。效果和默认方案一样，代价是多一次脚本执行。

## 升级

脚本来自 [Maasea/sgmodule](https://github.com/Maasea/sgmodule)，按 commit 固定在
`65075cdb388fc5e3094afd7e7314c67b243f3525`，不跟随 `master`。

这个脚本对 YouTube 全流量有 MITM 权限，固定住上游改动才不会无声生效。代价是 YouTube 改
协议时要手动改模块里的 commit 号。

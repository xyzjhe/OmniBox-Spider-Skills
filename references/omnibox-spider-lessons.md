# OmniBox Spider Lessons

## 近期高价值经验

### 1. 先确认是上游挂了，还是脚本坏了

- 不要一看到分类空、搜索空、推荐空就直接改脚本；先实测上游接口状态码和返回体。
- 如果分类、推荐、搜索接口同时 500/502，优先判断为上游异常，而不是 spider 逻辑错误。
- 给用户或群友反馈时要明确说清：是“上游挂了”，还是“脚本适配失效”。

### 2. 分类/详情优先信 live DOM 与页面内状态

- 先确认页面上是否真实有内容；如果前端页面有卡片、有章节，而 spider 返回空，优先排查自己的解析逻辑。
- 对 Nuxt/SSR 站点，优先找：
  - `__NUXT_DATA__`
  - 页面内 JSON
  - 真实 `a[href*='/albums/']`
  - `ul.chapter-list > li.chapter-item`
- 能靠 live DOM + 页面状态拿到数据时，优先走这条路，通常比硬怼私有加密接口更稳。

### 3. 听书站播放链要拆层

不要默认裸播放页（如 `/playing`）就是最佳 sniff 目标；它常常只是播放器壳。

更常见的可靠链路是：
1. `detail()` 拿 `albumId + chapterIdx`
2. `play()` 调真实接口（如 `/api/play_token`）拿直链
3. 拿不到再 fallback 到带章节上下文的页面（如 `/audios/{albumId}/{chapterIdx}`）去 sniff

对“集合壳页面”类站点，`detail()` 应先拿真实详情页/真实章节上下文，`play()` 再取直链，不能混成一层。

### 4. 页面真实播放页已知时，优先交给 SDK 嗅探

- 已知真实播放页 URL 时，优先 `OmniBox.sniffVideo(realPlayPageUrl)`，比自己手撸 Playwright 点击链更轻更稳。
- 但要注意：`sniffVideo` 也可能受运行环境 bug 影响，不能盲目把它当唯一方案；能直解接口就优先直解。

### 5. 遇到加密 API，先去前端 bundle 抄真逻辑

先从页面引用的 `_nuxt/*.js` 或前端 bundle 里搜：
- `payloadKey`
- `payloadVersion`
- `encryptRequest`
- `decryptResponse`
- `xchacha`
- `poly1305`

真实站点逻辑可能是：
- 请求 v1、响应 v2
- 响应体需要特殊处理，例如先 `reverse(cipher)` 再解

先抄前端真逻辑，再把 spider 里的解包对齐，效率远高于盲试 nonce/AAD 切片。

### 6. tingyou.fm 这次确认的关键点

- 分类与详情可以稳定依赖页面结构和 `__NUXT_DATA__`
- 播放主链应走 `/api/play_token`
- `/audios/{albumId}/{chapterIdx}` 更适合做 sniff fallback，而不是裸 `/playing`
- `play_token` 的 v2 响应需要按前端真实逻辑解包：
  - `nonce = bytes[1..24]`
  - `cipher = bytes[25..]`
  - 当 `version === 2` 时，先 `reverse(cipher)` 再解密

### 7. 交付默认最小补丁

- 用户或群友贴来原始脚本时，先固化原版基线，再做最小补丁。
- 不要随意改域名、头注释、依赖、站点配置块、线路顺序，除非问题本身要求。
- 验收前要核对 diff，防止“嘴上说最小改动，实际变成重写稿”。

### 8. 交付顺序：先文件，后说明

- 在群里交付爬虫修复时，优先直接发文件；说明尽量短，别长篇铺垫。
- 文件与说明不要分散太开：要么同一条一起发，要么先发文件再回复说明。

### 9. 日志策略：核心常驻，诊断可开关

常驻日志保留：
- `detail.out`
- `play.req`
- `play.out source=api/page`
- 关键错误

高频诊断日志如：
- `api.raw`
- `api.payload.meta`
- `decrypt.variant.fail`

应做成环境变量开关（例如 `TINGYOU_VERBOSE_API=1`），避免正常使用时刷屏。

### 10. 验收别只看语法

- `node --check` 只能验语法，不能保证运行时逻辑对。
- 关键链路仍要实测：分类、详情、播放至少各走一遍。

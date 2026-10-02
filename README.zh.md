# dsh-web-search-ddg

English | [中文](README.zh.md)

为 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)（DSH）web 能力 seam（`ctx.web`）提供的零 token 网络搜索提供方。

DSH 自带的搜索路由（`deepseek-official`）把每次 `web_search` 执行为一次 **`deepseek-v4-flash` 上的完整计费模型请求**——即使你会话模型选的是别的。本插件用两个零成本引擎按序替换它，先成功者胜出：

1. **Bing**——纯 fetch 请求 Bing 的 HTML 端点（带 cookie 引导）。不需要浏览器，健康时亚秒级响应。
2. **DuckDuckGo**——以 headless 模式驱动本地 Chrome/Edge/Chromium 抓取 DuckDuckGo 的 HTML 端点，解析 dump 出的 DOM。

- 每次搜索**零模型 token**——不需要 API key，不产生辅助模型请求
- **零依赖**——只用 Node 内置模块；不下载 Playwright/Puppeteer
- **引擎回退**——一个引擎被封锁或改版时，另一个接棒；只有全部失败才报错
- **保留自带提供方的注册**——切回只需一行配置，不用卸载

## 环境要求

- 提供 `ctx.web` seam 的 DSH 宿主（≥ `0.1.0-rc`）
- DuckDuckGo 引擎需要本地 Chromium 系浏览器：macOS（Chrome、Edge、Chromium）与 Linux（`/usr/bin/chromium`、`/usr/bin/google-chrome`）自动探测，可用 [`chromePath`](#配置) 覆盖。Bing 引擎无需浏览器。
- Node.js ≥ 18（可用时使用 `AbortSignal.any` 实现 fetch 超时）

## 安装

在你的 DSH profile 目录（如 `~/.dsh/profiles/web`）中，作为树外插件安装：

```bash
pnpm add dsh-web-search-ddg
```

然后在 profile 的 `package.json` 中把它声明为 profile bundle，让 DSH 以一等层加载（插件管理器可见、可干净卸载）：

```json
// profile package.json → dsh.profile.bundles
"bundles": [..., "dsh-web-search-ddg"]
```

接着在 profile 的 `cordis.patch.yml` 中选择提供方：

```yaml
# 为面向模型的 web_search 工具选择本提供方。
- id: web
  config:
    searchProvider: ddg-browser
```

> 注意：patch 行会**整体替换目标行的 config**（无深度合并），因此 `web` 行必须重述其拥有的全部键——自带行只拥有 `searchProvider`。`web-search-ddg` 自身的 `insert` 来自包的 `dsh.bundle` 补丁层，无需手写。

自带的 `web-search-deepseek` 行保持不动：其提供方仍注册且可用，切回只需一行（`searchProvider: deepseek-official`）。DSH 的选择是单一显式 ID，**不是**优先级链——设计上就没有静默回退。

不启动宿主即可验证组合后的配置树：

```bash
dsh --profile web --dump-config | grep -A2 searchProvider
```

重启宿主后生效（host 侧插件行不热更新）。

## 配置

所有键均可选；配置写在上面 `insert` 条目里。

| 键 | 默认值 | 说明 |
| --- | --- | --- |
| `engines` | `["bing", "duckduckgo"]` | 引擎执行顺序。支持 `"bing"`、`"duckduckgo"`，先成功者胜出 |
| `chromePath` | 第一个探测到的浏览器 | Chromium 系可执行文件绝对路径（仅 DuckDuckGo 引擎使用） |
| `timeoutMs` | `20000` | DuckDuckGo 引擎单次尝试的超时预算。超时时已缓冲的 DOM 输出仍算成功（Chrome 的 `--dump-dom` 进程打印后经常不退出）。每次搜索共两次尝试；保持 `2 × timeoutMs` 小于 `tool-web` 的 `searchTimeoutMs`（DSH 默认 60s） |
| `virtualTimeBudgetMs` | `8000` | Chrome 的 `--virtual-time-budget`——dump DOM 前页面允许的沉寂时长 |

## 行为说明

- **延迟**：Bing 引擎通常亚秒级返回；DuckDuckGo 引擎约 10–20 秒（DOM 很快就绪，但 Chrome 经常不自行退出，结果常在超时兜底时拿到——缓冲输出仍会被采纳）。
- **Bing 的结果质量依赖 cookie。** 没有 cookie 会话时，Bing 会返回忽略大部分查询词的降级结果。插件每个会话先访问一次 Bing 首页并复用其 cookie；结果为空时会重新引导一次。
- **搜索引擎会对激进 IP 限流。** 同一台机器高频自动化搜索可能同时被所有引擎掐连接或发挑战页。此时插件会报聚合错误、逐个列出各引擎的失败原因——可切换网络或等待标记解除（数小时），或将 `searchProvider` 切回 `deepseek-official`。绝不静默降级。
- **headless UA 被覆盖**为普通桌面 Chrome UA——DuckDuckGo 识别 `HeadlessChrome` 标记并据此拦截。
- 每次浏览器尝试都在 OS 临时目录下的一次性 `--user-data-dir` 中运行，进程结束后尽力清理。

## 工作原理

**Bing 引擎**——带桌面 Chrome 头和引导 cookie 的 `fetch` 请求结果页；解析 `<li class="b_algo">` 块提取标题与摘要；把结果链接（`/ck/a?…&u=a1<base64url>`）解包成真实目标 URL。

**DuckDuckGo 引擎**——对 `https://html.duckduckgo.com/html/?q=<query>` 以 headless 启动浏览器（`--headless --dump-dom --virtual-time-budget`，UA 覆盖）；带 kill 兜底地读取序列化 DOM（超时时已缓冲输出仍算成功）；解析 `<a class="result__a">` 标题与 `<a class="result__snippet">` 摘要，从跳转链接的 `uddg` 参数解码真实 URL 并按其配对。

两个引擎都通过 seam 返回 `{ sources: [{ url, title?, snippet? }], truncated: false }`——面向模型的 `web_search` 工具与结果卡片无需任何改动。

## 故障排查

`ddg-browser: all search engines failed (…)` 会逐个列出每个引擎的最后错误：

| 消息片段 | 含义 | 处理 |
| --- | --- | --- |
| `connection reset / rate-limited` | 你的 IP 被该引擎临时标记 | 切换网络（如手机热点）或等待数小时；标记会自动解除 |
| `anomaly/challenge page` | DuckDuckGo 反爬挑战页 | 同上；另检查 UA 字符串是否仍匹配当前 Chrome 版本 |
| `no parsable results/links` | 引擎页面结构变更 | 更新本插件，或提 issue |
| `browser did not finish within Nms` | 浏览器挂起未产出 DOM | 调大 `timeoutMs`；确认浏览器本身能启动 |

## 开发

```bash
npm test          # 端到端：注册到 stub ctx.web 并跑一次真实搜索
CHROME_PATH=/path/to/browser npm test
TEST_QUERY="换个查询" npm test
```

## 许可证

[MIT](LICENSE)

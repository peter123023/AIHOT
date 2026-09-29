# 技术细节

这份文档是逐行读代码整理出来的，回答的是「它到底是怎么跑起来的」这类问题，和
[架构](docs/architecture.md) 那份概览互补：概览讲结构，这份讲机制和取舍。

> **关于时效**：文中出现的第三方单价（极致了、模型调用量）是代码注释里记录的编写时数值，
> 服务商随时可能调价，以你的账号实际结算为准。版本号以 `package.json` 为准。

---

## 一、三个进程

| 进程 | 位置 | 端口 | 职责 |
|---|---|---|---|
| api | `apps/api/` | 3001 | Fastify。网页自用接口、公开 API、RSS、MCP、后台接口、图片代理、分享图、外部推送 |
| worker | `apps/worker/` | — | pg-boss 队列消费 + cron 定时任务。**唯一调模型、唯一抓信源的地方** |
| web | `apps/web/` | 3000 | React Router SSR 网页。只通过 HTTP 读 api，不碰数据库 |

一句话概括分工：**worker 生产内容，api 提供内容，web 展示内容**。三者通过同一个 PostgreSQL 通信。

数据库不是第四个进程，但它是真正的中心：业务数据在 `public` schema，任务队列在 `pgboss` schema
（pg-boss 把队列建在数据库里）。所以 **PostgreSQL 是硬依赖，MySQL 用不了**——全仓库 0 处 MySQL，
驱动是 `postgres.js`（`packages/backend/src/db.ts:1`），SQL 里大量使用 `jsonb`、`ON CONFLICT`、
`pg_advisory_xact_lock`、`make_interval(mins => ...)`、`FILTER (WHERE ...)`、`real[]` 数组。

### Docker 里的容器

`docker-compose.yml` 共 5 个容器（启用 `--profile https` 时 6 个）：

```
db (PostgreSQL 17) ──▶ setup (迁移+种子数据，跑完退出) ──▶ api ──▶ web ──▶ caddy (可选)
                                                        └─▶ worker
```

启动顺序是链式的：`db` healthy → `setup` 成功退出 → `api`/`worker` → `web` → `caddy`。

一个容易忽略的细节：

```58:60:docker-compose.yml
    # On stop the worker lets a paid call in flight finish (STOP_TIMEOUT_MS, 195 s); Docker's default
    # 10 s would kill it mid-call and leave "outcome unknown" receipts that are paid for again.
    stop_grace_period: 210s
```

worker 停止时要等在途的付费调用跑完。裸跑用 systemd 时**必须自己写 `TimeoutStopSec=210`**，
否则「钱付了、结果丢了、重跑再付一次」。

---

## 二、主流程：一条资料的六步

```
信源 ──▶ 采集（判重·抓正文）──▶ 判断与写作（预筛·两次评分·写作·结构化）
      ──▶ 发布投影 ──▶ 归组（事件·综述）──▶ 热度 ──▶ 日报/周报/月报
                                                  └──▶ 公开读取层 publication/ ──▶ 网页·RSS·API·MCP·llms
```

### 1. 采集（`packages/backend/src/sources/`）

六种信源类型（`sources/types.ts:6`，SQL CHECK 在 `database/migrations/0001_core.sql:13`）：

| kind | 读取器 | 需要什么 |
|---|---|---|
| `rss` | `sources/rss.ts` | 无（支持 ETag/Last-Modified → 304） |
| `web_list` | `sources/web-list.ts` | 写 CSS 选择器；抓不到时可经 Jina 渲染（付费） |
| `json_list` | `sources/json-list.ts` | 写字段路径 |
| `x_search` | `sources/x.ts` | SocialData 的 key（付费） |
| `mp_account` | `sources/mp.ts` | 极致了（Dajiala）的 key（付费） |
| `external` | 无读取器 | `INGEST_TOKEN`，只接收外部推送 |

**抓取频率自适应**（`sources/collect.ts` 的 `adaptIntervals()`）：按近 7 天该源产出算
`24*60/(perDay*3)`，再 clamp 到区间——`hot_signal` 最长 180 分钟；`x_search` 或走 Jina 的付费列表
120 分钟；其余 60 分钟；最短 15 分钟（付费源 60 分钟）。失败则指数退避并把 `health` 降级。

### 2. 入库与判重（`packages/backend/src/content/materials.ts`）

`upsertMaterial()` 是唯一入口：

- **身份键**：X 帖 `x:<tweetId>`；否则规范化 URL（去 hash、去 www、统一 https、去跟踪参数、
  微信只留 `__biz/mid/idx/sn`）；兜底 `src:<sourceId>:<sha256(url+title)>`
- **判重**：`INSERT ... ON CONFLICT (identity_key) DO NOTHING`
- **版本**：内容哈希（title|body|excerpt）变了才 `revision+1`，历史版本重现不算新
- **时间线**：`decideTimeline()` 把「发现时已发布超过 48 小时」或显式 backfill 的资料标记为历史，
  按原文时间归档，**不进「今天」、不进热度、不建事件**

相关表：`articles`、`article_revisions`、`article_discoveries`、`fetch_runs`。

### 3. 判断与写作（`packages/backend/src/editorial/analyze.ts`）

四步各一份提示词，都来自 `industry/prompts/`：

| 步骤 | 做什么 | 调用次数 |
|---|---|---|
| `prefilter` | 是否属于本行业（宽召回，只有 BLOCK 才停） | 1 |
| `score` | **同一份标准独立打两次分**（0–100） | 0 或 2 |
| `writing` | 中文标题、答案先行的摘要、推荐理由、标签 | 1 或 0 |
| `structure` | 分类、标签、主体、事实框（读者看不到） | 1 |

关键规则（`analyze.ts:46-54`、`normalizeAnalysis()`）：

- 入选条件：`score1 + score2 >= 2 × 门槛`，门槛按信源分级（`industry/selection.ts`：T1 60、T1_5 65、T2 76）
- 卡片上显示的分数是两次平均后**向下取整**，它从不单独决定半分
- 没入选但均分 > `understandFloor`（50）的，也用精选的写法（贵的 `understand`）
- 其余走便宜的 `summarize`；中文短帖直接原文照登，**0 次调用**
- `structure` 与评分**并行**跑，不阻塞

### 4. 发布投影（`packages/backend/src/publication/publish.ts`）

把 material + 最新 `analyses` + 人工覆盖 + 事件归属，投影成一行 `publications`
（visibility / eligible / selected / body_mode / indexable / search_text / story_id…），
同步 `pool_search` 和 `selected_ledger`。**纯重投影，不调模型。**

### 5. 归组与热度（`packages/backend/src/events/`）

**归组**（`group.ts`）队列是**串行**的（`localConcurrency: 1`），因为同一新事实的两篇报道不能各建一个事件。

1. 人工归属优先（`fact_articles.manual` 或 `grouping_overrides`）→ 直接短路，不调模型
2. 历史资料不建事件；非 editorial 源只挂故事、绝不新建
3. 已有自动归属则沿用（修订不重跑）
4. 向量召回：14 天窗口、余弦 ≥ 0.6、每个事实取最相似的一篇、Top 10；无 embedding key 时降级为字面 bigram
5. 模型**三分类**判断：`SAME_OCCURRENCE / SAME_STORY / UNRELATED / ROUNDUP`
6. 余弦 ≥ 0.85 直接合并；否则**换一家模型复核**（`groupReview`）才合并
7. 后续进展只挂在「故事根事实」上，不会链式生长

**人工归属永不覆盖**：所有证据性读取都过 `trusted()` 过滤（未决报道不作为他人证据），
写事务内在 article 行锁下**二次确认** manual。

**热度**（`hot.ts`）：规则 `heat-v1-48h-halflife24h`。

- 唯一输入是 `story_signals`，按**信源时间**（不是采集时间）算
- 48 小时窗口，**每个独立参与者只算一次**，`0.5^(Δh/24)` 衰减
- 对照 6 小时前：新参与者 ≥3 且 ≥50% 是新的 → 爆；`at - firstReportAt < 6h` → 新；6h 增幅 >15% → 上升
- 至少 2 个参与者且含 editorial 才进榜，Top 10 写 `hot_rankings`

重复抓取不会多算，一家媒体发十篇也只算一次。

### 6. 日报 / 周报 / 月报（`packages/backend/src/reports/compose.ts`）

- 日报窗口 = 北京时区 `[D-1 08:00, D 08:00)`
- 每个 fact 只留一条（first_party 优先，其次按分数）
- 按 `CATEGORIES[].section` 分节，每节 ≤8 条，溢出进「快讯」（≤12 条）
- 最近 7 期已出现的 fact 不重复
- 模型写导语 + highlights；周/月报取 top 40/60 走 `report-period` 提示词

---

## 三、前端：SSR + SPA 的混合

### 技术栈

不是 Next.js，是 **React Router 8.4.0**（框架模式，即原 Remix 的后继）+ React 19 + Vite 8 + Tailwind 4。

```11:20:apps/web/package.json
  "dependencies": {
    "@aihot/contracts": "*",
    "@aihot/industry": "*",
    "@react-router/node": "8.4.0",
    "isbot": "5.2.2",
    "motion": "13.4.4",
    "react": "19.3.0",
    "react-dom": "19.3.0",
    "react-router": "8.4.0"
```

而且它没有用框架自带服务器，是自己写的 Node HTTP 服务器
（`apps/web/server.ts`，核心是 `createRequestListener({ build, mode: "production" })`）——
因为要在 SSR 之前接管重定向表、静态资源、api 路径转发，并改写缓存头。

### 一次访问里两个进程各干什么

```
浏览器 ──GET /all──▶ web (3000)
                      │ ① loader 执行：fetch http://api:3001/api/site/pool
                      │ ② React 在服务端渲染成 HTML 字符串
                      │ ③ loaderData 序列化进 HTML（供水合用）
浏览器 ◀── 完整 HTML ──┘
              └─ 后续 /api/*、/feed.xml、/og/*、图片 → web 转发给 api
```

看 `apps/web/app/routes/all.tsx:14-30` 的 loader 就清楚了：`await loadOr404<PoolResponse>(...)` 里的
`await` 在**服务端**完成。

### `.data` 请求是什么

水合之后点击站内链接，React Router 发的是 `/all.data?_routes=…` 请求，返回 JSON，不再重新请求 HTML。
这就是你在 Network 里看到的那些请求。

**关键点**：`.data` 打到的是 **web 自己**，web 在服务端**再跑一遍 loader** 去问 api。
浏览器始终不知道 api 的地址，`API_BASE_URL` 只在服务端环境变量里。

`.data` 还会被**预取**触发——代码里到处是 `prefetch="intent"`，悬停或聚焦就提前拉数据。
`components/ui/IntentLink.tsx` 更讲究：悬停 100ms 才预取，手指滑动经过卡片时取消。

```
4:5:apps/web/app/components/ui/IntentLink.tsx
/** Hover, keyboard focus and a stationary touch prefetch; scrolling over a card does not. */
```

所以准确说法是：**首屏 SSR，水合之后变 SPA**。

### api 从不推数据给 web

通信严格单向（web 拉 api）：

- worker 算出新内容只**写数据库**，web 下次请求才读到，没有通知
- 后台「运行记录」页面的实时刷新是浏览器每 20 秒轮询（`routes/admin/runs.tsx:48-52`）
- `root.tsx:39` 直接 `shouldRevalidate: () => false`，自动重新验证是关掉的
- 缓存失效也不靠 purge：`releaseBoundCache()` 算出「下一个待发布内容的时刻」写成
  `X-Accel-Expires: @<时间戳>`，缓存到点自然失效

唯一的反向调用是 api 拉 web 渲染一次分享图（`media/prepare.ts:42-51`），best effort，失败只返回 false。

### 样式

Tailwind CSS 4.3.3 已经在用，通过 `@tailwindcss/vite` 接入，`app.css:1` 一行 `@import "tailwindcss"`。
v4 是 CSS-first，没有 `tailwind.config.js`，令牌写在 `@theme` 块里。

注意暗色模式不是 Tailwind 默认的 `.dark` 类，而是属性选择器：

```9:9:apps/web/app/app.css
@custom-variant dark (&:where([data-theme="dark"], [data-theme="dark"] *));
```

项目有自己的语义色体系（`--color-ink`、`--color-line`、`--color-accent`、`--color-surface`…）
和一套 `components/ui/`（11 个组件）。**不建议引入 shadcn/ui**：它的 `--background`/`--foreground`
命名体系跟这里不兼容，还会引入 Radix、CVA、tailwind-merge 等一批依赖，并打乱
`vite.config.ts:48-65` 手工调过的分包策略（注释里写了实测目标「每页 7–10 个 JS 文件」）。
需要个别复杂组件时，单独复制源码进来改类名即可。

---

## 四、SEO

**混合渲染不伤害 SEO。** 爬虫不点链接，它从 sitemap 或直接外链拿到 URL 后**直接 GET**，
而每个 URL 被直接请求时返回的都是完整 SSR HTML。对爬虫来说每个页面都是「首屏」。

项目配套的 SEO 设施：

- **sitemap.xml**（`publication/sitemap.ts`）：首页、`/all`、`/hot`、所有日报周报月报、所有主题页
  含分页、最近 500 个事件页、所有 `indexable` 的文章页、模型榜页
- **robots.txt**（`apps/api/src/routes/static.ts:56-68`）：屏蔽 `/api/`、`/admin/`，
  但**主动放行 `/api/v1/` 和 `/api/mcp`**——给 AI 爬虫留门
- **`pageMeta()`**（`lib/seo.ts:48-75`）：canonical、OG、Twitter card、JSON-LD（Organization、BreadcrumbList）
- canonical 会剔除 `utm_*` 等追踪参数（`listPath()`）；搜索结果页自动 `noindex`（`routes/all.tsx:40`）
- **IndexNow**：每天把新页面提交给 Bing（`seo.indexnow` 定时任务，默认关）
- **`/llms.txt`**：给大模型的站点说明
- **事件合并是 308 重定向**（`story_aliases`），旧链接权重不丢
- `<Link>` 渲染成标准 `<a href>`，爬虫不执行 JS 也能抓到全部链接

唯一可考虑加固的：robots.txt 目前没有屏蔽 `*.data`。如果发现 Search Console 里出现 `/all.data`
这类 URL，在 `robotsTxt()` 里加一行 `Disallow: /*.data$` 即可（实际上站内没有指向它的 `<a>`，
sitemap 里也没有，爬虫基本碰不到）。

---

## 五、付费与成本

### 采集服务（按次计费，默认不启用）

都不填 key 就完全不调用，站点照常跑。

| 服务 | 环境变量 | 默认熔断（分/时/天） | 用途 |
|---|---|---|---|
| `socialdata` | `SOCIALDATA_API_KEY` | 10 / 100 / 1000 | X 账号抓取；Codex 重置监控也靠它读 X |
| `dajiala` | `DAJIALA_KEY` | 5 / 60 / 500 | 公众号文章列表 + 正文 |
| `jina` | `JINA_API_KEY` | 5 / 50 / 300 | 抓不到正文时的渲染兜底（`web_list` 的 markdown 模式、正文兜底） |

（默认值见 `database/migrations/0022_receipt_attempts_budgets.sql:41-47`，后台「设置 → 预算」可改，**填 0 立即停用**。）

### 模型调用（主要成本，不可选）

熔断按服务商分：`llm` 300/6000/40000，`embedding` 300/6000/40000，
`zhipu`/`deepseek`/`dashscope`/`mimo` 各 100/2000/20000（`0036_open_source_defaults.sql`）。

量级参考（`docs/deploy.md:75`）：用示范信源本地试跑，首次导入 152 条资料约 930 次模型调用，
平均约 6 次/条。这个数包含重复导入、修订重跑、归组判断、事件综述，**日均会低于这个比例**。
精确数字看后台「模型与评测」页，按步骤统计调用次数和 token。

### 公众号：固定订阅，不是全网

**一个公众号 = 一条信源记录**，config 只认三个键（`sources/config-keys.ts:22`）：

```json
{ "ghid": "gh_xxxxxxxx", "nickname": "公众号名称" }
```

极致了只提供两个接口（`providers/dajiala.ts`）：`post_history`（按 ghid 拉该号列表）、
`article_detail`（按 URL 取正文）。**没有搜索、发现、推荐接口**，所以不存在「全网公众号」这回事。
调度端 `scheduleMpReconcile()` 只扫 `kind='mp_account' AND enabled` 的行。

单价（`providers/dajiala.ts:1-3`，以服务商实际结算为准）：

| 接口 | 单价 | 何时调用 |
|---|---|---|
| `post_history` | ¥0.14 / 次 | 每个账号每 `interval_minutes` 一次；同一 10 分钟窗口只付一次 |
| `article_detail` | ¥0.03 / 次 | 每篇**新**文章一次 |

**列表调用是主要成本，正文几乎可忽略。** 所以省钱的杠杆是调大 `interval_minutes`，而不是限制文章数。
每次检查最多处理 8 篇新文章；首次检查时 7 天以前的文章跳过。

一个缺口：`mp_account` **配不了 `ingestNoiseFilter`**（关键词前置过滤只对 `rss`/`web_list`/
`json_list`/`x_search` 生效，且它跑在 `collect.ts` 的通用路径上，公众号走独立的 `mp.ts`）。
所以公众号进来的文章一律要付一次预筛调用。高产但命中率低的号，可改用 RSS 出口（能配关键词过滤、
采集免费），或设成 `hot_signal`（不调模型）/ `EXCLUDE_MP`（不评分）。

### 付费护栏（`providers/receipts.ts`）

1. 逻辑请求有稳定键：任务 + 输入 revision + provider + model + prompt + 配置 + attemptTag
2. 调用前落占位行和 attempt 行，预算按 attempt 计数
3. **原始响应先落库再做业务写入**，恢复时复用已付过钱的结果
4. 结果不明的请求 caller 不重发，由 `ops.recover` 30 分钟后自动释放一次——最多多花一次

---

## 六、部署

### Docker 还是裸跑

**推荐 Docker**，尤其是服务器上已经有别的服务时——它能把 PostgreSQL 17 和 Node 24 与现有环境隔离。

| | Docker | 裸跑 |
|---|---|---|
| PostgreSQL 17 | 容器里，不动现有 MySQL | 要自己装、自己备份升级 |
| Node 24 | 打进镜像 | 要 nvm 另装；现有 pm2 可能还绑着旧 node |
| 停服安全 | compose 已设 `stop_grace_period: 210s` | 必须自己写 `TimeoutStopSec=210` |
| 更新 | `git pull` + `up -d --build` | 还要手动 `npm run build -w @aihot/web` |

两个硬前提：**Node ≥ 24.11**（`package.json` engines）、**必须 PostgreSQL**（见第一节）。

### 已有 MySQL + Node 的服务器

MySQL 用不上，但**可以和 PostgreSQL 共存**（5432 vs 3306）。要点：

- 用 Docker：`PORT=127.0.0.1:3002` 避免端口冲突；数据库在容器里
- 别启用 caddy（会抢 80/443，跟现有 nginx 打架），让现有 nginx 反代：

```nginx
location / {
    proxy_pass http://127.0.0.1:3002;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

`X-Forwarded-For` 别漏——`.env` 里设 `TRUST_PROXY=true` 后访客 IP 从这个头读，
影响后台登录次数限制和反馈限流。`SITE_URL` 必须写读者实际访问的地址（链接、RSS、分享图、MCP 都用它）。

上机前确认：`docker -v`、剩余内存（建议 ≥2G 空闲）、`ss -tlnp | grep -E '3000|3001|5432'`。

### Caddy 还是 Nginx

**用你已有的那个。** 性能差别在这个量级测不出来（瓶颈在后端，不在反代），纯粹是运维习惯问题。
Caddy 的优势是自动 HTTPS（申请+续期全自动），适合干净的机器从零搭建；你已经有 Web 服务器时，
引入 Caddy 要么抢端口要么要做二级代理，多一跳更绕。

### 多机拆分（web 在一台，api+worker 在另一台）

**技术上可行且被官方支持**：

```106:107:packages/contracts/src/http-policy.ts
 * Paths served by the api process; everything else is the web process. The web server proxies these to
 * the api (a reverse proxy in front may also route them straight to it).
```

web 会把所有 api 路径代理过去（`server.ts:134-149`），所以**只有 web 需要对外**。
配置：A 机 `API_BASE_URL=http://B内网:3001`；B 机 `LOCAL_ROUTER_URL=http://A内网:3000`（分享图预热，可省）。
A 机需要一个只含 web 的精简 compose（原版 `web depends_on api`，在没有 api 的机器上起不来）。

**但先算延迟账**：每页 SSR 有 1–2 次 web→api 调用（`api.server.ts:2`）。

| 两机之间 | RTT | 影响 |
|---|---|---|
| 同机房 / VPC | 0.3–2 ms | 可以接受 |
| 跨地域 / 公网 | 20–60 ms | 页面中位数从 ~10ms 变成 100ms+，**明显变慢** |

另外图片走 `/api/img-proxy`，字节流是「外网 → B → A → 浏览器」，在链路上跑两遍。

**如果目标是页面更快，更有效的办法是在 web 前加 nginx `proxy_cache`**——代码已经为共享缓存
准备好了 `s-maxage` 和 `X-Accel-Expires`，命中时根本不打 api，这是数量级的提升。
真正该拆分的理由是资源隔离和安全（worker 突发负载不影响 SSR、数据库不对外），而不是提速。

### 扩容

api 和 web 都无状态，可以多开实例。**worker 只能有一个**——pg-boss 的定时任务多实例会重复触发，
且归组队列刻意设了 `localConcurrency: 1`。

---

## 七、几条不变的规则

（整理自 `docs/architecture.md`，改动时不要破坏）

- **一个公开读取层**：网页、RSS、API、MCP、sitemap、llms、分享图全从 `packages/backend/src/publication/` 读
- **页面不调模型**：读者请求只读已落库结果；模型只在 worker 任务里
- **付费必有回执 + 预算熔断**
- **安全阀**：`COLLECT_ENABLED`、`MODEL_CALLS_ENABLED`、`FEISHU_*_ENABLED`、`INDEXNOW_SUBMIT_ENABLED`
  只决定「发不发出去」，不切换逻辑分支
- **失败可恢复**：`afterFailure()` 区分「等重试」和「永久失败」；`sweepUnprocessed()` 每 5 分钟兜底重排孤儿文章
- **来源可追溯**：每条精选链原文；默认只显示摘要，`site_fulltext` 决定能否展示全文
- **公开内容匿名**：管理员和访客看到的一样；读者数据留浏览器
- **旧文不刷屏**：发现时已超 48 小时的资料按原文时间归档
- **数据库迁移只做向后兼容的增量**

---

## 八、常见误解澄清

| 说法 | 实际情况 |
|---|---|
| 「前端是 Next.js」 | 不是，是 React Router 8（原 Remix 后继）+ 自建 Node 服务器 |
| 「React Router 不能用 Tailwind」 | 能用，项目已经在用 Tailwind 4 |
| 「api 会推数据给 web」 | 不会，严格单向，web 拉 api |
| 「SSR 之后浏览器还要去读 api」 | 不会，首屏数据在 HTML 里；`.data` 是水合后站内跳转/预取用的 |
| 「混合渲染影响 SEO」 | 不影响，爬虫直接 GET 每个 URL，拿到的都是完整 HTML |
| 「MySQL 也能跑」 | 不能，pg-boss 和大量 PG 专有 SQL 决定了必须 PostgreSQL |
| 「公众号是全网抓取」 | 不是，是显式白名单，一个号一条信源记录 |
| 「worker 可以多开加速」 | 不能，定时任务会重复触发；要加速调 `ANALYZE_CONCURRENCY` |
| 「拆到两台机器会更快」 | 通常更慢，除非在同一内网；提速应该用 nginx 缓存 |

# 下一步：`git clone https://github.com/Open-Dev-Society/OpenStock && cd OpenStock && pnpm install`，配 `.env` 里的 `MONGODB_URI` + `NEXT_PUBLIC_FINNHUB_API_KEY`，然后 `pnpm dev`。

> 快照日期：2026-09-21 | 来源：GitHub Trending daily（since=daily）| 当日仅 13 个项目（周一 UTC 午夜刚重置），非完整 20 个
> 抓取时间：2026-09-21 | Stars：17,047 | 当日新增：755 | 主语言：TypeScript
> 项目类别：B 类（代码架构 / Next.js 全栈金融应用）
> 仓库地址：https://github.com/Open-Dev-Society/OpenStock

**TL;DR：** OpenStock 是一个"贵的付费行情平台"的开源替代品——Next.js 15 + React 19 + Tailwind v4 + shadcn/ui，接 Finnhub 行情 + TradingView 图表 + Better Auth + MongoDB + Inngest 自动化（AI 欢迎邮件、每日新闻摘要）。AGPL-3.0，社区驱动，主打"知识不该被 paywall 锁死"的 Open Dev Society 宣言。

---

## 1. 场景问题：该项目主要解决什么问题

- [ ] **场景 A：学生/独立开发者想做自己的股票监控但不想付 Bloomberg/TradingView 费用**
  - 目标用户：金融爱好者 / 学习全栈的开发者
  - 痛点：付费行情平台贵、自托管方案要么是 Python 脚本要么是老 Rails app
  - 介入方式：Next.js 15 App Router 全栈模板，Finnhub 免费 tier 数据，TradingView 嵌入 widget
  - 效果：clone 下来改改 env 就能跑一个带 watchlist + 告警 + 新闻的行情站

- [ ] **场景 B：想给已有 app 加"个性化邮件 + AI 摘要"能力**
  - 目标用户：独立 SaaS 开发者
  - 痛点：自己写 cron + 邮件模板 + LLM 调用要一周
  - 介入方式：Inngest 已经配好 `app/user.created → AI 欢迎邮件` 和 `0 12 * * * → 每日新闻摘要` 两条 workflow
  - 效果：抄走 `lib/inngest/functions.ts` + `lib/nodemailer/` 就是一套完整模板

- [ ] **场景 C：学习 Next.js 15 + React 19 最新栈**
  - 目标用户：前端学习者
  - 痛点：官方 todo 示例学不到 App Router + Server Actions + Middleware 实战
  - 介入方式：真实业务——auth、watchlist、stock detail、cron job 全配齐
  - 效果：读 `app/(auth)/` + `lib/actions/` + `middleware/index.ts` 就是一套参考实现

---

## 2. 组成结构与技术组件

### 2.1 目录结构（来自 `git/trees/main`）

```
app/
  (auth)/          # sign-in / sign-up / forgot-password / reset-password 路由组
  (root)/          # 根布局 + stocks/[symbol] + watchlist + about + help + terms
  api/inngest/route.ts   # Inngest webhook 入口
components/
  ui/              # shadcn/ui 原语（button/dialog/command/input/select/popover/...）
  forms/           # InputField / SelectField / CountrySelectField / PasswordRequirements
  stocks/          # StockSentimentCard
  watchlist/       # WatchlistManager / AlertsPanel / CreateAlertModal / NewsGrid
  Header/Footer/SearchCommand/TradingViewWidget/UserDropdown...
database/
  models/watchlist.model.ts + alert.model.ts
  mongoose.ts
lib/
  actions/         # auth/finnhub/user/watchlist/alert/adanos Server Actions
  better-auth/auth.ts
  inngest/         # client + functions + prompts
  nodemailer/      # transporter + templates + reset-password
  ai-provider.ts   # gemini/minimax/siray 多 provider 抽象
hooks/             # useDebounce / useTradingViewWidget
scripts/           # 运维脚本（check-env / test-db / migrate-users / seed-inactive-user）
__tests__/         # Vitest 单测 + 集成测
```

### 2.2 技术栈（从 README + package.json 提取）

- **前端**：Next.js 15 App Router、React 19、TypeScript、Tailwind CSS v4（via @tailwindcss/postcss）、shadcn/ui + Radix UI、cmdk command palette、next-themes、lucide-react
- **后端/数据**：Better Auth（email/password + MongoDB adapter）、MongoDB + Mongoose、Next.js Server Actions、middleware 路由保护
- **外部 API**：Finnhub（行情/新闻/搜索）、TradingView 嵌入 widget、Adanos（可选情绪分析：Reddit/X/新闻/Polymarket）
- **自动化/AI**：Inngest（cron + workflow + Gemini 推理）、Nodemailer（Gmail transport）、AI provider 抽象（gemini/minimax/siray）
- **测试/工具**：Vitest、ESLint、Docker Compose（app + mongo 两服务）

### 2.3 核心数据流

```
用户登录:
  sign-up → Better Auth 写 MongoDB
         → Inngest `app/user.created` 事件
         → Gemini 生成个性化欢迎邮件 → Nodemailer 发送
日常:
  Cmd+K → SearchCommand → Finnhub 搜索
  → 加 watchlist (Server Action 写 MongoDB)
  → 股票详情页嵌 TradingView widget + Adanos 情绪卡
每日 12:00:
  Inngest cron → 读用户 watchlist → 个性化新闻摘要邮件
告警:
  CreateAlertModal → alert.model 存 MongoDB
  → Inngest 定时检查 → 触发邮件
```

---

## 3. 前 10 结构性优越点

### 优越点 1：全栈 Next.js 15 最新栈一次配齐
- 结构依据：README Tech Stack 段 "Next.js 15 (App Router), React 19, Tailwind CSS v4, shadcn/ui, Better Auth, MongoDB, Inngest"
- 为什么优越：不是 todo 级 demo，而是真实业务的 auth/watchlist/alert/cron/邮件全链路
- 对比维度：很多 Next.js 模板要么只做前端、要么 auth 用老 NextAuth，版本落后

### 优越点 2：Server Actions 组织清晰
- 结构依据：`lib/actions/` 下按域拆 `auth.actions.ts / finnhub.actions.ts / user.actions.ts / watchlist.actions.ts / alert.actions.ts`
- 为什么优越：action 按业务域分文件，不堆在一个 `actions.ts` 里；helpers 单独抽 `adanos.helpers.ts`
- 对比维度：很多项目把所有 server action 塞在 `app/actions.ts`，越写越乱

### 优越点 3：AI provider 抽象层
- 结构依据：`lib/ai-provider.ts` + env 注释 "Supported: gemini, minimax, siray"
- 为什么优越：欢迎邮件用 Gemini，但可一键切 MiniMax 或 Siray，不绑死单一 LLM 厂商
- 对比维度：多数 demo 直接 import `@google/genai`，换 provider 要改业务代码

### 优越点 4：Inngest 把 cron + workflow + LLM 调用合体
- 结构依据：`lib/inngest/{client,functions,prompts}.ts` + README "Workflows: app/user.created → AI Welcome Email; Cron 0 12 * * * → Daily News Summary"
- 为什么优越：不用自己写 node-cron + queue + retry，Inngest 自带幂等、重试、本地 dev server
- 对比维度：自己写 cron 经常漏失败重试和并发控制

### 优越点 5：Docker Compose 一键起全栈
- 结构依据：`docker-compose.yml` 含 openstock + mongodb 两服务，README 给完整 yaml 示例
- 为什么优越：新人 clone 不用先装本地 MongoDB，`docker compose up -d` 就跑
- 对比维度：很多全栈模板的 Docker 是后来补的，缺 healthcheck

### 优越点 6：watchlist + alert 数据模型分离
- 结构依据：`database/models/watchlist.model.ts` + `alert.model.ts` 两个独立 model
- 为什么优越：watchlist 是"我在看"，alert 是"到价通知"，语义不同分表，后续扩展不耦合
- 对比维度：很多项目把告警当 watchlist 字段，加类型时要迁移

### 优越点 7：Better Auth 而非 NextAuth
- 结构依据：README "Better Auth (email/password) with MongoDB adapter" + `lib/better-auth/auth.ts`
- 为什么优越：Better Auth 类型安全、MongoDB adapter 原生、比 NextAuth v4 现代
- 对比维度：NextAuth v5 仍在 beta，生态碎片化

### 优越点 8：middleware 做路由保护
- 结构依据：`middleware/index.ts` + README "Protected routes enforced via Next.js middleware"
- 为什么优越：不在每个 layout 里手写 `redirect`，集中在 middleware 一层
- 对比维度：很多项目在每个 page 里重复 `if (!session) redirect`

### 优越点 9：环境变量分两套模板
- 结构依据：README Environment Variables 段同时给 Atlas 远程 + Docker 本地两份 env 示例
- 为什么优越：新人不用猜"我该用哪个 URI"，直接抄对应段
- 对比维度：多数项目只给一份 env，本地开发要自己改

### 优越点 10：AGPL-3.0 强 copyleft 防闭源 SaaS 分叉
- 结构依据：README "AGPL-3.0; if you modify, redistribute, or deploy it (including as a web service), you must release your source"
- 为什么优越：防止大厂 fork 后包成付费 SaaS 锁用户，符合 Open Dev Society "知识不锁"的宣言
- 对比维度：很多开源金融项目用 MIT，被云厂商 fork 后闭源

---

## 4. 前 10 优化增强点

### 优化点 1：Finnhub 免费 tier 数据延迟
- 当前状态：README 自承 "Real-time data for non-US stocks is delayed by 15+ minutes on free tier"
- 优化方向：加 Polygon / Alpha Vantage 作为备选 provider，按 region 自动路由
- 预期收益：非美用户不用忍受延迟

### 优化点 2：邮件用 Gmail App Password 不安全
- 当前状态：README 建议用 Gmail App Password，生产不推荐
- 优化方向：默认切 Resend / Postmark，Gmail 仅作 dev fallback
- 预期收益：生产部署更稳，也避免 Gmail 封发件

### 优化点 3：测试覆盖率薄
- 当前状态：`__tests__/` 只有 5 个 test 文件（actions/ai-provider/reset-password/utils）
- 优化方向：给 watchlist/alert Server Actions 加集成测，覆盖未授权访问
- 预期收益：重构时有安全网

### 优化点 4：情绪分析依赖第三方 Adanos
- 当前状态：`ADANOS_API_KEY` 是可选，但 StockSentimentCard 直接调
- 优化方向：Adanos 不可用时前端 fallback 隐藏卡片，不要报错
- 预期收益：没 key 的用户不看到 broken UI

### 优化点 5：缺 Redis 缓存层
- 当前状态：每次搜索都打 Finnhub，没有缓存
- 优化方向：加 Upstash Redis 缓存 Finnhub 响应，5 分钟 TTL
- 预期收益：减少 API 配额消耗，响应更快

### 优化点 6：没有 e2e 测试
- 当前状态：只有 Vitest 单测，没有 Playwright/Cypress
- 优化方向：加 3 条 e2e（登录→加 watchlist→看详情）
- 预期收益：防止 UI 改动回归

### 优化点 7：TradingView 嵌入受自由 tier 限制
- 当前状态：README 明说 TradingView free tier 对 NSE/越南等新兴市场有限制
- 优化方向：在 MARKET_SUPPORT.md 加"哪些 symbol 只能用 TradingView"的明确清单
- 预期收益：用户不会看到空白图表才知道不支持

### 优化点 8：scripts/ 下脚本是 .mjs/.js 混着
- 当前状态：`test-db.mjs` / `check_db_name.js` / `test-db.ts` 同时存在
- 优化方向：统一到 .ts + tsx 运行，删 .mjs 旧版
- 预期收益：减少"我该跑哪个"的困惑

### 优化点 9：没接 OpenTelemetry / 错误监控
- 当前状态：README 没提 Sentry / Axiom
- 优化方向：加 Sentry SDK + Inngest function 失败告警
- 预期收益：生产出错能收到通知

### 优化点 10：缺移动端/响应式细节
- 当前状态：README 没提 PWA / mobile first
- 优化方向：加 manifest.json + 手势缩放，把 watchlist 做成 PWA
- 预期收益：手机上看盘体验提升

---

## 5. ProcessOn 全景图信息

- 文件夹名称：OpenStock
- 图表标题：Open-Dev-Society/OpenStock 结构性全景图 v5
- 图表链接：https://www.processon.com/view/link/6ab0f067c8280d5a5b22f21e （v5：12节点/11连线/4分组，techblue，auto_layout，连线路由避让：主链路solid蓝#2563eb/辅助dashed/反馈dot/跨层broken折线）
- 旧v4链接：https://www.processon.com/view/link/6ab0f067c8280d5a5b22f21e
- 图中应包含：Next.js App Router 路由组、Server Actions 层、MongoDB 数据模型、Inngest 自动化 workflow、外部 API（Finnhub/TradingView/Adanos）、Better Auth

## 6. 幕布文档信息

- 文档名称：OpenStock — 日榜研究
- 文档 ID：7BjnKoOEl2c
- 包含内容：场景问题 + 前10优越点 + 前10优化点 + ProcessOn 链接

---

## 2 分钟动作

- [ ] clone 后先跑 `pnpm test:db` 验证 MongoDB 连接（README 第 370 行）
- [ ] 打开 `lib/inngest/functions.ts` 看一眼两条 workflow 怎么写的，这是最值得抄的部分
- [ ] 如果你想学习 Next.js 15，从 `app/(root)/stocks/[symbol]/page.tsx` + `lib/actions/finnhub.actions.ts` 开始读

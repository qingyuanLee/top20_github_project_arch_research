# 下一步：`npm i @json-render/core @json-render/react`，30 行 defineCatalog + defineRegistry 跑通第一个 AI 生成 Dashboard。

> 数据说明：快照日期 2026-09-21；来源 GitHub Trending daily（since=daily）；当日仅 13 个项目（周一 UTC 午夜刚重置），非完整 20 个。
> 抓取时间：2026-09-21 | Stars：17,562 | 当日 +291 | 排名：#9 | 主语言：TypeScript
> 项目类别：B类（代码架构）
> 仓库地址：https://github.com/vercel-labs/json-render

TL;DR：Vercel Labs 出的 Generative UI 框架——AI 只能在你定义的 catalog 里挑组件出 JSON，你用任意渲染器（React/Vue/Svelte/Solid/RN/Next/PDF/Email/Ink/Three.js）安全渲染。

## 1. 场景问题：解决什么

- [ ] **场景 A — 企业 AI 客服 dashboard**：痛点是让 LLM 直接写 HTML 会出 XSS 和不可控样式。介入：defineCatalog 锁组件 + Zod schema 校验 props，AI 只能出合法 JSON。效果：流式渲染、永不崩溃。
- [ ] **场景 B — 多端一套 UI 定义**：痛点是 Web/iOS/邮件/PDF 各写一遍。介入：同一份 spec JSON，React/RN/react-pdf/react-email 各自渲染。效果：跨端只维护一份 catalog。
- [ ] **场景 C — AI Agent 输出结构化动作**：痛点是 agent 调用工具后界面怎么更新。介入：catalog 里 actions（export_report/refresh_data）+ watchers/setState 联动。效果：agent 输出即界面变化。

## 2. 组成结构与技术组件

### 2.1 目录结构（monorepo 实测）
- [ ] `packages/core/` — schema、catalog、AI prompt、SpecStream、动态 props 表达式
- [ ] `packages/react/` `vue/` `svelte/` `solid/` — 四套 Web 渲染器
- [ ] `packages/shadcn/` `shadcn-svelte/` — 36 个预制 shadcn/ui 组件
- [ ] `packages/react-native/` `react-three-fiber/` — 移动端与 3D（含 GaussianSplat）
- [ ] `packages/next/` `tanstack-start/` — 整页/路由/SSR/metadata 全 app
- [ ] `packages/remotion/` `react-pdf/` `react-email/` `image/` `ink/` — 视频/文档/邮件/OG 图/终端
- [ ] `packages/redux/` `zustand/` `jotai/` `xstate/` — 4 种状态库适配器
- [ ] `packages/directives/` — `$format/$math/$concat/$truncate/$pluralize/$t` 内置指令
- [ ] `packages/mcp/` — 对接 Claude/ChatGPT/Cursor/VS Code
- [ ] `packages/yaml/` `codegen/` `devtools*/` — YAML 线格式、代码导出、跨框架 devtools
- [ ] `apps/web/` — Next.js 文档站 + playground（docs/api/* 自动生成）

### 2.2 技术栈
- [ ] **语言**：TypeScript，pnpm workspace monorepo（README `pnpm install && pnpm dev`）
- [ ] **Schema 校验**：Zod（`defineCatalog(schema, { components: { Metric: { props: z.object(...) } } })`）
- [ ] **UI 底座**：Radix UI + Tailwind（shadcn），Satori（image），Remotion（video），Three.js/R3F（3D）
- [ ] **流式**：SpecStream 编译器（`createSpecStreamCompiler`，chunk-by-chunk patch）
- [ ] **表达式**：`$state/$cond/$template/$computed/$bindState` 声明式数据绑定
- [ ] **License**：Apache-2.0

### 2.3 核心数据流
- [ ] 开发者 defineCatalog（组件 props schema + actions）
- [ ] `catalog.prompt()` 自动生成 system prompt 喂给 LLM
- [ ] LLM 流式吐 JSON Spec → SpecStreamCompiler 增量 patch
- [ ] Renderer 按 registry 把 JSON 树渲染成真实组件
- [ ] 用户交互 → emit action → setState/watchers → 重算 visibility/dynamic props → UI 更新

## 3. 前 10 优越点

### 优越点 1：Catalog + Zod 双护栏
- 结构依据：README "Define Your Catalog" 代码（`defineCatalog(schema, { Card: { props: z.object(...) } })`）
- 为什么优越：AI 输出不仅受组件名白名单约束，props 还被 Zod 校验，非法字段运行时拒绝。
- 对比：Vercel AI SDK 的 generative UI 只靠 prompt 约束。

### 优越点 2：一份 Spec 跨 12+ 渲染端
- 结构依据：README Packages 表（react/vue/svelte/solid/react-native/next/tanstack-start/remotion/react-pdf/react-email/image/ink/react-three-fiber）
- 为什么优越：同一 JSON 在 Web、移动、PDF、邮件、视频、终端、3D 都能渲染，复用度极高。
- 对比：Adaptive Cards 只覆盖 Web/Teams。

### 优越点 3：SpecStream 渐进式渲染
- 结构依据：README "Streaming (SpecStream)" + `createSpecStreamCompiler().push(chunk)`
- 为什么优越：不等 LLM 输出完整 JSON 就先画骨架，用户体感秒开。
- 对比：普通 JSON 解析要等完整响应。

### 优越点 4：声明式动态 props 表达式
- 结构依据：README "Dynamic Props"（`$state/$cond/$template/$computed`）
- 为什么优越：UI 树本身能响应状态变化，不用写命令式 JS。
- 对比：手写 React 要 useMemo/useEffect。

### 优越点 5：watchers 副作用模型
- 结构依据：README "State Watchers"（Select 变化触发 loadCities）
- 为什么优越：表单级联、数据加载都在 spec 里声明，AI 也能生成。
- 对比：要手写 onChange 逻辑。

### 优越点 6：36 个 shadcn/ui 开箱
- 结构依据：README "@json-render/shadcn — 36 pre-built shadcn/ui components (Radix UI + Tailwind)"
- 为什么优越：不丑、可定制、跟 shadcn 生态同步。
- 对比：自绘组件库设计拉胯。

### 优越点 7：状态库适配器全家桶
- 结构依据：packages/redux、zustand、jotai、xstate
- 为什么优越：无论项目用哪个状态库，StateStore 都能接，不用改业务代码。
- 对比：多数 generative UI 框架自带私有 store。

### 优越点 8：devtools 跨框架
- 结构依据：packages/devtools + devtools-{react,vue,svelte,solid} + `Ctrl/Cmd+Shift+J`
- 为什么优越：spec 树/state editor/action log/stream log/DOM picker 一板通用。
- 对比：调试 AI UI 只能 console.log。

### 优越点 9：MCP Apps 集成
- 结构依据：packages/mcp "MCP Apps integration for Claude, ChatGPT, Cursor, VS Code"
- 为什么优越：JSON UI 直接当 MCP 工具返回值，agent 生态即插即用。
- 对比：需要自写 MCP server。

### 优越点 10：Vercel 背书 + Apache-2.0
- 结构依据：README Vercel Labs badge + LICENSE Apache-2.0
- 为什么优越：可商用、无 copyleft 风险，Vercel 自身 Next.js 生态保证持续维护。
- 对比：很多 AI UI 框架是个人项目。

## 4. 前 10 优化增强点

### 优化点 1：包数量爆炸
- 当前状态：packages/ 下 33 个子包
- 优化方向：把 renderer adapter 合并到 `@json-render/{framework}` 单包多入口
- 预期收益：减少依赖树体积与版本对齐负担。

### 优化点 2：Jev 实验性
- 当前状态：README 标 "Experimental Jev composition"
- 优化方向：把 Jev 从实验转正，或明确废弃
- 预期收益：避免用户踩半成品。

### 优化点 3：SSR/Next App spec 模型还不成熟
- 当前状态：`createNextApp` 只能生成 routes/layouts
- 优化方向：支持 server actions / RSC 直接进 spec
- 预期收益：AI 能生成全栈应用而非仅 UI。

### 优化点 4：安全沙箱未内建
- 当前状态：actions 由开发者自己实现
- 优化方向：内置 action allowlist + 权限分级 + 审计
- 预期收益：企业接 LLM 生成 UI 时不被误调用。

### 优化点 5：性能在大 spec 下未压测
- 当前状态：README 无 benchmark
- 优化方向：出 1000 节点 spec 的渲染 perf 表
- 预期收益：大屏 dashboard 场景可放心用。

### 优化点 6：i18n 只靠 $t directive
- 当前状态：`$t` 是内置 directive
- 优化方向：内置 ICU message format + 复数/性别
- 预期收益：国际化业务直接用。

### 优化点 7：错误恢复弱
- 当前状态：流式 JSON 一旦中途损坏无法重试
- 优化方向：SpecStream 支持错误分支 + 局部 patch 重试
- 预期收益：长 AI 输出不崩页。

### 优化点 8：样式系统绑定 Tailwind
- 当前状态：shadcn 套件依赖 Tailwind
- 优化方向：提供 CSS 变量/DSS 无关主题包
- 预期收益：不用 Tailwind 的项目也能用。

### 优化点 9：移动端组件覆盖薄
- 当前状态：react-native 只列 25+ 标准组件
- 优化方向：加手势/导航/键盘处理组件
- 预期收益：RN 生产可用。

### 优化点 10：文档站与代码版本漂移风险
- 当前状态：apps/web 文档随代码发版，但 playground 用本地 localhost
- 优化方向：把 playground 部署到公开 sandbox，自动从 CDN 拉最新版
- 预期收益：新用户 10 秒在线试。

## 5. ProcessOn 全景图信息

- 文件夹名称：json-render
- 图表标题：vercel-labs/json-render 结构性全景图
- 图中应包含：顶层（Generative UI 框架：护栏化 AI 出 UI）；中层（defineCatalog/Zod schema / SpecStream 流式编译 / Renderer 调度 / StateStore+watchers）；底层（core 包、12+ 渲染器（React/Vue/Svelte/Solid/RN/Next/PDF/Email/Video/Ink/3D/Image）、状态适配器（Redux/Zustand/Jotai/XState）、MCP、Zod、Radix+Tailwind）。
- 图表链接：https://www.processon.com/view/link/6ab0d0cbcb92f406f7f3654b

## 6. 幕布文档信息

- 文档名称：json-render — 日榜研究
- 文档 ID：1U-W3koQmyc
- 包含内容：场景问题 + 前10优越点 + 前10优化点 + ProcessOn 链接

---

2 分钟动作：`git clone` 后 `pnpm install && pnpm dev`，开 http://json-render.localhost:1355 进 playground 输入"一个显示 LTV 和 Churn 的双卡片 dashboard"，看 AI 直接出 spec。

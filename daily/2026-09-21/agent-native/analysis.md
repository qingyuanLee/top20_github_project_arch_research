<!-- 数据说明：快照日期 2026-09-21，来源 GitHub Trending daily（since=daily）。当日仅 13 个项目（周一 UTC 午夜刚重置），非完整 20 个。本项目排名 #12。 -->

# BuilderIO/agent-native — 架构研究分析

> 抓取时间：2026-09-21 | Stars：5,411 | 排名：#12 | 主语言：TypeScript
> 项目类别：B 类（代码架构 — BuilderIO 开源的 agentic app 全栈框架）
> 仓库地址：https://github.com/BuilderIO/agent-native

**下一步可执行：** 跑 `npx --yes @agent-native/core@latest create my-agent --standalone --template chat` 10 秒起一个 chat 模板，然后打开 `actions/hello.ts` 改第一个 action。

**TL;DR：** BuilderIO 出的 TypeScript 框架，核心主张是 **"一个 action 同时是 agent 的 tool 和 UI 的函数"**——用 `defineAction({schema, http, run})` 定义一次，React 用 `useActionQuery` 调它、agent 把它当 tool 调它、HTTP / MCP / A2A / CLI 全部自动暴露。自带 chat UI、auth、PostgreSQL/PGlite、automations、agent teams、9 个官方模板 app。

---

## 1. 场景问题：该项目主要解决什么问题

### 场景 1：独立开发者想做"有 UI 的 agent"而不是"聊天框"
- **目标用户：** 全栈/前端开发者，想做一个能自主干活但又有完整界面的 AI app
- **痛点：** 现在做 agent 要么是纯文本聊天（LangChain/LlamaIndex demo），要么是把 agent 塞进现有 UI 让它"点按钮"——两者都割裂
- **如何介入：** 框架的核心抽象是 **shared actions**：agent 调 action 当 tool，React 调 action 当函数；同一份 validation / permission / implementation
- **效果：** 写一个 action，UI 和 agent 自动对齐，不会出现"UI 上点了 agent 不知道"

### 场景 2：企业想把内部工具做成 agent 可调用
- **目标用户：** 平台工程师、内部工具团队
- **痛点：** 已经有 REST API + React 后台，想加一个"AI 助手"能调这些 API；通常要重新写一遍 tool 定义
- **如何介入：** 把每个业务操作写成 `defineAction`，框架自动暴露 HTTP + MCP + A2A；agent 直接用
- **效果：** 不需要为 agent 单独维护一套 tool schema

### 场景 3：团队要做多 agent 协作产品
- **目标用户：** 做"agent 团队"产品的创业公司
- **痛点：** 多 agent 编排（delegate、handoff）通常要自己搭 orchestrator、消息总线、状态同步
- **如何介入：** 框架内置 agent teams（同 workspace 内 specialist agent 或跨 connected agents）、shared application state、checkpoints/resume
- **效果：** 不用从零搭 orchestration 层

### 场景 4：需要本地开发 + 生产 PostgreSQL 切换
- **目标用户：** 想本地零依赖起、上生产用 PG 的团队
- **痛点：** 本地装 PG 麻烦；生产用 SQLite 不保险
- **如何介入：** 开发期 PGlite（嵌入式 WASM Postgres），生产切 PostgreSQL；同一个 Nitro 兼容 host
- **效果：** `pnpm dev` 零配置，生产改环境变量就切

---

## 2. 组成结构与技术组件

### 2.1 目录/内容结构（基于实际 `gh api contents/` 输出）

```
agent-native/
├── packages/                 # pnpm monorepo
│   ├── core/                 # 框架核心（defineAction / agent engine / run-loop）
│   │   └── src/
│   │       ├── action.ts             # defineAction 主抽象
│   │       ├── agent/               # agent engine / run-loop / resume
│   │       ├── a2a/                 # A2A 协议
│   │       ├── automation/          # schedules / events
│   │       ├── authorization/       # auth & permissions
│   │       ├── chat-threads/
│   │       ├── checkpoints/         # run resume
│   │       └── ... (50+ 模块)
│   ├── agentkit/             # agent 运行时
│   ├── dispatch/             # 多 agent 调度
│   ├── code-agents-ui/       # 内置 chat UI
│   ├── desktop-app/          # 桌面壳
│   ├── mobile-app/           # 移动壳
│   ├── vscode-extension/
│   ├── agent-browser-extension/ / agent-chrome-extension/
│   ├── browser-control-extension-core/
│   ├── embedding/            # 向量/embedding 层
│   ├── scheduling/           # 定时任务
│   ├── skills/               # skills 系统
│   ├── toolkit/              # 工具集
│   ├── docs/
│   └── ...
├── templates/                # 17 个官方 app 模板
│   ├── chat / analytics / assets / calendar / clips /
│   ├── content / crm / design / dispatch / factory /
│   ├── forms / mail / plan / slides / tasks / videos / brain
├── community-templates/
├── mcp-registry/  registry/
├── docs/  e2e/
├── scripts/
├── agent-native.json        # 框架配置
├── pnpm-workspace.yaml
└── package.json             # root (private, type: module)
```

### 2.2 技术栈/工程化清单

- **语言：** TypeScript（ESM, `"type": "module"`）
- **Monorepo：** pnpm workspace + `pnpm-lock.yaml`
- **Format/Lint：** oxlint + oxfmt（Rust 工具链）+ prettier
- **测试：** Vitest（`vitest.shared.ts` + 各包 `vitest.config.ts`），区分 `.db.test.ts` / `.integration.spec.ts` / `.e2e.spec.ts` / `.live.spec.ts` / `.perf.spec.ts`
- **Runtime：** Nitro 兼容 host；`@agent-native/core` 包
- **DB：** 开发期 PGlite（WASM 嵌入式 PG），生产 PostgreSQL
- **LLM 抽象：** AI SDK 集成（源码里有 `translate-ai-sdk.integration.spec.ts`），"bring your LLM"
- **协议出口：** HTTP + MCP + A2A + CLI（一个 action 同时暴露 4 个协议）
- **部署：** Electron desktop、mobile app、VS Code extension、Chrome extension、EAS/Amplify 配置（`amplify.yml`、`eas.json`）
- **License：** MIT

### 2.3 核心数据流 / Action 生命周期

```
开发者写 actions/hello.ts：
  defineAction({ description, schema: z.object(...), http: {method:"GET"}, run: async ({name}) => ... })
        │
        ▼
框架静态分析（action-type-inference.ts）
        │
        ├──► Agent 侧：action 注册为 tool（description + schema 进 LLM tool list）
        ├──► React 侧：useActionQuery("hello", {name}) 自动调
        ├──► HTTP 侧：自动生成 GET /hello 路由
        ├──► MCP 侧：自动暴露为 MCP tool
        ├──► A2A 侧：自动暴露为 A2A skill
        └──► CLI 侧：自动生成命令
        │
        ▼
所有调用走同一份 run() + 同一份 permission/validation
        │
        ▼
执行结果写 PostgreSQL/PGlite（shared data）
        │
        ▼
agent 看到 UI 的 application state（当前页/选中记录/视图）
```

---

## 3. 前 10 结构性优越点

### 优越点 1：单一 action 抽象驱动 6 个 surface
- **结构依据：** README："One action powers every app surface: UI, agent, HTTP, MCP, A2A, and CLI." 源码 `packages/core/src/action.ts` + `a2a/` 目录
- **为什么优越：** 业务逻辑只写一次，6 个入口自动同步；不会出现"UI 改了 agent tool 没改"
- **对比：** 传统做法要分别写 React query hook + OpenAPI 路由 + LangChain tool + MCP server，4 处维护

### 优越点 2：agent 不"点 UI"，而是走 action 层
- **结构依据：** README："The agent does not click through the UI. It works through the same action layer as the UI."
- **为什么优越：** 避免了"让 LLM 截图+坐标点按钮"这种脆弱模式；agent 和 UI 共用同一组强类型操作
- **对比：** browser-use / OpenHands 类靠视觉点 UI，每改一次 UI 就要重新训练

### 优越点 3：shared data + shared app state
- **结构依据：** 文档链接 `server-database` + `context-awareness`；源码 `application-state/`、`checkpoints/`
- **为什么优越：** agent 知道"用户当前在哪个页面、选了哪条记录"，不需要每次把上下文贴进 prompt；UI 改了数据 agent 立刻看到
- **对比：** 多数 agent 框架只给黑盒 chat，不知道 UI 在干嘛

### 优越点 4：PGlite 开发 + PG 生产零切换
- **结构依据：** README："Use PostgreSQL in production and PGlite for local development on any Nitro-compatible host."
- **为什么优越：** 新人 clone 下来 `pnpm install && pnpm dev` 就跑，不需要先装 PG；上生产只改 DATABASE_URL
- **对比：** 全栈框架本地 SQLite、生产 PG，SQL 方言差异要踩坑

### 优越点 5：17 个官方模板 app 直接抄
- **结构依据：** `templates/` 下 chat / analytics / assets / calendar / clips / content / crm / design / dispatch / factory / forms / mail / plan / slides / tasks / videos / brain
- **为什么优越：** 不是空 hello-world，而是覆盖会议记录、设计、PPT、分析、日历、邮件、素材、CRM 等真实场景；每个都是可跑的参考实现
- **对比：** 多数 agent 框架只有一个 chat template

### 优越点 6：agent teams 原生内置
- **结构依据：** README："Delegate work to specialist agents in the same workspace or across connected agents." 源码 `packages/dispatch/`
- **为什么优越：** 多 agent 协作不用自己搭 orchestrator；同 workspace 内 delegate 和跨 connected agents 都支持
- **对比：** CrewAI / AutoGen 要自己写 group chat 协议

### 优越点 7：automations（schedule + event）一等公民
- **结构依据：** README "Automations: Run agent work on schedules or events." 源码 `automation/`、`scheduling/`
- **为什么优越：** 不只是聊天式 agent，能定时/事件触发跑任务；从"对话工具"升级为"工作流"
- **对比：** 多数 agent 框架只支持"用户发消息才动"

### 优越点 8：auth & permissions 内建
- **结构依据：** README "Authentication and permissions: Control who can access and change shared work." 源码 `authorization/`
- **为什么优越：** 多用户 app 不用自己叠一层 auth；action 级权限和用户级权限一起管
- **对比：** 很多 agent 框架默认单用户 demo，加团队就要重做

### 优越点 9：跨平台外壳全套
- **结构依据：** `packages/desktop-app`、`mobile-app`、`vscode-extension`、`agent-browser-extension`、`agent-chrome-extension`、`browser-control-extension-core`
- **为什么优越：** 写一套 action 层，能同时出桌面/移动/IDE/浏览器扩展四个壳；不是 web-only demo
- **对比：** 多数 TS agent 框架只跑浏览器

### 优越点 10：测试矩阵按层次切分
- **结构依据：** root package.json scripts 把测试分成 `test:fast`（排除 db/integration/e2e/live/perf）、`test:content-db`、`test:content-row-mutations-postgres`、`test:core-integration` 等多档
- **为什么优越：** 开发者改一行只跑 fast 子集；CI 才跑全量；大型 monorepo 不会被测试拖死
- **对比：** 很多框架 `npm test` 一把跑 20 分钟，改完不敢跑

---

## 4. 前 10 优化增强点

### 优化点 1：核心包 API 尚未稳定
- **当前状态：** root `version = 1.0.0` 但 `agent-native.eject.json`、`migration-manifest.json` 表明仍在迭代
- **优化方向：** 发一个稳定的 semver policy（0.x 还是 1.x），并在 README 标注 breaking change 节奏
- **预期收益：** 生产团队敢升级，不用每次追 main

### 优化点 2：LLM provider 抽象未在 README 说清
- **当前状态：** README 只说 "Bring your LLM"，但没列默认支持哪些 provider、怎么切
- **优化方向：** 在 Getting Started 加一张 provider 矩阵（Anthropic / OpenAI / OpenRouter / local vLLM）和切换代码示例
- **预期收益：** 新人 5 分钟接好自己的 key，不会卡在"怎么配 LLM"

### 优化点 3：MCP/A2A 暴露缺少安全默认值
- **当前状态：** action 自动暴露 6 个 surface，但默认权限模型未在 README 详述
- **优化方向：** 默认 MCP/A2A 出网要显式 opt-in；README 加一段"生产部署 checklist"
- **预期收益：** 避免把内部 action 误暴露成公网 MCP server

### 优化点 4：模板间依赖关系未文档化
- **当前状态：** 17 个模板并列，没说哪个是 starter、哪个是 production-grade
- **优化方向：** 给每个模板标 star level（example / starter / production reference）和依赖关系
- **预期收益：** 用户不会从 slides 模板开始想做日历 app

### 优化点 5：缺少可观测性方案
- **当前状态：** 没看到 OpenTelemetry / LangSmith / Logfire 集成默认项
- **优化方向：** 在 `core` 里加 `tracing` 接口，默认接 OTel；文档列三个推荐后端
- **预期收益：** 生产排障不用自己包一层 logger

### 优化点 6：checkpoint/resume 语义未公开
- **当前状态：** 源码有 `checkpoints/`、`run-loop-with-resume.integration.spec.ts`，但 README 只字未提
- **优化方向：** 文档加一节"agent 崩溃后怎么 resume"，给一个 `pnpm resume` CLI 示例
- **预期收益：** 长任务（跑 1 小时）用户不用从头来

### 优化点 7：DB migration 工具未提及
- **当前状态：** 用 PostgreSQL/PGlite，但 README 没说 migration 怎么写
- **优化方向：** 加 `@agent-native/core db:migrate` 命令 + 一份 migration 文件示例
- **预期收益：** 团队迭代 schema 不用手搓 SQL

### 优化点 8：community-templates 目录空/未激活
- **当前状态：** 有 `community-templates/` 但没看到贡献者指南
- **优化方向：** 加 `CONTRIBUTING-templates.md`：模板提交要求（README + demo + license）
- **预期收益：** 第三方模板生态能长起来

### 优化点 9：本地开发端口/配置未约定
- **当前状态：** README 只给 `npx create`，没说默认端口、环境变量、`.env.example`
- **优化方向：** 每个模板带 `.env.example`，文档列全变量
- **预期收益：** 新人不会因为缺 `DATABASE_URL` 报错半小时

### 优化点 10：e2e/live 测试门槛高
- **当前状态：** `test:fast` 把 live/e2e/perf 全排除，本地几乎跑不到
- **优化方向：** 加 `pnpm test:demo`：起一个 docker compose 跑核心 e2e，30 秒出结果
- **预期收益：** 贡献者改 action 后能本地验证端到端，不用 push 到 CI 才知道挂

---

## 5. ProcessOn 全景图信息

- 文件夹名称：`agent-native`
- 图表标题：`BuilderIO/agent-native 结构性全景图`
- 图表链接：https://www.processon.com/view/link/6ab0d1be6a29601cdfc490f3
- 图中应包含：
  - 顶层定位：Agent-Native 框架（一个 action = UI + Agent + HTTP + MCP + A2A + CLI）
  - 中层：核心抽象（defineAction）、Agent engine/run-loop、Dispatch/agent teams、Automations、Auth/permissions、Chat UI
  - 底层：TypeScript/pnpm/Vitest/Nitro、PGlite/PostgreSQL、17 个官方模板、Desktop/Mobile/VS Code/Browser 外壳

## 6. 幕布文档信息

- 文档名称：`agent-native — 日榜研究`
- 文档 ID：`7TnMgEYqtyc`（[打开](https://mubu.com/doc/7TnMgEYqtyc)）
- 包含内容：场景问题 + 前 10 优越点 + 前 10 优化点 + ProcessOn 链接

---

**2 分钟收尾动作：** 今天就 `npx --yes @agent-native/core@latest create my-agent --standalone --template chat` 起一个空项目，5 分钟后你就会看到第一个自动暴露的 action。
